---
title: 'GSoC coding period: The beginning'
date: '2019-06-01'
tags: jenkins, gsoc, java, gitlab
---

On Monday, May 27, the Google Summer of Code 2019 Coding Period started. Our team had been regularly doing sync ups to plan how to carry out the project successfully in the next 3 months. The project workflow had been agreed upon. It has been an exciting month since this is my first Google Summer of Code. I had fairly less experience in `Java` or `Object Oriented Design` per se but I wanted a new challenge. I have been following a few Java books and they have helped me be better at Java:

1. `Java 8 in Action` by Raoul, Mario and Alan
2. `Core Java for Impatient` by Cay S. Horstmann
3. `Effective Java` by Joshua Bloch

A friend knew I would need these books for GSoC so he gifted it to me. A shout out to my friend [Shibasis Pattnaik](https://github.com/shibasis0801/).

## Project Tools

We use JIRA `scrum boards`, `sprints` and `epics` to track our progress. Other resources include `mailing list` and `gitter channels`. For meetings, we use `Zoom` because it supports upto 100 users, has clients for desktop, browser, phone etc. Jenkins project page for gsoc 2019 is also live which give an overall idea of what's new.

## Before Community Bonding Work

I finished some of the nit things that would be helpful for our project. One such thing was the `GitLab API Plugin`. In Jenkins it is recommended to wrap any API inside a new plugin so that all the plugins depend on the same version of API. The API releases can be controlled by plugin owner himself. Basically it abstracts the API release testing for CVEs etc in one place rather than doing manually in each dependent plugin.

The problem with existing GitLab Plugin is that it doesn't have a separate API plugin and defines the API inside itself. This cause following problems:

1. Dedicated efforts to maintain an API for GitLab would required which is different from the scope of our plugin goals

2. Other plugins might not be able to reuse the GitLab Java APIs defined inside GitLab Plugin. Even if they extend this plugin, they will also be inheriting excess functionality and that might be a problem.

3. Keeping the APIs inside the plugin makes the codebase larger. Plugins should be lightweight and limited to their functionality. 

This has been correctly implemented by GitHub Plugins in Jenkins. We plan to follow the similar convention of 3 SCM plugins:

1. `GitHub API Plugin` - Wraps GitHub Java API 

2. `GitHub Plugin` - Build Trigger and Webhook Management

3. `GitHub Branch Source Plugin` - To support Multibranch Pipeline Jobs and Folder organisation

So, I wanted to implement a Java API for GitLab or maybe extend one. But that would have taken a lot of time as Oleg Nenashev suggested. To our good luck, I found a well maintained GitLab Java API repository by gmessner. This guy has been doing excellent work to maintain the codebase:

1. Responds to Issues within 24 hours

2. Listen to user's requests 

3. Releases frequent fixes and improvements

4. Implements some utils that are not part of GitLab API inherently e.g. creating `Personal Access Token` from Username/Password credentials.

This is the benefit of Open source that you get to reuse the work of others. So wrapping that API was fairly simple. Although we faced some problems with Maven because there were some dependencies that were targeting Java 9 and our Java level in Maven was set to Java 8. For this reason, `Maven Enforcer Plugin` caused an error during builds. After having discussions with Mentors and GitLab API owner, we decided to skip this enforcing check. If you find something like `Found Banned Dependency` error during the compilation of plugin then you might be interested to look at `gitlab-api` [POM](https://github.com/jenkinsci/gitlab-api-plugin/blob/master/pom.xml).

#### Releasing the GitLab API Plugin

On April 25, The plugin reached it's releasable state. After preparing a Wiki [page](https://wiki.jenkins.io/display/JENKINS/GitLab+API+Plugin), creating a JIRA [issue](https://issues.jenkins-ci.org/browse/HOSTING-751) under `HOSTING` project and sending a [pull request](https://github.com/jenkins-infra/repository-permissions-updater/pull/1128) to request persmission to host plugin in [jenkinsci](https://github.com/jenkinsci/) github organisation.

There was a plugin with the same name but the repository was abandoned. So the org admins decided to rename the existing plugin, archive it and fork our plugin into the org. So the final release happened on `April 29`. In case you want to host your own plugin you may refer to this [guide](https://jenkins.io/doc/developer/publishing/requesting-hosting/).

To release your plugin to maven central or jenkins.ci, use `mvn release:prepare release:perform -Dusername=<username> -Dpassword=<password>`. All the other configurations are setup my plugins parent POM. If you want to avoid typing credentials in terminal you may set the server configuration in the `~/.m2/settings.xml`.

## Community Bonding Work

On May 6, the results were declared. We had 7 projects selected in Jenkins project this year. And this was already a big year for Jenkins Project since there were more project ideas and gsoc participants than any previous gsoc editions. We had a long list of community bonding period action items like finalising meeting slots, preparing design document, get introduced to key stakeholders etc. 

The most important work was the design document. Every software development starts with planning. It helps you answer questions like 

1. What features wil you to implement? 
2. What will the Tech Stack consist of?
3. How will the users use the software?
4. How will "xyz" feature be implemented?
5. How will the timeline look like?

So I prepared a design document that aimed at containing the following things:

1. Objectives and expectations based on proposal 
2. Adjustments like reality mapping, technical details
3. Answer above questions
4. Create diagrams of operation  
5. Write mini user guides as if project is already done

Mostly everthing was covered sans a few things that could only be realised after implementing.

## Coding Phase I

This is when the fun begins or maybe a little bit of stress because you have a responsibility of pushing stable codes. The plan for this phase is to implement the `GitLab Branch Source Plugin`. This plugin will handle the pipeline builds for multiple branches. The current version of `GitLab Plugin` has a few problems and it does not fully support multibranch pipeline. If you want to know more about the project you may take a look at our project decription [page](https://jenkins.io/projects/gsoc/2019/gitlab-support-for-multibranch-pipeline/).

The coding work actually started a few days ahead as the design document, project workflow and other nit things were completed. I started writing code on May 23. My main motive of working on this plugin is to make it lightweight and make the code more readable. Also follow a general trend that is common with rest of SCM plugins. Most of the developers I interacted with suggested to take a look at `Gitea Plugin`. This plugin was developed recently like 2 years back and has a nice clean codebase. It is a plugin with very consistent coding style so easier to read. My work so far has been inspired from `Gitea Plugin` and `Github Plugin`. 

I am learning a lot of cool stuffs by reading their codebases. I have made an experimental release that sports GitLab Server Configurations in Jenkins to authenticate Jenkins Server with the GitLab Server to communicate via REST APIs.

You may take a look at the release [here](https://github.com/baymac/gitlab-branch-source-plugin/releases/tag/gitlab-branch-source-v0.0.1-SNAPSHOT) and test it. It lacks support for webhooks and a documentation right now. These updates will be pushed by the end of this week. 

The issue tracker can be found [here](https://issues.jenkins-ci.org/browse/JENKINS-57803?jql=project%20%3D%20JENKINS%20AND%20component%20%3D%20gitlab-branch-source-plugin). 

The source code can be found [here](https://github.com/baymac/gitlab-branch-source-plugin/)

### Interesting things implemented

1) `Use of Java 8 streams wherever possible` - This will make the code look more concise and easy to use for multi-threading. I am implementing it on my plugin as I am learning new stream APIs from `Java 8 streams` book. I have used behaviour design pattern which lets you modify the function at the function call itself. User requirements constantly change so using this pattern lets you easily modify the function with lesser probability of errors. For example, I am using a predicate to compare the GitLab Server URL with our predefined URL.

Previously:
```java
private synchronized boolean removeServer(@CheckForNull String serverUrl) {
    List<GitLabServer> endpoints = new ArrayList<>(getServers());
    modified = false;
    for (Iterator<GitLabServer> iterator = endpoints.iterator(); iterator.hasNext(); ) {
        if (serverUrl.equals(iterator.next().getServerUrl())) {
            iterator.remove();
            modified = true;
        }
    }
}
```

After using predicate:
```java
private synchronized boolean removeServer(@CheckForNull String serverUrl, Predicate<GitLabServerPredicate> p) {
    List<GitLabServer> endpoints = new ArrayList<>(getServers());
    modified = false;
    for(GitLabServer endpoint : endpoints) {
        if(p.test(serverUrl, endpoint)) {
            endpoints.remove(endpoint);
            modified = true;
        }
    }
}

private interface GitLabServerPredicate {
    boolean test(String serverUrl, GitLabServer gitLabServer);
}
```

You can now pass a lambda at function call. In this case, `(serverUrl, gitLabServer) -> serverUrl.equals(gitLabServer.getServerUrl())`. In future if you want to pass a new lambda, you only need to define a new predicate interface and pass that predicate in the `removeServer(String, Predicate<>)` method. You will not have to mess inside the function code itself which is proven to be more error prone. If you are new to predicate it may look complex at first but when you start implementing yourself you get the hang of it.

2) `Generating Personal Access Token from inside Jenkins` - This is a feature implemented by GitHub plugin. It lets you create personal access token to access your GitLab Server APIs without leaving Jenkins. You can use either text field to supply username/password to GitLab server or use the credentials plugin to persist username/password securely for future use. This feature is not inherently supported by GitLab APIs. These had to be implemented by doing some hacky stuff. Thankfully the GitLab4J APIs I am using had implemented it and all I had to do was make a function call. 

```java
String tokenName = UUID.randomUUID().toString();
String token = AccessTokenUtils.createPersonalAccessToken(
        defaultIfBlank(serverUrl, GitLabServer.GITLAB_SERVER_URL),
        login,
        password,
        tokenName,
        GL_PLUGIN_REQUIRED_SCOPE
);
createCredentials(serverUrl, token, login, tokenName); // creates an id credentials of PersonalAccessToken type and persists in Jenkins
```

At this point I realised why the previous plugin was a jumbled codebase. The developers had to implement these complex functions inside the plugin itself. But it really makes sense to abstract away these API calls.

![gitlab-token-creator](/assets/2019-06-01-gsoc-coding-period-the-beginning/gitlab-token-creator.png)
`GitLab Personal Access Token Creator`

3) `Use of groovy instead of jelly` - I preferred using `groovy` to write all the UI elements over `jelly`. This is because it felt more readable and groovy is a language that is highly flexible language and used in many aspects of Java development e.g. gradle, jenkinsfile etc. So I think this will be an opportunity to get started. Although it comes down to your personal preference. To learn about the tags library, refer to this [guide](https://reports.jenkins.io/core-taglib/jelly-taglib-ref.html).

4) `Detect only credentials that belongs our plugin` - This is a small bug that I discovered in Gitea Plugin. When the function calls the credentials plugin to fill the credentials items in the credentialsId list box, it passes a matcher for the authentication interface to find matches which leads to discovery other plugin's credentials implemented a similar interface. It should rather pass a matcher to check if it is the credentials belong to the credentials implementation class in our plugin. 

```java
 public ListBoxModel doFillCredentialsIdItems(@QueryParameter String serverUrl, @QueryParameter String credentialsId) {
    if(!Jenkins.getInstance().hasPermission(Jenkins.ADMINISTER)) {
        return new StandardListBoxModel().includeCurrentValue(credentialsId);
    }
    return new StandardListBoxModel()
        .includeEmptyValue() // also added this empty value to show none option when no credentials is added.
        .includeMatchingAs(ACL.SYSTEM,
                Jenkins.getInstance(),
                StandardCredentials.class,
                fromUri(serverUrl).build(),
                credentials -> credentials instanceof PersonalAccessTokenImpl
                // AuthenticationTokens.matcher(GiteaAuth.class) - method passed by Gitea Plugin
    );
}
```

### Caveats

I implemented a large part of the code without testing it and sent a pull request of almost 1200+ lines. This is a horrible style of coding which led to a loss of almost 3 entire days in search of one single bug. Trying all sorts of methods to see where it went wrong only to realise I missed an `@Extension` annotation over a `DescriptorImpl` method. `:facepalms:` It is a good idea to solve the problem in chunks, test it then implement the next part. This makes it easier to isolate bugs and also for mentors to review the code. Henceforth, I will follow a Test Driven Development (TDD) to find bugs at the compile time itself.

### Tools to aid development

If you have tried `JShell` then you might already know how much handy it can be in writing Java codes. It lets you define simple functions and run. It can be great for debugging. If using IntelliJ, you will find it under `Tools` menu. It also requires JDK 9+.

I started using Debugger for the first time and it actually helped me single out the bug in the code. A shout out to Justin, who introduced me to it.

### Work for this week

1. Prepare presentation of last week's work
2. Implement webhooks support, upgrade to latest gitlab-api-plugin
3. Prepare documentation

## Acknowledgements

Our team consists of:
1. [LinuxSuRen](https://github.com/LinuxSuRen) (The originator of this project idea)
2. [Marky](https://github.com/markyjackson-taulia) (The org admin who manages our project meetings to code reviews)
3. [Joseph](https://github.com/casz) (The chief code reviewer)
4. [Justin](https://github.com/justinharringa) (The mentor who helps with tech)
5. [Jeff](https://github.com/jeffpearce) (The mentor who helps with detailed code reviews)

Honorable Mentions:
6. [Oleg](https://github.com/oleg-nenashev) (The org admin who helps with all technical issues)
7. [Greg](https://github.com/gmessner) (The creator of GitLab4J APIs)
8. [Stephen](https://github.com/stephenc) (The creator of SCM related Plugins)

Also thanks to people who have helped me Jesse, Robert, Matt, Ullrich, Martin, Gavin.
