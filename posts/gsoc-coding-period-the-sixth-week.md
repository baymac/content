---
title: 'GSoC coding period: The sixth week'
date: '2019-07-07'
tags: jenkins, gsoc, gitlab
---

This week has been exciting. The branch source part of the plugin that I was working on is finally ready to be merged. It took a lot of efforts to make it work. After a series of fixing bugs, undoing unwanted changes, adding new features the branch source has some features working currently. 

What works and what doesn't?

1) Multibranch Pipeline Jobs work perfectly fine. 

2) GitLab Group Jobs has bugs.

## Issues Fixed:

1) Move to `serverName` based listbox than `serverUrl` in Multibranch Pipeline Jobs

The listbox that allows user to select a GitLab server in Multibranch Pipeline Jobs had value of `serverUrl`. This caused issue when 2 server entries with same server url to only select the server first selected. Now the listbox has value `serverName` and desired server entry can be choosen.

|  Issue 	| 
|---	    |
|[JENKINS-58261](https://issues.jenkins-ci.org/browse/JENKINS-58261) | 

2) Fix GitLabSCMFile api call to obtain Jenkinsfile without checkout

During branch builds, the GitLabSCMFile is responsible for fetching the contents of Jenkinsfile. There was a wrong api call made which caused commit to be not found.

|  Issue 	| 
|---	    |
|[JENKINS-58263](https://issues.jenkins-ci.org/browse/JENKINS-58263) | 

3) The Project listbox in Multibranch Pipeline doesn't repopulate

When a project was selected from the project listbox in Multibranch Pipeline job the project was refetched from the server. This was happening as `doProjectFillItems` method depended on the project field itself. So I removed it from the list of it's query parameters.

|  Issue 	| 
|---	    |
|[JENKINS-58264](https://issues.jenkins-ci.org/browse/JENKINS-58264) | 

4) Try to add Pipeline declarative dependencies to run pipeline jobs with hpi:run

I added the Pipeline declarative dependency:

```xml
<dependency>
    <groupId>org.jenkinsci.plugins</groupId>
    <artifactId>pipeline-model-definition</artifactId>
    <verison>1.3.9</version>
</dependency>
```

But that was a bit complicated since it created a lot of dependency management issues. I feel it is inessential atm, so didn't spend time fixing it. To test the plugin with `mvn hpi:run` we can use Scripted Pipeline. If declarative pipeline used then install `Pipeline: Declarative` from update centre in your Jenkins instance.

|  Issue 	| 
|---	    |
|[JENKINS-58268](https://issues.jenkins-ci.org/browse/JENKINS-58268) | 

5) Cleaned up log in Branch Indexing

The logs during branch indexing displayed an extra `looking up from repository..` due to an additional logging in `retrieve(c,o,e,l)` method. It was only required in `retrieveActions(e, l)` method.

|  Issue 	| 
|---	    |
|[JENKINS-58270](https://issues.jenkins-ci.org/browse/JENKINS-58270) | 

6) GitLab Server without credentials can also be tested with Test connection

If no credentials is specified in GitLab Server configuration then Test connection gave error. Now, if no credentials is specified it gives a message `Valid GitLab Server but no credentials specified`.


|  Issue 	| 
|---	    |
|[JENKINS-58356](https://issues.jenkins-ci.org/browse/JENKINS-58356) |

7) Fixed GitLab Browser projectUrl getter

`GitLabBrowser` extends `GitRepositoryBrowser` but repository in GitLab is known as project. So the `repoUrl` field in our `GitLabBrowser` was made `projectUrl`. The browser was required to supply the `projectUrl` but it contained `getRepoUrl()` which caused error. Fixed it by:

```java
public String getProjectUrl() {
    return super.getRepoUrl();
}
```

|  Issue 	| 
|---	    |
|[JENKINS-58365](https://issues.jenkins-ci.org/browse/JENKINS-58365) | 

> NOTE: All above issues were fixed in [PR-20](https://github.com/baymac/gitlab-branch-source-plugin/pull/20).