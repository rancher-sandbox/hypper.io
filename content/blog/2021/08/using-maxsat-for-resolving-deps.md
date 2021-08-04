# Using a MaxSAT for resolving Hypper chart dependencies on Kubernetes

[Hypper](http://hypper.io) is a Kubernetes package manager based on Helm. It
handles charts in a similar way as Helm does, with a twist: it tries to resolve
dependencies between charts to reuse existing resources and deployments.
For more info, see the two articles on this blog by @mattfarina: [Introducing
Hypper](https://community.suse.com/posts/15893197) and
[Hypper v3 - Now with a SAT solver](https://community.suse.com/posts/14072167).

Hypper is designed to work with shared dependencies: dependencies installed once
on a cluster that several charts depend on. Think of Hypper as bringing the
usual Linux package management to Kubernetes Helm charts.

Checking all the charts so see if their dependencies are correctly installed or
there's some missing is not an easy task. This is called a "Satisfiability
problem", or SAT, for friends. In fact, solving a SAT problem it's an [NP-Complete
problem](https://en.wikipedia.org/wiki/Boolean_satisfiability_problem#Unrestricted_satisfiability_(SAT))!.

## How to solve a SAT problem?

Finding a solution to that problem may be somewhat easy by reducing the problem
space (like Go modules), or using custom heuristics to be fast but without an
assurance of finding an optimal solution (like Apt).

To find an optimal solution to a SAT problem fast enough so it's useful to use,
we need a focused approach. That comes from a SAT solver algorithm: it gets fed
with facts of a problem, and it finds the optimal combination of facts that can
satisfy the problem or a demonstration that such combination doesn't exist.

In Hypper's case, we also want the possibility to depend on Helm charts outside
of our curated repositories. Therefore, we would benefit from knowing if a
system can be upgraded or not, regardless of our ability on curating the
repositories. Hence, we use a SAT solver directly when performing those
upgrades.

## What kind of SAT solvers exist?

Let's walk through an example on the following paragraphs and see. We will use
the example of installing `Wordpress` with a dependency to `Mariadb` (the names
are just placeholders for a potential real-life scenario).

### Pure SAT

A Pure SAT is the simplest implementation of a SAT solver. It receives
first-order logic statements (facts). For example, a fact specifying that
`Wordpress` depends on `Mariadb` would be `not(Wordpress) or Mariadb`.

The SAT solver will tell you if there's a combination of facts that work together
(are satisfiable, for example you can install `Wordpress` because `Mariadb` is
already present), or if there's no solution, and why.

Pure SAT solvers give you a solution on which dependencies you need to install
(several versions of `Mariadb` for example), or walk around conflicts on several
charts (`Mariadb` conflicts with `Mysql`), but one still needs to write their
own heuristics to pick the suitable versions.

An example of a package manager using a Pure SAT solver is
[Zypper](https://en.opensuse.org/Portal:Zypper)).

### Pseudo-Boolean SAT

In addition to the normal "hard" clauses of a Pure SAT solver, a Pseudo-Boolean
SAT operates with soft Pseudo-Boolean equations. In our example,  the
Pseudo-Boolean equation specifying that `Wordpress` depends on `Mariadb` would
be:

```
Mariadb - Wordpress >= 0
```

There, `Mysql` and `Mariadb` can be 1 (installed) or 0 (not installed).

If you want to install `Wordpress` (`Wordpress` is `1`):
```
Mariadb - Wordpress >= 0  ; 
Mariadb - 1         >= 0  ; 
Mariadb >= 0 + 1          ;
Mariadb >= 1              ;
Mariadb =  1
```
In words, `Mariadb` needs to be installed.

These equations allow to easily list several dependencies:
```
Mariadb + Mysql - Wordpress >= 0
```

Pseudo-Boolean SAT solvers simplify even more solving dependencies on several
charts, (several versions of `Mariadb` for example), but one needs still needs
to write their own heuristics to pick the suitable versions.

### MaxSAT

MaxSAT solvers are a generalization of Pseudo-Boolean ones. These kind of
solvers allow you to add weights to constraints in the equations, and give you
the result that maximizes your preferences. A MaxSAT solver can even find a
maximal possible partial solution when there's no solution for the whole input
set.

This turns out to be an even more complex problem,
[NP-Hard](https://en.wikipedia.org/wiki/Maximum_satisfiability_problem).

An example of using a MaxSAT equation to install `Wordpress` and its dependency
`Mariadb` on the latest [Semver](https://semver.org/) version would be:

```
max( (Mariadb, version: 10.5.9,  weight: 1) + 
     (Mariadb, version: 10.5.10, weight: 2) + 
     (Mariadb, version: 10.5.11, weight: 3)   )
  - Wordpress >= 0
```

## Why pick MaxSAT?

In our case, we have several strategies we want to apply to our problem:
- **Minimize package removal**.
- **Minimize number of package charts**.
- **Maximize freshness of packages**. By assigning a distance between package
  versions, we can optimize for those that have the smaller distance. Doing a
  full upgrade of the system means selecting to maximize freshness of all
  installed packages, without asking for installing new packages by default (but
  allowing for packages to be pulled into the installed list).
- **Minimize install of optional packages**.
- **Maximize install of optional packages**.
- **Minimize changes to installed packages**: Configuration changes in
  values.yaml, treat charts with K8s CRDs differently, etc.
  
With a MaxSAT solver, we can tackle those. For example we can have certain
preferences set by weighting them and solve for the following criteria:
- Upgrade and minimize changes to installed packages.
- Upgrade only up to minor or patch Semver versions.
- Install with the minimum set of dependencies required.

Usually such problems are computationally more intensive than using a normal
SAT. But since we have an overseeable problem space (`10^1` charts instead of
`10^3` packages), we still can manage to solve this in an acceptable time frame. 

## And how does this look?

Here is Hypper installing `Wordpress`, with its dependency on `Mariadb` and an
optional dependency on `Prometheus`.  `Prometheus` has an optional dependency on
`Grafana`, which is already installed.

``` console
$> hypper shared-dep list path/to/local/chart/prometheus
NAME     VERSION REPOSITORY                       STATUS      NAMESPACE   TYPE
grafana  ^8.0.0  https://our-hypper-charts/repo   installed   grafana     shared-optional

$> hypper install path/to/local/chart/wordpress_with_mariadb_and_grafana
❓ Install optional shared dependency "prometheus" of chart "wordpress"? [Y/n]:
y
The following charts are going to be installed:
wordpress v5.8.0
 ├─ mariadb v10.5.11
 └─ prometheus v2.27.1

🛳  Installing chart "mariadb" as "mariadb" in namespace "databases"…
🛳  Installing chart "prometheus" as "prometheus" in namespace "metrics"…
🛳  Installing chart "wordpress" as "wordpress" in namespace "wordpress"…
👏 Done!
```

If you are left hungry for a more mathematical explanation, have a look at our
docs on this topic
[here](https://github.com/rancher-sandbox/hypper/blob/main/docs/design/sat-solver.md).
