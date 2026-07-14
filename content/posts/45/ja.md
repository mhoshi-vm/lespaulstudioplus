---
title: "Greenplum の Dockerfile を作り、Testcontainersから起動"
date: "2026-07-16T00:00:00+09:00"
tags: ["Greenplum", "Testcontainers", "Spring"]
thumbnail: "img.png"
---

Greenplum をベースのアプリケーション開発にはPostgresなどで代替することが多かったです。
ただ、Greenplumにしかないクエリとかを試すのに不便でした。
そこでGreenplumをコンテナで起動できるDockerfileを開発しました。
おまけでTestcontainersでSpringと連携したよ。
<!--more--> 

## はじめに

GreenplumはPostgresをベースとしたRDBです。
Postgresで開発したアプリをGreenplumで動かす、は大抵OKなのですが、Greenplumにしかないクエリをベースに開発したい時に困ります。

いままでは、テスト環境を作り、そこで検証、といった手順がメインでしたが、コンテナでできないものか？ということで開発しました。

## コード

ここにあります。READMEに細かい手順も書いているので、そちらを参照。
PostGIS, MADLib などの拡張やPXFなどの機能をいれています。

[https://github.com/mhoshi-vm/greenplum-learn/tree/main/docker/gp78](https://github.com/mhoshi-vm/greenplum-learn/tree/main/docker/gp78)

なお、このDockerfileはあくまで検証向けであり、それを使って本番とかでは使わないでください。

## SpringのTestcontainersで試す

Docker イメージがあれば、Springのテストコンテナから呼び出すことも可能です。

pom.xmlにはこの依存関係が必要です。

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-testcontainers</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers-postgresql</artifactId>
  <scope>test</scope>
</dependency>
```

こんな感じのコードを入れます。

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.postgresql.PostgreSQLContainer;
import org.testcontainers.utility.DockerImageName;

@TestConfiguration(proxyBeanMethods = false)
class GreenplumTestcontainersConfiguration {

	@Bean
	@ServiceConnection
	PostgreSQLContainer postgresContainer() {
		return new PostgreSQLContainer(DockerImageName.parse("gp7").asCompatibleSubstituteFor("postgres"))
			.waitingFor(Wait.forLogMessage("^.*statement: GRANT ALL PRIVILEGES ON DATABASE.*\\n", 1))
			.withReuse(true);

	}

}
```

## おわりに

DockerでGreenplumで起動できれば、Testcontainersで使えて便利です。