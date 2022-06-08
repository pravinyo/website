---
title: Tips to test (end to end) library changes - Gradle based Project
subtitle: Some people are still struggling with this thing (including me)
thumbnail-img: /assets/img/tips-to-test-library-changes/thumb-nail.jpeg
share-img: /assets/img/tips-to-test-library-changes/header.jpeg
img_path: /assets/img/tips-to-test-library-changes/
tags: [gradle, softwareengineering, backenddevelopment, testing]
date: 2022-06-03 14:10:00 +0800
categories: [Blogging, Article]
readtime: true
image:
  path: header.jpeg
  width: 1000   # in pixels
  height: 400   # in pixels
  alt: Image by Michal Jarmoluk from Pixabay
---

## **The Problem**  
I had made multiple changes in 2 dependencies of the spring application. These changes are on the local machine, and I need to test the functionality of the change end to end with the dependent application before pushing the artifact to a remote artifact repository like Artifactory.  

I know this is a simple task, but the reason is that some people are still struggling with this thing
(including me), and now even popular IDE like IntelliJ also support that. But, which approach to choose is still
people get confused.

<p>&nbsp;</p>
## **Setup** 
Here, I am using Kotlin, Gradle, and IntelliJ development IDE. For the sake of simplicity, I am going to
use a simple project, but the challenges are still the same.

![Let’s start][funny-gif-1]
_Let’s start_

I ask about above problem every time to myself, whenever I am working on a project that has some changes in one of the dependencies, and I know there are many ways to handle it. Let's see some of them.

<p>&nbsp;</p>
## Approach 1: _Upload the changes to a remote artifact repository (Artifactory) and use that to test_ {#approach1}

This is straightforward. Here, we simply publish our change to remote Artifactory and update the version of the dependency in `build.gradle` file,

For example,
```diff
-implementation("dev.pravin:projectb:1.0.1")
+implementation("dev.pravin:projectb:1.0.2")
```
  {: .nolineno file="build.gradle" }

**_I do not suggest this approach._** If you are unsure about your change, it is better to avoid this approach. You could follow either of the below approaches to test locally. Later go ahead and follow this approach. Mostly this is taken care of by _CI build pipeline_.

Other things to know:
* Once the build is available in remote Artifactory, we have to update the library version in the `build.gradle` file and use that to test the changes.
* **_It takes a lot of time_** as now we have to update the build version push those changes to the _version control system (GitHub, Bitbucket, etc.)_, create a new build with _CI pipeline_, and later push it to _Artifactory_. Update the `build.gradle` file in the dependent app to test it. As we add changes, we have to follow the same steps again.
* It is easy to accomplish.
* The artifact repository becomes messy as there is no way to tell stable and unstable build.
* It creates more confusion for other folks who will be using it.

![funny gif][funny-gif-0]

<p>&nbsp;</p>
## Approach 2: _Add changes as external jar dependency_ {#approach2}
It is another way to add a dependency to the project, but it requires a few extra steps to configure the dependency. If you update the dependency, you need to remove the older one, then add the latest one.  
These steps keep repeating for each change. 

![External Jar Dependency][approach2gif]
_steps to add external jar_

Other things to know:
* We have to uncomment the current dependency for the `build.gradle` file and _import the jar as an external dependency_.
* **_We must create a jar of the library_**, and later follow steps to manually remove the old and add the updated jar.
* This approach can be frustrating if you have to test changes multiple times for the same library.
* It is easy to recall those steps, and anyone can easily do it.

![funny image][funny-gif-2]
<p>&nbsp;</p>
## Approach 3: _publish changes to the local maven repository(~/.m2) and using that to test changes_ {#approach3}
> Let’s say, Project A **_dependents on_** Project B.  

Changes in **Project B** _(Assuming Project B already has required changes)_:
* Open the `build.gradle` file and add the following changes:  
    **_Increment project version_**  
    We can locally use it to identify artifacts by incrementing the version (like from 1.0.**1** to 1.0.**2**).  
    
    **Add _maven-publish_ plugin**
    ```gradle
    plugins {
        ...
        id "maven-publish"
    }
    ..
    ```
    {: .nolineno file="build.gradle" }

    **Add below config for _publishing_**
    ```gradle
    ...
    publishing {
        publications {
            maven(MavenPublication) {
                from components.java
            }
        }
    }
    ```
    {: .nolineno file="build.gradle" }
    ![Screenshot of build.gradle][projectb-image-1]{: .shadow }
    _Screenshot of build.gradle in project B_

* open the terminal in the **root folder** of the project and run `gradle publishToMavenLocal`.  
    This command, creates a new build of the project with the updated version, and publishes the jar artifact in the _local repository of the maven_.  
    This repository is used by other projects to resolve the dependency.

    ![Screenshot of terminal][projectb-image-2]{: .shadow }
    _Screenshot of terminal in project B_

Changes in **Project A**:
* we have to make small changes in `build.gradle` file.  
    check for `repositories` and add "mavenLocal()" as the first entry.  
    Next, we must update the dependency version to the one we published in the previous step.  
    
    ```diff
     - implementation("dev.pravin:projectb:1.0.1")
     + implementation("dev.pravin:projectb:1.0.2")
    ```
    {: .nolineno file="build.gradle"}
    
    ![Screenshot of build.gradle][projecta-image-1]{: .shadow }
    _Screenshot of build.gradle in project A_

* Reload the `build.gradle` changes and run the project.  
    This time Gradle will check the maven local repository (~/.m2) and use that to resolve dependencies. We should now be able to run the application with the latest changes.

    ![Screenshot of Project A execution output ][projecta-image-2]{: .shadow }
    _Screenshot of project A execution output_

* Here is the complete step animation:
    ![complete steps animation][projectab-gif]
    _complete steps for both project changes_


> It is much simpler and avoids unnecessary artifact upload to a remote artifact repository for each change.  
{: .prompt-tip }

Other things to know:
* It has some code changes like adding the _maven Local plugin_, updating the _version_, and config related to _publishing to maven local_ in `build.gradle` of the application.
* It comparatively **_takes less time_**, and it will be simple as we make changes.  
  We can reuse the same version while publishing to maven local. In the end, add code changes, publish to maven local with the same version, and reload `build.gradle` in the dependent application.
* There are small code changes, and it is easy to follow.

![funny image][funny-gif-3]
<p>&nbsp;</p>
## Approach 4: _Add the library as a Module dependency in the application and test it_ {#approach4}

This pattern is followed by most Gradle projects where we add projects (Project A, Project B) as a **_subproject in the multi-project setup_**. We have to create top-level `settings.gradle` which has details of both subprojects. 

>In our problem, this method doesn’t provide much benefit as it requires more changes in gradle files to test the  code changes. It feels like using a **_bazooka to kill a fly_**.
{: .prompt-warning }

>For step-by-step details, [follow this link][external-link-gradle-multi-project]
{: .prompt-info }

Other things to know:
* We have to make a few changes in the Gradle file (`build.gradle` and `settings.gradle`) to achieve this.
* It takes time to set up for beginners, and even for non-beginner, it still requires following some documentation (**_In short, it requires effort to remember steps_**).
* Understanding how this works takes some time.
* It is done as a one-time setup. Due to more steps involved if wrongly followed, it may cause other problems like dependency not being visible, and there could be more.
* _It is not simple_, and checking changes with more applications **_could be time-consuming_**.

![funny image][funny-gif-4]
<p>&nbsp;</p>
## Conclusion {#conclusion}
After going through all the above approaches, it is clear that Approaches 1 and 4 are not a good choice. **_Both approaches 2 and 3 are better for this problem statement_**.  

If there are _small changes_ that we need to test, we can go with _Approach 2_, and it is ok as that involves _fewer gradle file changes compared to approach 3_.  

If there are _many changes_ that you need to test, it would be _better to go with Approach 3_. Even though approach 3 has few extra changes compared to approach 2, we know that there is a good possibility that we again have to make changes and redo the same process. So, _approach 3 reduces the hassle of manual importing that approach 2 does_.  

For this problem statement, **_approach 3 is the better option as I need to make further changes and update the code_**.

> You have something to share, go ahead and add in **comments** &#128521; >> Happy Learning!!
{: .prompt-tip }


![reading gif][funny-gif-5]

<!-- Reference Link -->
[approach2gif]: external-jar-dependency.gif
[projectab-gif]: maven-local.gif
[projectb-image-1]: projectb-image-1.png
[projectb-image-2]: projectb-image-2.png
[projecta-image-1]: projecta-image-1.png
[projecta-image-2]: projecta-image-2.png

[funny-gif-0]: https://media.giphy.com/media/w89ak63KNl0nJl80ig/giphy.gif
[funny-gif-1]: https://media.giphy.com/media/Pmv6m86yGQCjkLjmqx/giphy.gif
[funny-gif-2]: https://media.giphy.com/media/3oKIPbNb1vWdftiVLq/giphy.gif
[funny-gif-3]: https://media.giphy.com/media/5tf0OxYC9Cv6IQar40/giphy.gif
[funny-gif-4]: https://media.giphy.com/media/LKTTAzGboJGzC/giphy.gif
[funny-gif-5]: https://media.giphy.com/media/l0Exe5NhestV2TfC8/giphy.gif

[external-link-gradle-multi-project]: https://docs.gradle.org/current/userguide/multi_project_builds.html#sec:adding_subprojects