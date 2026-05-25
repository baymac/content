---
title: 'GSoC coding period: The second presentation week'
date: '2019-07-27'
tags: jenkins, gsoc, gitlab
---

This week we had our second coding phase evaluation. We had to present our project on Jenkins official YouTube channel.

The project demo can be seen [here](https://youtu.be/tnoObQqGhyM).

The presentation slides can be found [here](https://docs.google.com/presentation/d/1fMiDiLi3L39hoaFz-qLLhWQXwb1U9864_Per3vTc1dk/edit?usp=sharing).

This week we also had some important patches and features in our plugin.

## Issues fixed:

1) Improvements to Form Validation in GitLab Group

Previously GitLab Group `projectOwner` field displayed an error message that the field wasn't able to fetch the owner but didn't show the message. The `fetchOwner` method was reused here which was implemented for `GitLabSCMSource`. This makes the code more readable and reduces clutter.

Also fixed the duplicate SCM Traits which appeared in listbox. For some reason in the super class multiple traits are fetched, it needs to be filtered out. 

Fixed by

```java
Set<SCMTraitDescriptor<?>> dedup = new HashSet<>();
for (Iterator<SCMTraitDescriptor<?>> iterator = all.iterator(); iterator.hasNext(); ) {
    SCMTraitDescriptor<?> d = iterator.next();
    if (dedup.contains(d)
            || d instanceof GitBrowserSCMSourceTrait.DescriptorImpl) {
        // remove any we have seen already and ban the browser configuration as it will always be github
        iterator.remove();
    } else {
        dedup.add(d);
    }
}
```

![casz-error](/assets/2019-07-28-gsoc-coding-period-second-presentation-week/casz-error.png)

![casz-error-expanded](/assets/2019-07-28-gsoc-coding-period-second-presentation-week/casz-error-expanded.png)

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58481](https://issues.jenkins-ci.org/browse/JENKINS-58481) | [#9](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/9) 

2) Add subgroup filter traits

This new subgroup filter trait allows `GitLab Group` Job type to fetch the sub projects from the `projectOwner` specified (should be a group/subgroup). This was contributed by Joseph.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58653](https://issues.jenkins-ci.org/browse/JENKINS-58653) | [#10](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/10)

3) Plugin released to Experimental Update Center

The plugin was released with Alpha 2 version to Experimental Update Center. It can installed in your Jenkins instance by:

i. [Plugin Management Tool](https://github.com/jenkinsci/plugin-installation-manager-tool)

```bash
$ java -jar plugin-management-tool.jar 
    -p gitlab-branch-source:experimental 
    -d <path-to-default--jenkins-plugins-directory> 
    -w <path-to-jenkins-war>
```

ii. Changing update center URL

You can install plugins from Experimental Update Center by changing the JSON URL used to fetch plugins data. Go to Plugin Manager, then to the Advanced tab, and configure the update center URL https://updates.jenkins.io/experimental/update-center.json. Submit, and then select Check Now. Experimental plugin updates will be marked as such on the Available and Updates tabs of the Plugin Manager.

The plugin release JSON looks like [this](https://codebeautify.org/jsonviewer/cbca3bb7).

4) Jenkins CI Incrementals Error Fixed

The plugin was forked from my user account. The POM still contained the repository link to the old repository. For the release I had to update the new repository (now hosted in `jenkinsci` org). Incidentally, it fixed the incremental error returned by Jenkins CI Server. The problem was with the vague error message `bad http request` that was returned by the CI. Gavin might fix it, it is an open issue anyway. Now CI builds return the incrementals link.

5) Add Trusted Members strategy from Forked Merge Request

This issue was suggested by Oleg. The merge request can be made by any user so they can also modify the `Jenkinsfile` to their favour and cause data leak like environment variables etc. So the merge request build by our plugin should provide a strategy that allow Fork Merge Requests that are only made by trusted members like members with Developer/Maintainer/Owner permission.

While adding this feature, I got hit with another unknown bug that was caused the swapping of functioning of Origin MR discovery and Fork MR discovery. 

So I fixed both origin mr discovery and fork mr discovery traits to work correctly. Then added a new strategy in `ForkMergeRequestTrait` called `TrustPermission` that builds Merge Requests from Forks which are made only by users who are trusted members of the project. The strategy currently detects whether a MR is trusted or not but I haven't yet figured out how to skip the build for the untrusted MRs.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58666](https://issues.jenkins-ci.org/browse/JENKINS-58666), [JENKINS-58680](https://issues.jenkins-ci.org/browse/JENKINS-58680) | [#12](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/12)

6) Added slides for Jenkins World GSoC Report

Added 3 slides on GSoC Report Jenkins World which can be found [here](https://drive.google.com/file/d/1F-ETgGHnBG6BeqUY-dJ0y_YbPskH58lH/view?usp=sharing).

## Pending Work

There are a few issues with the same order of their priorty:

1. Make Webhook Work - [JENKINS-58593](https://issues.jenkins-ci.org/browse/JENKINS-58593).

2. Fix Trusted Member Strategy - [JENKINS-58666](https://issues.jenkins-ci.org/browse/JENKINS-58666)

3. Fix indexing log in GitLab Group Job Type - [JENKINS-58446](https://issues.jenkins-ci.org/browse/JENKINS-58446)

4. Add Tag Build Strategy - [JENKINS-58477](https://issues.jenkins-ci.org/browse/JENKINS-58477)

Once the above issues are fixed, we can make our stable release to Jenkins Update Center. Also will be adding blog about the plugin in the `jenkins.io`.

Ciao.