---
title: "Adding the Paketo Python Buildpack to Tanzu Application Service (beta)"
date: 2020-10-14T21:30:12+09:00
tags: ["Tanzu Application Service"]
thumbnail: "2020-10-15T02-49-32.png"
---

## Correction

A more correct procedure was covered in [this post](../12). The procedure introduced here fails to persist the buildpack, so it is incomplete. There are still useful points in it, however, so I am keeping it for reference.

## Introduction

Tanzu Application Service for Kubernetes (AKA TAS4K8S) is the commercial version of [Cloud Foundry for Kubernetes](https://github.com/cloudfoundry/cf-for-k8s). At the time of writing this product is in Beta.
For building ) to Kubernetes.

At the time of writing, buildpacks for the following languages are supported:

* java
* nodejs
* go
* .Net core
* php
* nginx
* httpd
* procfile

And the following buildpacks are still community-grade and deliberately excluded from the installation:

* Ruby
* python

They are expected to be supported at or after GA, but can we try them now anyway?... In this post I actually enabled the Python buildpack on TAS4K8S.

## Caution

The method below is not supported by VMware.
We use the following buildpack maintained by [Paketo](https://paketo.io/):

https://github.com/paketo-community/python

As noted in the Paketo Slack channel, this Python buildpack is considered alpha and major changes are expected in upcoming releases.
Excerpt from the Slack comment:

https://paketobuildpacks.slack.com/archives/CULAS8ACD/p1600791481010300?thread_ts=1600789945.010200&cid=CULAS8ACD
> The Python buildpack is not anywhere near the same level of support as Ruby. Ruby is getting promoted into the official buildpacks org while Python is slated to be rewritten from scratch in the next few months. It is essentially alpha quality software at this point.

## Environment

TAS4K8S Beta 0.4.0

The latest publicly available Beta version is 0.3.0, but the steps should be nearly identical.

## Preparation

First, download the kp CLI for operating kpack.
Download the kp-* file matching your workstation OS from Tanzu Network:

https://network.pivotal.io/products/build-service/#/releases/667179

Note: at least at the time of writing, use kp version 0.2.0. Version 1.0.0 introduced major changes that TAS4K8S has not caught up with yet.
After installing the CLI, load your kubeconfig and confirm the output looks right:

```
% export KUBECONFIG=~/.tas
% kp store list
NAME                  READY
cf-buildpack-store    True
```

## Steps

Here is the overall flow. kpack has two building blocks:

* **Store** (renamed **ClusterStore** in the latest version): registers information about buildpacks
* **CustomBuilder**: defines which buildpacks are used to build images from source code

TAS4K8S hardcodes and uses `cf-buildpack-store` as the Store and `cf-default-builder` as the CustomBuilder.

So there are two main steps:
1. Register the Python buildpack in `cf-buildpack-store`
1. Redefine `cf-default-builder` to use the Python buildpack

Each is described below.

#### 1. Register the Python buildpack

First, patch `cf-buildpack-store` as shown below. Without this, later steps are known to fail. Set `REPO/PROJECT/IMAGE` to values matching your private registry such as Harbor.

```
kubectl patch stores cf-buildpack-store --patch '{"metadata": { "annotations" : { "buildservice.pivotal.io/defaultRepository" : "REPO/PROJECT/IMAGE" }}}' --type=merge
```

If it succeeds, you'll see:

```
store.experimental.kpack.pivotal.io/cf-buildpack-store patched
```

Next, add the Python buildpack to the Store.

```
kp store add cf-buildpack-store gcr.io/paketo-community/python
```

Success looks like `paketo-community/python` appearing in this command's output:

```
% kp store status cf-buildpack-store
BUILDPACKAGE ID                  VERSION
paketo-buildpacks/php            0.0.8
paketo-buildpacks/procfile       1.4.0
paketo-community/python          0.0.3
tanzu-buildpacks/go              1.0.3
tanzu-buildpacks/nodejs          1.0.4
tanzu-buildpacks/java            2.5.0
paketo-buildpacks/dotnet-core    0.0.8
```
#### 2. Redefine the builder to use the Python buildpack

Use a file like the one below. As the comment line `From below here...` says, buildpack names can differ per version, so cross-check with the output of `kp store status cf-buildpack-store` and adjust.

```
cat ~/packeto-python-order.yaml
---
- group:
  - id: paketo-community/python
## From below here should match the output of `kp store status cf-buildpack-store`
- group:
  - id: tanzu-buildpacks/java
- group:
  - id: tanzu-buildpacks/nodejs
- group:
  - id: tanzu-buildpacks/go
- group:
  - id: paketo-buildpacks/dotnet-core
- group:
  - id: paketo-buildpacks/php
- group:
  - id: paketo-buildpacks/httpd
- group:
  - id: paketo-buildpacks/nginx
- group:
  - id: paketo-buildpacks/procfile
```

Once the file is created, run:

```
kp cb patch -n cf-workloads-staging cf-default-builder -o ~/packeto-python-order.yaml
```

Success looks like Python being added to the list in the verification command:


```
% kp cb status -n cf-workloads-staging cf-default-builder
...
DETECTION ORDER
Group #1
  paketo-community/python
Group #2
  tanzu-buildpacks/java
Group #3
  tanzu-buildpacks/nodejs
Group #4
  tanzu-buildpacks/go
Group #5
  paketo-buildpacks/dotnet-core
Group #6
  paketo-buildpacks/php
Group #7
  paketo-buildpacks/httpd
Group #8
  paketo-buildpacks/nginx
Group #9
  paketo-buildpacks/procfile
```

## Trying it out

Paketo's Hello World test app for the buildpack is published here:

https://github.com/paketo-community/python/tree/v0.0.3/integration/testdata

Download it to your workstation and `cf push`.

```
% cf push pypy
Pushing app pypy to org demo / space demo as admin...
Getting app info...
Creating app with these attributes...
+ name:       pypy
  path:       /Users/mhoshino/python/integration/testdata/pip
  routes:
...
```

If all goes well, the build runs and the app starts.

```
Waiting for app to start...

name:                pypy
requested state:     started
isolation segment:   placeholder
routes:              pypy.apps.SYSTEMDOMAIN
last uploaded:       Thu 15 Oct 10:55:58 JST 2020
stack:
buildpacks:

type:            web
instances:       1/1
memory usage:    1024M
start command:   gunicorn server:app

     state     since                  cpu    memory    disk      details
#0   running   2020-10-15T01:57:01Z   0.0%   0 of 1G   0 of 1G
```

Confirm you can reach it with `curl` as well.

```
% curl  -k https://pypy.apps.SYSTEMDOMAIN/
Hello, World with pip!
```

Nice — success.

## Bonus: vendoring dependencies in advance

When you `cf push`, TAS4K8S automatically resolves your application's dependencies. For Python that means the contents of `requirements.txt`.
In environments without internet access, or when you want faster builds, you may need to download dependencies ahead of time.

You can use the approach supported by the traditional Python buildpack:

https://docs.cloudfoundry.org/buildpacks/python/index.html#vendoring

Create a directory called `/vendor` in your source directory and put the dependencies inside:

```
% mkdir -p vendor
% pip download -r requirements.txt --no-binary=:none: -d vendor
Collecting Flask==0.12.3
  Downloading Flask-0.12.3-py2.py3-none-any.whl (88 kB)
     |████████████████████████████████| 88 kB 299 kB/s
  Saved ./vendor/Flask-0.12.3-py2.py3-none-any.whl
Collecting Jinja2==2.7.2
  Downloading Jinja2-2.7.2.tar.gz (378 kB)
     |████████████████████████████████| 378 kB 854 kB/s
  Saved ./vendor/Jinja2-2.7.2.tar.gz
Collecting MarkupSafe==0.21
  Downloading MarkupSafe-0.21.tar.gz (13 kB)
  Saved ./vendor/MarkupSafe-0.21.tar.gz
Collecting Werkzeug==0.10.4
  Downloading Werkzeug-0.10.4-py2.py3-none-any.whl (293 kB)
     |████████████████████████████████| 293 kB 5.8 MB/s
  Saved ./vendor/Werkzeug-0.10.4-py2.py3-none-any.whl
Collecting gunicorn==19.5.0
  Downloading gunicorn-19.5.0-py2.py3-none-any.whl (113 kB)
     |████████████████████████████████| 113 kB 7.4 MB/s
  Saved ./vendor/gunicorn-19.5.0-py2.py3-none-any.whl
Collecting click>=2.0
  Using cached click-7.1.2-py2.py3-none-any.whl (82 kB)
  Saved ./vendor/click-7.1.2-py2.py3-none-any.whl
Collecting itsdangerous>=0.21
  Using cached itsdangerous-1.1.0-py2.py3-none-any.whl (16 kB)
  Saved ./vendor/itsdangerous-1.1.0-py2.py3-none-any.whl
Successfully downloaded Flask Jinja2 MarkupSafe Werkzeug gunicorn click itsdangerous
```

Then during `cf push`, the contents of `/vendor` take precedence and no dependency downloads happen during the build.

```
% cf push pypy2
...

Staging app and tracing logs...
...
   pip installing from vendor directory
   Looking in links: file:///workspace/vendor
   Processing ./vendor/Flask-0.12.3-py2.py3-none-any.whl
   Processing ./vendor/Jinja2-2.7.2.tar.gz
   Processing ./vendor/MarkupSafe-0.21.tar.gz
   Processing ./vendor/Werkzeug-0.10.4-py2.py3-none-any.whl
   Processing ./vendor/gunicorn-19.5.0-py2.py3-none-any.whl
   Processing ./vendor/click-7.1.2-py2.py3-none-any.whl
   Processing ./vendor/itsdangerous-1.1.0-py2.py3-none-any.whl
```



# Summary

This time we added the Python buildpack to the Beta TAS4K8S, and it turned out to be very simple.
It was also a good reminder of how flexible kpack is.
