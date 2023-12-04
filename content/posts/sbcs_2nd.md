---
title: "Tanzu Observabilityを使ったSaaS型のKubernetes観測"
date: 2020-11-24T21:30:12+09:00
categories: ["Tanzu Application Service"]
tags: ["Tanzu Application Service"]
thumbnail: "images/7/2020-10-15T02-49-32.png"
draft: true
---

[前回](https://licensecounter.jp/devops-hub/blog/tanzu-observability/)の記事でTanzu Observabilityの概要を紹介させていただきました。今回は一歩踏み込んで昨今話題のKubernetes観測について取り上げます。

なお、この記事では前回に習い、意図的に「監視」ではなく「観測」(Observe)という表現を使います。

![](../../images/sbcs_2nd/2020-12-14T07-54-00.png)
## Prometheusから脱却?SaaS型のKubernetes観測のメリットとは

Kubernetesの登場により、より柔軟なインフラストラクチャ、そしてアプリケーションをオンデマンドに展開やスケールアップが可能になりました。しかし、それと同時に課題になるのが観測の手法です。

Kubernetesの観測と聞き、多くの方が最初に思い浮かべるソリューションが[Prometheus](https://prometheus.io/)だと思います。Prometheusは、[Cloud Native Computing Foundation(CNCF)のプロジェクトの一つであり](https://landscape.cncf.io/selected=prometheus)、Kubernetes観測の代表格です。特徴としてエージェントに依存せず、サービスディスカバリによって観測対象を自動的にみつけだし収集できることと、[PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/)という高速に必要なデータを取り出すクエリ言語をサポートしている点です。きっと皆様も、PoCや本番環境で利用されたことがあると思います。

しかし、Prometheusの導入によって新たな問題が発生していきます。最初の問題がスケーラビリティです。まず、Prometheusが高可用性(HA)の構成が取りにくいという点です。結果として、複数のPrometheusをKuberntesごとに作るなどの構成になってきてしまいます。当然Prometheusの数に伴い運用の難易度が高くなります。さらに頭を悩ませるのがストレージの容量です。Prometheusのデータ量はKubernetesの規模によって大きくなり、数テラバイト〜数ペタバイドの容量になりうります。

もう一つの問題がセキュリティです。Prometheusは、 [ユーザー認証やアクセス制御の仕組みは機能が充実していません](https://prometheus.io/docs/operating/security/#authentication-authorization-and-encryption)。そのため、Prometheusを複数チームで共有することや、オペレーター以外のユーザーに公開することは難しくなります。
![](../../images/sbcs_2nd/2020-12-14T07-47-48.png)
（KubernetesのPrometheus連携とその課題)

こういった点に疑問を感じ始めたときこそが、SaaS型のKubernetes観測を検討するべきタイミングです。SaaSにすることによって、オンプレミス側の運用を簡素化していきます。さらには、データ容量もSaaSのサービスによって保証されるだけでなく、アップタイムもService Level Agreement(SLA)で保証されます。ユーザー認証もプラットフォームが提供する機能を利用することができます。

Tanzu ObservabilityはSaaS型プラットフォームであり、Kubernetes環境の観測を次のステップへと昇格させることができます。この記事では、この点の詳細をみていきます。


## Tanzu ObservabilityとPrometheusの連携

冒頭で取り上げたように既にPrometheusをお使いのお客様も多いと思います。Tanzu ObservabilityはPrometheusの運用で培った経験や資産を活かしつつ連携することができます。

![](../../images/sbcs_2nd/2020-12-14T07-52-23.png)
(PrometheusとTanzu Observabilityの連携)

それを実現する一つ目の機能が[Prometheus Storage Adapter](https://docs.wavefront.com/prometheus.html)です。この機能を使うことにより、PrometheusのデータをTanzu Observabilityに転送をすることができます。この構成のメリットがオンプレミス側のPrometheusを小さくできることです。あくまで、オンプレミスのPrometheusをキャッシュストレージとして使い、データの長期保管をTanzu Observability側にまかせることができます。(Tanzu Observabilityはデータを18ヶ月保管することができます。)

もう一つの機能がTanzu Observability上での[PromQL](https://docs.wavefront.com/wavefront_prometheus.html)のサポートです。今まで使ってきたPromQLをベースとしTanzu Observabilityのダッシュボードやアラートの作成に使うことができます。なお、PromQL自体知名度が高く、使っている方も多いでしょうが、最終的にはTanzu Observabilityが提供するクエリ言語である[Wavefront Query Language](https://docs.wavefront.com/query_language_reference.html)に移行することがお勧めです。この言語をつかうことによって100を超える分析手法を中心にTanzu Observabilityの機能を全てつかえるようになるためです。

さて、複数のPrometheusを一つのSaaSプラットフォームでまとめあげたとき、チームごとのアクセス管理が気になるかもしれません。もしかしたら、チーム間ではみられて困るかもしれないデータがあるかもしれません。この時に役にたつのが[Metrics Security](https://docs.wavefront.com/metrics_security.html)機能です。この機能によってPrometheusから転送されるデータそのものにユーザーアクセス制御ができることです。よって複数のPrometheusを１つのプラットフォームにまとめても、マルチテナントやユーザーアクセスを制御できます。

## Wavefront Collector for Kubernetesによるデータ収集

Prometheusをまだ使っていない、もしくはTanzu Observabilityの使い方になれたら次に検討していただきたいのが、[Wavefront Collector for Kubernetes](https://docs.wavefront.com/kubernetes.html)をつかい直接Kubernetesをデータを転送することです。

![](../../images/sbcs_2nd/2020-12-14T07-58-54.png)
(Wavefront Collectorを使ったTanzu Observabilityの直接連携)

この機能をつかうことがお勧めする一つの理由が、Tanzu Observabilityのデフォルトのダッシュボードをつかうことができる点です。このデフォルトのダッシュボードはクラスタ、ノード、ポッドなど階層化されたものになっています。それぞれのダッシュボードから気になる箇所をクリックしていき、ドリルダウンしていくことができます。そうすることで、自分のKubernetesの環境を様々な断面でみていくことができます。
![](../../images/sbcs_2nd/2020-12-14T08-08-27.png)
(Cluster, Nodes, Podのダッシュボードの遷移)

さらには[TroubleShootingダッシュボード](https://docs.wavefront.com/wf_kubernetes_troubleshooting.html)とよばれる問題解決に特化したダッシュボードを提供していることです。こうすることで、Kubernetesの問題解決に役立てることができます。

![](../../images/sbcs_2nd/2020-12-14T08-02-30.png)
(Kubernetes TroubleShootingダッシュボード)

もう一つお勧めする理由がKubernetes以外のミドルウェアとの観測するためのダッシュボードを提供している点です。多くのお客様ではKubernetesの上でApache, MySQL, Memcachedなどのミドルウェアを稼働させていると思います。[前回の記事](https://licensecounter.jp/devops-hub/blog/tanzu-observability/)でも紹介したようTanzu Observabilityは250近いインテグレーションを持っています。そしてWavefront Collectorはそれら対応したミドルウェアを自動的にディスカバー、つまり見つけ出し、メトリクスを収集することができます。そして、それらもまたデフォルトのダッシュボードを提供しています。こうすることで、Kubernetesだけでなく、ミドルウェアのレイヤーまで観測を広げることができます。

![](../../images/sbcs_2nd/2020-12-14T08-00-53.png)
(MySQLダッシュボード)

Wavefront Collectorのインストールですが、Kubernetesの標準的なデプロイツールである[Helm](https://helm.sh/)の他に、[Tanzu Mission Control](https://tanzu.vmware.com/mission-control)経由でインストールできます。Tanzu Mission Controlは（本記事では詳細に取り上げませんが)Kubernetesのライフサイクル管理を行うことができるSaaSプラットフォームであり、Wavefront Collectorを簡単にインストールするためのインテグレーション機能を提供します。以下の画面ショットにあるように、Tanzu Observabilityの認証情報(APIキー)を登録するだけで利用できます。こうすることでKubernetesのライフサイクル管理と、観測を同時に行うことができます。

![](../../images/sbcs_2nd/2020-12-14T03-16-52.png)
(Tanzu Mission ControlとTanzu Observabilityの連携)

## さらにアドバンスに、Horizontal Pod Autoscalerとの連携

既に盛り沢山の内容ですが、最後にもう一つだけ機能を紹介します。それはHorizotal Pod Autoscaler(HPA)との連携です。

![](../../images/sbcs_2nd/2020-12-14T08-17-55.png)
(Tanzu ObservabilityとHPAの連携)

HPAとは、Kubernetesにおける自動スケリーング機能です。従来の自動スケーリングは[CPUとメモリの消費量にしか対応していない、そして観測プラットフォームとは別のメトリクスをトリガーとする課題](https://github.com/kubernetes-sigs/metrics-server)がありました。

Tanzu Observabilityの[HPA Adapter](https://docs.wavefront.com/wavefront_kubernetes.html#kubernetes-scaling-with-wavefront)を使うと、CPUやメモリだけでなくTanzu Observabilityからの値で自由にスケーリングのトリガーを定義できます。さらは、Wavefrontは[API稼働率99.95%をSLA](https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/support/vmw-wavefront-service-level-agreement.pdf)を保証しています。よって信頼できる観測基盤として安心してアプリケーションで利用することができます。

このHPAの使い方は[個人のブログ](https://blog.lespaulstudioplus.info/posts/wf-demanabu-hpa/)などで紹介されているので合わせてご覧ください。

## まとめ

以上、Kubernetesの観測基盤としてTanzu Observabilityを使うメリットを紹介させていただきました。
ぜひ、Tanzu Observabilityを使いよりアドバンスなKubernetes観測に役立ててください。
