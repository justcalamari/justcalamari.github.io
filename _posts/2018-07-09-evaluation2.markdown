---
layout: post
title: "Going Beyond Version Bumping"
author: Justin Calamari
date: 2018-07-09 10:00 -0400
categories: jekyll update
---

In my [last post]({{ site.baseurl }}{% post_url
2018-06-11-introduction.markdown %}}), I introduced the
[cf-autotick-bot][cf-scripts], which tracks the version of packages on the
conda-forge channel and automatically updates recipes with the most recent
version of the source code. As I mentioned in my previous post, the bot
achieves this by storing the conda-forge dependency graph along with
metadata for each feedstock. With all of this information and the ability to
PR to any feedstock, the bot has the potential for much more than just
bumping the version.

## The Migrator Class

Recently [@CJ-Wright][CJ] added the `Migrator` class to bot's source code.
Now, instead of explicity running the version bumping code on all
feedstocks, the bot runs a set of `Migrator` instances on each feedstock. By
putting the version bumping code in a subclass of `Migrator`, the bot can
perform its original version bumping procedure, as well as migrations from
any other inheritor of `Migrator`. This new paradigm allowed for the
development of new functionality for the bot.

## Conda-build 3 Compiler Syntax

Conda-build 3 defines a Jinja2 function to specify compiler packages.
Recipes can include for example
{% raw %}
```
requirements:
  build:
    - {{ compiler("c") }}
```
{% endraw %}
which means the package needs to be built with C compilers.

Anaconda 5.0 switched from OS-provided compiler tools to their own
compilers. Conda-forge does not yet use these compilers, but the core
development team would like to switch to them. To do this, all recipes must
first be using the new compiler syntax. Since the bot tracks every feedstock
and issues PRs, it is a good tool to move recipes to this new syntax.

To give the bot this functionality, I wrote a `Migrator` subclass to handle
this. The migrator first checks if a recipe needs to move to the new syntax
by checking if it uses old compilers (toolchain or gcc), and then runs a
[conda-smithy][smithy] command that updates the recipe with the new syntax.
This is then PRed to the feedstock for the recipe maintainers to review.

I also wrote a migrator which simply bumps the build number of all recipes.
This is useful when the switch is made to the new compilers, since all
packages with compilers as build requirements would need to be rebuilt. The
bot also makes sure that packages are rebuilt in dependency order, that is
packages are only rebuilt after all of its requirements are rebuilt.

This rebuild migrator is useful in other situations besides this compiler
switch. Whenever a pinning in conda-forge's [global pinning][pinning] file
is changed, packages must be rebuilt to use the new pinning. This migrator
allows the bot to trigger rebuilds in the correct order of the descendants
of the package in the global pinning.

## Noarch

Conda-forge allows pure-Python packages to be built noarch (no
architecture), which mean the package only needs to built once (on
CircleCI), making the build faster and freeing resources for other packages.
A package can be built noarch simply by including `noarch: python` in the
build section of the meta.yaml; however, there are many recipes on
conda-forge which can be build with noarch, but do not include this in the
build section. It would be ideal if any package is build noarch if possible,
to reduce CI usage, especially in the age of the bot, which issues PRs in
bulk, possible consuming a lot of CI resources.

Using the `Migrator` class, the can easily issue PRs adding `noarch: python`
to the build section, so I wrote a migrator to do this. There are many
requirements for a package to be noarch, summarized in this list

* No compiled extensions
* No post-link or pre-link or pre-unlink scripts
* No OS specific build scripts
* No python version specific requirements
* No skips except for python version. (If the recipe is py3 only, remove
  skip statement and add version constraint on python)
* 2to3 is not used
* Scripts argument in setup.py is not used
* If entrypoints are in setup.py, they are listed in meta.yaml
* No activate scripts
* Not a dependency of `conda`

The bot checks some of this criteria and skips PRing to feedstocks it knows
cannot be noarch, but some of these requirements the bot cannot check with
the metadata it currently collects. Therefore, the noarch PR is just a
suggestion, and the PR message displays a checklist with the items above and
asks the recipe maintainers to only merge if all items are satisfied.
Otherwise, the recipe maintainers can close the PR and the bot will not
issue another PR suggesting noarch.


[cf-scripts]: https://github.com/regro/cf-scripts
[CJ]: https://github.com/CJ-Wright
[smithy]: https://github.com/conda-forge/conda-smithy
[pinning]: https://github.com/conda-forge/conda-forge-pinning-feedstock/blob/master/recipe/conda_build_config.yaml
