---
title: 'GSoC coding period: The fourth week'
date: '2019-06-21'
tags: jenkins, gsoc, jcasc, gitlab
---

This week work was to develop `Branch Source` part of the plugin. In Jenkins, <SCM>-Branch-Source Plugins implement SCM API Plugin to support configuration of the SCM specific jobs. SCM stands for source control system. A SCM can be `Git`, `Subversion`, `CVS` etc. There are other plugins as well that will help us implement our GitLab Branch Source Plugin. Here's the list of dependencies plugins that contains extension points to be implemented in branch source plugins:

1. `SCM API Plugin` - It contains a set of extension points such as `SCMSource`(Project), `SCMNavigator`(Group), `SCMHead`(Branch), `SCMEventListener`(Events like push, pulls) etc. to implement SCM related structures in Jenkins. They have very good documentation available [here](https://github.com/jenkinsci/scm-api-plugin/tree/master/docs). They also have SCM traits that lets you configure your CI operations on SCM, you can see all the traits [here](https://github.com/jenkinsci/scm-api-plugin/tree/master/src/main/java/jenkins/scm/api/trait).

2. `Branch API Plugin` - It contains extension points to implement the branching structure of your source control system in Jenkins. SCM projects(or repositories) can have multiple branches. They can decide which branches needs to be continuously integrated. To allow such multiple branch configuration `Branch API Plugin` has extension points such as `BranchBuildStrategy`, `BranchProjectFactory`, `BranchProperty` and more. You can see the extension points [here](https://jenkins.io/doc/developer/extensions/branch-api/). They also a proper documentation available [here](https://github.com/jenkinsci/branch-api-plugin/tree/master/docs).

3. `Git Plugin` - There are multiple SCMs that are based on Git e.g. GitHub, GitLab, Gitea etc. So there is a lot of commonality that is implemented by Git Plugin which is then extended by SCM specific plugins. Used in `GitLabSCMBuilder`, `GitLabBrowser` etc. There is actually a lot of common codes between both Git Plugin and GitLab Branch Source Plugin.

The list of `packages/classes/enums` implemented this week it's associated issues and takeaways:

### GitLabSCMSource

* `getRemote()` method builds the checkout url using `uri templates`.

* In `retrieve(head,listener)` using a different version of `getBranch(project, head.getName())` and not supplying `username` like gitea plugin. So .

* Unable to get sha of gitlab mr head. No api call for that. 

    ^ Fixed it by getting the head sha by `mr.getDiffRef().getHeadSha()`.

* Used enum for checking if mr is opened. 

* Check the path of `project` field. It has to be of the format `{username}/{project name}`.

* In `retrieve(criteria, observer, event, listener)` method, there is a need to check if the project is forked from some other repository but that can be tested only if GitLabApi is authenticated. Without authentication all forked projects return `null` for the `forkedFromProject` field.

In Gitea:

```java
if (request.isFetchPRs()) {
    if (giteaRepository.isMirror()) {
        listener.getLogger().format("%n  Ignoring pull requests as repository is a mirror...%n");
    } else {
        request.setPullRequests(c.fetchPullRequests(giteaRepository));
    }
}
```
A workaround:

```java
if (request.isFetchMRs()) {
    // If not authenticated GitLabApi cannot detect if it is a fork
    // If `forkedFromProject` is null it doesn't mean anything
    if (gitlabProject.getForkedFromProject() == null) {
        listener.getLogger()
                .format("%n  Unable to detect if it is a mirror or not still fetching MRs anyway...%n");
        request.setMergeRequests(gitLabApi.getMergeRequestApi().getMergeRequests(project));
    }
    else {
        listener.getLogger().format("%n  Ignoring merge requests as project is a mirror...%n");
    }
} 
```

* Extra check if PAT is supplied to GitLabApi

```java
// To verify the supplied PAT is valid
if(!gitLabApi.getAuthToken().equals("")) {
    gitLabApi.getUserApi().getCurrentUser();
}
```

* Checking branch using uri template

```java
listener.getLogger().format("%n Checking branch %s%n",
        HyperlinkNote.encodeTo(
                UriTemplate.buildFromTemplate(gitlabProject.getHttpUrlToRepo())
                        .literal("/blob")
                        .build()
                        .set("branch", b.getName())
                        .expand(),
                b.getName()
        )
);
```

* Origin project name will always the same as the source project name. Even if forked user changes the name of the project, for the api the name is always the same. So keeping originProject = project in merge request strategy(line ~305).

```java
String originOwner = m.getAuthor().getUsername();
// Origin project name will always the same as the source project name
String originProject = project;
```

* Skipped checking if the `project` and `originProject` are equal while creating a `MergeRequestSCMHead`. Since they are always equal.

```java
if (request.process(new MergeRequestSCMHead(
        "MR-" + m.getIid() + (strategies.size() > 1 ? "-" + strategy.name()
                .toLowerCase(Locale.ENGLISH) : ""),
        m.getIid(),
        new BranchSCMHead(m.getSourceBranch()),
        ChangeRequestCheckoutStrategy.MERGE,
        originOwner.equalsIgnoreCase(projectOwner)
                ? SCMHeadOrigin.DEFAULT
                : new SCMHeadOrigin.Fork(originOwner + "/" + originProject),
        originOwner,
        originProject,
        m.getTargetBranch()), ...
```

* While creating a new BranchSCMHead for the MR, using `m.getTargetBranch()` for target and  `m.getSourceBranch()` for originName. (~315)

* DiffRefs doesn't populate for a list of MergeRequests, it is only fetched when called MR individually by id. So making extra api calls to fetch them individually after fetching a list:

Previously

```java
for(final MergeRequest m : gitLabApi.getMergeRequestApi().getMergeRequests(project, Constants.MergeRequestState.OPENED)) {
    ...
}
```
After

```java
List<MergeRequest> mrs = gitLabApi.getMergeRequestApi().getMergeRequests(project, Constants.MergeRequestState.OPENED);
for (MergeRequest mr : mrs) {
    // Since by default GitLab4j do not populate DiffRefs for a list of Merge Requests
    // It is required to get the individual diffRef using the Iid.
    final MergeRequest m = gitLabApi.getMergeRequestApi().getMergeRequest(project, mr.getIid());
    ...
}
```

* Cannot call `diffRefs` on `request` since it will always return null. 

* What should be the urls for the icons in the `DescriptorImpl`?

### GitLabSCMBuilder

The purpose is to build a `GitSCM` for `GitLabSCMSource` 

* Generating remoteUrl for the constructor using `checkoutUriTemplate` method.

* RefSpec for MR is `merge-requests`.

* `checkoutUriTemplate` method returns the checkout uritemplate (ssh or https/http). The uri template needs to variable to be populated. E.g. `https://gitlab.com/<owner>/<project>.git`.

```java
checkoutUriTemplate(null, source.getServerUrl(), null, null)
                        .set("owner", source.getRepoOwner())
                        .set("repository", source.getRepository())
                        .expand()
```

* `build` method sets the remote and head and revision based on refspec of either Merge Request or Branch

### MergeRequestSCMHead

Represents a Merge Request in Jenkins. Merge requests are treated as branches with different properties.

* Need to implement an enum of RefSpec (later work)

* What is `refSpec`?

    See:<br>
    [git-scm](https://git-scm.com/book/en/v2/Git-Internals-The-Refspec)<br>
    [stackoverflow](https://stackoverflow.com/questions/44333437/git-what-is-refspec)

### BranchSCMHead

Represents a Git Branch in Jenkins

### GitLabBrowser

Implemented my own `GitLabBrowser` instead of using the one defined in `Git Plugin` since I wanted use uri template and the implemented version methods were not required as I am only tragetting GitLab Server version 10.0.0 onwards.

* Implements methods `diffLink`, `getFileLink`, `getDiffLink`, `getChangeSetLink`.

* Modifed the set for `diff` method.

### MergeWithGitSCMExtension

* What work does it do?

### MergeRequestSCMRevision

Implements revision for Merge Requests, very similar to BranchSCMRevision.

### BranchSCMRevision

Implements revision for branch.

### GitLabSCMSourceContext

It is a context of GitLabSCMSource required when using SCM Traits e.g. `SSHCheckout` etc.

### GitLabSCMSourceRequest

Required by SCM Traits.

* See how add the counterpart of `GiteaConnection` as a member variable. Using GitLabApi for now.

* Add support for TAGs

### WebHookRegistration - Enum

Contains different type of webhook registration in Jenkins.

### GitLabSCMFileSystem

To build file system of GitLab Project's repository.

* `lastModified()` returns the last activity in the project (maybe irrelevant if it tracks activities such as comment, issues etc). It returns a `long`. 

* `BuilderImpl` has methods `supportsDescriptor(scmDescriptor)` & `supportsDescriptor(scmSourceDescriptor)` which returns false. What these methods are required for?

### GitLabSCMFile

To represents files, directories or links in Project's repository.

* Need to make `children()` method work. It returns an `Iterable<SCMFile>`.

* Need to make `lastModified()` work as only know method in API is last activity.

* In overrided method `type()` trying to get file, throws error is not file. So there is no point of checking if it is a file. Fixed by returning `nonexistent` type in the `catch` block.

### GitLabSCMNavigatorContext

It is a context of GitLabSCMNavigator required for SCM Traits.

### GitLabWebhookListener 

* In our plugin's SCM API version `localconfiguration` is marked `@NonNull` earlier it was `@CheckForNull`.

* Received GitLab enterprise account, will test the Group webhooks and Project webhooks.

* GitLab API doesn't support api calls on Group webhooks. See https://github.com/gitlab4j/gitlab4j-api/issues/393

```java
// Since GitLab doesn't allow API calls on Group WebHooks. 
// So fetching a list of web hooks in individual projects inside the group
List<ProjectHook> projectHooks = new ArrayList<>();
for(Project p : gitLabGroup.getProjects()) {
    gitLabApi.getProjectApi().getHooksStream(p).forEach(projectHooks::add);
}
```

* When create webhook provide these functionality add secret token(see gitlab plugin), add more events give option for sslVerification

* In `register(owner, navigator, mode, credentialsId)` method check if webhook on USER projects can be supported. For now returning if `gitlabOwner` is a `USER`.

### GitLabSCMNavigator

* Unable to fetch GitLab Owner (User or Group) in one member variable. So using an enum instead.

```java
private GitLabOwner fetchOwner(GitLabApi gitLabApi) {
    try {
        Group group = gitLabApi.getGroupApi().getGroup(projectOwner);
        return GitLabOwner.GROUP;
    } catch (GitLabApiException e) {
        try {
            User user = gitLabApi.getUserApi().getUser(projectOwner);
            return GitLabOwner.USER;
        } catch (GitLabApiException e1) {
            e1.printStackTrace();
        }
        e.printStackTrace();
    }
    return null;
}
```

Now moved `fetchOwner(gitLabApi, projectOwner)` to `GitLabOwner` enum.

* In `visitSource(observer)` method, if `projectOwner` is a `subgroup`, it will only return projects in the subgroup and its subgroups. And if the `projectOwner` is a `user`, it will also return the projects in the groups and subgroups owned by user.

* Find if there can be a better data structure for `gitlabOwner`

* In `visitSource(observer)` method, using the following way to to skip user's group owned projects

```java
if(gitlabOwner == GitLabOwner.USER && p.getNamespace().getKind().equals("group")) {
    // skip the user repos which includes all organizations that they are a member of
    continue;
}
```

* In `visitSource(observer)` method, find if there can be an api call to check the project is empty. But now I am using a `getTree` method on the project and it throws error if the repository is empty.

```java
try {
    gitLabApi.getRepositoryApi().getTree(p);
} catch (GitLabApiException e) {
    observer.getListener().getLogger().format("%n    Ignoring empty repository %s%n",
            HyperlinkNote.encodeTo(p.getWebUrl(), p.getName()));
    continue;
}
```

* Printing `User` website Url instead of `Group` website url. Since GitLab doesn't have website for groups. But unfortunately this is not working our api. So API needs to be fixed. 

```java
if (gitlabOwner == GitLabOwner.USER) {
    String website = null;
    try {
        // This is a hack since getting a user via username finds user from a list of users
        // and list of users contain limited info about users which doesn't include website url
        User user = gitLabApi.getUserApi().getUser(projectOwner);
        website = gitLabApi.getUserApi().getUser(user.getId()).getWebsiteUrl();
    } catch (GitLabApiException e) {
        e.printStackTrace();
    }
    if (StringUtils.isBlank(website)) {
        listener.getLogger().println("User Website URL: unspecified");
        listener.getLogger().printf("User Website URL: %s%n",
                HyperlinkNote.encodeTo(website, StringUtils.defaultIfBlank(fullName, website)));
    }
}   
```

* Modernize `doFillServerUrlItems(context, serverUrl)` jenkins permission checks. Need to be checked elsewhere as well.

* `setTraits(List)` clashing with the parent class. Can you override a `@DataboundSetter` method? Fixed it by using `GitHub BS Impl`.

```java
@DataBoundSetter
public void setTraits(@CheckForNull SCMTrait[] traits) {
    this.traits = new ArrayList<>();
    if (traits != null) {
        for (SCMTrait trait : traits) {
            this.traits.add(trait);
        }
    }
}
@Override
public void setTraits(@CheckForNull List<SCMTrait<? extends SCMTrait<?>>> traits) {
    this.traits = traits != null ? new ArrayList<>(traits) : new ArrayList<SCMTrait<? extends SCMTrait<?>>>();

}
```

### GitLabSCMSourceBuilder

Required for building `GitLabSCMSource` for `GitLabSCMSourceNavigator`.

### helpers - package

Contains classes that do not contribute directly to the plugin.

### GitLabOwner enum

Contains types of GitLab owner i.e. `USER` or `GROUP`.

* Added `fetchOwner(gitLabApi, projectOwner)` as a static method.

### GitLabAvatar

Creating avatar of GitLab in Jenkins.

### GitLabAvatarCache

Builds Avatar url and creates a cache of the entries.

### BranchDiscoveryTrait 

SCM API trait to define branch properites.

### OriginMergeRequestDiscoveryTrait 

SCM API trait to define Merge Request (from Origin Project) properites.

### ForkMergeRequestDiscoveryTrait 

SCM API trait to define Merge Request (from Forked Project) properites.

### SSHCheckoutTrait

SCM API trait to allow checking out repository over SSH.

### GitLabNotifier

Notifies GitLab about the status of the build.

### GitLabMergeRequestSCMEvent

Defines an event when a Merge Request is created in GitLab.

### GitLabPushSCMEvent

Defines an event when a push is made in GitLab.

### AbstractGitLabSCMHeadEvent

Defines commmon events methods for the above 2 events.

## Present State

The present state of the GitLab branch source is very unstable. Although the basic functions such as indexing, building is working. But secondary functions like avatar is broken. Folder organisation is also broken. One of the important thing that was missing in GitLab Plugin was that it was unable to detect merge requests, now we are able to detect and build merge requests from GitLab.

![mr-detection](/assets/2019-06-22-gsoc-coding-period-the-fourth-week/mr-detection.png)