---
title: 'GSoC coding period: The plugin hosting week'
date: '2019-07-13'
tags: jenkins, gsoc, gitlab
---

This week the branch source part of the plugin has taken some shape, a few more important bugs were fixed and features were added. GitLab Branch Source Plugin was also hosted on `jenkinsci` GitHub organisation.

## Issues Fixed:

1) Request to host our plugin on Jenkins official GitHub organisation:

To initiate the process to move our plugin into `jenkinsci` org, I raised a ticket. There is a conflict due existence of another plugin with the same name but we are expecting it to be fixed as the old plugin is not maintained and also not released to Jenkins update centre.

On Friday, our plugin got hosted in the `jenkinsci` organisation. I am hoping to fix a couple of issues and making my first release on Monday.

The official `GitLab-Branch-Source-Plugin` repository can be found [here](https://github.com/jenkinsci/gitlab-branch-source-plugin/).

|  Issue 	|
|---	    |
|[HOSTING-795](https://issues.jenkins-ci.org/browse/HOSTING-795) |

2) Fixed issues in GitLab Group job type

There were couple of bug fixes in this pr:

i. `GitLabSCMNavigator` now takes `serverName` instead of `serverUrl`.

ii. `GitLabSCMBuilder` also takes `serverName` to create a `GitLabSCMSource` object.

iii. In `GitLabSCMNavigator` the `retrieve(c, o, e, l)` method was called for branch indexing directly. Since the `gitlabProject` was fetched in `retrieveActions(e, l)`, it was giving null pointer exception. So refetched the GitLab project:

```java
if(gitlabProject == null) {
    gitlabProject = gitLabApi.getProjectApi().getProject(projectOwner+"/"+project);
}
```

iv. There is another issue that I am trying to fix here is that default SCM Traits are not getting populated for `GitLab Group` Job type. See pr for more info.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58318](https://issues.jenkins-ci.org/browse/JENKINS-58318) | [#21](https://github.com/baymac/gitlab-branch-source-plugin/pull/21)

3) Added Build Status Notifier feature

Added Impl of `GitLabBuildStatusNotifier` class that contains a method `sendNotifications` which communicates with the GitLab Server to publish the commit status (either `SUCCESS`, `FAILED`, `PENDING` etc.). There are 3 other extensions:

`JobCompletedListener`, `JobCheckOutListener` - that call `sendNotification` method based on the build status 

`JobScheduledListener` - that sends PENDING status notification to GitLabServer.

Another issue fixed was the `enqueue` error. I made a mistake of changing the status of an already pending pipeline to a pending again when build started. Changed it to running. 

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58400](https://issues.jenkins-ci.org/browse/JENKINS-58400) | [#22](https://github.com/baymac/gitlab-branch-source-plugin/pull/22)

4) Added support for GitLab subgroups

We moved form `projectName` to `projectPath` variable. Instead of only project name now the `projectPath` contains project path with namespace.

Another issue fixed, was when the sourceproject for a mr is missing. The trait will compare different mrs to either exclude branch or not, there is a chance that the desired mr actually has the source project id but an mr without `source project id` is encountered will result in an NPE. So when adding mrs to the request, I am removing the mrs with null source project id in GitLabSCMSource `retrieve(c, o, e, l)`.

```java
List<MergeRequest> mrs = gitLabApi.getMergeRequestApi().getMergeRequests(gitlabProject);
mrs = mrs.stream().filter(mr -> mr.getSourceProjectId() != null).collect(Collectors.toList());
request.setMergeRequests(mrs);
```

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58266](https://issues.jenkins-ci.org/browse/JENKINS-58266), [JENKINS-58269](https://issues.jenkins-ci.org/browse/JENKINS-58269), [JENKINS-58455](https://issues.jenkins-ci.org/browse/JENKINS-58455) | [#23](https://github.com/baymac/gitlab-branch-source-plugin/pull/23)


5) Add support Tags

Tags support was added by Impl of `GitLabTagSCMHead` and `TagDiscoveryTrait`. And required indexing in `GitLabSCMSource`.

Fixed a NPE which was caused due to a missing implementation of `GitLabSCMSFileSystem` was added in 
a06cfe5.

This pr also fixed issue which caused closed mr branches not to be built.

This issue was not totally fixed, see [JENKINS-58477](https://issues.jenkins-ci.org/browse/JENKINS-58477).

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58265](https://issues.jenkins-ci.org/browse/JENKINS-58265), [JENKINS-58309](https://issues.jenkins-ci.org/browse/JENKINS-58309), [JENKINS-58467](https://issues.jenkins-ci.org/browse/JENKINS-58467) | [#24](https://github.com/baymac/gitlab-branch-source-plugin/pull/24)

6) Release permission for our plugin:

Sent pr to `repository-permissions-updater` to get required permission to release our plugin to Jenkins update center.

|  Pull Request 	| 
|---	    |
|[#1218](https://github.com/jenkins-infra/repository-permissions-updater/pull/1218) | 

## Issues Pending:

1) Decide the strategy for GitLab Pipeline Status Notification

There are 2 cases to consider when reporting build status to GitLab.

Case I:

GitLab Pipeline status is not similar to GitHub (where it is a simple boolean field). GitLab Pipeline status are similar to Jenkins Pipeline and has stages. For example,

It contains
* `#1 Declarative checkout passed`
* `#2 Stage 1 passed`

This can be seen in the following screenshot.
![Screenshot from 2019-07-11 11-20-01](https://user-images.githubusercontent.com/23079344/61025323-f7a63980-a3cd-11e9-9915-cacfbc1e027f.png)
This is how it is implemented in [ArgelBargel's GitLab Branch Source Plugin](https://github.com/Argelbargel/gitlab-branch-source-plugin/).

The problem:

1. If the job is canceled. The stage details is not passed on to the Pipeline Status. It doesn't provide appropriate details. Below is the screenshot of that.

![Screenshot from 2019-07-11 11-23-51](https://user-images.githubusercontent.com/23079344/61025497-97fc5e00-a3ce-11e9-882b-8e530da63bc6.png)

The `Stage 1` failed but GitLab Pipeline is not notified about it.

2. All the stages link to the build. Meaning they don't link to their respective stages because Jenkins doesn't have a dedicated action for stages. You can only have a popup menu to see the log of a particular stage.

Case II:

This is what I have implemented. We are notifiy GitLab pipeline about the Job status instead of Pipeline stages.

![Screenshot from 2019-07-11 11-12-47](https://user-images.githubusercontent.com/23079344/61025884-aa2acc00-a3cf-11e9-9c55-d6d25406188d.png)

This is done with following assumptions:

1. GitLab User doesn't change the job name for the same project. You can see multiple pipeline statuses for a single pipeline status as they are made by different jobs.

2. GitLab User goes to Jenkins to see the stages details rather than GitLab itself.

If the above assumptions are wrong then it can be problems. Let me know your thoughts.

|  Issue 	| 
|---	    |
|[JENKINS-58445](https://issues.jenkins-ci.org/browse/JENKINS-58445) | 

2) Add support for GitLab branch source in Skip notification trait

Sometimes user might require to not get notified about the Pipeline status. So a specific plugin called `skip-notification-trait-plugin` exist that provides the skip notification behaviour to Multi Branch Pipeline Job types.

|  Issue 	| 
|---	    |
|[JENKINS-58480](https://issues.jenkins-ci.org/browse/JENKINS-58480) | 

3) GitLab Pipeline Status Notification cannot transition via run from running

This issue is similar to the `enqueue` error solved in [JENKINS-58400](https://issues.jenkins-ci.org/browse/JENKINS-58400). 

|  Issue 	| 
|---	    |
|[JENKINS-58466](https://issues.jenkins-ci.org/browse/JENKINS-58466) | 

4) Add webhooks support and testing

This weekend I will be implementing the webhooks support which will enable server to send push events to our Jenkins instance for auto indexing and building.

|  Issue 	| 
|---	    |
|[JENKINS-57760](https://issues.jenkins-ci.org/browse/JENKINS-57760), [JENKINS-58361](https://issues.jenkins-ci.org/browse/JENKINS-58361) |

Dasvidaniya.