---
title: 'GSoC coding period: The plugin release week'
date: '2019-07-20'
tags: jenkins, gsoc, gitlab
---

This week we are planning to cut a release to Jenkins Update Center. This really depends on the progress we make in this week. We have some important bugs to fix before we make the release.

I have requested help from ArgelBargel and his plugin users in this [issue](https://github.com/Argelbargel/gitlab-branch-source-plugin/issues/105).

## Alpha release

I cut an alpha release which can be found [here](https://github.com/jenkinsci/gitlab-branch-source-plugin/releases/tag/gitlab-branch-source-v0.0.4-SNAPSHOT).

## Issues

1) Web Hooks support:

Currently I am try to add Web Hooks support to the plugin. Webhooks registration on GitLab Server for `Navigator` and `Source` is working

![Screenshot from 2019-07-13 17-09-05](https://user-images.githubusercontent.com/23079344/61171176-8fa05080-a591-11e9-91af-3beb13519ed0.png)

Although we have a few issues here:

* Unable to listen to post request on Jenkins instance. Any idea how to implement this?

* GitLab API doesn't support webhooks creation on a group. Although Group webhooks are available in Enterprise Edition but cannot take advantage of it. Currently it is a WIP in GitLab. So for Navigator, individually setting webhooks on all projects (so it takes a longer period to process).

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57760](https://issues.jenkins-ci.org/browse/JENKINS-57760) | [#25](https://github.com/baymac/gitlab-branch-source-plugin/pull/25)

2) Fix Jenkins CI build error

Jenkins CI build on GitLab Branch Source Plugin returned the following error:
```
[2019-07-14T03:45:08.934Z] [ERROR] Make sure `git status -s` is empty before using -Dset.changelist: [docs/img/gitlab-credentials.png, docs/img/gitlab-section.png, docs/img/gitlab-server.png, docs/img/gitlab-token-creator.png, src/main/webapp/images/16x16/icon-branch.png, src/main/webapp/images/16x16/icon-commit.png, src/main/webapp/images/16x16/icon-fork.png, src/main/webapp/images/16x16/icon-gitlab.png, src/main/webapp/images/16x16/icon-merge-request.png, src/main/webapp/images/16x16/icon-project.png, src/main/webapp/images/16x16/icon-tag.png, src/main/webapp/images/24x24/icon-branch.png, src/main/webapp/images/24x24/icon-commit.png, src/main/webapp/images/24x24/icon-fork.png, src/main/webapp/images/24x24/icon-gitlab.png, src/main/webapp/images/24x24/icon-merge-request.png, src/main/webapp/images/24x24/icon-project.png, src/main/webapp/images/24x24/icon-tag.png, src/main/webapp/images/32x32/icon-branch.png, src/main/webapp/images/32x32/icon-commit.png, src/main/webapp/images/32x32/icon-fork.png, src/main/webapp/images/32x32/icon-gitlab.png, src/main/webapp/images/32x32/icon-merge-request.png, src/main/webapp/images/32x32/icon-project.png, src/main/webapp/images/32x32/icon-tag.png, src/main/webapp/images/48x48/icon-branch.png, src/main/webapp/images/48x48/icon-commit.png, src/main/webapp/images/48x48/icon-fork.png, src/main/webapp/images/48x48/icon-gitlab.png, src/main/webapp/images/48x48/icon-merge-request.png, src/main/webapp/images/48x48/icon-project.png, src/main/webapp/images/48x48/icon-tag.png] (use -Dignore.dirt to make this nonfatal) -> [Help 1]
```

For more info, see this discussion [thread](https://groups.google.com/forum/#!topic/jenkinsci-dev/Df_hT_qeINo).

Remove `webapp/` from repository and added to `.gitignore`

When the plugin is compiled, it generates the webapp images of different sizes from the SVGs. So Jenkins CI builds failed. Fixed it in [`513150c`](https://github.com/jenkinsci/gitlab-branch-source-plugin/commit/513150cf4a5f3c62c7b5143525997046cae6e671) & [`d3982fb`](https://github.com/jenkinsci/gitlab-branch-source-plugin/commit/d3982fb56905e800a0340b01618fb4afda5fe11b). 

The above assumption was false. Previously, I used `SVN Rasterizer` Maven Plugin which generate the images from SVGs. But that is not the case anymore since I removed it after adding once. Another thing is that nothing in `src/` should ever be in `.gitignore`. So the idea of adding webapp directory to `.gitignore` was scraped.

We can also ignore the error by adding a flag `-Dignore.dirt` in our `maven.config` file. But still not the best way to fix this.

There were still confusions with what caused the build to fail. I tried readding the SVGs and PNGs. In the process removed additional SVGs of Merge Request, Tags etc. Kept only logo, branch and project. Still that didn't fix the issue. 

Later, Gavin suggested maybe the `.gitattributes` caused the changes to the binary files during build. So we need to specify the binary files (by globbing) in `.gitattributes` to skip modifying. Based on this [doc](https://help.github.com/en/articles/configuring-git-to-handle-line-endings) added the following to `.gitattributes`:

```
# Denote all files that are truly binary and should not be modified.
*.png binary
*.jpg binary
```

Currently the deploy stage fails as the archives cannot be deployed incrementally. Haven't been able to figure it out yet. Raise a ticket and awaiting response.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58510](https://issues.jenkins-ci.org/browse/JENKINS-58510), [INFRA-2183](https://issues.jenkins-ci.org/browse/INFRA-2183) | [#1](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/1)

3) Suggestion by Justin on GitLab Pipeline Status Reporting

Justin suggested we can give user the option to get the Pipeline Stages status to GitLab or only the Job status. This is a planned feature and I will implement this soon. Presently the Job Status Reporting has been improved. Previously there were multiple entries for different jobs. Now the Job Names are moved into description. The names of the pipeline entries are now based on branch heads such as `jenkinsci/branch`, `jenkinsci/mr`, `jenkinsci/tag`.

![job-status-1](/assets/2019-07-20-gsoc-coding-period-plugin-release-week/job-status-1.png)

![job-status-2](/assets/2019-07-20-gsoc-coding-period-plugin-release-week/job-status-2.png)

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58525](https://issues.jenkins-ci.org/browse/JENKINS-58525) | [#3](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/3)


4) Readded missing icons for heads

Due to sending fix for the CI build issue, I removed the icons are readded a different set of icons and some icons were missing. We required a the missing icons that would be a part of various actions in our plugin. Added 3 more icons `mr`, `tag` and `commit`.

![icon-branch](/assets/2019-07-20-gsoc-coding-period-plugin-release-week/icon-branch.png)

![icon-tag](/assets/2019-07-20-gsoc-coding-period-plugin-release-week/icon-tag.png)

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58526](https://issues.jenkins-ci.org/browse/JENKINS-58526) | [#4](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/4)

5) Fix for SSH checkout of project(s)

SSH checkout had a bug as the uri scheme that was used to dissect the ssh remote url gave NPE. In order to fix it moved to a simpler implementation of getting the sshRemote.

```java
public static UriTemplate checkoutUriTemplate(@CheckForNull Item context,
                                                  @NonNull String serverUrl,
                                                  @CheckForNull String sshRemote,
                                                  @CheckForNull String credentialsId,
                                                  @NonNull String projectPath) {

        if (credentialsId != null && sshRemote != null) {
            URIRequirementBuilder builder = URIRequirementBuilder.create();
            URI serverUri = URI.create(serverUrl);
            if (serverUri.getHost() != null) {
                builder.withHostname(serverUri.getHost());
            }
            StandardUsernameCredentials credentials = CredentialsMatchers.firstOrNull(
                    CredentialsProvider.lookupCredentials(
                            StandardUsernameCredentials.class,
                            context,
                            context instanceof Queue.Task
                                    ? Tasks.getDefaultAuthenticationOf((Queue.Task) context)
                                    : ACL.SYSTEM,
                            builder.build()
                    ),
                    CredentialsMatchers.allOf(
                            CredentialsMatchers.withId(credentialsId),
                            CredentialsMatchers.instanceOf(StandardUsernameCredentials.class)
                    )
            );
            if (credentials instanceof SSHUserPrivateKey) {
                return UriTemplate.buildFromTemplate("ssh://" + sshRemote)
                        .build();
            }
        }
        return UriTemplate.buildFromTemplate(serverUrl+'/'+projectPath)
                .literal(".git")
                .build();
    }
```

`sshRemote` is populated at the time of `retrieve(c, o, e, l)` method in `GitLabSCMSource`.

Also added `checkout credentials` in the Job creation UI instead of `credentials`.

This issue was reported by Justin and fixed by me.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58531](https://issues.jenkins-ci.org/browse/JENKINS-58531) | [#6](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/6)

6) Added Skip Notification in the plugin itself

`Skip Notification` is a trait which can be selected at the time of job creation. With this trait the GitLab Server will not be notified about the build status. There is a plugin called `Skip Notifcation Trait Plugin` which provides the trait for `BitBucket Branch Source Plugin`. Initially the plan was to extend `Skip Notification Trait Plugin` to add support for our plugin. But I felt it wasn't worth to reside in a different plugin as the `Skip Notification Trait Plugin` had only 270 installs in over a year time. So I decided to implement it inside our plugin itself.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58480](https://issues.jenkins-ci.org/browse/JENKINS-58480) | [#8](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/8)

7) Reduced API calls in fetch owner

I had implemented some messed up code in `GitLabSCMNavigator`. The `retrieveActions` method had to fetch the owner based on the `projectOwner` provided by user. Joseph noted that multiple API calls are being made to fetch the same details of the owner like `fullName`, `web_url` etc. So he implemented Model classes for `GitLabUser` and `GitLabGroup` which stored the necessary details. It helped me learn about Object Oriented Programming more. Due to this fix the `GitLab Group` job type now performs faster. 

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-58535](https://issues.jenkins-ci.org/browse/JENKINS-58535) | [#7](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/7)

8) Added GitLab4J groupfilter to include subgroups

This fix was again made by Joseph as he would suggested we can provide option to include a trait that allows `subgroup` projects to be included.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[#409](https://github.com/gitlab4j/gitlab4j-api/issues/409) | [#410](https://github.com/gitlab4j/gitlab4j-api/pull/410)

9) Added Jenkins interface to receive POST request from GitLab Server

The webhooks support is yet to be completely implemented. Some progress has been made with the implementation of `GitLabWebhookAction` which processes the HTTP request received from the GitLab Server where the webhook is created upon job creation.

![post-request](/assets/2019-07-20-gsoc-coding-period-plugin-release-week/post-request.png)
`Request received by Jenkins`

![webhook-execution](/assets/2019-07-20-gsoc-coding-period-plugin-release-week/webhook-execution.png)
`GitLab Server says the test request was successfully executed`

Also added implementation of `WebhookRegistrationTrait` that helps you override the default webhook management and choose if you want to use a different context (say Item) or disable it altogether.

See [1fef8766](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/2/commits/1fef8766c0a82ebc06224ff6d29a966567e3f967).

See [acc83f96](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/2/commits/acc83f96a60dcec7f549f284dc13d5722c6ffe71).


## Other work:

Justin will be taking a closer look at [#5](https://github.com/jenkinsci/gitlab-branch-source-plugin/pull/5) to see if he can find some cue to fix it.

A new release of `GitLab API Plugin` was made this week which can be found [here](https://github.com/jenkinsci/gitlab-api-plugin/releases/tag/gitlab-api-1.0.3).

Mock Presentation for second coding phase will be on 22nd July, Monday at about 4:00 pm UTC. 

A top-level look at the presentation:

* 15 mins in total
    * Recap what was done in the 1st eval (2-3 mins)
    * Touch new added since the 1st eval (2-3 slides) (2-3 mins)
    * Light demo (5-8 mins)

## Pending Work:

1. Complete Web Hook feature implementation

2. All the remaining bugs are optional before the 2nd phase evaluation on 23rd

3. Prepare presentation

4. Send PR to update the project page

5. Write blog for `jenkins.io`

6. Update repository documentation

7. Release plugin to update center
