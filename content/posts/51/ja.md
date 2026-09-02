---
title: "Spring Enterprise : Application Advisor 解説 (Upgrade Plan 編)"
date: "2026-09-01T19:00:00+09:00"
tags: ["Application Advisor", "Spring", "Spring Enterprise"]
thumbnail: img.png
---
Spring Enterprise についてくる Application Advisor のマニュアルを補足する形で紹介します。
[前回](../50) の `patch apply` に続いて、このエントリでは `upgrade-plan apply` の機能に絞って紹介します。
<!--more-->

## はじめに

前回も書いたとおり、Application Advisor は大きく二つのパッチ機能を提供します。

- `advisor patch apply`:
ホットフィックスおよびパッチバージョン専用に使用
- `advisor upgrade-plan apply`:
マイナーバージョンおよびメジャーバージョンのアップグレードに使用

今回は後者です。マニュアルは [こちら(v1.6)](https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/application-advisor/1-6.html) です。

![img.png](img.png)

`patch apply` が「3.1.4 → 3.1.5」のようなパッチバージョンだけを見るのに対して、
`upgrade-plan apply` は「Spring Boot 2.7 → 3.5」のような世代をまたぐアップグレードを担当します。

なお、ここでいう、パッチやマイナーアップグレードは、Java の各依存関係の [Semantic Versioning](https://semver.org/) を想定しています。
つまり、以下のような例で考えます。

- 3.1.4 → 3.1.5 : パッチ適用
- 3.1.4 → 3.2.2 : マイナーアップグレード
- 3.1.4 → 4.0.3 : メジャーアップグレード

パッチ適用の作りの前提としては、ユーザーコードには影響がないものと想定しています。
そのため、`patch apply` は特にユーザーコードは弄らず、依存関係の更新を行うものでした。
ところが、メジャー・マイナーアップグレードはユーザーコードへの影響がありうるため、`patch apply` とは違う作りが必要になってきます。

## レシピとマッピング座標とは？

`upgrade-plan apply` コマンドには、`patch apply` にはない概念が登場します。
それが、`レシピ(Recipe)` と `マッピング座標(Mapping Coordinates)` です。

以下の概念を持ちます。

- `レシピ(Recipe)`：メジャー・マイナーアップグレード間で発生する非互換なコードを修正するため、OpenRewrite フレームワークをベースに書かれたコード
- `マッピング座標(Mapping Coordinates)` : 各依存関係のバージョンの対応表の役割をしており、`step` の回数を求めるのに使われます。

Application Advisor の CLI には、レシピとマッピング座標が同梱されています。
マニュアルの以下にその詳細が記載されています。

- [Application Advisor 1.6 のレシピ一覧](https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/application-advisor/1-6/app-advisor/recipes-index-index-boot-recipes.html)
- [Application Advisor v1.6.7 のマッピング座標](https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/application-advisor/1-6/app-advisor/coverage-1-6-7.html)

シンプルなアップグレードのケースでは、以下のようになります。

![img_1.png](img_1.png)

Application Advisor はマッピング座標を使い、Step と呼ばれる実行の単位を算出します。
この実行の単位のたびに `upgrade-plan` はレシピを実行して正常終了します。上の絵の例では、2 Step の実行です。
（注意：実際に 2.7 からアップグレードすると、ステップ数はもっと多くなります。）
マニュアルでは、[Spring Petclinic を例にこれが取り上げられています](https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/application-advisor/1-6/app-advisor/how-to-guides-upgrade-boot.html)。

Application Advisor は、このレシピとマッピング座標をカスタマイズすることが可能です。
これをすることで、以下のようなアップグレードもサポートが可能になります。

![img_2.png](img_2.png)

この絵の例では、ある Spring を同梱したカスタムフレームワークを想定しています。
カスタマイズしたマッピング座標をもとに、そのカスタムフレームワークのバージョンごとの Spring Boot の対応を知ります。
これをすることで、アップグレードまでの必要なステップが再計算されます。この絵で言えば、1 Step 増えます。
また、そのステップごとに独自のレシピを実行することもできます。

なお、レシピは [OpenRewrite](https://docs.openrewrite.org/) をもとに記述します。
デフォルトに含まれるレシピを使った軽微な修正から、独自のレシピを Java で書き上げる方法まで、さまざまなやり方をサポートしています。

これをすることで、標準の Spring Boot だけでなく、カスタムのパッケージのアップグレードを行うことが可能です。
また、ここまで Spring Boot という前提で書いていますが、Spring Framework や他の Java フレームワークでも利用することができます。

さて、いよいよ、ここからカスタムのレシピとマッピング座標について解説していきます。
ただし、どうしても最初は難易度が高いものです。実際に行う場合は、Broadcom 社からなんらかのサポートがある体制で臨むことをおすすめします。

# TERASOLUNA の自動アップグレードに挑戦

今回は前の手順でのアップグレードを実践するために、日本国産フレームワークの [TERASOLUNA](https://www.nttdata.com/jp/ja/lineup/terasoluna/) をアップグレードできるかやってみます。

## TERASOLUNA とバージョンと Spring Framework の対応関係

TERASOLUNA と Spring Framework は以下の対応関係（私調べ）を持っています。

| TERASOLUNA | spring-framework | spring-data-commons | spring-security |
| --- | --- | --- | --- |
| `5.7.x` | 5.3.39 | 2.7.18 | 5.7.13 |
| `5.8.x` | 6.0.3 | 3.0.0 | 6.0.1 |
| `5.9.x` | 6.1.3 | 3.2.2 | 6.2.1 |
| `5.10.x` | 6.2.15 | 3.5.7 | 6.5.7 |
| `5.11.x` | 7.0.3 | 4.0.2 | 7.0.2 |

見ての通り、5.7 → 5.8 が Spring Framework 5系から6系へと大きな変更が伴っています。
これを Application Advisor でアップグレード可能かやってみます。

## サンプルコード

ここに配置しました。

https://github.com/mhoshi-vm/simple-terasoluna

## カスタマイズなしで upgrade-plan 実行

まず、カスタマイズなしで upgrade-plan を実行します。

### 1. CLI のダウンロード

[前回](../50) と同じです。マニュアルの手順、または Spring Enterprise Repository を構成していれば以下でも取れます。
(以下は Mac ARM64用の 1.6.7 をダウンロードする )

```
mvn -U dependency:get -Dartifact=com.vmware.tanzu.spring:application-advisor-cli-macos-arm64:1.6.7:tar -Dtransitive=false
```

### 2. プランの確認

ソースコードのルートディレクトリに移動して
`upgrade-plan` を実行します。

```
advisor upgrade-plan get
```

執筆時点の最新 1.6.7 では以下のような出力になりました。

```bash
advisor upgrade-plan get
     
🚀 Existing build-configuration is already up-to-date


🏃 Fetching and processing upgrade plan details [00m 01s] ok

Projects discovered:
        - junit: 6.0.x (no upgrades available)
        - junit-platform: 6.0.x (no upgrades available)

The projects ["spring-framework", "spring-data-commons"] could not be included in the Upgrade Plan because they are used as transitive dependencies for other projects, and no upgrades are configured for them.
Please request your administrator to configure the projects of the following dependencies:

        - org.terasoluna.gfw:terasoluna-gfw-common
                uses:
                        - spring-framework
                        - spring-data-commons
                blocking upgrades for:
                        - spring-data-commons
        - org.terasoluna.gfw:terasoluna-gfw-web
                uses:
                        - spring-framework
                        - spring-data-commons
                blocking upgrades for:
                        - spring-data-commons
        - org.terasoluna.gfw:terasoluna-gfw-jodatime
                uses:
                        - spring-framework
                        - spring-data-commons
                blocking upgrades for:
                        - spring-data-commons

In order to learn more about publishing upgrade mappings, visit https://techdocs.broadcom.com/us/en/vmware-tanzu/spring/tanzu-spring/commercial/spring-tanzu/app-advisor-custom-upgrades.html

No upgrade plans available - your project seems to be up to date.
```

出力の `could not be included in the Upgrade Plan because they are used as transitive dependencies for other projects, and no upgrades are configured for them.` は
このコンテキストにあわせると、以下のような意味となります。

- ["spring-framework", "spring-data-commons"] は TERASOLUNA(for other projects 部分)の推移的依存関係(Transitive Dependency)
- これら推移的依存関係と TERASOLUNA の対応関係がわからない
- よって、アップグレードができない

この出力は、TERASOLUNA の関連ライブラリが更新できないことを示唆しています。

```
        - org.terasoluna.gfw:terasoluna-gfw-common
                uses:
                        - spring-framework
                        - spring-data-commons
                blocking upgrades for:
                        - spring-data-commons
        - org.terasoluna.gfw:terasoluna-gfw-web
                uses:
                        - spring-framework
                        - spring-data-commons
                blocking upgrades for:
                        - spring-data-commons
        - org.terasoluna.gfw:terasoluna-gfw-jodatime
                uses:
                        - spring-framework
                        - spring-data-commons
                blocking upgrades for:
                        - spring-data-commons
```

つまり、エラーになって何もアップグレードできなかったというものです。
特にマッピングは指定していないので、ここまでは想定通りです。

## カスタマイズのマッピングで upgrade-plan 実行

以下のカスタムのマッピングを任意のディレクトリにつくります。
ファイル名も任意ですが、`terasoluna.json` にします。

```json
{
  "slug": "terasoluna-gfw",
  "coordinates": [
    "org.terasoluna.gfw:terasoluna-gfw-common",
    "org.terasoluna.gfw:terasoluna-gfw-web",
    "org.terasoluna.gfw:terasoluna-gfw-jodatime"
  ],
  "repositoryUrl": "https://github.com/terasolunaorg/terasoluna-gfw",
  "rewrite": {
    "5.7.x": {
      "recipes": [],
      "nextRewrite": {
        "version": "5.8.x"
      },
      "requirements": {
        "supportedGenerations": {
          "spring-data-commons": "2.7.x",
          "spring-framework": "5.3.x",
          "spring-security": "5.7.x"
        },
        "excludedArtifacts": []
      }
    },
    "5.8.x": {
      "recipes": [],
      "nextRewrite": {
      },
      "requirements": {
        "supportedGenerations": {
          "spring-data-commons": "3.0.x",
          "spring-framework": "6.0.x",
          "spring-security": "6.0.x"
        },
        "excludedArtifacts": []
      }
    }
  }
}
```

中身としては、以下がポイントになります。

- 元のコードが解決できなかったマッピングを `coordinates` に記載
- `rewrite` に依存関係のバージョンを、`requirements` に依存する他の Spring ライブラリを記載
- `nextRewrite` に次のバージョンを記載

なお、本来は TERASOLUNA 5.9.x 以降も記載すべきですが、今回は省略しています。

そして以下の環境変数を export します。

```bash
export SPRING_ADVISOR_MAPPING_CUSTOM_0_FILEPATH=<配置ディレクトリ>/terasoluna.json
```

この状態で Advisor CLI v1.6.7 を実行すると、以下のようになります。

```bash
advisor upgrade-plan get
     
🚀 Existing build-configuration is already up-to-date


🏃 [ 1 / 2 ] Validating syntax of upgrade mappings [00m 01s] ok
🏃 [ 2 / 2 ] Fetching and processing upgrade plan details [00m 01s] ok

Projects discovered:
        - junit: 6.0.x (no upgrades available)
        - junit-platform: 6.0.x (no upgrades available)
        - terasoluna-gfw: 5.7.x → 5.8.x
        - spring-data-commons: 2.7.x → 4.1.x
        - spring-framework: 5.3.x → 7.0.x

Upgrade Plan for your Dependencies:
        - Step 1:
                * Upgrade terasoluna-gfw from 5.7.x to 5.8.x
                * Upgrade spring-data-commons from 2.7.x to 3.0.x
                * Upgrade spring-framework from 5.3.x to 6.0.x
        - Step 2:
                * Upgrade spring-framework from 6.0.x to 6.1.x
                * Upgrade spring-data-commons from 3.0.x to 3.2.x
        - Step 3:
                * Upgrade spring-framework from 6.1.x to 6.2.x
                * Upgrade spring-data-commons from 3.2.x to 3.5.x

Some upgrades were not included in the upgrade plan.
Please, upgrade and release if needed the following projects:
        * terasoluna-gfw:5.8.x
                Last version of spring-framework is 6.0.x
        * terasoluna-gfw:5.8.x
                Last version of spring-data-commons is 3.0.x
```

以下がポイントです。

```diff
Upgrade Plan for your Dependencies:
        - Step 1:
+               * Upgrade terasoluna-gfw from 5.7.x to 5.8.x
                * Upgrade spring-data-commons from 2.7.x to 3.0.x
                * Upgrade spring-framework from 5.3.x to 6.0.x
        - Step 2:
                * Upgrade spring-framework from 6.0.x to 6.1.x
                * Upgrade spring-data-commons from 3.0.x to 3.2.x
        - Step 3:
                * Upgrade spring-framework from 6.1.x to 6.2.x
                * Upgrade spring-data-commons from 3.2.x to 3.5.x
```

この出力にもあるように、マッピング座標ファイルにより、Step が定義できています。
特に `Upgrade terasoluna-gfw from 5.7.x to 5.8.x` とあるように、カスタムライブラリの更新もマッピング座標ファイルによって定義されました。
なお、以下の出力が出ているのは、マッピング座標ファイルに TERASOLUNA 5.9.x 以降を記載していないためです。

```
Some upgrades were not included in the upgrade plan.
Please, upgrade and release if needed the following projects:
        * terasoluna-gfw:5.8.x
                Last version of spring-framework is 6.0.x
        * terasoluna-gfw:5.8.x
                Last version of spring-data-commons is 3.0.x
```

この状態で、`apply` を実行して実際の変更をかけます。

```bash
advisor upgrade-plan apply
```

Advisor CLI 1.6.7 で実行すると以下のような出力になりました。

```bash
🚀 Existing build-configuration is already up-to-date


🏃 [ 1 / 4 ] Validating syntax of upgrade mappings [00m 01s] ok
🏃 [ 2 / 4 ] Validating the license of rewrite artifacts [00m 13s] ok
🏃 [ 3 / 4 ] Fetching and processing upgrade plan details [00m 01s] ok

Projects to upgrade:
        * terasoluna-gfw from 5.7.x to 5.8.x
        * spring-data-commons from 2.7.x to 3.0.x
        * spring-framework from 5.3.x to 6.0.x

🔨 [ 4 / 4 ] Upgrading sources... [00m 18s] ok

👍 Successfully applied upgrade.


⚠️  Warnings:

* Application Advisor might produce a partial upgrade and will incrementally cover all the required changes to upgrade all the Spring projects. If you have questions or are experimenting issues upgrading your applications, please request our help or support in https://support.broadcom.com
```

実際の変更を `git diff` でみると以下が見えます。

まず、TERASOLUNA のバージョンが上がっています。
あわせて、Spring Framework 6 系が Java 8 のサポートを廃止したことにより、Javax > Jakarta ドメインの変更と最新化も行えています。

```diff
diff --git a/pom.xml b/pom.xml
index c170406..1d7b1e3 100644
--- a/pom.xml
+++ b/pom.xml
@@ -14,7 +14,7 @@
   <properties>
     <maven.compiler.release>17</maven.compiler.release>
     <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
-    <terasoluna.version>5.7.4.RELEASE</terasoluna.version>
+    <terasoluna.version>5.8.1.RELEASE</terasoluna.version>
   </properties>
 
   <dependencyManagement>
@@ -60,8 +60,8 @@
     </dependency>
 
     <dependency>
-      <groupId>javax.servlet</groupId>
-      <artifactId>javax.servlet-api</artifactId>
+      <groupId>jakarta.servlet</groupId>
+      <artifactId>jakarta.servlet-api</artifactId>
       <scope>provided</scope>
     </dependency>

```

さらに、Spring Framework 6 から URL 末尾の "/" が既定でマッチしなくなったため、その補助もやっています。
（これは Advisor CLI の同梱レシピによるものです。）

```diff
diff --git a/src/main/java/com/example/demo/GreetingController.java b/src/main/java/com/example/demo/GreetingController.java
index b7d903f..09f6305 100644
--- a/src/main/java/com/example/demo/GreetingController.java
+++ b/src/main/java/com/example/demo/GreetingController.java
@@ -13,7 +13,7 @@ public class GreetingController {
         this.service = service;
     }
 
-    @GetMapping("/greet")
+    @GetMapping({"/greet", "/greet/"})
     public String greet(@RequestParam(defaultValue = "world") String who) {
         return service.greet(who);
     }
```

過去には、Spring Framework で JavaConfig ではなく XML による Bean 定義が主流だった時代がありましたが、それに関しても変更ができています。

```diff
diff --git a/src/main/resources/applicationContext.xml b/src/main/resources/applicationContext.xml
index a527888..f799a1e 100644
--- a/src/main/resources/applicationContext.xml
+++ b/src/main/resources/applicationContext.xml
@@ -5,11 +5,11 @@
        xmlns:mvc="http://www.springframework.org/schema/mvc"
        xsi:schemaLocation="
          http://www.springframework.org/schema/beans
-         http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
+         https://www.springframework.org/schema/beans/spring-beans.xsd
          http://www.springframework.org/schema/context
-         http://www.springframework.org/schema/context/spring-context-4.3.xsd
+         https://www.springframework.org/schema/context/spring-context.xsd
          http://www.springframework.org/schema/mvc
-         http://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd">
+         https://www.springframework.org/schema/mvc/spring-mvc.xsd">
 
   <!-- The whole application context. TERASOLUNA blank projects are wired in
        XML, not with @Configuration classes. -->
diff --git a/src/main/webapp/WEB-INF/web.xml b/src/main/webapp/WEB-INF/web.xml
index 21d02bd..82263d6 100644
--- a/src/main/webapp/WEB-INF/web.xml
+++ b/src/main/webapp/WEB-INF/web.xml
@@ -1,9 +1,8 @@
 <?xml version="1.0" encoding="UTF-8"?>
-<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
+<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
-         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
-                             http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
-         version="4.0">
+         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
+         version="6.0">
 
   <servlet>
     <servlet-name>dispatcher</servlet-name>
```


なお、さらに、`sql-error-codes.xml` と `src/main/resources/META-INF/spring.factories` というファイルができています。
これらは、Spring Framework 5の動作維持のために作られます。この記事の本質ではないので、ぜひ自分で試してみてください。

## カスタマイズのレシピで upgrade-plan 実行

ここではさらにカスタマイズのレシピを入れて実行してみます。
ここでは、TERASOLUNA のガイドにある[2.15. [Step 15] Joda Time から JSR-310 への移行](https://github.com/terasolunaorg/terasoluna-gfw/wiki/Migration-Guide-5.8.1_ja#step-15-joda-time-%E3%81%8B%E3%82%89-jsr-310-%E3%81%B8%E3%81%AE%E7%A7%BB%E8%A1%8C)を実装してみます。

まず、先ほどの変更を一旦リセットします。

```bash
git reset --hard HEAD
```

### マッピングファイルにレシピを登録

以下のマッピングファイルにレシピを登録します。
```diff
{
  "slug": "terasoluna-gfw",
  "coordinates": [
    "org.terasoluna.gfw:terasoluna-gfw-common",
    "org.terasoluna.gfw:terasoluna-gfw-web"
  ],
  "repositoryUrl": "https://github.com/terasolunaorg/terasoluna-gfw",
  "rewrite": {
    "5.7.x": {
      "recipes": [
+       {
+         "name": "org.openrewrite.java.dependencies.UpgradeDependencyVersion",
+         "params": {
+           "groupId": "org.terasoluna.gfw",
+           "artifactId": "*",
+           "newVersion": "5.8.x"
+         }
+       },
+       {
+         "name": "org.openrewrite.java.ChangeType",
+         "params": {
+           "oldFullyQualifiedTypeName": "org.terasoluna.gfw.common.date.jodatime.JodaTimeDateFactory",
+           "newFullyQualifiedTypeName": "org.terasoluna.gfw.common.date.ClassicDateFactory"
+         }
+       },
+       {
+         "name": "org.openrewrite.xml.ChangeTagAttribute",
+         "params": {
+            "elementName": "bean",
+            "attributeName": "class",
+            "oldValue": "org.terasoluna.gfw.common.date.jodatime.DefaultJodaTimeDateFactory",
+            "newValue": "org.terasoluna.gfw.common.date.DefaultClassicDateFactory"
+          }
+        },
+        {
+          "name": "org.openrewrite.java.dependencies.RemoveDependency",
+          "params": {
+            "groupId": "org.terasoluna.gfw",
+            "artifactId": "terasoluna-gfw-jodatime"
+          }
+        }
      ],
      "nextRewrite": {
        "version": "5.8.x"
      },
      "requirements": {
        "supportedGenerations": {
          "spring-data-commons": "2.7.x",
          "spring-framework": "5.3.x",
          "spring-security": "5.7.x"
        },
        "excludedArtifacts": []
      }
    },
    "5.8.x": {
      "recipes": [],
      "nextRewrite": {
      },
      "requirements": {
        "supportedGenerations": {
          "spring-data-commons": "3.0.x",
          "spring-framework": "6.0.x",
          "spring-security": "6.0.x"
        },
        "excludedArtifacts": []
      }
    }
  }
}
```

一つ前のマッピング座標ファイルをもとに、`5.7.x` の `recipes` を追加しています。
ここでは TERASOLUNA の移行ガイドにあるように、依存関係の差し替えをレシピによって表現しています。

再度実行します。

```
export SPRING_ADVISOR_MAPPING_CUSTOM_0_FILEPATH=.advisor/mappings/terasoluna.json
advisor upgrade-plan apply  
```

実行結果は省略して、`git diff` をみてみます。
すると先ほどの結果に加え以下が実行されており、レシピが実行されたことがわかります。

```diff
-    <!-- Joda-Time support. TERASOLUNA drops this at 5.8: the migration guide's
-         [Step 15] replaces it with the JSR-310 ClockFactory. -->
     <dependency>
-      <groupId>org.terasoluna.gfw</groupId>
-      <artifactId>terasoluna-gfw-jodatime</artifactId>
-      <version>${terasoluna.version}</version>
-    </dependency>
```

コードも書き変わっています。

```diff
diff --git a/src/main/java/com/example/demo/GreetingService.java b/src/main/java/com/example/demo/GreetingService.java
index b9c704e..0cc8b2d 100644
--- a/src/main/java/com/example/demo/GreetingService.java
+++ b/src/main/java/com/example/demo/GreetingService.java
@@ -2,7 +2,7 @@ package com.example.demo;
 
 import java.util.Date;
 import org.springframework.stereotype.Service;
-import org.terasoluna.gfw.common.date.jodatime.JodaTimeDateFactory;
+import org.terasoluna.gfw.common.date.ClassicDateFactory;
 
 /**
  * TERASOLUNA's guideline marks service classes with Spring's @Service
@@ -14,9 +14,9 @@ import org.terasoluna.gfw.common.date.jodatime.JodaTimeDateFactory;
 @Service
 public class GreetingService {
 
 @Service
 public class GreetingService {
 
-    private final JodaTimeDateFactory dateFactory;
+    private final ClassicDateFactory dateFactory;
 
-    public GreetingService(JodaTimeDateFactory dateFactory) {
+    public GreetingService(ClassicDateFactory dateFactory) {
         this.dateFactory = dateFactory;
     }
```

これで必要なアップグレードは自動化できました。
簡単なテストも入れていますが、`./mvnw test` を実行すると問題なく完了します。
**（注意：ここではかなり単純な Joda Time の移行を行っていますが、実際にはガイドに従い、より綿密な計画が必要と思われます。）**
このように、Advisor CLI が知らない TERASOLUNA のアップグレードを行えました。

ここまで行った全ての変更は以下でみられます。

https://github.com/mhoshi-vm/simple-terasoluna/pull/1/changes

## upgrade-plan を本番で使うためには？

その他、これを本番で使っていくためのコツを書いていきます。

### レシピとマッピング座標を管理するの面倒なのだけど・・・

Broadcom としては、可能な限り、レシピとマッピング座標はユーザーが作らずともデフォルトで動くように努めています。
まずは、Broadcom に対して、サポートチケットをあげることをお勧めします。
ある程度、理にかなっていれば、レシピとマッピングを以後のリリースに同梱することは可能です。

今回紹介しているカスタムのレシピとマッピングはあくまで、世に知られていないプライベートな依存関係や、一時的なワークアラウンドの目的で作るものです。

### マッピング座標を作るの面倒なのだけど・・・

マッピング座標ファイルは以下のコマンドで自動で作れます。

```
advisor mapping create -c='org.terasoluna.gfw:terasoluna-gfw-common' < /dev/null
```

`-c` に渡すのはMaven 座標（`groupId:artifactId`）です。

なお、このコマンドは、実行した場所から Maven リポジトリを解決して、座標ファイルを生成します。
以下は実行前に知っておくとよい点です。

- 全てのバージョンに対して座標を作ろうとするので、1プロジェクトで10分近くかかることがある
- バージョン間のレシピは空の状態で提供される
- **標準入力を待つ**ので、非対話（CI やバックグラウンド実行）では `< /dev/null` を付けないと無言でハングします。出力も出ないので、ハングなのか時間がかかっているだけなのか区別がつきません
- 座標を1つ渡せば、そのプロジェクトの関連 artifact をまとめて拾ってくれます。「所属不明の artifact ごとに1回」ではなく「プロジェクトごとに1回」で十分です

どのプロジェクトが未対応かを先に調べたいときは `advisor mapping search --prefix <prefix>` が速いです。

### レシピを作るの面倒なのだけど・・・

今なら、レシピは AI ・ LLM で作ることをお勧めします。
OpenRewrite の知見は十分にインターネット上にあるので、それなりの精度でつくれます。

### エアギャップ環境はいける？

Advisor CLI はエアギャップ環境に対応しています。

### Advisor CLI の更新頻度は？

`upgrade-plan` は、常に最新のレシピとセットで提供されます。
いいかえると、アップグレードしないと、最新パッチのレシピが含まれない（例：1.6.5 には、Spring Boot 4.1 にあげるレシピが含まれない）ので、特定のバージョンからアップグレードできなくなります。
よって、`upgrade-plan` を運用するうえでは、高頻度での更新が推奨されます。
また、Advisor CLI 自体のバグが修正されている可能性もあるので、可能な限り最新を試すのがおすすめです。

## もっと勉強がしたい！

以下のコンテキストが App Advisor をまた違った観点で解説しています。
ユーザー登録は必要ですが、無料です。なお、残念ながらコンテンツは英語です。

[Tanzu Academy : Spring Application Advisor Introduction](https://spring.academy/guides/app-advisor-intro)

## まとめ

Application Advisor の `upgrade-plan` について書きました。
