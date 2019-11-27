---
category: software development
tags: [git, gitflow, processes, tutorial]
---

# What is GitFlow and how to work with it.

![gitflow](/software-development/2019-11-27-gitflow-short-info/gitflow.png)

More detailed information you may find [here](https://habr.com/ru/post/106912/).


## Features

When you want to create a feature for our awesome project, you have to create a new branch from develop. A good practice is give a name to the branch like ‘feature/<project>-<ticket number>’. For example, we work with project MRF, and we want to develop a new feature for ticket 1488, so we create a new branch with a name ‘feature/MRF-1488’. 

When you finish your work, you have to create a pull request with a develop as a target.


## Develop

Develop is a main branch for the team. It contains all features, bugfixes and hotfixes that were contributed by developers. It should contain a project that is able to work. If someone finds a bug, he has to notify the other teammates about the issue, and someone should create a bug ticket for the issue.


## Release

Release is a branch that will be deployed on a production environment very soon. A name of the branch should contain a version number of a future release. Example release branch name: release/1.0.0


## Bugfix

If we find a bug in release or develop branch, we create a new branch to fix the issue. A parent branch for the bugfix depends on the environment where the bug was found. A name should be like: bugfix/MRF-1488

If we merge the bugfix into a release, we have to merge last release updates into the develop branch too.


## Master

*Master* branch contains a ready-to-deploy code, so after we prepare a project for deployment in release branch, we merge the release into a master. Master always should contain a code that ready to be deployed without any condition.

## Hotfix

Hotfixes are branches that we create to fix bugs that were wound on production env or master branch (if it was not deployed yet). When hotfix is finished, it should be merged into a master ASAP. Hotfixes merge after we merge last updates into a latest *release* branch and develop too.

## Tag

We mark a version that we deploy on production env by *tags*. A name of the tag should represent a version like *1.0.0* and also the tag should contain a list of features, bugfixes and hotfixes that were deployed on the production.
