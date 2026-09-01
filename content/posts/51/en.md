---
title: "Spring Enterprise: A Guide to Application Advisor (Upgrade Plan)"
date: "2026-09-01T19:00:00+09:00"
tags: ["Application Advisor", "Spring"]
thumbnail: img.png
---
This post introduces Application Advisor, which comes with Spring Enterprise, as a supplement to the official manual.
Following on from `patch apply` in [the previous post](../50), this entry focuses on the `upgrade-plan apply` feature.
<!--more-->

## Introduction

As I wrote last time, Application Advisor provides two major patching features.

- `advisor patch apply`:
used for hotfixes and patch versions only
- `advisor upgrade-plan apply`:
used for minor and major version upgrades

This time it's the latter. The manual is [here (v1.6)](https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/application-advisor/1-6.html).

![img.png](img.png)

Where `patch apply` only looks at patch versions such as "3.1.4 → 3.1.5",
`upgrade-plan apply` takes care of upgrades that cross generations, such as "Spring Boot 2.7 → 3.5".

Note that the patches and minor upgrades I refer to here assume [Semantic Versioning](https://semver.org/) for each Java dependency.
In other words, think of it in these terms:

- 3.1.4 → 3.1.5 : applying a patch
- 3.1.4 → 3.2.2 : a minor upgrade
- 3.1.4 → 4.0.3 : a major upgrade

Patching is built on the assumption that user code is unaffected.
That is why `patch apply` updates dependencies without touching user code.
Major and minor upgrades, on the other hand, can affect user code, so they need a different design from `patch apply`.

## What are recipes and mapping coordinates?

The `upgrade-plan apply` command brings in concepts that `patch apply` does not have:
`Recipes` and `Mapping Coordinates`.

They cover the following:

- `Recipe`: code written on top of the OpenRewrite framework to fix incompatible code that comes up across major and minor upgrades
- `Mapping Coordinates`: acts as a correspondence table of versions for each dependency, and is used to work out the number of `step`s

The Application Advisor CLI ships with recipes and mapping coordinates.
The manual describes them in detail here.

- [The recipe index for Application Advisor 1.6](https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/application-advisor/1-6/app-advisor/recipes-index-index-boot-recipes.html)
- [Mapping coordinates for Application Advisor v1.6.7](https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/application-advisor/1-6/app-advisor/coverage-1-6-7.html)

For a simple upgrade, it looks like this.

![img_1.png](img_1.png)

Application Advisor uses the mapping coordinates to calculate units of execution called Steps.
For each of these units, `upgrade-plan` runs the recipes and completes successfully. In the diagram above, that is two Steps.
The manual [walks through this using Spring Petclinic as an example](https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/application-advisor/1-6/app-advisor/how-to-guides-upgrade-boot.html).

Application Advisor lets you customize these recipes and mapping coordinates.
Doing so makes upgrades like the following possible.

![img_2.png](img_2.png)

The example in this diagram assumes a custom framework that bundles Spring.
Based on the customized mapping coordinates, Advisor learns which Spring Boot version goes with each version of that custom framework.
With that, the steps required to reach the upgrade are recalculated. In this diagram, that adds one Step.
You can also run your own recipes at each of those steps.

This makes it possible to upgrade not only plain Spring Boot but custom packages as well.

I have written this on the assumption of Spring Boot so far, but it can also be used with Spring Framework and other Java frameworks.

# Attempting an automated TERASOLUNA upgrade

To put the approach above into practice, let's see whether we can upgrade [TERASOLUNA](https://www.nttdata.com/jp/ja/lineup/terasoluna/), a Japanese-made framework.

## How TERASOLUNA versions map to Spring Framework

TERASOLUNA and Spring Framework line up as follows (based on my own research).

| TERASOLUNA | spring-framework | spring-data-commons | spring-security |
| --- | --- | --- | --- |
| `5.7.x` | 5.3.39 | 2.7.18 | 5.7.13 |
| `5.8.x` | 6.0.3 | 3.0.0 | 6.0.1 |
| `5.9.x` | 6.1.3 | 3.2.2 | 6.2.1 |
| `5.10.x` | 6.2.15 | 3.5.7 | 6.5.7 |
| `5.11.x` | 7.0.3 | 4.0.2 | 7.0.2 |

As you can see, 5.7 → 5.8 comes with a major change, from the Spring Framework 5 line to the 6 line.
Let's see whether Application Advisor can carry out that upgrade.

## Sample code

I put it here.

https://github.com/mhoshi-vm/simple-terasoluna

## Running upgrade-plan without customization

First, let's run `upgrade-plan` without any customization.

### 1. Downloading the CLI

Same as [last time](../50). Follow the steps in the manual, or if you have the Spring Enterprise Repository configured you can also fetch it with the following.
(This downloads 1.6.7 for Mac ARM64.)

```
mvn -U dependency:get -Dartifact=com.vmware.tanzu.spring:application-advisor-cli-macos-arm64:1.6.7:tar -Dtransitive=false
```

### 2. Checking the plan

Here I run `upgrade-plan`.

```
advisor upgrade-plan get
```

With 1.6.7, the latest at the time of writing, the output was as follows.

```bash
Build configuration file does not exist! Generating it...
🏃 [ 1 / 6 ] Resolving build tool version [00m 01s] ok
🏃 [ 2 / 6 ] Resolving dependencies with mvnw [00m 03s] ok
🏃 [ 3 / 6 ] Resolving JDK version [00m 02s] ok
🏃 [ 4 / 6 ] Resolving Git repository [00m 01s] ok
🏃 [ 5 / 6 ] Resolving application modules [00m 01s] ok
🏃 [ 6 / 6 ] Obtaining dependency management artifacts [00m 02s] ok

🚀 The build-configuration has been generated in /Users/mh013301/terasoluna-gfw/target/.advisor/build-config.json

🏃 Fetching and processing upgrade plan details [00m 01s] ok

The projects ["apache-commons-collections", "apache-commons-lang", "spring-framework", "spring-data-commons"] could not be included in the Upgrade Plan because they are used as transitive dependencies for other projects, and no upgrades are configured for them.
Please request your administrator to configure the projects of the following dependencies:

        - commons-beanutils:commons-beanutils
                uses:
                        - apache-commons-collections
        - com.github.dozermapper:dozer-core
                uses:
                        - apache-commons-lang
                        - apache-commons-collections
        - org.terasoluna.gfw:terasoluna-gfw-common
                uses:
                        - spring-framework
                        - spring-data-commons
                blocking upgrades for:
                        - spring-data-commons
        - org.terasoluna.gfw:terasoluna-gfw-web
                uses:
                        - spring-framework
                        - spring-data-commons
                blocking upgrades for:
                        - spring-data-commons

In order to learn more about publishing upgrade mappings, visit https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/tanzu-spring/commercial/spring-tanzu/app-advisor-custom-upgrades.html

No upgrade plans available - your project seems to be up to date.

```

Mapped onto this context, the `could not be included in the Upgrade Plan because they are used as transitive dependencies for other projects, and no upgrades are configured for them.` line in the output means the following.

- ["apache-commons-collections", "apache-commons-lang", "spring-framework", "spring-data-commons"] are transitive dependencies of TERASOLUNA (the "for other projects" part)
- The relationship between those transitive dependencies and TERASOLUNA is unknown
- Therefore the upgrade cannot be performed

In other words, it errored out and nothing could be upgraded.
Since no mapping has been supplied, this is as expected so far.

## Running upgrade-plan with a custom mapping

Create the following custom mapping in a directory of your choice.
The file name is up to you as well; I will use `terasoluna.json`.

```json
{
  "slug": "terasoluna-gfw",
  "coordinates": [
    "org.terasoluna.gfw:terasoluna-gfw-common",
    "org.terasoluna.gfw:terasoluna-gfw-web"
  ],
  "repositoryUrl": "https://github.com/terasolunaorg/terasoluna-gfw",
  "rewrite": {
    "5.7.x": {
      "recipes": [],
      "nextRewrite": {
        "version": "5.8.x"
      },
      "requirements": {
        "supportedGenerations": {
          "spring-data-commons": "2.7.x",
          "spring-framework": "5.3.x",
          "spring-security": "5.7.x"
        },
        "excludedArtifacts": []
      }
    },
    "5.8.x": {
      "recipes": [],
      "nextRewrite": {
      },
      "requirements": {
        "supportedGenerations": {
          "spring-data-commons": "3.0.x",
          "spring-framework": "6.0.x",
          "spring-security": "6.0.x"
        },
        "excludedArtifacts": []
      }
    }
  }
}
```

Then export the following environment variable.

```bash
export SPRING_ADVISOR_MAPPING_CUSTOM_0_FILEPATH=<directory>/terasoluna.json
```

Running Advisor CLI v1.6.7 in this state gives the following.

```bash
mh013301@PJQ72XCV5C terasoluna-gfw % advisor upgrade-plan get
     
🚀 Existing build-configuration is already up-to-date


🏃 [ 1 / 2 ] Validating syntax of upgrade mappings [00m 01s] ok
🏃 [ 2 / 2 ] Fetching and processing upgrade plan details [00m 01s] ok

Projects discovered:
        - terasoluna-gfw: 5.7.x → 5.8.x
        - spring-data-commons: 2.7.x → 4.1.x
        - spring-framework: 5.3.x → 7.0.x

The projects ["apache-commons-collections", "apache-commons-lang"] could not be included in the Upgrade Plan because they are used as transitive dependencies for other projects, and no upgrades are configured for them.
Please request your administrator to configure the projects of the following dependencies:

        - commons-beanutils:commons-beanutils
                uses:
                        - apache-commons-collections
        - com.github.dozermapper:dozer-core
                uses:
                        - apache-commons-lang
                        - apache-commons-collections

In order to learn more about publishing upgrade mappings, visit https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/tanzu-spring/commercial/spring-tanzu/app-advisor-custom-upgrades.html


Upgrade Plan for your Dependencies:
        - Step 1:
                * Upgrade terasoluna-gfw from 5.7.x to 5.8.x
                * Upgrade spring-data-commons from 2.7.x to 3.5.x
                * Upgrade spring-framework from 5.3.x to 6.2.x

Some upgrades were not included in the upgrade plan.
Please, upgrade and release if needed the following projects:
        * terasoluna-gfw:5.8.x
                Last version of spring-framework is 6.0.x
        * terasoluna-gfw:5.8.x
                Last version of spring-data-commons is 3.0.x
```

Errors are still showing as before, but here is the key part.

```
Upgrade Plan for your Dependencies:
        - Step 1:
                * Upgrade terasoluna-gfw from 5.7.x to 5.8.x
                * Upgrade spring-data-commons from 2.7.x to 3.5.x
                * Upgrade spring-framework from 5.3.x to 6.2.x
```

As this shows, the mapping coordinates file has made it possible to define the first Step.
From here, run `apply` to make the actual changes.

```bash
advisor upgrade-plan apply
```

Running it with Advisor CLI 1.6.7 produced the output below.
All sorts of errors show up, but in the end it does perform the version upgrade.

```bash
     
🚀 Existing build-configuration is already up-to-date


🏃 [ 1 / 2 ] Validating syntax of upgrade mappings [00m 01s] ok
🏃 [ 2 / 2 ] Fetching and processing upgrade plan details [00m 01s] ok

Projects discovered:
        - terasoluna-gfw: 5.7.x → 5.8.x
        - spring-data-commons: 2.7.x → 4.1.x
        - spring-framework: 5.3.x → 7.0.x

The projects ["apache-commons-collections", "apache-commons-lang"] could not be included in the Upgrade Plan because they are used as transitive dependencies for other projects, and no upgrades are configured for them.
Please request your administrator to configure the projects of the following dependencies:

        - commons-beanutils:commons-beanutils
                uses:
                        - apache-commons-collections
        - com.github.dozermapper:dozer-core
                uses:
                        - apache-commons-lang
                        - apache-commons-collections

In order to learn more about publishing upgrade mappings, visit https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/tanzu-spring/commercial/spring-tanzu/app-advisor-custom-upgrades.html


Upgrade Plan for your Dependencies:
        - Step 1:
                * Upgrade terasoluna-gfw from 5.7.x to 5.8.x
                * Upgrade spring-data-commons from 2.7.x to 3.5.x
                * Upgrade spring-framework from 5.3.x to 6.2.x

Some upgrades were not included in the upgrade plan.
Please, upgrade and release if needed the following projects:
        * terasoluna-gfw:5.8.x
                Last version of spring-framework is 6.0.x
        * terasoluna-gfw:5.8.x
                Last version of spring-data-commons is 3.0.x
mh013301@PJQ72XCV5C terasoluna-gfw % 
mh013301@PJQ72XCV5C terasoluna-gfw % 
mh013301@PJQ72XCV5C terasoluna-gfw % 
mh013301@PJQ72XCV5C terasoluna-gfw % advisor upgrade-plan apply
     
🚀 Existing build-configuration is already up-to-date


🏃 [ 1 / 4 ] Validating syntax of upgrade mappings [00m 01s] ok
🏃 [ 2 / 4 ] Validating the license of rewrite artifacts [00m 13s] ok
🏃 [ 3 / 4 ] Fetching and processing upgrade plan details [00m 01s] ok

The projects ["apache-commons-collections", "apache-commons-lang"] could not be included in the Upgrade Plan because they are used as transitive dependencies for other projects, and no upgrades are configured for them.
Please request your administrator to configure the projects of the following dependencies:

        - commons-beanutils:commons-beanutils
                uses:
                        - apache-commons-collections
        - com.github.dozermapper:dozer-core
                uses:
                        - apache-commons-lang
                        - apache-commons-collections

In order to learn more about publishing upgrade mappings, visit https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/tanzu-spring/commercial/spring-tanzu/app-advisor-custom-upgrades.html


Projects to upgrade:
        * terasoluna-gfw from 5.7.x to 5.8.x
        * spring-data-commons from 2.7.x to 3.5.x
        * spring-framework from 5.3.x to 6.2.x

🔨 [ 4 / 4 ] Upgrading sources... [00m 14s] ok

👍 Successfully applied upgrade.


⚠️  Warnings:

* Application Advisor might produce a partial upgrade and will incrementally cover all the required changes to upgrade all the Spring projects. If you have questions or are experimenting issues upgrading your applications, please request our help or support in https://support.broadcom.com
```

Looking at the actual changes with `git diff`, here is what we see.

First, it bumps the TERASOLUNA version.

```
diff --git a/pom.xml b/pom.xml
index 801bfd3..6fe5e3e 100644
--- a/pom.xml
+++ b/pom.xml
@@ -15,7 +15,7 @@
   <properties>
     <maven.compiler.release>17</maven.compiler.release>
     <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
-    <terasoluna.version>5.7.4.RELEASE</terasoluna.version>
+    <terasoluna.version>5.8.1.RELEASE</terasoluna.version>
   </properties>
    
   <dependencies>
```

On top of that, because the Spring Framework 6 line dropped Java 8 support, it also carries out the javax → jakarta namespace change and brings the version up to date.

```bash

@@ -38,9 +38,9 @@
     </dependency>
 
     <dependency>
-      <groupId>javax.servlet</groupId>
-      <artifactId>javax.servlet-api</artifactId>
-      <version>4.0.1</version>
+      <groupId>jakarta.servlet</groupId>
+      <artifactId>jakarta.servlet-api</artifactId>
+      <version>6.1.0</version>
       <scope>provided</scope>
     </dependency>
   </dependencies>
```

Furthermore, since Spring Framework 6 no longer matches a trailing "/" in URLs by default, it helps out with that too.
(This one comes from a recipe bundled with the Advisor CLI.)

```bash
diff --git a/src/main/java/com/example/demo/TodoController.java b/src/main/java/com/example/demo/TodoController.java
index 1813c87..d5ae2f5 100644
--- a/src/main/java/com/example/demo/TodoController.java
+++ b/src/main/java/com/example/demo/TodoController.java
@@ -18,7 +18,7 @@ public class TodoController {
         this.beanMapper = beanMapper;
     }
 
-    @PostMapping("/todos")
+    @PostMapping({"/todos", "/todos/"})
     public Todo create(@RequestBody TodoForm form) {
         return beanMapper.map(form, Todo.class);
     }
```

It also creates files called `sql-error-codes.xml` and `src/main/resources/META-INF/spring.factories`.
These are generated to preserve Spring Framework 5 behavior. That is not the point of this article, so do try it yourself.

## Running upgrade-plan with a custom recipe

Now let's go further and run it with a custom recipe added.
Here I will implement [2.14. [Step 14] Migrating from Dozer to MapStruct](https://github.com/terasolunaorg/terasoluna-gfw/wiki/Migration-Guide-5.8.1_ja#step-14-dozer-%E3%81%8B%E3%82%89-mapstruct-%E3%81%B8%E3%81%AE%E7%A7%BB%E8%A1%8C) from the TERASOLUNA guide.

First, reset the earlier changes for now.

```bash
git stash
```

### Registering a recipe in the mapping file

Register the recipes in the mapping file as follows.

```json
{
  "slug": "terasoluna-gfw",
  "coordinates": [
    "org.terasoluna.gfw:terasoluna-gfw-common",
    "org.terasoluna.gfw:terasoluna-gfw-web"
  ],
  "repositoryUrl": "https://github.com/terasolunaorg/terasoluna-gfw",
  "rewrite": {
    "5.7.x": {
      "recipes": [
        {
          "name": "org.openrewrite.java.dependencies.UpgradeDependencyVersion",
          "params": {
            "groupId": "org.terasoluna.gfw",
            "artifactId": "*",
            "newVersion": "5.8.x"
          }
        },
        {
          "name": "org.openrewrite.java.dependencies.ChangeDependency",
          "params": {
            "oldGroupId": "com.github.dozermapper",
            "oldArtifactId": "dozer-core",
            "newGroupId": "org.mapstruct",
            "newArtifactId": "mapstruct",
            "newVersion": "1.5.3.Final"
          }
        }
      ],
      "nextRewrite": {
        "version": "5.8.x"
      },
      "requirements": {
        "supportedGenerations": {
          "spring-data-commons": "2.7.x",
          "spring-framework": "5.3.x",
          "spring-security": "5.7.x"
        },
        "excludedArtifacts": []
      }
    },
    "5.8.x": {
      "recipes": [],
      "nextRewrite": {
      },
      "requirements": {
        "supportedGenerations": {
          "spring-data-commons": "3.0.x",
          "spring-framework": "6.0.x",
          "spring-security": "6.0.x"
        },
        "excludedArtifacts": []
      }
    }
  }
}
```

The important point here is that I write `UpgradeDependencyVersion` as well as `ChangeDependency`.

**Once `recipes` is no longer empty, it "replaces" the version updates that Advisor performs automatically.**
Trying three variants on the same application gave this.

| `recipes` for `5.7.x` | `terasoluna.version` | Dozer |
| --- | --- | --- |
| `[]` | `5.7.4` → `5.8.1.RELEASE` | unchanged |
| `ChangeDependency` only | **stays at `5.7.4`** | → `mapstruct` |
| `UpgradeDependencyVersion` + `ChangeDependency` | `5.7.4` → `5.8.1.RELEASE` | → `mapstruct` |

Run it again.

```
export SPRING_ADVISOR_MAPPING_CUSTOM_0_FILEPATH=.advisor/mappings/terasoluna.json
advisor upgrade-plan apply  
```

I will skip the console output and look at `git diff` instead.
On top of the earlier result, the following has been applied, which shows that the recipe ran.

```bash
     <!-- The bean mapper TERASOLUNA 5.8.1 replaces with MapStruct. -->
     <dependency>
-      <groupId>com.github.dozermapper</groupId>
-      <artifactId>dozer-core</artifactId>
-      <version>6.5.2</version>
+      <groupId>org.mapstruct</groupId>
+      <artifactId>mapstruct</artifactId>
+      <version>1.5.3.Final</version>
     </dependency>
```

I would like to say the upgrade succeeded here, but at this stage the change is only half done, so the actual code errors out as shown below.

![img_3.png](img_3.png)

Going any further would stray from the point, but this case needs a somewhat more elaborate recipe.
Something along the following lines would be required.

| What you want to do | Recipe |
| --- | --- |
| Swap the field and constructor types | `org.openrewrite.java.ChangeType` |
| `map(form, Todo.class)` → `map(form)` | `org.openrewrite.text.FindAndReplace` (regex) |
| Adding the `@Mapper` interface | Append to the end of the existing file with `FindAndReplace` |
| Configuring `annotationProcessorPaths` | Inject into `pom.xml` with `FindAndReplace` |

I will not go any deeper into recipes in this blog, but either way, we managed to upgrade TERASOLUNA, which the Advisor CLI knows nothing about.

## What does it take to use upgrade-plan in production?

Here are some other tips for taking this into production.

### Ugh... writing recipes and mapping coordinates is a hassle...

Broadcom strives, as much as possible, to make recipes and mappings work by default so that users do not have to write them.
So my first recommendation is to raise a support ticket with Broadcom.
As long as the request is reasonable, bundling the recipes and mappings into a later release is possible.

Custom recipes and mappings are really only for private dependencies that the world does not know about, or as a temporary workaround.

### Writing mapping coordinates is a hassle...

You can generate the mapping coordinates file automatically with the following command.

```
advisor mapping create -c='org.terasoluna.gfw:terasoluna-gfw-common' < /dev/null
```

What you pass to `-c` is a Maven coordinate (`groupId:artifactId`).

Note that this command resolves the Maven repository from where you run it and generates the coordinates file.
Here are a few things worth knowing before you run it.

- It tries to build coordinates for every version, so a single project can take close to 10 minutes
- The recipes between versions are provided empty
- **It waits on standard input**, so in a non-interactive setting (CI or a background run) it hangs silently unless you add `< /dev/null`. There is no output either, so you cannot tell whether it has hung or is simply taking a while
- Pass a single coordinate and it picks up the sibling artifacts of that project together. One run per project is enough, rather than one run per unattributed artifact

When you want to find out up front which projects are not covered, `advisor mapping search --prefix <prefix>` is quicker.

### Writing recipes is a hassle...

These days I would recommend having AI/LLMs write the recipes.
There is plenty of OpenRewrite knowledge on the internet, so you can get reasonable accuracy.

### Does it work in an air-gapped environment?

The Advisor CLI supports air-gapped environments.

### How often is the Advisor CLI updated?

`upgrade-plan` is always shipped as a set with the latest recipes.
Put another way, if you do not upgrade, the recipes for the newest patches are not included (for example, 1.6.5 does not include the recipe for moving to Spring Boot 4.1), which means you cannot upgrade from certain versions.
So when operating `upgrade-plan`, updating frequently is recommended.
Bugs in the Advisor CLI itself may also have been fixed, so it is best to try the latest version whenever you can.

## Wrap-up

I wrote about Application Advisor's `upgrade-plan`.
