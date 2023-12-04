---
title: "TAS4K8Sにcf-k8s-prometheusをインストールして、さらにWavefrontと連携する"
date: 2020-11-16T21:30:12+09:00
categories: ["Tanzu Observability"]
tags: ["Tanzu Application Service", "Tanzu Observability", "Wavefront"]
thumbnail: "images/11/2020-11-17T03-07-46.png"
draft: true
---

TAS4K8Sにcf-k8s-prometheusをインストールして、それをさらに、Wavefrontと連携できます。<!--more-->

## はじめに

TAS4K8Sのアプリごとのメトリクスですが、[cf-for-k8s-metrics](https://github.com/cloudfoundry/cf-for-k8s-metric-examples)でやり方が紹介されているよう、各デプロイメントのアノーテーションを追加することで、スクレイプのEndpointを通知できるようになります。あとはこれはPrometheusなどによってスクレイプさせればいいのですが、問題はTAS4K8Sで使われているIstioのサービスメッシュです。サービスメッシュの特徴としてmTLSが有効になったサービスメッシュ内からはサービスメッシュ外の通信は簡単に許可しません。これは、[cf-for-k8s-metrics](https://github.com/cloudfoundry/cf-for-k8s-metric-examples)でもかかれていますが、以下の特徴のためです。

> Be deployed in namespace that has the label "istio-injection=enabled". This injects istio sidecars onto prometheus's pods. The recommended namespace is cf-system.

>Have a network policy in place that allows prometheus to scrape your app's pod.

>Have the necessary certs available in prometheus's istio sidecar.

[cf-k8s-prometheus](https://github.com/cloudfoundry/cf-k8s-prometheus#how-to-deploy-in-cf-for-k8s)では、これらの問題を解決した状態でのPrometheusのインストールの自動化を行ってくれます。

さて、今回はこれをWavefront(Tanzu Observability)のPrometheus Storage Adapterと連携してデータを送る方法を紹介します。絵で紹介すると以下のような内容です。

![](../../images/11/2020-11-17T02-24-25.png)

## 注意点

以下の方法はまだ正式サポートが表明されていない手段です。
また執筆時点では、TAS4K8SがBetaの頃に記載されています。

## 前提

* TAS4K8Sの環境　（筆者は0.4.0を使用)
* Wavefront(Tanzu Observability)のアカウント取得済み

## 手順

### 1. Prometheus Storage Adapterのインストール

ここの[Helmチャート](https://github.com/wavefrontHQ/helm/tree/master/prometheus-storage-adapter)通りにインストールすることがおすすめです。

以下のコマンドでインストールします。なお、`--set adapter.prefix=prom`をいれておくことをおすすめします。あとでWavefront側でのメトリクスが判別しやすくなります。

```
kubectl create namespace prom-adapter
helm install prom-adapter wavefront/prometheus-storage-adapter --namespace prom-adapter \
		--set wavefront.wavefront.url=https://<wavefront url> \
		--set wavefront.wavefront.token=<wavefront proxy> \
		--set adapter.prefix=prom
```

PODが起動することを確認します。

```
% kubectl get po -n prom-adapter
NAME                                                       READY   STATUS    RESTARTS   AGE
prom-adapter-prometheus-storage-adapter-655ddb5dc9-dv8dz   1/1     Running   0          30m
prom-adapter-wavefront-proxy-77f4986bf6-dfbkd              1/1     Running   0          30m
```

### 2. TAS4K8SのManifestを更新

以下の内容で`../configuration-values/prometheus-wavefront-proxy.yaml`を用意します。

```
#@ load("@ytt:overlay", "overlay")
#@ load("@ytt:yaml", "yaml")

#@ def wavefront_proxy_config():
#@overlay/match missing_ok=True
#@overlay/match-child-defaults missing_ok=True
remote_write:
- url: http://prom-adapter-prometheus-storage-adapter.prom-adapter.svc.cluster.local:1234/receive
#@ end

#@overlay/match by=overlay.subset({"kind": "ConfigMap","metadata": {"name": "prometheus-server"}})
---
data:
  #@overlay/replace via=lambda a,_: yaml.encode(overlay.apply(yaml.decode(a), wavefront_proxy_config()))
  prometheus.yml:
```
#### バージョンによっては必要な手順

のちのステップで以下のようなエラーが出る場合があります。
```
ytt: Error:
- cannot load /namespaces.star: Expected to find file 'namespaces.star' (hint: only files included via -f flag are available)
    in <toplevel>
      add-prometheus-node-exporter.yml:2 | #@ load("/namespaces.star", "system_namespace")
```
その場合は、以下のような`../configuration-values/namespaces.star`ファイルを作成してください。

```
def system_namespace():
  return "cf-system"
end

def workloads_namespace():
  return "cf-workloads"
end

def workloads_staging_namespace():
  return "cf-workloads-staging"
end
```

### 3. Manifestファイルの更新

まず、cf-k8s-prometheusのソースをダウンロードします。

```
cd <tas install dir>
git clone https://github.com/cloudfoundry/cf-k8s-prometheus/ ../cf-k8s-prometheus
```

そして以下のようにManifestファイルを更新します。

オンラインでインストールした場合は以下のコマンド
```
 ytt -f config -f ../configuration-values -f ../cf-k8s-prometheus/config | kbld -f- > /tmp/final_deployment_tmp.yml
```

プライベートレポジトリー経由でインストールした場合は以下のコマンド

```
 ytt -f config -f ../configuration-values -f ../cf-k8s-prometheus/config | kbld -f- -f relocated_images.yml > /tmp/final_deployment_tmp.yml
```



### 4. Manifestの適用

以下でManifestを適用します。

```
kapp deploy -a cf -f /tmp/final_deployment_tmp.yml
```

## 確認

ここまで設定すると、WavefrontにPrometheusのメトリクスが見えてくるはずです。
メトリクスは以下のロジックでみえるようになります。

* `_`アンダースコアを`.`ドットに切り替える
* 接頭語に`-prefix`を指定したものが付与される。

以下は`prometheus_remote_storage_sent_bytes_total`を確認した結果

![](../../images/11/2020-11-17T03-05-53.png)

## まとめ

TAS4K8SをPrometheusでスクレイプし、さらにWavefrontに転送することは可能です。
