---
title: 'GSoC coding period: The mock presentation week'
date: '2019-06-15'
tags: jenkins, gsoc, jcasc, gitlab
---

This week there were 2 new releases of GitLab Branch Source Plugin. Both of them were enhancement and fix releases. One was `v0.0.2-SNAPSHOT` for last week's work and for this week's work `v0.0.3-SNAPSHOT` was released. It seems like the GitLab Server Configuration in Jenkins has been resolved and we can move on to GitLab Branch Source implementation. 

The meeting was about to happen on Tuesday but due to Marky's other commitments it was shifted to Friday. I had some more time to improve my presentation and also work on other issues. It was an overall effective week with some major bug fixes and team discussions to made on pending issues. 

## Issues Fixed

1) Merged Joseph's PR on JCasC and other tools

This PR was sent by Joseph Peterson which included support for Jenkins Code as Configuration (JCasC), incremental tools,
maven checkstyle, remove deprecated methods, add travis CI etc. It took me a few days to fully understand how JCasC works. 

I was looking for ways to improve handling secrets in configuration yaml file. I had confusions with Global Property and Environment Variables. Joseph later cleared on difference, JCasC only able to access environment variables and not global properties. JCasC documentation suggested using environment variables pointing to the secrets but I was looking at something that did not require use of any external services like docker secrets, azure secrets or kubernetes vault etc. In the mailing list, I was suggested we can simply store a plain text credentials in an environment variable and using it to populate the credentials field in JCasC. Although it should work just fine but storing secrets in plain text might need not be a very secured way of storing credentials even if it is not exposed in the front end. There's got to be a better way. I have ideas similar to Travis AES encryption of files. When you run travis-encrypt on a file, it generates an encoded file which can added to the repository. The keys to decrypt the files are added to travis as environment variable. Using openssl they file can decrypted. Later, once my project takes shape I will work on this.

JCasC is super cool. It lets you configure your Jenkins instance via a yaml file and you do not have to sift through the UI to find your settings. So our settings can now be configured as follows:

```yaml
credentials:
  system:
    domainCredentials:
      - credentials:
          - gitlabPersonalAccessToken:
              scope: SYSTEM
              id: "i<3GitLab"
              token: "XfsqZvVtAx5YCph5bq3r" # gitlab personal access token

unclassified:
  gitLabServers:
    servers:
      - credentialsId: "i<3GitLab"
        manageHooks: true
        name: "gitlab.com" # optional
        serverUrl: "https://gitlab.com"
```

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57886](https://issues.jenkins-ci.org/browse/JENKINS-57886) | [#7](https://github.com/baymac/gitlab-branch-source-plugin/pull/7)

2) Added README

Our repository lacked a README. So the README file contains all the required configurations to setup a GitLab Server in your Jenkins instance. Also some related information to get started with Jenkins.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57803](https://issues.jenkins-ci.org/browse/JENKINS-57803) | [#11](https://github.com/baymac/gitlab-branch-source-plugin/pull/11)

3) Bumped checkstyle version from 8.17 to 8.18

There was some security warns related to the 8.17 version. So upgraded to a later version. This was done by GitHub dependabot.

|  Pull Request 	|   
|---	    |
| [#12](https://github.com/baymac/gitlab-branch-source-plugin/pull/12)

4) Rename Package

The GitLab Server Configuration package was again repackaged under a different name which is `gitlabserverconfig`. It was previously `gitlabserver`. 

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57943](https://issues.jenkins-ci.org/browse/JENKINS-57943) | [#13](https://github.com/baymac/gitlab-branch-source-plugin/pull/13)

5) Used regex to verify JCasC test for name field

The name field in GitLab server configuration is auto generated with a random alphanumeric numbers where the last 4 digits are random. As the program doesn't know the previously what random number will be generated so we need to match pattern. So the expected output yaml looks like:

```yaml
servers:
- credentialsId: "i<3GitLab"
  manageHooks: true
  name: "gitlab-[0-9]{4}"
  serverUrl: "https://gitlab.com"
```
In the `should_support_configuration_export()` method in configuration test we use:

```java
assertThat(exported, matchesPattern(expected));
```

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57948](https://issues.jenkins-ci.org/browse/JENKINS-57943) | [#14](https://github.com/baymac/gitlab-branch-source-plugin/pull/14)

6) Add HashMap for GitLab filter methods

This PR was closed without getting merged. There was previously confusion with HashMap as this data structure is not supported by JCasC, see [here](https://github.com/jenkinsci/configuration-as-code-plugin/pull/900). Joseph said,  I feel like your pre-optimizing for a use case that is never going to be relevant. Most users will have two servers, gitlab.com and their private instance. With 10 or 100 servers in the list, even though that will most likely be an unrealistic scenario, an array lookup is still going to be pretty fast." So we decided that adding HashMap isn't the most quintessential thing to do.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57889](https://issues.jenkins-ci.org/browse/JENKINS-57889) | [#15](https://github.com/baymac/gitlab-branch-source-plugin/pull/15)

7) Add Java 8 streams and remove HashMap

Since this PR was made from `remove-hashmap` which was branched from the previous branch `hash-servers` so there was a requirement for removing HashMaps implementation. This branch should have been done from the `develop` branch instead. In this PR I added Java 8 streams for the filter methods using only a `List` Collection. Some minor changes to the README fixing links and other JCasC instructions. 

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57976](https://issues.jenkins-ci.org/browse/JENKINS-57976), [JENKINS-57686](https://issues.jenkins-ci.org/browse/JENKINS-57686) | [#16](https://github.com/baymac/gitlab-branch-source-plugin/pull/16)

8) Fixed UI to reflect the state of the object

GitLab server configurations required a unique name to co-exist. If there are conflicting names the first seen GitLab server entry is persisted. But the problem was in the `GitLabServers` which extended `GlobalConfiguration` that when the entries was saved, it was serialized into XML as expected but the UI didn't reflect the same even if it was refreshed. It needed to be restarted to reflect the persistent state of the object of `GitLabServers` which is a list of servers. 

This issue was fixed by Joseph. This required an upgrade to Jenkins version `2.150.3` which brings `PersistentDescriptor` interface which allows configuration to persist fields with getters and setters. So there was no need to override the `configure` method to persist our configuration. So now the persistent xml data is loaded from the disk everytime the object is instantiated which is when refresh is called on the current page. The `load()` method is annotated as `PostConstruct` so it get automatically invoked after constructor and field injection.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57988](https://issues.jenkins-ci.org/browse/JENKINS-57976), [JENKINS-57988](https://issues.jenkins-ci.org/browse/JENKINS-57686) | [#16](https://github.com/baymac/gitlab-branch-source-plugin/pull/16)

9) Added Validation for GitLab Server Url

Previously GitLab Server Url only validated if it was of type `gitlab.com` community edition. GitLab Servers for community self-hosted, gold and ultimate servers was not be validated. Now the validation occurs in two phases:

  i. First, it checks if the URL is a valid URL scheme else it raises an error of type `Malformed URL Exception`.

  ii. Second, if the URL scheme is valid. It makes an API call to the GitLab Server to fetch a single project from the public GitLab server `/project` endpoint. If no project is found then it returns null. In either ways if no exception is raises, it suffices to be a valid GitLab Server endpoint.

The code looks like:

```java
public static FormValidation doCheckServerUrl(@QueryParameter String value) {
            Jenkins.get().checkPermission(Jenkins.ADMINISTER);
            try {
                new URL(value);
            } catch (MalformedURLException e) {
                LOGGER.error("Incorrect url - %s", value);
                return FormValidation.error("Malformed url (%s)", e.getMessage());
            }
            if (GITLAB_SERVER_URL.equals(value)) {
                LOGGER.info("Community version of GitLab - %s", value);
            }
            // Fetch a single project using an empty string as the access token
            GitLabApi gitLabApi = new GitLabApi(value, ""); 
            try {
                gitLabApi.getProjectApi().getProjects(1, 1); // fetches 1 project in 1 page
                return FormValidation.ok();
            } catch (GitLabApiException e) {
                LOGGER.info("Unable to validate serverUrl - %s", value);
                return FormValidation.error(Messages.GitLabServer_invalidUrl(value));
            }
        }
```

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57747](https://issues.jenkins-ci.org/browse/JENKINS-57976), [gitlab4j-#384](https://github.com/gitlab4j/gitlab4j-api/issues/384)| [#17](https://github.com/baymac/gitlab-branch-source-plugin/pull/17)

10) Fixed git history by rebasing

I had been handling git poorly e.g. adding commits for small changes, taking out branches from feature branches, adding merge commits etc. This let to a bad commit history. It is a good practice to keep Git commit history linear by using rebase. So Joseph suggested for a meeting to show how `git rebase` can help with the modifying the git commits by droping, squashing, editing etc.

So when the first PR was made from `add-validation` branch, here's what the history looked like:

![bad-git-history](/assets/2019-06-15-gsoc-coding-period-the-mock-presentation-week/bad-git-history.png)

`Bad Git History`

The `add-validation` branch was taken from `hash-servers` branch which was never merged. So there was no need for commits that would never be merged into `develop` branch. So we decided to drop those commits. So we rebased `add-validation` onto `develop`:

```bash
git checkout add-validation
git rebase develop
```

Rebasing `add-validation` onto `develop` solve all merge conflicts by picking all changes from develop's branch side. which means skipping/dropping commits for the part in this case. otherwise resolve the merge conflicts to the desired effect. `git status` to get a hint of what current conflict is. `git rebase --continue`, until there is no conflict usually git GUI's can be somewhat more helpful. 

The result:

![good-git-history](/assets/2019-06-15-gsoc-coding-period-the-mock-presentation-week/good-git-history.png)

`Good Git History`

Now the git history for the `add-validation` PR merge looks linear. We dropped and squashed commits while rebasing `develop` onto `add-validation` and then a fast forward merge into `develop`.

So we decide on 3 things from here:

i. Take branches only from base branch i.e. `develop` branch.

ii. Perform `git rebase` before merging in order to avoid merge commits and keep commit history linear.

iii. Squash git commits into one if those are related changes.

|  Pull Request 	|   
|---	            |
| [#18](https://github.com/baymac/gitlab-branch-source-plugin/pull/18)


## Presentation

So Friday arrived and I had a little practise before giving the presentation. I also added Speaker notes in the Google slides to help me recollecting my points. It was a mock presentation on the work done so far. Marky and Justin were present. Everything was fine sans the internet issues. Apart from that it was fun and I also had a practise before the phase 1 evaluations.

Find my presentation slides [here](https://drive.google.com/open?id=1Rv8e6mzGi8aHLSSMboQsazQv_9G0NsvG).

View my presentation video [here](https://drive.google.com/file/d/1KmWZNV2tqfGsNKBAi9_8eao1JnId3-Wx/view?usp=sharing).