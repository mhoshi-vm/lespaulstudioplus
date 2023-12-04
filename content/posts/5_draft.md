---
title: "Tanzu Service Mesh(Istio)でBitnamiのWordpressを動かす"
date: 2020-09-09T21:30:12+09:00
categories: ["Tanzu Service Mesh"]
tags: ["Tanzu Service Mesh", "Tanzu Application Service", "Wordpress"]

draft: true
---

# TL;DR

BitnamiのWordpressをIstio環境下で動かす場合の注意点が複数あります。

# はじめに


Bitnami社(現在はVMware社に買収）には、OSSの様々なツールのDockerイメージやHelmチャートを提供しています。

そんな中、今日は以下をWordpressをインストールします。

https://github.com/bitnami/charts/tree/master/bitnami/wordpress

しかし、ただインストールするだけでなく、それをIstio環境でインストールします。
なお、今回は本家のIstioではなく、Tanzu Service Meshを使いました。

## WordpressをIstio下に配置することで、何がうれしいの？

そこまで嬉しくはないかもしれないですが、細かい点としては、以下ができます。

* 分散トレーシングができる
* Wordpressのアクセス数などをモニターできる
* MySQLの接続もモニターできる
* mTLSがうれしい（かも？）

ネットで検索したら、あまりIstio + Wordpressをやっているものがなかったので、一応記事かしました。

# 今回の構成

絵で表すといかになります。

![](../../images/tsm_wordpress/2020-09-17T12-25-01.png)


## MariaDB(Istio有効）

WordpressにはMySQLデータベースが必要のため、これまたBitnamiのHELMチャートで構成します。
ここにあるものを使います。

https://github.com/bitnami/charts/tree/master/bitnami/mariadb

なお、後述しますが、本当はmariadb-galeraのチャートを使いたかったのですが、今回は諦めています。

## NFS(Istio無効)

いかで解説されていますが、Wordpressはレプリカ構成の場合、RWXのボリュームが必要です。

https://engineering.bitnami.com/articles/scaling-wordpress-in-kubernetes.html

なので、ここではNFSをつかって構成します。
以下のまたBitnamiのHELMチャートを使います。

https://github.com/helm/charts/tree/master/stable/nfs-server-provisioner

ここに限って言えば、Istio化するメリットはまったくないようなので、有効にしていません。

## Wordpress(Istio有効)

ここをそのまま使います。

https://github.com/bitnami/charts/tree/master/bitnami/wordpress

## Galeraはあきらめた

いかに報告されている事象にあるよう、GaleraがいくらやってもIstioを有効にするとアクセスができなくなってしまいました。

https://github.com/bitnami/charts/issues/3510

よって今回はGaleraはあきらめました。

# 手順

## NFSのデプロイ

NFSのデプロイは以下の通りです。
まずNFSサーバーのYamlファイルを以下のように用意します。
`storageClass`に入る値は環境ごとに調整してください。

```yaml
persistence:
  enabled: true
  storageClass: default
  size: 50Gi
```

そして、NFSをデプロイします。

```
kubectl create ns bitnami-nfs
kubectl config set-context --current --namespace=bitnami-nfs
helm install -f nfs-server.yaml nfs-server  stable/nfs-server-provisioner
```

## MariaDBのデプロイ

mariaDBのデプロイは以下の通りです。
まずYamlファイルを以下のように用意します。
`storageClass`に入る値は環境ごとに調整してください。

```Yaml
rootUser:
  forcePassword: true

db:
  user: "wordpress"
  name: wordpress
  forcePassword: true

metrics:
  enabled: true

master:
  persistence:
    storageClass: default
slave:
  persistence:
    storageClass: default
```

さらに、パスワードを保管した別のsecret.yamlを用意します。


そしてMariaDBをデプロイします。

```
# Install MariaDB
kubectl create ns mariadb
kubectl config set-context --current --namespace=mariadb
kubectl label namespace mariadb istio-injection=enabled
helm install -f mariadb.yaml -f secrets/mariadb.yaml wordpress tac/mariadb
```
