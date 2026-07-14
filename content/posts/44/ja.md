---
title: "Oracle GoldenGate を RabbitMQ の JMS クライアントを使いデータ連携"
date: "2026-07-16T00:00:00+09:00"
tags: ["Oracle", "Golden Gate", "RabbitMQ", "JMS"]
thumbnail: "img.png"
---
今回は、Oracle GoldenGate を利用して Oracle Database の変更データ（CDC: Change Data Capture）を RabbitMQ へ連携（レプリケーション）する方法を試していきます。
個人的にはあまり使ったことがない、RabbitMQのJMSクライアントを経由しています。

対象のレポジトリはこちらです。
[mhoshi-vm/oracle-gg-to-rabbitmq](https://github.com/mhoshi-vm/oracle-gg-to-rabbitmq)

## はじめに

Oracleのデータベースをリアルタイムに更新を取得（これをCDCと呼んだりします）するには、GoldenGateを使うことが多いです。 さらに、そのデータをマイクロサービスと連携する場合、メッセージングサービスKafka が選ばれることが普通です。
Kafkaと本質的に一緒なRabbitMQでも同様にできるはずなのですが、[Oracle GoldenGateのターゲット](https://docs.oracle.com/en/database/goldengate/big-data/26/gadbd/target.html)にRabbitMQが記載されていないです。
そのせいで長らく、「GoldenGateはRabbitMQをサポートしない」と認識されていたと思います。

ところが、[Oracle GoldenGateはJMSプロトコルをサポート](https://docs.oracle.com/en/database/goldengate/big-data/26/gadbd/java-message-service-jms1.html#GUID-66A87483-8428-4B13-B043-F6ABFD74E7D3)しており、
[RabbitMQもJMSをサポート](https://www.rabbitmq.com/client-libraries/jms-client)しています。
ということは実はこれを組み合わせれば、夢にまで見たOracleのCDCをRabbitMQに流すことができる！？を試してみました。


Oracle GoldenGate (OGG) の Big Data Adapters と RabbitMQ のJMSクライアントを使って連携をします。
ここでは、その連携を行うための構成や設定方法を解説していきます。

## アーキテクチャ

冒頭にも書きましたが、OracleとRabbitMQが双方でサポートするJMSを使って連携します。
やり方は以下です。


1. Oracle Database 内のトランザクションログ (Redoログ) を OGG の Extract プロセスがキャプチャ
2. 変更データ（Trailファイル）を OGG の Replicat プロセス (Big Data Adapter) が読み取る
3. OGG Big Data Adapter が JMS (Java Message Service) を介して RabbitMQ の Exchange / Queue に JSON 形式等で Publish する

```
┌─────────────────────────┐     redo      ┌──────────────────────┐
│   Oracle DB Free 23ai   │─────logs─────▶│  GoldenGate Capture  │
│   (CDB / FREEPDB1 PDB)  │   (LogMiner)  │  goldengate-oracle   │
└─────────────────────────┘               └──────────┬───────────┘
                                                     │ trail files
                                              shared Docker volume
                                                     │
                                          ┌──────────▼───────────┐
                                          │  GoldenGate DAA      │
                                          │  goldengate-daa      │
                                          │  (JMS handler)       │
                                          └──────────┬───────────┘
                                                     │ JMS
                                          ┌──────────▼────────────────────┐
                                          │  RabbitMQ 4.3                 │
                                          │  jms.durable.queues exchange  │
                                          │  oracle.cdc                   │
                                          └───────────────────────────────┘
```

## レポジトリの内容と前提条件

今回作成したレポジトリ [mhoshi-vm/oracle-gg-to-rabbitmq](https://github.com/mhoshi-vm/oracle-gg-to-rabbitmq) には、検証環境をサクッと立ち上げるための Docker Compose ファイル群と、OGG 側のプロパティファイル（`.prm` / `.properties`）のサンプルを格納しています。
基本的なセットアップも README.md に記載しています。

### 必要な環境

*   Docker および Docker Compose
*   Oracle GoldenGate for Big Data の**商用ライセンス**(無償ライセンスではできないです)

### 結果

レポジトリにも書いていますが、セットアップできればOracleで行ったSQLがRabbitMQに伝搬されていきます。

今回のセットアップでは、`jms.durable.queues` > `oracle.cdc` の経路で溜まっていきます。

![img_1.png](img_1.png)

ここにさらにテストの結果を載せていますが、10万SQLの以下も試しました。

[https://github.com/mhoshi-vm/oracle-gg-to-rabbitmq/blob/main/TESTING_RESULT.md](https://github.com/mhoshi-vm/oracle-gg-to-rabbitmq/blob/main/TESTING_RESULT.md)

INSERT、UPDATE、DELETEなどといった操作を拾いつつ、順番も守れていることが確認できました。

## おわりに

Oracle GGとRabbitMQが連携できる、を証明できて安心です。