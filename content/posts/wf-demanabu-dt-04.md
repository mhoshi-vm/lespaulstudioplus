---
title: "Wavefrontで学ぶ分散トレーシング Part-4"
date: 2020-08-17T21:30:12+09:00
categories: [Tanzu Observability]
tags: [Wavefront,Tanzu Observability, Distributed Tracing]
thumbnail: "/images/Wavefront-Logo-Square-512x512.png"
---

この文章は、Wavefrontで学ぶ分散トレーシング　シリーズの第四回目です。<!--more-->

# シリーズ

第一回　：　[概要編](../wf-demanabu-dt)  
第二回　：　[Spring Bootで分散トレーシング](../wf-demanabu-dt-02)   
第三回　：　[REDメトリクスって何？](../wf-demanabu-dt-03)  
第四回　：　サービスをつなげてみる**← いまここ**  
第五回　：　[Pythonで分散トレーシング](../wf-demanabu-dt-05)  
第六回　：　[AMQPで分散トレーシング](../wf-demanabu-dt-06)  
第七回　；　[サービスメッシュで分散トレーシング](../wf-demanabu-dt-07)  


# 始めに

[第二回](../wf-demanabu-dt-02)や[第三回](../wf-demanabu-dt-03)で簡単なアプリで分散トレーシングをみました。

今回は２つのサービスをつなげてその時の動作をみたいと思います。
以下を目指します。
![](../../images/wf_demanabu_dt_04/2020-09-16T02-10-12.png)

# 準備編

第二回と同じ[手順](../wf-demanabu-dt-02)でアプリを作ってください。
正しく構成されていれば、以下のURLで"Hello World"もしくは"Good Bye World"が表示されるはずです。

```
curl localhost:8082/hello
```

なお、混乱をさけるため、このアプリは記事では**REDアプリ**と呼称します。
今回はこのREDアプリに加え、もうひとつアプリをつくり、くっつけてみます。

新しく作る方を**HUBアプリ**と呼びます。

# ソースコード

ここに公開しています。

https://github.com/mhoshi-vm/wf-demanabu-dis-tracing/tree/master/4


# HUBアプリの準備

ここからはHUBアプリの作り方です。

(もうほぼテンプレですが)
準備ができたら、以下のURLにアクセスしてください。

[start.spring.io](https://start.spring.io)

![](../../images/wf_demanabu_dt_04/2020-09-16T02-11-59.png)

ログイン後、以下を実施します。

* Add Dependencies を選択
* Spring Webを検索して追加
* Sleuth も同様に追加
* Wavefront も同様に追加

最後にGenerateをクリックします。
zipファイルは現在のアプリと重複しない場所に展開してください。

お気に入りのエディターで以下のファイルを開いてください。

```
mhoshino@mhoshino demo % vi src/main/java/com/example/demo/DemoApplication.java
```

これを以下の内容に差し替えてください。

```
package com.example.demo;

import java.util.Map;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpMethod;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}

@RestController
class HubController{

    private static final Logger LOGGER = LoggerFactory.getLogger(HubController.class);

	@Value("${hub.urls}")
	private String urls;

    @Autowired
    RestTemplate restTemplate;

    @Bean
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

    @GetMapping(value="/hub")
    public String MultiRest(@RequestHeader Map<String, String> header) {
		printAllHeaders(header);

		for( String url : urls.split(",") )
		{
			LOGGER.info(String.format("URLS = %s", url));
			restTemplate.exchange(url, HttpMethod.GET, null, new ParameterizedTypeReference<String>() {}).getBody();
		}
        return "REST Complete";
    }

	private void printAllHeaders(Map<String, String> headers) {
		headers.forEach((key, value) -> {
			LOGGER.info(String.format("Header '%s' = %s", key, value));
		});
	}
}
```

さらに以下のファイルを開きます。

```
mhoshino@mhoshino demo % vi src/main/resources/application.properties
```

そして以下の内容を追記します。

```
management.endpoints.web.exposure.include=wavefront
server.port=8083
wavefront.application.name=demo3
wavefront.application.service=Hub

hub.urls=http://localhost:8082/hello
```

コード編集は以上です。

# サービスをくっつけてみる

さて、アプリを起動します。以下のコマンドをREDアプリとHUBアプリ両方で実行します。（同じコマンドなのでディレクトリーを混乱しないように）

```
./mvnw sprint-boot:run
```

そしてしばらく、以下のURLに対してコマンドを何回か実行します。

```
curl localhost:8083/hub
```

しばらくしたら、WavefrontのURLにログインしてみましょう。

http://localhost:8083/actuator/wavefront

そして、Wavefrontにログイン後、[Applications] > [Application Map(Beta)]を選択します。
![](../../images/wf_demanabu_dt_04/2020-09-16T02-12-24.png)

うまくいってれば、以下のように図がつながっているはずです。

![](../../images/wf_demanabu_dt_04/2020-09-16T02-12-40.png)

[Applications] > [Traces]を開くと、Traceログがみれるはずです。

![](../..images/wf_demanabu_dt_04/2020-09-16T02-12-50.png)


# 何が起きている？

さて、いつもどおり後回しの解説です。
今回作ったHUBアプリは非常に単純に、前回作ったREDアプリへREST APIを送るシンプルなアプリです。
コードでいうと以下の箇所


```
    @GetMapping(value="/hub")
    public String MultiRest(@RequestHeader Map<String, String> header) {
		printAllHeaders(header);

		for( String url : urls.split(",") )
		{
			LOGGER.info(String.format("URLS = %s", url));
			restTemplate.exchange(url, HttpMethod.GET, null, new ParameterizedTypeReference<String>() {}).getBody();
		}
        return "REST Complete";
    }
```

どのURLへ送るかは`application.properties`の`hub.urls`パラメーターで指定します。

```
hub.urls=http://localhost:8082/hello
```

もうすこし詳細な挙動をみてみましょう。

HUBアプリのログを見てみます。
以下のようなログになっているかと思います。

```
2020-07-20 09:18:40.787  INFO [,5f14e2e08374d381d7534b5d89939527,d7534b5d89939527,true] 34562 --- [nio-8083-exec-2]
2020-07-20 09:18:40.787  INFO [,5f14e2e08374d381d7534b5d89939527,d7534b5d89939527,true] 34562 --- [nio-8083-exec-2]
2020-07-20 09:18:40.787  INFO [,5f14e2e08374d381d7534b5d89939527,d7534b5d89939527,true] 34562 --- [nio-8083-exec-2]
2020-07-20 09:18:40.787  INFO [,5f14e2e08374d381d7534b5d89939527,d7534b5d89939527,true] 34562 --- [nio-8083-exec-2]
```

注目するのが`[,5f14e2e08374d381d7534b5d89939527,d7534b5d89939527,true] ` の箇所で、第一回目で説明したよう、`5f14e2e08374d381d7534b5d89939527`がTrace ID
で`d7534b5d89939527`がスパンIDです。

REDアプリのログをみてみます。
以下のようなログになっているかと思います。

```
[hellorest,5f14e2e08374d381d7534b5d89939527,7ce53e353e48c2aa,true] 34007 --- [nio-8082-exec-2]
[hellorest,5f14e2e08374d381d7534b5d89939527,7ce53e353e48c2aa,true] 34007 --- [nio-8082-exec-2]
[hellorest,5f14e2e08374d381d7534b5d89939527,7ce53e353e48c2aa,true] 34007 --- [nio-8082-exec-2]
[hellorest,5f14e2e08374d381d7534b5d89939527,7ce53e353e48c2aa,true] 34007 --- [nio-8082-exec-2]
```

ここで着目すると

* `5f14e2e08374d381d7534b5d89939527`のTrace IDが一致している
* Span ID がREDアプリとHUBアプリの間で異なる

さらに、このログを右の箇所注目するとこのようになっていると思います


```
2020-07-20 09:18:40.789  INFO Header 'accept' = text/plain, application/json, application/*+json, */*
2020-07-20 09:18:40.789  INFO Header 'x-b3-traceid' = 5f14e2e08374d381d7534b5d89939527
2020-07-20 09:18:40.789  INFO Header 'x-b3-spanid' = 067b8ed01abdb759
2020-07-20 09:18:40.789  INFO Header 'x-b3-parentspanid' = d7534b5d89939527
2020-07-20 09:18:40.789  INFO Header 'x-b3-sampled' = 1
2020-07-20 09:18:40.789  INFO Header 'user-agent' = Java/14.0.1
2020-07-20 09:18:40.789  INFO Header 'host' = localhost:8082
2020-07-20 09:18:40.789  INFO Header 'connection' = keep-alive
```

これは、HUBアプリからやってきた、HTTPのヘッダーを出力しています。
ここからわかることが

* x-b3-trace-id にTrace IDが含まれている
* x-b3-parentspanid にHUBアプリのSpan IDが含まれている

## x-b3?

さて、x-b3 ってなんぞや？ですが、これはb3-propagationとよばれるZipkinで定義されているTrace情報を共有するためのHTTPヘッダーです。

詳しくは、[ここ](https://github.com/openzipkin/b3-propagation)を参照ください。

Spring Bootではこのx-b3のアップストームからの展開とダウンストリームへの挿入をSluethがほぼコードからは透過的に実施します。
以下の③の箇所で定義したときのものです。

![](../../images/wf_demanabu_dt_04/2020-09-16T02-13-10.png)

# まとめ

* サービス同士をくっつけるには、Spring Bootでは単純にREST APIを実行すればよい
* サービス同士はTrace ID、Span IDをHTTPの"x-b3-*"ヘッダーでやり取りしている
* "x-b3-*"ヘッダーは、Spring BootではSluethが勝手にやってくれるのでうれしい

さて、次回はこのx-b3-ヘッダーの取り扱いがどれだけややこしいか、pythonをつかって説明したいと思います。
次回は「[Pythonで分散トレーシング](../wf-demanabu-dt-05)」です。
