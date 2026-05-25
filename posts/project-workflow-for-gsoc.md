---
title: 'Project Workflow for GSoC'
date: '2019-05-24'
tags: jira, git, jenkins, gsoc
---

For my GSoC project, I will be developing plugins for Jenkins Community. We required two things:

1. Track objectives and issues in one place
2. Manage code reviews and plugin releases

## Issue Tracker

For the first problem, we started using `GitHub Project boards` for objectives and `GitHub issues` for issues. Project boards were a bit clumsy so we preferred using a Google Doc instead. For issues, the GitHub issues are just okay but we felt keeping discussions and codes separate is a better idea.

For a good reason, Jenkins community hosts it's own [JIRA server](https://issues.jenkins-ci.org) where this can be managed quite well. JIRA issues can be more verbose while being separate from the code itself. JIRA issues can take various forms, namely:

1. `Bug` - For filing bug in the code
2. `Epic` - For categorising issues in one place
3. `Task` - For creating a task for the future
4. `Story` - For stories about a user's experience with product or feature
5. `New Feature` - For adding a new feature
6. `Improvement` - For improving an existing feature

There are multiple other configurations that can help organising issues, to learn more see [here](https://issues.jenkins-ci.org/secure/ShowConstantsHelp.jspa?decorator=popup#IssueTypes).

JIRA also provides support for project boards to track progress in the form of scrum boards or kanban boards. Scrum boards are basically like trello that lets use divide a virtual board into various sections such as TODO, In Progress, In Review, Done etc. and place issues, which take the form of cards, in the stage depending on their progress. Also allows you to create sprints which are bounded by time. More about it later. Kanban boards are also similar but with higher level of customisation that can lead to ambiguity in case of small team projects. It comes down to your preference. You can learn about the differences [here](https://www.atlassian.com/agile/kanban/kanban-vs-scrum).

For our project, we decided to use:

1. `JIRA Issues` - These can exist in all possible forms (task, story, bug, improvement etc) based on the requirement

2. `EPIC` - To organise issues of each GSoC phases in separate categories. So we created four EPICs, namely,

    * `Phase 0: Community Bonding` - [link](https://issues.jenkins-ci.org/browse/JENKINS-57541)
    * `Phase 1: Multibranch Pipeline Support For GitLab` - [link](https://issues.jenkins-ci.org/browse/JENKINS-57445)
    * `Phase 2: Multibranch Pipeline Support For GitLab` - [link](https://issues.jenkins-ci.org/browse/JENKINS-57540)
    * `Phase 3: Multibranch Pipeline Support For GitLab` - [link](https://issues.jenkins-ci.org/browse/JENKINS-57538)
<br/><br/>
3. `Scrum Board` and `Sprints` - To keep a track of our progress by both mentors and students.

    A scrum was created with the name:
    
    * `Google Summer of Code 2019` - [link](https://issues.jenkins-ci.org/secure/RapidBoard.jspa?rapidView=591)<br/>
  
    Four sprints were created to track the issues of all GSoC 2019 projects under Jenkins, namely:

    * `GSoC 2019. Community Bonding`
    * `GSoC 2019. Code Phase 1`
    * `GSoC 2019. Code Phase 2`
    * `GSoC 2019. Code Phase 3`

JIRA Scrum boards comes with another section called `Backlog` which has a list of features, defects, experiment, enhancement etc that needs to be done in the future which is not part of a time bound sprint. 

![Backlog Section](/assets/2019-05-24-project-workflow-for-gsoc/backlog-scrum-board.png)
`The Backlog Section of Scrum Board`

![Active Sprints Section](/assets/2019-05-24-project-workflow-for-gsoc/active-sprint-scrum-board.png)
`The Active Sprints Section of Scrum Board`

To add issues to the sprints you can either add the desired sprint's link at the time of issue creation or simply drag and drop from the backlog. There are also other sections that can be helpful like releases, tests, components etc.

One of the interesting feature of JIRA is JIRA Query Language (JQL). This allows you to define filter for searches. You can have predefined filters for the board as well. 

Select `Board` (top right) then `Configure` then `Quick Filter`. For example:

![quick-filters](/assets/2019-05-24-project-workflow-for-gsoc/quick-filters.png)
`Quick Filters in our scrum board`

You can define your own filters. JIRA also offers lots of other customisations options like card colours, columns, issue detail view etc. You can explore them based on your needs.

## Git Workflow

For our project we chose Git over any other SCM like Subversion or Mercurial since all Jenkins repositories are hosted on GitHub and Git makes merging simpler than others. There multiple Git Workflow Models like Feature Branch Workfow, Gitflow Workflow etc. We decided to follow Gitflow Workflow. The idea is to 3 mainline branches and temporary feature branches:

1. `master` - Only tagged commits from `develop` branch are merged into this branch
2. `develop` - When a feature branch is complete it is merged into this branch
3. `release` - After a feature implementation passes all tests and reviewed then it is merged into this branch
4. `<feature-branch>` - A branch checked out from `develop` branch when a new feature has to be implemented

This stricting branching model can be tedious to maintain through git CLI. To aid in this development cycle a tool called [`Gitflow`](https://github.com/petervanderdoes/gitflow-avh) is used.

![git-flow-model](/assets/2019-05-24-project-workflow-for-gsoc/git-flow-model.png)
`A visualisation of the gitflow model` [[1]](https://nvie.com/posts/a-successful-git-branching-model/)

Git Flow is simply a wrapper around Git that abstracts the underlying branch modelling with intuitive commands.

> NOTE: This Gitflow Workflow is slightly modified in order to use Pull Requests for git merge instead of a plain git merge.

On Ubuntu, install `git-flow` using:
```
sudo apt-get install git-flow
```
For other OS, see [this](https://github.com/petervanderdoes/gitflow-avh/wiki/Installation).

To start the workflow, run:
```
git flow init
```
Then follow the on-screen instructions for setup.

![git-flow-init](/assets/2019-05-24-project-workflow-for-gsoc/git-flow-init.png)<br/>

### To start a feature branch

```
git flow feature start <feature-branch>
```

In the background it does a `git checkout develop` and `git checkout -b <feature-branch>`.

Push commits to the feature branch.

### Start a pull request 

This is a custom step and not a part of Gitflow workflow, if your repository doesn't need you to send PRs then feel free to move on to the next step. 

Push the _`feature`_ branch to GitHub and start a pull request to merge into the _`develop`_ branch. After merging this PR, do a local sync using `git pull origin develop` and manually delete the feature branch using `git branch -D <feature-branch>` (to delete local branch) and `git push origin :<feature-branch>`. You might also want to do `git fetch --all --prune` on other machines.

### To finish a feature branch [Skip if sending PR] 

```
git flow feature finish <feature-branch>
```

In the background it does a `git checkout develop` and `git merge <feature-branch>`.

### To start a release branch

This step is to fork a _`release`_ branch off of _`develop`_ branch after enough features are acquired for a release.

We only push bug fixes, documentation and other release-oriented tasks to ~`release`_ branch.

Once it is ship ready, the _`release`_ branch gets merged into master and tagged with a version number. In addition, it should be merged back into develop.

Using a dedicated branch to prepare releases makes it possible for one team to polish the current release while another team continues working on features for the next release.

To create a release branch:

```
git flow release start <version-number>
```

In the background it does a `git checkout develop` and `git checkout -b release/<version-number>`.

Push commits to the release branch.

### To finish a release branch

```
git flow release finish '<version-number>'
```

In the background it does a `git checkout master`, `git merge release/<version-number>`, `git checkout develop` and `git merge release/<version-number>` and `git branch -d release/<version-number>`.

### Sending pull request from release branch

We cannot send PRs from _`release`_ branch and use the gitflow release finish step without breaking the PR workflow. If you do not merge your PR and follow the gitflow release finish step then the PR will be marked closed as the release branch will be deleted. If you want to do it then follow along, at the end there are manual steps to do it using the Git CLI. 

### The overall flow of Gitflow

1. A _`develop`_ branch is created from _`master`_

2. A _`release`_ branch is created from _`develop`_

3. _`Feature`_ branches are created from _`develop`_

4. When a _`feature`_ is complete it is merged into the _`develop`_ branch

5. When the _`release`_ branch is done it is merged into _`develop`_ and _`master`_

6. If an issue in _`master`_ is detected a _`hotfix`_ branch is created from _`master`_

7. Once the _`hotfix is`_ complete it is merged to both _`develop`_ and _`master`_

### Graphical Tool to visualise the git flow

We will also use `GitKraken Linux Client` to manage commits, tags, branches etc. Personally I find it helpful. 

There are other options like `GitHub Desktop`, `Smart Git`, `Tower 2` etc.

### Using plain Git to replicate the GitFlow Workflow

This workflow is slightly modified to support merging with the help of Pull Requests.

{% highlight shell %}

git init

git checkout -b develop

git checkout -b feature/<feature-branch>

# Push commits required for the feature

git push origin feature/<feature-branch>

# Send PR to develop branch

# Merge PR after review by mentors

git pull origin develop

git branch -d feature/<feature-branch>

git push origin :feature/<feature-branch>

git checkout -b release/<version-number>

# Push commits to your release branches like bug fixes, documentation etc

git push origin release/<version-number>

# Send PR to develop branch

# Merge PR after review by mentors

git pull origin develop

git checkout release/<version-number>

git tag -a <tag-name> -m <your message>

git checkout master 

git merge <tag-name>

git push origin master

git push origin --tags

git branch -d release/<version-number>

git push origin :release/<version-number>

# NOTE: In gitflow the release branch was merged with master and 
# tag was merged with develop, but we doing opposite here
# since the PR has to be merged with the develop branch to see the
# files changed 

git checkout develop

# start next development iteration
{% endhighlight %}

You can also check other simpler workflows:

* [GitLab Flow](https://about.gitlab.com/2014/09/29/gitlab-flow/)
* [GitHub Flow](https://guides.github.com/introduction/flow/)

Resources:

[Atlassian GitFlow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)