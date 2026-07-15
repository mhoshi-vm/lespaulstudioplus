---
title: "Spring の gRPC + mTLS 認証の証明書の更新を自動認識させる"
date: "2026-07-16T02:00:00+09:00"
tags: ["Spring", "gRPC", "mTLS"]
thumbnail: "img.png"
draft: true
---

Spring の gRPC の認証では、mTLS 認証がおすすめです。
mTLS はTLS証明書をつかいますが、それには期限があり、一定期間ローテートさせる必要があります。
ただ、これを Spring gRPC で認識させようとおもうと、「え、こんな方法しかないの？」状態だったので、備忘として残しました。
<!--more--> 

この記事公開後の未来では、以下の実装が完了しておりこの記事のやり方以外があるのだろうと予想しています。

[Support for gRPC server TLS certificate rotation #49833](https://github.com/spring-projects/spring-boot/issues/49833)
[Support for gRPC client TLS certificate rotation #49831](https://github.com/spring-projects/spring-boot/issues/49831)

## はじめに

マイクロサービス間の連携に gRPC を使うはとてもおしゃれな手段です。
SpringでもSpring gRPCプロジェクトによってそれがサポートされています。

http://spring.io/projects/spring-grpc

そのgRPCの通信を認証させるための一つの手段が mTLS です。

[https://docs.spring.io/spring-grpc/reference/client.html#_mutual_tls](https://docs.spring.io/spring-grpc/reference/client.html#_mutual_tls)

mTLS はTLS証明書をつかいますが、それには期限があり、一定期間ローテートさせる必要があります。
ただ、これを Spring gRPC で認識させようとすると（執筆時点では）意外とむずかしい、というそんな話です。

## コード

コードはここです。

[https://github.com/mhoshi-vm/play-w-gprc-mtls](https://github.com/mhoshi-vm/play-w-gprc-mtls)

## 試していく

### 事前準備

まず、証明書をつくっていきます。

```
bash certificate.sh -x
```

サーバーを起動します。(compileは必須)

```
cd server
./mvnw compile
./mvnw spring-boot:run
```

別プロンプトでクライアントを起動します。(こっちもcompileは必須)

```
cd client
./mvnw compile
./mvnw spring-boot:run
```

その後、以下のコマンドを打ちます。

```
curl localhost:8081
```

client側のプロンプトでいかが出力されれば、gRPC成功

```
message: "Hello ==> hello"
```

### クライアント証明書を更新



## おわりに

DockerでGreenplumで起動できれば、Testcontainersで使えて便利です。