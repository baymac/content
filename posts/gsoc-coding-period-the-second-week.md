---
title: 'GSoC coding period: The second week'
date: '2019-06-08'
tags: jenkins, gsoc, jcasc, gitlab
---

This post contains summary of work done during the second week of GSoC Coding period from 3rd - 8th June. Most of the work was around fixing some important bugs and implementing JCasC support for our plugin. No new releases were published. 

Monday and Tuesday were ineffective days due to my personal reasons. Work started Wednesday onwards.

### Fixed Issues:

1) Add databound setters for the non-mandatory fields

This fix was made in order to keep Data Bound Construtor to have minimal parameters. In future, if there was a need to change of signature, there will be unlikely a binary incompatibility issue. So we decided to only use `serverUrl` as the mandatory parameter and set it is as `final` variable. Later, `name` field was also moved into the Data Bound Constructor since `name` was marked `@NonNull` SpotBugs wants it to be instanied by constructor as well. 


|  Issue 	|   Pull Request	| 
|---	    |---	            |
| [JENKINS-57894](https://issues.jenkins-ci.org/browse/JENKINS-57894) | [#8](https://github.com/baymac/gitlab-branch-source-plugin/pull/8)	|


2) Add Loggers to GitLabServer and GitLabServers class

Added Simple Logging Facade for Java (SLF4J) logging for debugging. To enable debug logging in the plugin: 

```
i) Go to Jenkins -> Manage Jenkins -> System Log

ii) Add new log recorder

iii) Enter your desired name

iv) On the next page, enter 'io.jenkins.plugins.gitlabserver' for Logger, set log level to FINEST, and save

vi) After performing GitLab server configuration, you should see output in the log page.
```

|  Issue 	|   Pull Request	| 
|---	    |---	            |
| [JENKINS-57900](https://issues.jenkins-ci.org/browse/JENKINS-57900) | [#10](https://github.com/baymac/gitlab-branch-source-plugin/pull/10)|

3) Remove synchronisation from GitLabServers methods

GitLabServers methods `addServer`, `removeServer` etc were synchronized. As Jeff pointed out, there could be a race condition would be two admins making changes to the config at the same time. Justin pointed out that synchronistaion cause performance bottlenecks or, worse, deadlocks. We decided there isn't an urgent requirement of synchronisation and remove it unless there seems to be a need for it. 

|  Issue 	|   Pull Request	| 
|---	    |---	            |
| [JENKINS-57892](https://issues.jenkins-ci.org/browse/JENKINS-57892) | [#9](https://github.com/baymac/gitlab-branch-source-plugin/pull/9)

4) Provide a unique name for GitLab Server configuration

Previously, all the GitLab Servers function used the `serverUrl` to distinguish between different gitlab server configurations. But user might want to setup a multiple configurations for one GitLab Server with different authentications. So we decided to filter GitLab servers based on an unqiue id. So I added the following method to generate a unique name for GitLab Server:

```java
private String getRandomName() {
        return String.format("%s-%s", SCMName.fromUrl(this.serverUrl, COMMON_PREFIX_HOSTNAMES),
                RandomStringUtils.randomNumeric(SHORT_NAME_LENGTH));
    }
```

Generates names like `gitlab-4531`, `gitlab-7632`, `gitlab-8343` etc.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57880](https://issues.jenkins-ci.org/browse/JENKINS-57880) | [#5](https://github.com/baymac/gitlab-branch-source-plugin/pull/5)

5) Upgrade to new API plugin release 1.0.2

GitLab API Plugin upgraded to a new GitLab4J release `4.11.4`. It contains Scope list of APIs as enum. Before I was using `ApplicationScope`. Now an API scope enum was defined within `AccessTokenUtil` class itself.

Previously:

```java
List<String> GL_PLUGIN_REQUIRED_SCOPE = ImmutableList.of(
    Constants.ApplicationScope.API.toValue(), // api
    Constants.ApplicationScope.READ_USER.toValue() // read_user
);
```

Later:

```java
private static final List<AccessTokenUtils.Scope> GL_PLUGIN_REQUIRED_SCOPE = ImmutableList.of(
        AccessTokenUtils.Scope.API,
        AccessTokenUtils.Scope.READ_REGISTRY,
        AccessTokenUtils.Scope.READ_USER,
        AccessTokenUtils.Scope.READ_REPOSITORY
    );
```

Now the `createToken` method from `AccessTokenUtils` take a list of `Scope` enum instead of a list of `String`.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57850](https://issues.jenkins-ci.org/browse/JENKINS-57850) | [#2](https://github.com/baymac/gitlab-branch-source-plugin/pull/2)

6) Remove client api model classes

There were some model classes defined initially to store Username/password or Personal Access Token as objects. But since we depend on an API Plugin these classes were not required. We can directly class from Credentials Plugin classes such as `StandardUsernamePasswordCredentials` or our `PersonalAccessTokenImpl` class to pass a matcher or a pass the credentials itself.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57884](https://issues.jenkins-ci.org/browse/JENKINS-57884) | [#6](https://github.com/baymac/gitlab-branch-source-plugin/pull/6/)

7) Repackaging to separate GitLabServer and GitLabBranchSource codebase

The server codebase was repackaged into `gitlabserver` package name. And branch source fucntionality will be packaged into `gitlabbranchsource`. If in future we decide to take out this plugin then it would be easier to do it when we have separated codebases.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
| [JENKINS-57853](https://issues.jenkins-ci.org/browse/JENKINS-57853) | [#3](https://github.com/baymac/gitlab-branch-source-plugin/pull/3)

8) Fix delete server configuration code:

Previously:

```java
@Override
public boolean configure(StaplerRequest req, JSONObject json) throws FormException {
    req.bindJSON(this, json);
    return true;
}
```

```java
@Override
public boolean configure(StaplerRequest req, JSONObject json) throws FormException {
    servers = req.bindJSONToList(GitLabServer.class, json.get("servers"));
    save();
    return super.configure(req, json);
}
```

The user wasn't able to delete a server from configuration when only one configuration persisted in Jenkins. I didn't had time to find the exact cause, I will debug later to see what caused this. But this issue was solved with the above code changes.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
| [JENKINS-57751](https://issues.jenkins-ci.org/browse/JENKINS-57751) | [#4](https://github.com/baymac/gitlab-branch-source-plugin/pull/4)

9) Adding JCasC support and CI

This PR was sent by Joseph who initially suggested we can add support JCasC in our plugin. Implemented a simple test that confirms you can configure gitlabservers which is a global configuration and export. So the plugin can be configured using a `yaml` file. 

The PR also consisted of a basic travis `yaml` file to build our plugin. And some other refractories that either served compatability with JCasC or removed deprecated codes.

|  Issue 	|   Pull Request	| 
|---	    |---	            |
|[JENKINS-57886](https://issues.jenkins-ci.org/browse/JENKINS-57886) | [#7](https://github.com/baymac/gitlab-branch-source-plugin/pull/7)

### Pending Issues:

Mostly trivial issues weren't fixed. Maybe require more research or technical expertise to implement these.

1) Restrict user from setting name field with an existing name

Although `name` field could still be modified by user. That can be a problem. Although there isn't much incentive for user to change the field.

|  Issue 	|   
|---	    |
| [JENKINS-57920](https://issues.jenkins-ci.org/browse/JENKINS-57920)

2) Removing name field from user view

Jeff suggested that maybe user never require to see the gitlab-server name. Although this depends on how our plugin develops and what our user requirements would be. As not sure about that, we are keeping the name field as it is, taking a note that we can remove it if user never requires it.

|  Issue 	|   
|---	    |
| [JENKINS-57883](https://issues.jenkins-ci.org/browse/JENKINS-57883)

3) Adding HashTable or HashMap for faster GitLab servers filters

Suggested by Jeff, we can use one of the above data structure to save user's time while performing functions like adding a server, updating a server or removing a server. Later, there were JCasC incompatibility issues that cropped up so this has been saved for future.

|  Issue 	|   
|---	    |
| [JENKINS-57883](https://issues.jenkins-ci.org/browse/JENKINS-57883)

### Work for upcoming week

1. Prepare presentation for Tuesday Meeting
2. Add web hooks management impl
3. Learn about JCasC and merge Joseph's PR
4. Add documentation to the plugin