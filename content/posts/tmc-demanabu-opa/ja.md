---
title: "Tanzu Mission Controlで学ぶOpen Policy Agent Part-1"
date: 2020-09-24T21:30:12+09:00
tags: ["Tanzu Mission Control", "Open Policy Agent"]
thumbnail: "2020-09-24T13-09-22.png"
featured: true
---

この記事はTanzu Mission Controlで学ぶ Open Policy Agentシリーズです。<!--more-->

# シリーズ

第一回 : 概要 **< いまここ**  
第二回 : [TMCのOpen Policy Agentを遊んでみる](../tmc-demanabu-opa-02)  
第三回 : [TMCのOpen Policy Agentを解剖する](../tmc-demanabu-opa-03)  
第四回 : [TMCからOPAポリシーを自作する](../tmc-demanabu-opa-04)  

# Tanzu Mission Controlって何？

Tanzu Mission ControlとはVMwareがリリースしたマルチKuberenetesを管理するための仕組みです。
省略では「TMC」と呼びます。

Tanzu Mission Controlを使うといろんなKubernetesを一元的に管理できます。
EKS,AKS,GKEが管理できるのも面白いです。
その中で面白い機能なのが、Open Policy Agentの機能が実装された点です。

# Open Policy Agentってなに？

## まずは以下をみてくれたまえ

Open Policy Agentを説明する前に、以下のGifをみてみてください。


![render1596759963723.gif](render1596759963723.gif)

これは、あるpodに対してkubectlを使いログインして色々操作している点です。
さて、気づきますでしょうか？３行目以降の`root [ / ]#`。。。

なんと**ホストOSへ昇格してrootシェルを奪っています。**

## なぜホストOSのRootシェルが奪えたか

さて、知っている人は知っているし、大したことではないのですが、これには２つの仕掛けがPodになされています。

* Podに対して、Privileged権限を付与している。これによってHost OSへの全ての操作がゆるされる。
* `HostPID: True`を指定し、ホストOSのプロセステーブルをみれるようにしてしまう。

Yamlとしては、以下の内容です。

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: verybad
spec:
  hostPID: true　# !!注目
  containers:
    - name: verybad
      image: alpine
      command: [ "sleep", "3600" ]
      securityContext:
          privileged: true # !!注目
```
これに加えて以下のnsenterコマンドで以下のフラグを指定することによってホストのプロセスID 1へ`BASH`プロンプトをくっつけてしまいます。

```
/usr/bin/nsenter -t 1 -m -u -n -i -- bash
```

## え？でもそれってPSPで守れるよね？

そのとおり、Pod Security Policyで守れます。ちゃんと権限設定をすれば、以下のようにOS権限昇格しようとしても怒られます。

```
mhoshino@mhoshino ~ % kubectl exec -it verybad sh
/ # /usr/bin/nsenter -t 1 -m -u -n -i -- bash
nsenter: can't open '/proc/1/ns/ipc': Permission denied
/ #
```
しかし、まだあまり知られていないですが、**PSPは近いうちにDeprecatedになる予定です。**

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Big news <a href="https://twitter.com/kubernetesio?ref_src=twsrc%5Etfw">@kubernetesio</a> pod security policy is likely to be deprecated in 2020 with the entire project likely moving to <a href="https://twitter.com/OpenPolicyAgent?ref_src=twsrc%5Etfw">@OpenPolicyAgent</a> Gatekeeper <a href="https://twitter.com/hashtag/KubeCon?src=hash&amp;ref_src=twsrc%5Etfw">#KubeCon</a> SIG Auth <a href="https://t.co/vmkJp52A9z">pic.twitter.com/vmkJp52A9z</a></p>&mdash; Sean Kerner (@TechJournalist) <a href="https://twitter.com/TechJournalist/status/1197658440040165377?ref_src=twsrc%5Etfw">November 21, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

このツイートに補足する形でもGithub上でもKubernetes v1.22でPSPがなくなる予定が宣言されました。

https://github.com/kubernetes/enhancements/issues/5#issuecomment-656120326
> The deprecation schedule for the current beta version in 1.22 is independent of whether or not an in-tree implementation of the standard pod security profiles will be provided. That has not yet been determined.

## Open Policy Agent PSPの代替として登場

PSPがなくなるという衝撃的なニュースとともに、静かに騒がれているのが、PSPの代替です。
そこでOpen Policy Agentです。
![](2020-09-24T13-25-05.png)


Open Policy AgentとはCNCFのプロジェクトの一つであり、より簡易およびポリシーを定義することができるツールです。
さらにKubernetesは[Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)を組み合わせることによって、なにかのリソースが作られる前にインターセプトして、拒否するなどができるようになります。

絵で表すと以下のような感じです。

![](2020-09-24T13-25-22.png)


# このシリーズはなんぞや？

Open Policy Agentとは、でてからまだ日が浅いですが、なんとTanzu Mission Controlではプロダクトとしてつかえるようになりました。そしてこれはPSPの代替として、つかわれるようになります。

このシリーズではTMCのOpen Policy Agentがどのように実装されているか解剖していくシリーズです。
次回は、「[TMCのOpen Policy Agentを遊んでみる](../tmc-demanabu-opa-02) 」です。
