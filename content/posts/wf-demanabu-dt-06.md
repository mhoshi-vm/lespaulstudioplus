---
title: "Wavefrontで学ぶ分散トレーシング Part-6"
date: 2020-08-19T21:30:12+09:00
categories: [Tanzu Observability]
tags: [Wavefront,Tanzu Observability, Distributed Tracing]
thumbnail: "images/Wavefront-Logo-Square-512x512.png"
---

この文章は、Wavefrontで学ぶ分散トレーシング　シリーズの第六回目です。<!--more-->

# シリーズ

第一回　：　[概要編](../wf-demanabu-dt)  
第二回　：　[Spring Bootで分散トレーシング](../wf-demanabu-dt-02)   
第三回　：　[REDメトリクスって何？](../wf-demanabu-dt-03)  
第四回　：　[サービスをつなげてみる](../wf-demanabu-dt-04)  
第五回　：　[Pythonで分散トレーシング](../wf-demanabu-dt-05)  
第六回　：　AMQPで分散トレーシング**← いまここ**   
第七回　；　[サービスメッシュで分散トレーシング](../wf-demanabu-dt-07)  


# 始めに

過去の回では、Spring BootおよびPythonを使った分散トレーシングを紹介しました。
もう一度、それらの復習をすると

 * 分散トレーシングのキモはTrace IDとSpan ID
 * HTTPヘッダーをもとにTrace IDとSpan IDをサービス間で共有することでサービスがつながる
 * アプリケーション側では、HTTPヘッダーからTrace IDとSpan IDを展開するようなコーディングを含まないといけない

さて、この中で2点目ですが、いままでの検証ではずっと「HTTPリクエスト」をベースにやってきました。
そこで、すこし疑問に思うかもしれないのが「HTTP以外のリクエスト」でも、これがうまくいくのかという点です。

そこで今回は、RabbitMQを使い、AMQPでも分散トレーシングを行えるか検証します。

# 準備

今回はRabbitMQを使いますので、PC上にインストールします。
手順はOSによって、違うので以下をもとにインストールおよび起動をしてください。

https://www.rabbitmq.com/download.html

今回はSpring Bootのアプリを使って、RabbitMQにキューを仕込むおよび取り出すコードを書いていきます。
いままで同様、これには最低限以下をインストールしてください。

* Java JDK 8+

[Oracle JDK](https://www.oracle.com/java/technologies/javase-downloads.html)に従いJDKをインストールしてください

# ソースコード

ここに公開しています。

https://github.com/mhoshi-vm/wf-demanabu-dis-tracing/tree/master/6

# アプリの準備

必要なアプリは２つです、ProducerアプリとConsumerアプリです。

## Producerアプリ

さて、第六回にもなったので、いままで通り、[start.spring.io](https://start.spring.io)からダウンロードするのではなく、CLIをつかって直接必要なパッケージをダウンロードします。

```
curl https://start.spring.io/starter.tgz \
       -d artifactId=producer \
       -d baseDir=producer \
       -d dependencies=web,amqp,cloud-starter-sleuth,wavefront \
       -d packageName=com.example \
       -d applicationName=ProducerApplication | tar -xzvf -
```

以下のように出力されれば成功です。

```
x producer/.mvn/
x producer/.mvn/wrapper/
x producer/.mvn/wrapper/maven-wrapper.properties
x producer/.mvn/wrapper/MavenWrapperDownloader.java
x producer/.mvn/wrapper/maven-wrapper.jar
x producer/.gitignore
x producer/HELP.md
x producer/mvnw.cmd
x producer/src/
x producer/src/main/
x producer/src/main/resources/
x producer/src/main/resources/templates/
x producer/src/main/resources/application.properties
x producer/src/main/resources/static/
x producer/src/main/java/
x producer/src/main/java/com/
x producer/src/main/java/com/example/
x producer/src/main/java/com/example/producer.java
x producer/src/test/
x producer/src/test/java/
x producer/src/test/java/com/
x producer/src/test/java/com/example/
x producer/src/test/java/com/example/producerTests.java
x producer/pom.xml
```

これに二つのコードを追加します。
まず`producer/src/main/java/com/example/ProducerRestController.java`というファイルを作り以下の内容にします。

```
package com.example;

import java.util.Map;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProducerRestController {

        private static final Logger LOGGER = LoggerFactory.getLogger(ProducerRestController.class);

        @Autowired
        Producer producer;

        @GetMapping("/amqp")
        public ResponseEntity<String> hello (@RequestHeader Map<String, String> header){
                printAllHeaders(header);
            producer.send();
            return ResponseEntity.ok("Hello World!");
        }

        private void printAllHeaders(Map<String, String> headers) {
                headers.forEach((key, value) -> {
                        LOGGER.info(String.format("Header '%s' = %s", key, value));
                });
        }
}
```

さらに`producer/src/main/java/com/example/Producer.java`を作ります。

```
package com.example;

import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

@Component
public class Producer{

    @Bean
    public Queue hello() {
        return new Queue("hello");
    }

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private Queue queue;

    public void send() {
        String message = "Hello World!";
        this.template.convertAndSend(queue.getName(), message);
        System.out.println(" [x] Sent '" + message + "'");
    }
}
```

最後に`producer/src/main/resources/application.properties`を以下の内容にアップデートします。

```
spring.application.name=producer

management.endpoints.web.exposure.include=wavefront
server.port=8086

wavefront.application.name=demo6
wavefront.application.service=producer

spring.rabbitmq.host=localhost
```


## Consumerアプリ

もうひとつConsumerアプリを作ります。
まず以下のCurlコマンドを実行します。

```
curl https://start.spring.io/starter.tgz \
       -d artifactId=consumer \
       -d baseDir=consumer \
       -d dependencies=web,amqp,cloud-starter-sleuth,wavefront \
       -d packageName=com.example \
       -d applicationName=ConsumerApplication | tar -xzvf -
```

これに`consumer/src/main/java/com/example/Consumer.java`を作ります。

```
package com.example;

import java.util.Map;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.messaging.handler.annotation.Headers;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
@RabbitListener(queues = "hello")
public class Consumer {

    private static final Logger LOGGER = LoggerFactory.getLogger(Consumer.class);

    @RabbitHandler
    public void receive(@Payload String body, @Headers Map<String,Object> headers) {
		LOGGER.info(String.format(" [x] Received '" + body + headers + "'"));
    }

}
```

最後に`consumer/src/main/resources/application.properties`を以下の内容にアップデートします。

```
spring.application.name=consumer

management.endpoints.web.exposure.include=wavefront
server.port=18086

wavefront.application.name=demo6
wavefront.application.service=consumer

spring.rabbitmq.host=localhost
```
# アプリの起動

準備ができたらアプリを起動していきます。
まず前提としてRabbitMQの起動が必要です。
これはOSによって異なりますが、私はMacなので以下で起動できます。

```
/usr/local/sbin/rabbitmq-server
```

次にProducerアプリを起動します。

```
cd producer
./mvnw spring-boot:run
```

さらに、Consumerアプリを起動します。

```
cd ..
cd consumer
./mvnw spring-boot:run
```

両方起動したら、以下のURLにcurlを流します。

```
curl localhost:8086/amqp
```

うまくいけば、` [x] Sent 'Hello World!'`と表示されるはずです。
しばらく何回か`curl`を実行して下さい。

そしてしばらくしたら以下のURLをアクセスします。

http://localhost:8086/actuator/wavefront

そして Applications > Application Map(Beta)を選択してください。

うまくいけば、以下のように繋がるはずです。

![](../../images/wf_demanabu_dt_06/2020-09-16T02-22-38.png)

つまり、なんとAMQPでも分散トレーシングができていることになります。
Applications > Tracesをみると、ちゃんとTrace IDが一致した状態で、オブジェクトがつながっているのがわかります。

![](../../images/wf_demanabu_dt_06/2020-09-16T02-22-47.png)


# なにが起きている？

さて、一体何がなんやらと思うので、説明します。
まず、状況をわかりやすくするため、Consumerアプリをとめてください。(Ctrl + c)

Producerアプリを起動したまま、以下を三回程度実行します。

```
curl localhost:8086/amqp
```

この状態で、RabbitMQの管理コンソールにログインします。

http://localhost:15672/#/　
ユーザー・パスワードはguest/guestです。

ログイン後、Queueタブをみると、helloというQueueにメッセージがcurlを行った回数分溜まっていることがわかります。

![](../../images/wf_demanabu_dt_06/2020-09-16T02-23-01.png)

つまりProducerアプリは、curlがくるたびに`hello`キューにメッセージを貯める非常にシンプルなアプリです。
対しConsumerアプリは、この`hello`キューのメッセージを読み取るようなアプリケーションです。

## x-b3ヘッダーは？

前回から繰り返しているよう、x-b3ヘッダーがTrace IDなどのやりとりをおこなう仕組みでした。
AMQPはこれはどうやっているのか？

先ほどの管理コンソールから、`hello` キューを選択して、`Get Messages`を選択します。
するとメッセージの中に`b3`ヘッダーが表示されます。

![](../../images/wf_demanabu_dt_06/2020-09-16T02-23-15.png)

これはつまり、AMQPでもヘッダーをつかって、Trace IDのやりとりが行われていることがわかります。
なお、これもまた`Sleuth`が透過的にSpring Bootの場合に行ってくれるおかげです。

https://docs.spring.io/spring-cloud-sleuth/docs/current-SNAPSHOT/reference/html/#sleuth-with-zipkin-over-rabbitmq-or-kafka

# まとめ

* AMQPでも分散トレーシングはできる
* AMQPでもヘッダーをつかってTrace IDなどをやりとりしている
* Spring Bootだと相変わらず、コードから透過的に分散トレーシングしてくれる

さて、次回はいよいよ「[サービスメッシュで分散トレーシング](../wf-demanabu-dt-07)」です。
