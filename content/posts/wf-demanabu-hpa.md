---
title: "Wavefrontで学ぶHorizontol Pod Autoscaler Part-1"
date: 2020-08-21T21:30:12+09:00
categories: [Tanzu Observability]
tags: ["Wavefront","Tanzu Observability", "Horizontal Pod Autoscaler", "featured"]
thumbnail: "images/hpa.png"
featured: true
---

この文章は、Wavefrontで学ぶHorizontal Pod Autoscaler　シリーズの第一回目です。<!--more-->
# シリーズ

第一回　：　概要編　**← いまここ**  
第二回　：　[Wavefront情報からスケールする](../wf-demanabu-hpa-02)  
第三回　：　[サーバーレスもどきを実装する](../wf-demanabu-hpa-03)

# Wavefrontとはなんぞや？

WavefrontとはいうのはSaaSベースのクラウドおよびアプリケーションの監視プラットフォームです。
現在はVMwareに買収されており、Tanzu Observabilityと呼ばれています。
このシリーズでは、旧称のWavefrontを多用しますが同じものです。

# Horizontal Pod Autoscalerとはなんぞや？

Horizontal Pod Autoscaler(HPA)とは、KubernetesでデプロイしたPODを負荷をモニターしてスケールアウトする機能です。

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

# HPAを普通に使って何が悪いの？

さて、この機能でよく勘違いされるのが、k8sをインストールした直後にOOTB（Out of the Box)で使えないという点です。

通常はHPAを使えるようになるためには、なんらかのメトリクスを収集する別のツールが必要です。
有名なのが、metrics-serverです。

https://github.com/kubernetes-sigs/metrics-server

ただし、かかれている通り、CPU、Memoryにのみの対応です。
さらによくある問題として、「ほんとうにCPUがそのとき高かったの？」問題です。結局のところOpsチームはモニターダッシュボードをみていて、そのCPU、MemoryがうまくAutoScaleの事象に合わないといらぬコミュニケーション問題に発展しかねないです。

# Prometheusは？

PrometheusをHPAのソースとして使うというのは昨今外国のブログで紹介されています。
例えば以下です。

https://towardsdatascience.com/kubernetes-hpa-with-custom-metrics-from-prometheus-9ffc201991e

ただし、これも注意しないといけないのが、Prometheusを本番業務に影響を与えるコンポーネントに昇格させてしまう点です。Prometheusの運用に慣れていない場合やHAの構成をとれていない場合、あまりお勧めができない構成となってしまいます。

# じゃあどうするの？

そこでWavefrontですよ。
Wavefrontでは昨今のTanzu情勢から、kubernetesの連携を強めており、正式にHPAのアダプターをサポートしています。

https://github.com/wavefrontHQ/wavefront-kubernetes-adapter

英語ですが以下の動画でも取り上げて説明されています。

[![IMAGE ALT TEXT HERE](https://i3.ytimg.com/vi/nZnbdNHFNyU/hqdefault.jpg)](https://www.youtube.com/watch?v=nZnbdNHFNyU)  

https://www.youtube.com/watch?v=nZnbdNHFNyU


これを使うことのメリットとして

* Opsが見ているダッシュボードと同じメトリクスでHPAができる
* CPU、Memory以外の好きなメトリクスで自由に定義できる
* SaaSならではの、可用性(%99.95 Availability）とパフォーマンス保証

# このシリーズはなんぞや？

HPAというものを長らく存在を知っておきながら、いまだkubernetesの飛び道具的存在であり、手を出すの躊躇していたのはきっと私だけではないはず。

このシリーズでは、WavefrontというSaaSモデルでサポートされているシステムでHPAを構成してみて、今一度HPAがどのように動くものなのか理解を深めるというものです。

次回は「[Wavefront情報からスケールする](../wf-demanabu-hpa-02)」です。
