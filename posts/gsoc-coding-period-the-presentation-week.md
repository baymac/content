---
title: 'GSoC coding period: The presentation week'
date: '2019-06-29'
tags: jenkins, gsoc, evaluation, gitlab
---

This week the GSoC Coding Phase 1 officially ended. We had a presentation on Tuesday (9:00 am UTC). It was a nice experience demonstrating your work done to others. After that I spent this week mostly reflecting things that were accomplished and things that could be improved in the phase 1.

A few takeaways from coding phase 1:

1. My contribution to the code:

    ![code-contribution-1](/assets/2019-06-29-gsoc-coding-period-the-presentation-week/code-contribution-1.png)

    As much as I spent writing codes, I also spent a bunch of time undoing the changes. That is how we improved our codebase, making it more readable, faster, modular with every new changes.<br>

    [GitLab Branch Source Plugin](https://github.com/baymac/gitlab-branch-source-plugin):

    * 18 PRs opened (17 merged)
    * 35 issues raised (29 fixed)
    * 3 pre releases<br><br>

    [GitLab API Plugin](https://github.com/jenkinsci/gitlab-api-plugin)

    * 1 PRs opened(1 merged)
    * 2 issues raised (2 fixed)
    * 2 releases<br><br> 
    
    [GitLab4J](https://github.com/gitlab4j/gitlab4j-api)

    * 7 issues raised (6 fixed)<br><br>

2. A clean git repository and agile development:

    I learnt about git a lot more than I did from any of my previous projects. It is important to have a clean git
    history. You can find the relevant changes for whatever you are looking for in the future.

    * Used a lot of `git rebase` to modify my project history (either squashing commits).
    * Always take new branches from base branch that will be sending pr to.
    * Keep commits consise and also write sensible commmit message which you can look back in future and undestand what it does.
    * Squash commits with related or minor changes into one.
    * `git commit --amend --no-edit` was also used heavily.
    * `git reset` is very risky and only to be done in local repositories (not when commits are published in public repo).
    * Narrow down on your issues, write clearly about it, screenshot it. Then start working on that. Planning is more
    important than actually coding. I was very easily tempted to just focusing on the code and it was clearly a bad idea
    when at times I had to undo hours worth of code.
    * Agile is just a jargon. You basically have a scrum board or kanban board. You raise your issues there explaining it and then start working on a pr related it. Try to keep it as concise as possible.<br><br>

3. Presentation:

    The presentation was done right. The mock presentation hadn't gone very well as my internet was unstable. This time
    there were no such issues. I received good appraisal from mentors. The demo can be found [here](https://t.co/NfahzzdMgb). My blog on jenkins.io is also live which can be found [here](https://jenkins.io/blog/2019/06/29/phase-1-multibranch-pipeline-support-for-gitlab/). The good news came on Friday that I passed my first phase evaluation with a nice constructive feedback from my mentors.

4. Reflections:

    Looking back in time I really feel good that I came this far accomplishing things I wasn't so sure I could. I think we need to believe first and act second, that way the problem becomes easier to solve. Jenkins has excellent documentation and community members which really helped me to build my project. This one month I learnt a lot about how jenkins works, debugged large codebase, learnt Java design patterns, interacted with people from different countries.

    In 2nd phase, I will try to learn from my mistakes. I think phase 1 was not absolutely efficient. The productivity could have increased had it been for better management. I am sometimes swayed by the motivation of fixing an issue that I almost end up undoing a lot of those fixes. I need to stick to agile development. Ask for help wherever necessary rather than trying to fix it all on my own. It can save me a lot of time.

    The presentation could have been improved as well. The initial few slides on overview of the project could have been more specific rather than moving away from the target. Also I improve my oration for better expression of my work. I hope to fix all these in phase 2.

## Updates on GitLab Branch Source

After last week I made a lot of improvements in the codebase at one go. But since I messed up a bit, I decided to return to my past state and fix issues one by one. I raised a really really long [PR-20](https://github.com/baymac/gitlab-branch-source-plugin/pull/20). The plan is to do a minimal code review and merge it. I fixed some simple bugs like:

* Removed additional constants from GitLabSCMBuilder.

* Fixed checkouturitemplate and mr refspec in GitLabSCMBuilder.

* Added icons.

* Fetching StandardUsernamePassword instead of PAT in GitLabSCMNavigator.

* Fixed checkout credentials in GitLabSCMSource.

* Filtering based on serverUrl will move to serverName soon.

Here is the working of GitLab Branch Source Plugin (in pictures):

![new-job](/assets/2019-06-29-gsoc-coding-period-the-presentation-week/new-job.png)

GitLab Branch Source will add support for two types of job:

* Folder Organisation (GitLab Group) - _performs Multibranch Pipeline Jobs for multiple repositories inside a User or a Group._
* Multibranch Pipeline Jobs - _performs CI on repositories with multiple branches e.g feature branches for MRs._

Select `Multibranch Pipeline Jobs` | Enter `Item Name` | Select `Ok`

![add-branch-source](/assets/2019-06-29-gsoc-coding-period-the-presentation-week/add-branch-source.png)

Assuming you have configured the GitLab Servers either using `Configure System` UI or JCasC yaml.

![config-branch-source](/assets/2019-06-29-gsoc-coding-period-the-presentation-week/config-branch-source.png)

Select `Server` -> If you have multiple server entries with same server url then it doesn't matter whichever server you select, it will always select the first one it finds in the list. It is because the server filter is based on server url. [JENKINS-58261](https://issues.jenkins-ci.org/browse/JENKINS-58261)

Select `Credentials` -> This is checkout credentials, this will be either `SSH Username with private key` type or `Username with password`.

Enter `Owner` -> This is can be username only at the moment. Based on this field the `Project` listbox is populated. E.g. 
    
I am trying to add support for `groups` and `subgroups`. E.g.
* If you want a group project -> enter `<groupname>`
* If you want a subgroup project -> enter `<groupname>\..\<subgroupname>` 
See [JENKINS-58269](https://issues.jenkins-ci.org/browse/JENKINS-58269)

If no `Owner` is entered, no projects is populated. This UI needs to be fixed see [JENKINS-58265](https://issues.jenkins-ci.org/browse/JENKINS-58265)

Select the `project` you want to perform the job on. There is one minor issue here. See [JENKINS-58264](https://issues.jenkins-ci.org/browse/JENKINS-58264).

![traits-branch-source](/assets/2019-06-29-gsoc-coding-period-the-presentation-week/traits-branch-source.png)

There are multiple configurations (which is known as `traits` in Jenkins) that you can add to the branches, merge requests and more. The above three are pretty basic for configuration on `branch`, `origin merge request` and `fork merge request`. You can test more traits by selecting `Add`. I will write a separate blog posts on how to leverage these traits in Multibranch Pipeline Jobs.

Select `Save`.

![indexing-branch-source](/assets/2019-06-29-gsoc-coding-period-the-presentation-week/indexing-branch-source.png)

Here the branch indexing is done successfully. It checks if `JenkinsFile` is present in the branch. It also detects an MR and skips indexing the branch that was used for sending the MR (in this case `master`).

![overview-branch-source](/assets/2019-06-29-gsoc-coding-period-the-presentation-week/overview-branch-source.png)

Select `Status` to get an overview of detected branches and merge requests. Tags support hasn't been added yet see [JENKINS-58265](https://issues.jenkins-ci.org/browse/JENKINS-58265).

Select a branch (e.g. `develop`, `MR-2` etc.) | Select any build history (e.g. #1, #2 .. etc.) | Select `Console Output` 

![build-output](/assets/2019-06-29-gsoc-coding-period-the-presentation-week/build-output.png)

While building the plugin should check that checkout repository contains `Jenkinsfile` but there is some problem in the implementation of `GitLabSCMFile` see [JENKINS-58263](https://issues.jenkins-ci.org/browse/JENKINS-58263).

But without the check the pipeline job on the branch should work fine because the during checkout branches were detected based on `Jenkinsfile` being found in their root dir. My build fails because my `Jenkinsfile` has invalid pipeline as code.

`GitLab Group` job type is not tested. It should have very similar bugs. Right now focusing on only the `Multibranch Pipeline jobs` would make it easier and faster to fix bugs and replicate the same changes for `GitLab Group` job type.

Webhook support is also broken see [JENKINS-57760](https://issues.jenkins-ci.org/browse/JENKINS-57760).

## WIP

I have started using Agile for project development.

Currently I am working on 2 important improvements:

|  Issues 	|   
|    ---	|
| [JENKINS-58261](https://issues.jenkins-ci.org/browse/JENKINS-58261), [JENKINS-58263](https://issues.jenkins-ci.org/browse/JENKINS-58263) |

More issues [here](https://issues.jenkins-ci.org/browse/JENKINS-57540).

I hope to fix them by this weekend and make an alpha/beta release.