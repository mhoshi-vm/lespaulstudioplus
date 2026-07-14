---
title: "Concourse and Github Compatibility Notes"
date: 2021-03-03T21:30:12+09:00
tags: ["Concourse", "Github"]
thumbnail: "2021-04-06T08-15-02.png"
---
A summary of Concourse and Github<!--more-->

## Introduction

This is a summary of how well Github works with Concourse when using Concourse as your main CI tool.
I'll keep updating it bit by bit as I verify things.

## Trigger

Monitoring a Github repository and triggering on commits is easy.
By adding `trigger: true` inside Jobs, Concourse runs the task the moment it detects an update.

```
resources:
- name: upstream-source-code
  type: git
  source:
    uri: https://github.com/<project>/<repo>
    branch: <branch>
jobs:
- name: Code updated
  plan:
    - get: upstream-source-code
      trigger: true
    - task: something
      // Do Something
```



## Push

This is also easy.
The point is to put values into `outputs`, which ultimately results in a push to the repository. See:

https://github.com/concourse/git-resource#out-push-to-a-repository

The private_key used for pushing to Github and the like should probably be turned into variables backed by an external credential manager:

https://concourse-ci.org/creds.html



```
resources:
- name: source-code
  type: git
  source:
    uri: https://github.com/<project>/<repo>
    branch: <branch>
    private_key: ((sourceKey))
    private_key_user: ((sourceUser))
jobs:
- name: merge-upstream
  plan:
    - get: source-code
    - task: commit-push
      config:
        platform: linux
        image_resource:
          type: docker-image
          source:
            repository: alpine/git
            tag: latest
        inputs:
          - name: source-code
        outputs:
        - name: source-code
        run:
          path: /bin/bash
          args:
            - -c
            - |
              set -x
              cd source-code
              git config --global user.name "concourse"
              git config --global user.email "concourse@local"
              // Modify Code
              git add .
              git commit -m "Commit and Push"

```

## Pull Request

Not an official one, but the following resource seems to be the equivalent:

https://github.com/telia-oss/github-pr-resource

## Status Badge

It exists.  
Often included in Readmes, this feature displays the latest status of a specific job.

For example, the `build/passing` badge included in the Readme of [Spring Petclinic](https://github.com/spring-projects/spring-petclinic):

![](2021-04-06T08-19-46.png)

I assumed this feature surely wouldn't exist, but an API is properly provided.
As introduced in the API documentation below, you can pick up not only the state of the whole pipeline but also the status of each Job:

https://concourse-ci.org/observation.html#badges
