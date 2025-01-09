---
title: "Tanzu Mission Controlで学ぶOpen Policy Agent Part-2"
date: 2020-09-30T21:30:12+09:00
categories: [Tanzu Mission Control]
tags: ["Tanzu Mission Control", "Open Policy Agent"]
thumbnail: "/images/tmc_demanabu_opa/2020-09-24T13-09-22.png"
---

この記事はTanzu Mission Controlで学ぶ Open Policy Agentシリーズです。<!--more-->

# シリーズ

第一回 : [概要](../tmc-demanabu-opa)     
第二回 : TMCのOpen Policy Agentを遊んでみる  **< いまここ**  
第三回 : [TMCのOpen Policy Agentを解剖する](../tmc-demanabu-opa-03)  
第四回 : [TMCからOPAポリシーを自作する](../tmc-demanabu-opa-04) 

# はじめに

[前回](../tmc-demanabu-opa)の記事にあるよう、TMCを使うとOPAが気軽に試せます。
今回はまずこの機能を試してみます。

TMCでは、OPAをSecurity Policyとして実装しています。
この中身をみていきます。

# 準備編

TMCの環境が必要になります。アカウントをもっていない場合、HOL（Hands on Lab)が一番早く用意できます。
環境はこの記事をベースに確認してください。

[Tanzu Mission Controlを気軽に試す(via HOL)](https://qiita.com/hmachi/items/a2f5eeb5abd7c72a4873)

テストには、前回でも紹介したよう、悪意のあるPodの権限昇格の手段として使えた以下のYamlを使います。

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: verybad
spec:
  hostPID: true
  containers:
    - name: verybad
      image: alpine
      command: [ "sleep", "3600" ]
      securityContext:
          privileged: true
```
まず、特にセキュリティ設定していないk8sで走らせると、想定通りですが、動いてしまいます。

```
% kubectl create -f verybadsecurity.yaml
pod/verybad created
```

# 検証

## とりあえず試す

まずTMC側でPolicyを有効にします。

左ペインの[ Policies ] > [Assignment] > [ Security ]タブを選択します。
![](../../images/tmc_demanabu_opa_02/2020-09-30T13-40-00.png)

対象のClusterGroupを選択して[ CREATE SECURITY POLICY ]を選択します。

![](../../images/tmc_demanabu_opa_02/2020-09-30T13-43-46.png)

その他のオプションは、のちに説明します。
とりあえず、これで[ Create Policy ] で作ってしまいます。

すると、さきほど起動が成功してしまった、昇格権限をもったPodがいっぱい怒られて、上がらなくなります。
![](../../images/tmc_demanabu_opa_02/2020-09-30T13-46-41.png)

これのすごい点がClusterAdminで上の操作をしていますが、それでもTMCで設定したPodの設定が有効になります。
さらに、新しくNamespaceを作ってもそれが有効になります。以下の例では`foo`というネームスペースを作った場合にも有効になっていることを確認しています。

![](../../images/tmc_demanabu_opa_02/2020-09-30T13-47-19.png)

## Namespaceに例外を追加する

例えば、特定のNamespaceで例外を作りたいとします。
もう一度TMCの画面にもどって、作成したポリシーの一番下の`Exclude specific namespaces (optional)`を選択します。

![](../../images/tmc_demanabu_opa_02/2020-09-30T13-49-36.png)

 ここに、除外するネームスペースのルールをいれます。

![](../../images/tmc_demanabu_opa_02/2020-09-30T13-50-40.png)

先ほどのつくった`foo`のネームスペースにこのLabelを追加します。

```
kubectl label namespace foo security=false
```

この状態でもう一度違反Podを`foo`ネームスペースで作ると以下のように今度はエラーにならず実行できます。Labelに該当しないネームスペースは引き続き失敗します。

![](../../images/tmc_demanabu_opa_02/2020-09-30T13-58-54.png)

# Introducing Policy Insights!

さらにすごいのが、最近のリリースで追加されたPolicy Insightsの機能。
もし、ポリシー違反の状態でうごいているものがあった場合に画面一覧で表示する機能があります。

![](../../images/tmc_demanabu_opa_02/2020-09-30T14-05-52.png)

# あれ？で、OPAは？

さて、ここまでOPAが一切登場していないですが、実はこの裏で動いていた技術こそがOPAでした。
TMCのSecurity Policyを有効にした環境では、以下の`constrainttemplates`CRDが作られます。

```
kubectl get constrainttemplates
NAME                                              AGE
vmware-system-tmc-allowed-host-paths-v1           23m
vmware-system-tmc-allowed-users-v1                23m
vmware-system-tmc-allowed-volumes-v1              23m
vmware-system-tmc-block-host-namespace-v1         23m
vmware-system-tmc-block-privilege-escalation-v1   23m
vmware-system-tmc-block-privileged-container-v1   23m
vmware-system-tmc-enforce-host-networking-v1      23m
vmware-system-tmc-linux-capabilities-v1           23m
```

さらに、上のリソース名で、`kubectl get`をすると同じく何かみえてきます。

```
kubectl get vmware-system-tmc-block-host-namespace-v1
NAME             AGE
tmc.cgp.strict   24m
```

この設定がOPAに関連します。

さて、より細かい話は次回にとっておきたいと思います。
いまはTMCを使うとOPAが裏で動いてなんかすごいことをしている程度に思ってください。

# まとめ

* TMCのSecurity Policyを使うと不正な権限昇格を阻止できる
* Policy Insightsを使うと、ポリシー違反を一目でわかる
* TMCは裏でOPAをつかっている

次回は「[TMCのOpen Policy Agentを解剖する](../tmc-demanabu-opa-03)」です。
