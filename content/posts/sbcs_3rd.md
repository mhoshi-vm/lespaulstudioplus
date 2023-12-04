---
title: "Tanzu ObservabilityとSpring Bootで始めるアプリケーション観測"
date: 2021-01-01T21:30:12+09:00
categories: ["Tanzu Application Service"]
tags: ["Tanzu Application Service"]
thumbnail: "images/7/2020-10-15T02-49-32.png"
draft: true
---


[第1回](https://licensecounter.jp/devops-hub/blog/tanzu-observability/)および、[第2回](https://licensecounter.jp/devops-hub/blog/-tanzu-observability2/)の記事でTanzu Observabilityを紹介させていただきました。このシリーズ最後の記事では、Tanzu Observabilityでアプリケーション観測を実現する「分散トレーシング」を紹介させていただきます。

なお、この記事では初回から引き続き、意図的に「監視」ではなく「観測」(Observe)という表現を使います。

![](../../images/sbcs_2nd/2020-12-14T07-54-00.png)

## 分散トレーシングとは

「分散トレーシング」という言葉に馴染みがない方々にこれがどういった機能か解説します。

分散トレーシングとは、アプリケーション可視化の手法の一つであり、Tanzu Observabilityだけでなく、他のベンダー製品やオープンソース(ZipkinやJaegerが代表格)でも使われています。分散トレーシングのキーとなる用語がTraceとSpanです。これをより理解するために、あるREST APIを想像してください。このREST APIがリクエストを受け取り、なんらかの出力を出すまでに流れ、これを分散トレーシングではTraceと呼びます。その一つのTraceが実行される間の個別の処理(e.g. DBへのクエリ、演算、外部サービスの連携)をSpanと呼びます。分散トレーシングとは平たく言えば、このTraceとSpanを見える化して、問題箇所を特定するという技術です。

![](../../images/sbcs_3rd/2021-01-06T00-22-16.png)
(TraceとSpanの例)

分散トレーシングが注目されるようになった背景が、(また)昨今のマイクロサービスの流れです。マイクロサービスでは、小さな単位のアプリケーションがネットワークで相互に通信を行い、一つの大きなサービスを形成していくことを目指していきます。この際の通信の見える化およびボトルネックの特定に期待されているのが、分散トレーシング技術です。

![](../../images/sbcs_3rd/2021-01-06T00-53-19.png)

## Tanzu Observabilityの分散トレーシング機能

Tanzu Obserabilityで使われている分散トレーシングの主な3つの機能を紹介させていただきます。

#### トレースビュー

トレースビューでは先の分散トレーシングで説明したTraceとSpanの流れを以下のように図式化された状態でみることがことができます。こうすることにより、どの処理にどれくらいの時間がかかったのかなどがみることができるようになっています。また[スパンログ](https://docs.wavefront.com/trace_data_details.html#span-logs)とよばれる特定のエラーメッセージを保管し、このビューから確認することができます。

![](../../images/sbcs_3rd/2021-01-06T01-17-49.png)
(トレースビュー)

#### マップビュー

マップビューは、[2020年の最後のアップデートでGAされた機能](https://docs.wavefront.com/2020.38.x_release_notes.html)です。トレースビューでは時系列でTraceとSpanの流れを表示することができますが、マップビューでは複数のアプリケーションがどのように連携しているかを図式化されるようになります。また[External Service](https://docs.wavefront.com/tracing_external_services.html)機能を使うことにより、データベースやSaaSの通信を可視化することもできようになっており、それらのトラブルシュートにも役立てることができます。

![](../../images/sbcs_3rd/2021-01-06T01-13-59.png)
(マップビュー)

#### オフライントレース

オフライントレースとは、分散トレーシングの情報を一度ローカルに保存し、[後に参照することができます](https://docs.wavefront.com/tracing_view_offline_traces.html)。Traceは非常に高速に大量のデータが転送されるため、重要なものが流れてしまう危険性があります。これをつかうことのよって、データが流れきってしまう前に、ローカルにトレーシング情報を保管できます。そして後に参照することで問題判別に役立てることができます。

![](../../images/sbcs_3rd/2021-01-06T01-54-14.png)
(オフライントレース)

以上Tanzu Observabilityの分散トレーシングの3つの機能を紹介させていただきました。

## Tanzu ObservabilityとSpring Boot：分散トレーシングをより手ごろに

分散トレーシングは、アプリケーション観測を行うための様々なメリットを提供します。しかし、分散トレーシングを始める上での懸念点の一つが、敷居の高さです。

まず、最初の悩みが、アプリケーションのコードの改変が必要な点です。ここまで説明してきたTraceやSpanですが、自動的に付与されません。そのため、アプリケーション開発者がSpanの開始点を見極めコーディングすることが要求されます。Tanzu Observabilityの場合は、マニュアルにも紹介されていますが、[専用のSDKを使いコードのアップデート](https://docs.wavefront.com/tracing_instrumenting_frameworks.html)が必要です。

もう一点悩ませるのが環境の用意です。アプリケーション開発の初期段階、検証フェーズの際にTanzu Observabilityのような有償製品を利用は躊躇するかもしれません。そして検証もできず、本番環境の利用も躊躇するという悪循環が起きるかもしれません。

この懸念点ははTanzu Observability固有のものではなく分散トレーシングをサポートする他の製品でも同様に存在します。そのため、分散トレーシングは多くの企業で着手ができていない観測項目の一つだと思われます。Tanzu Observabilityでは、この懸念に対し他社と比較し独特かつ一線を画すソリューションを提供しています。それはSpring Bootとの強力な連携です。

Spring Bootとは、[開発者の中でもっとも支持されているJavaのフレームワークです](https://www.jrebel.com/blog/2020-java-technology-report#:~:text=The%20far%20and%20away%20favorite,framework%20for%20several%20years%20now.)。そして、VMwareでは[このSpring Bootを強力に推し進めています](https://tanzu.vmware.com/spring-app-framework)。

Tanzu Observabilityでは、Spring Boot開発者達が即座に分散トレーシングを開始できるよう以下の機能を提供しています。

#### Tanzu Observabilityの機能をライブラリとして組み込み

[2020年の5月にSpring BootとTanzu Observabilityの連携機能がアナウンスされました](https://spring.io/blog/2020/05/07/tanzu-observability-by-wavefront-spring-boot-starter)。この連携と同時にSpring Bootのアプリケーションの雛形を作るサイト、[start.spring.io](https://start.spring.io/)にて、Tanzu Observabilityの依存関係追加が可能となりました。これによりベンダーロックインなSDKでの開発や専用のエージェントのインストールをすることなく、アプリケーションの標準ライブラリとしてTanzu Observabilityと連携できるようになりました。

![](../../images/sbcs_3rd/2021-01-06T02-50-00.png)
(Spring Bootとの連携)

さらにポイントなのが、追加のコーディングがほぼなく、分散トレーシングが使えるようになった点です。Spring Bootではもともと、[Spring Cloud Slueth](https://spring.io/projects/spring-cloud-sleuth)とよばれる分散トレーシングを自動で追加するモジュールが提供されていました。Tanzu Observabilityはこれと連携し、アプリケーションのコードを改変せずとも分散トレーシングを実現できる技術を提供しています。

#### サインアップ不要の無料ライセンス

上記に加えSpring Bootを使う全てのユーザーがアクセス可能な、Tanzu Observabilityの[Freemium(無料)](https://docs.wavefront.com/wavefront_springboot.html#sending-data-from-spring-boot-into-wavefront)ライセンスが提供されています。このFreemiumライセンスはメールアドレスすら必要としない、サインアップ不要なものです。

利用には、先ほど紹介したTanzu Observabilityの依存関係の追加をするだけです。Spring Bootアプリケーション起動時に以下のようなメッセージがコンソールに出力されます。内容にあるよう、自動的にアカウントが作成され、そして専用のログインURLも提供されます。

```
A Wavefront account has been provisioned successfully and the API token has been saved to disk.

To share this account, make sure the following is added to your configuration:

	management.metrics.export.wavefront.api-token=xxxx-xxxx-xxxx-xxxx
	management.metrics.export.wavefront.uri=https://wavefront.surf

Connect to your Wavefront instance using this one-time use link:
https://wavefront.surf/us/example
```
それを使ってTanzu Observabilityにログインすることができ、かつここまで紹介した分散トレーシング機能も全て使えます。

このFreemiumライセンス自体は様々な[制約](https://docs.wavefront.com/wavefront_spring_boot_faq.html)が設けられていますが、検証段階のアプリケーションを試験的にテストするためには最適な方法です。これにより、環境の用意のハードルの敷居がさがります。

以上、Tanzu ObservabiityとSpring Bootを使ったアプリケーション観測について紹介しました。なお、この機能に関するより詳細および実機で試したい場合、[公式のチュートリアル](https://docs.wavefront.com/wavefront_springboot_tutorial.html)や[個人のブログ](https://blog.lespaulstudioplus.info/posts/wf-demanabu-dt/)を参照ください。Spring Bootについても今後さらにとりあげる予定です。

## まとめ

分散トレーシングの概要、Tanzu Observabiityの機能そして、SpringBootの関係を紹介させていただきました。また、このシリーズも計3回に渡り、Tanzu Observabilityを紹介させていただきました。監視ではなく「観測」を実現するため役立てられれば幸いでした。
