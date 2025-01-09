---
title: "Wavefrontで学ぶHorizontal Pod Autoscaler Part-3"
date: 2020-08-23T21:30:12+09:00
categories: [Tanzu Observability]
tags: ["Wavefront","Tanzu Observability", "Horizontal Pod Autoscaler"]
thumbnail: "/images/hpa.png"
---

この文章は、Wavefrontで学ぶHorizontal Pod Autoscaler　シリーズの第三回目です。<!--more-->
# シリーズ

第一回　：　[概要編](../wf-demanabu-hpa)  
第二回　：　[Wavefront情報からスケールする](../wf-demanabu-hpa-02)  
第三回　：　サーバーレスもどきを実装する **← いまここ**


# 始めに

前回の[前提知識](../wf-demanabu-hpa)にもあったよう、Kubernetesは、HPAとして3つのAPIをサポートしています。

その中で、Wavefrontは`external.metrics.k8s.io`を使った自由度の高いメトリクスの定義をサポートしています。

以下により詳細の説明があります。

https://github.com/wavefrontHQ/wavefront-kubernetes-adapter/blob/master/docs/introduction.md#externalmetricsk8sio

ここでのポイントが実質Wavefrontのメトリクスをどれでもつかって自由にHPAができるというものです。
今回はこれをつかって、「サーバーレスもどき」のアプリケーションを作ってみます。

# 検証方法

今回は以下の図示されているような構成で検証します。

![](../images/wf_demanabu_hpa_03/2020-09-16T14-43-22.png)

つまり、Kubernetesとは別に稼働しているWebアプリ(`/counter`)に対して、リクエストを送信してスケールアップをするような構成です。特にリクエストがやってこない場合、徐々にスケールダウンしていきます。

なお、「サーバーレスもどき」と書いたわりに、必ず最低1つのPodが常に起動し続けます。
これは、`HPAScaleToZero`とよばれるHPAのminインスタンスの値`0`を許可するが、執筆時点でまだAlphaの実装であり、GAとされていないためです。

https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-gates-for-alpha-or-beta-features

これがGAされれば、より「サーバーレスもどき」に近づくのですが、今回は諦めます。

# 準備

前回の[準備編](https://qiita.com/hmachi/items/cfeeb53ea37f12ac27af#準備編)と[インストール](https://qiita.com/hmachi/items/cfeeb53ea37f12ac27af#インストール)を参照してください。

# アプリの準備

今回は、すでに持っているWavefrontのアカウントに対してMetricsを送付するようなアプリケーションを書きます。様々な方法がありますが、ここでは一番手っ取り早いSpring Bootを使ったアプリを作ります。

なお以下のアプリはWavefrontにアクセスできる端末であればどこで起動しても問題ないです。PC上でもいいです。最低限OpenJDKがインストールされている必要があります。

https://github.com/mhoshi-vm/wf-demanabu-hpa

このレポジトリーをクローンします。

```
git clone https://github.com/mhoshi-vm/wf-demanabu-hpa
cd wf-demanabu-hpa
```

そして以下のファイルを編集します。

```
vi src/main/resources/application.properties
```

中身はWavefrontのアカウントとAPIキーを登録してください。

```
wavefront.freemium-account=false
management.metrics.export.wavefront.uri=https://<account>.wavefront.com
management.metrics.export.wavefront.api-token=<API key>
```

準備ができたら起動します。

```
./mvnw spring-boot:run
```

ただしく起動されたら、別のプロンプトでしばらく、URLへリクエストを送信します。

```
curl localhost:8080/counter
```

`Count Complete`と出れば問題ないです。

この状態でWavefrontのUIにログインします。
そして以下のQueryを実行します。

```
mdiff(5m, ts(custom.metrics.counter))
```

すると以下のように、curlを実行した回数分、5分間停滞するような画面になっていればOKです。

![](../../images/wf_demanabu_hpa_03/2020-09-16T14-43-53.png)

アプリは起動したままで次のステップに移ります。

# HPAの準備

前回のゴミ掃除も兼ねて、作業用のネームスペースを再作成します。

```
kubectl delete ns hpa
kubectl create ns hpa
```

KubernetesのDeploymentを作成します。

```
kubectl create deployment --image=nginx hpa-pods -n hpa
```

そして、今回は以下のHPAを作ります。

```
cat <<EOF | kubectl apply -n hpa -f -
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: example-hpa-external-metrics
  annotations:
    wavefront.com.external.metric/scale_counter: 'mdiff(5m, ts(custom.metrics.counter))'
spec:
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: External
    external:
      metricName: scale_counter
      targetAverageValue: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-pods
EOF
```

キモがいかです。

* 参照したいメトリクスをAnnotationをつかって表現している。この場合は、`wavefront.com.external.metric/scale_counter: 'mdiff(5m, ts(custom.metrics.counter))'`
* spec.metricsをAnnotationで指定した名前を定義します。この場合は`scale_counter`

定義としてはこれだけです、作成されたHPAをみてみましょう

```
kubectl get hpa -n hpa
NAME                           REFERENCE             TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
example-hpa-external-metrics   Deployment/hpa-pods   0/1 (avg)   1         5         1          25m
```

この状態で、`curl localhost:8080/counter`を3回実行します。
しばらくするとTARGETSの値がCurlを実行した`1`に近い値に落ち着くと思います。またReplica数も3になります。

```
kubectl get hpa -n hpa
NAME                           REFERENCE             TARGETS        MINPODS   MAXPODS   REPLICAS   AGE
example-hpa-external-metrics   Deployment/hpa-pods   967m/1 (avg)   1         5         3          26m
```

これは、つまりPodに対する平均値が計算され、`3(curlの回数) / 3(podの値) = 1`となっているためです。

そして、５分後、また`0`になり、ゆっくりReplica数も`1`に近づいていくと思います。

```
kubectl get hpa -n hpa
NAME                           REFERENCE             TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
example-hpa-external-metrics   Deployment/hpa-pods   0/1 (avg)   1         5         1          35m
```

# まとめ

* Wavefrontの`external.metrics.k8s.io`を使えばどんなメトリクスをつかってもHPAができます。

全三回のシリーズから、Wavefront + HPAの実装がシンプルに実現できることが体験できました。
