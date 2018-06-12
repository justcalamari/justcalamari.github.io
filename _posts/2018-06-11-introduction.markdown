---
layout: post
title: "Conda-forge Autotick Bot and Dependency Graph"
date: 2018-06-11 10:00 -0400
categories: jekyll update
---

## Autoticking Bot

[Conda-forge][conda-forge] is a community led collection of recipes, build 
infrastructure and distributions for the conda package manager
which makes it easy for developers to build and upload packages to the 
conda-forge channel. This allows users to install and start using packages
simply by running `conda install <package_name>`. Conda-forge stores each
package's recipe in a separate GitHub repository, called a feedstock. When a
recipe is updated and pushed to the feedstock, a build of the conda package
is triggered, allowing users to conda install the most recent version or build
of the package. Traditionally, updating the recipe on each release has been
the job of a group of recipe maintainers. Putting this responsibility solely
on recipe maintainers, however, results in some feedstocks being neglected and
left outdated, as maintainers may not remember to bump the feedstock version
when a new version of the source code is released or may not even be aware
of a new version since in many cases recipe maintainers are not developers
on the source code. To solve this, the [conda-forge autotick bot][cf-scripts]
was created to track out-of-date feedstocks and issue pull requests with
updated recipes.

The bot tracks and updates out-of-date feedstocks in four steps:
1. Find the names of all feedstocks on conda-forge.
2. Compute the dependency graph of packages on conda-forge found in step 1.
3. Find the most recent version of each feedstock's source code.
4. Open a PR into each out-of-date feedstock updating the meta.yaml for the
   most recent upstream release.

These steps are run on Travis CI from four different GitHub repos as daily
cron jobs, so a different worker runs every six hours. The dependency graph
is saved in the [regro/cf-graph][cf-graph] repository, and any changes made
to the graph while the bot runs are pushed to this repo using
[doctr][doctr]. While doctr is generally used to deploy docs from
Travis CI to GitHub pages, it is able to push the contents of any directory
to a specified repo. So, by including `doctr deploy --token --built-docs .
--deploy-repo regro/cf-graph --deploy-branch-name master .` in the
.travis.yml of the worker repos, the bot can push the files it modified to
the master branch of the regro/cf-graph repository.

## Dependency Graph

To keep track of the packages on conda-forge and determine if they need to
be updated, the bot stores the dependency graph of conda-forge packages. The
dependency graph is a directed graph in which each node represents a package
and the edges represent dependencies between packages. For example, scipy
requires numpy to run, so the dependency graph has an edge from numpy to
scipy. Here is the dependency graph for all packages on conda-forge,
visualized using [HoloViews][holoviews] and [Bokeh][bokeh]:

{% include graph.html %}

The graph is created using [NetworkX][networkx], which allows additional
attributes to be stored in each node. This lets the bot to keep track of
all important feedstock metadata in the graph. So, the graph is able to
store the current version of the feedstock on conda-forge and the most
recent version of the source code, letting the bot determine which packages
need to be updated.

Storing this information in a graph allows us to observe the relationships
between packages on conda-forge. This can give interesting information such
as the [most depended upon packages][top_100]. Storing the dependency graph
is also useful if feedstocks need to be visited in dependency order, i.e. a
package must be visited before any package that depends on it. This comes up
in [triggering rebuilds of dependency chains][rebuilds]. When a package is
rebuilt, if ABI compatibility is not guaranteed, other packages that depend
on it must be rebuilt as well, but a needs to be rebuilt after all of its
dependencies, so it is important to traverse the graph in order.

In conda-smithy 3, a recent release of conda-smithy, a [central
configuration file][central_pinning] is used for the build matrices and
versions of specific packages. This configuration file handles pinning of
packages, so recipes no longer need to manually pin versions. When a new ABI
version of a package in central pinning is released, the central pinning
file will be updated, and feedstocks with a dependency on that package need
to be rebuilt.

To make sure that feedstocks are visited in the correct order, nodes are
grouped by the length of their longest path to the source node from central
pinning. Any nodes that have the same longest path length cannot depend on
each other, so it is safe to rebuild them simultaneously. So to trigger
rebuilds properly, the bot can issue PRs into all feedstocks with the same
longest path length updating the build number (thus triggering a rebuild),
and wait for all PRs to be merged before moving to the next path length. I
recently opened a [PR][path_lengths] which was merged to find the longest
path to each node in the graph from a specified starting node, allowing the
bot to properly group feedstocks.

[conda-forge]: https://conda-forge.org/
[cf-scripts]: https://github.com/regro/cf-scripts
[cf-graph]: https://github.com/regro/cf-graph
[00]: https://github.com/regro/00-find-feedstocks
[01]: https://github.com/regro/make-cf-graph
[02]: https://github.com/regro/graph-upstream
[03]: https://github.com/regro/cf-auto-tick
[doctr]: https://github.com/drdoctr/doctr
[holoviews]: http://holoviews.org/
[bokeh]: https://bokeh.pydata.org/en/latest/
[networkx]: https://networkx.github.io/
[top_100]: https://github.com/regro/cf-graph/blob/master/top_100.txt
[rebuilds]: https://github.com/regro/cf-scripts/issues/44
[central_pinning]: https://github.com/conda-forge/conda-forge-pinning-feedstock/blob/master/recipe/conda_build_config.yaml
[path_lengths]: https://github.com/regro/cf-scripts/pull/161
