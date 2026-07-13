---
title: "Learning Open Policy Agent with Tanzu Mission Control Part-1"
date: 2020-09-24T21:30:12+09:00
categories: [Tanzu Mission Control]
tags: ["Tanzu Mission Control", "Open Policy Agent"]
thumbnail: "2020-09-24T13-09-22.png"
featured: true
---

This article is part of the Learning Open Policy Agent with Tanzu Mission Control series.<!--more-->

# Series

Part 1 : Overview **< you are here**  
Part 2 : [Playing with TMC's Open Policy Agent](../tmc-demanabu-opa-02)  
Part 3 : [Dissecting TMC's Open Policy Agent](../tmc-demanabu-opa-03)  
Part 4 : [Writing your own OPA policies from TMC](../tmc-demanabu-opa-04)  

# What is Tanzu Mission Control?

Tanzu Mission Control is a system released by VMware for managing multiple Kubernetes clusters.
Abbreviated "TMC".

With Tanzu Mission Control you can manage all kinds of Kubernetes centrally.
Being able to manage EKS, AKS and GKE is interesting too.
Among its features, the interesting one here is that it implements Open Policy Agent.

# What is Open Policy Agent?

## First, watch this

Before explaining Open Policy Agent, watch this Gif:


![render1596759963723.gif](render1596759963723.gif)

It shows someone logging into a pod with kubectl and doing various things.
Now, do you notice? From line 3 onward: `root [ / ]#`...

They have **escalated to the host OS and grabbed a root shell.**

## Why could the host OS root shell be taken?

Those in the know already know, and it's no big trick — two things were rigged on this Pod:

* The Pod is granted Privileged. This allows every operation against the Host OS.
* `HostPID: True` is specified, exposing the host OS process table.

As YAML, it looks like this:

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: verybad
spec:
  hostPID: true　# !!note
  containers:
    - name: verybad
      image: alpine
      command: [ "sleep", "3600" ]
      securityContext:
          privileged: true # !!note
```
On top of this, the nsenter command below with these flags attaches a `BASH` prompt to the host's process ID 1:

```
/usr/bin/nsenter -t 1 -m -u -n -i -- bash
```

## But wait, PSP protects against that, right?

Exactly — Pod Security Policy protects against it. With proper permissions configured, OS privilege escalation attempts get scolded like this:

```
mhoshino@mhoshino ~ % kubectl exec -it verybad sh
/ # /usr/bin/nsenter -t 1 -m -u -n -i -- bash
nsenter: can't open '/proc/1/ns/ipc': Permission denied
/ #
```
However — not widely known yet — **PSP is scheduled to be deprecated in the near future.**

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Big news <a href="https://twitter.com/kubernetesio?ref_src=twsrc%5Etfw">@kubernetesio</a> pod security policy is likely to be deprecated in 2020 with the entire project likely moving to <a href="https://twitter.com/OpenPolicyAgent?ref_src=twsrc%5Etfw">@OpenPolicyAgent</a> Gatekeeper <a href="https://twitter.com/hashtag/KubeCon?src=hash&amp;ref_src=twsrc%5Etfw">#KubeCon</a> SIG Auth <a href="https://t.co/vmkJp52A9z">pic.twitter.com/vmkJp52A9z</a></p>&mdash; Sean Kerner (@TechJournalist) <a href="https://twitter.com/TechJournalist/status/1197658440040165377?ref_src=twsrc%5Etfw">November 21, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Supplementing this tweet, the plan for PSP to disappear in Kubernetes v1.22 was also declared on Github:

https://github.com/kubernetes/enhancements/issues/5#issuecomment-656120326
> The deprecation schedule for the current beta version in 1.22 is independent of whether or not an in-tree implementation of the standard pod security profiles will be provided. That has not yet been determined.

## Enter Open Policy Agent as the PSP replacement

Along with the shocking news of PSP going away, the quiet buzz is about its replacement.
That's where Open Policy Agent comes in.
![](2020-09-24T13-25-05.png)


Open Policy Agent is a CNCF project — a tool that lets you define policies more simply.
Moreover, combined with Kubernetes [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/), it can intercept resources before they are created and reject them.

Pictured, it looks like this:

![](2020-09-24T13-25-22.png)


# So what is this series?

Open Policy Agent hasn't been around long, yet Tanzu Mission Control has made it usable as a product. And it will come to be used as the PSP replacement.

This series dissects how TMC's Open Policy Agent is implemented.
Next up: "[Playing with TMC's Open Policy Agent](../tmc-demanabu-opa-02)".
