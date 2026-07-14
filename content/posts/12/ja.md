---
title: "Tanzu Application Service(beta)にPacketo Python Buildpackを追加する(改訂版)"
date: 2020-11-24T21:30:12+09:00
tags: ["Tanzu Application Service"]
thumbnail: "2020-10-15T02-49-32.png"
---

[前回](../7)のやり方で、Tanzu Application Service(Beta)にPython Buildpackをインストールする方法を紹介しました。しかし、この方法だと、インストールスクリプト(kapp)を流す度に永続化されず、再実行する必要がある問題がありました。この記事では改訂版でBuildPackのインストールを永続化する方法を紹介します。

## 注意

引き続き正式な手順ではないのでご注意ください。


## 手順

### 1. 使用したいPythonのBuildPackをみつける

以下のURLにログインします。

https://console.cloud.google.com/gcrpython?gcrImageListsize=30

そして、今回使いたいPython Buildpackを選択してURLを保存します。

![](2020-11-19T14-15-12.png)

この例では、version 0.0.3を前提にすすめます。

### 2. Pythonのビルドパックで使用したいバージョンにオーバーライド

以下のようなYamlファイル`../configuration-values/python-buildpack.yaml`として用意します。`image:`以降の値を前手順でコピーしたものに置き換えます。

```yaml
#@ load("@ytt:overlay", "overlay")

#@overlay/match by=overlay.subset({"kind": "Store","metadata": {"name": "cf-buildpack-store"}})
---
spec:
  sources:
  #@overlay/match by=overlay.all
  #@overlay/match by=lambda x, y, z: y["image"].startswith("gcr.io/paketo-community/python"), expects="1+"
  #@overlay/replace
  - image: gcr.io/paketo-community/python@sha256:e6546f3072c49336ce99a2d8297716b748a69da9128c5afb1606c2b73a18a317
```

さらに以下のファイルが存在した場合、`config-optional`に写します。

```
mv config/omit-community-buildpacks.yml config-optional/
```

### 3. インストールを再実行する

以下のInstallを再実行します。

https://docs.pivotal.io/tas-kubernetes/0-3/installing-tas-for-kubernetes.html#install-tas-for-k8s-from-network

## 確認

以下のように`Store`にpythonのビルドパックが表示されれば成功です。

```yaml
$ kubectl get store cf-buildpack-store -o yaml
...
- api: "0.2"
	buildpackage:
		id: paketo-community/python
		version: 0.0.3
	diffId: sha256:10c2a652e1089dfc757f8041099ee287dbeb24645dc6810585e09c76cf7cf3be
	digest: sha256:a99f8b9f7854ebd520ce682aa23733dca6635b848fcaa61779fdfcd4b012ce22
	id: paketo-community/pip
	size: 5023166
	stacks:
	- id: io.buildpacks.stacks.bionic
	- id: org.cloudfoundry.stacks.cflinuxfs3
	storeImage:
		image: gcr.io/paketo-community/python@sha256:e6546f3072c49336ce99a2d8297716b748a69da9128c5afb1606c2b73a18a317
	version: 0.0.139
```

## まとめ

 Pythonのビルドバックの永続化も行えます。
