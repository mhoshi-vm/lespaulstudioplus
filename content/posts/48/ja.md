---
title: "Spring gRPC を Spring Authorization Server で認証機能をつける（oauth2Login編）"
date: "2026-07-18T14:00:00+09:00"
tags: ["Spring", "gRPC", "OIDC", "OAUTH2"]
thumbnail: "img.png"
---
[前回](../47)までは、gRPCの通信がclient, server しか登場しない前提で書いていました。
ただし、必ずしも、認証の起点をgRPCである必要がなく、もしWebブラウザを起点にできるなら、gRPCの認証をかなり楽にできます。
この記事では、それを試してみました。
<!--more--> 

## はじめに

[前々回](../46)・[前回](../47)で扱った client_credentials と private_key_jwt は、どちらもgRPCのクライアントとサーバー（と認証サーバー）という登場人物しかいない前提で書いていました。
これはいわゆるなMachine-to-Machine(M2M)の認証です。

ただし、前回までの記事でもそうであったように、最初に`curl` を打って通信を開始していました。
すなわち、Webクライアントを認証に起点にできるなら話が変わってきます。

つまり、Webクライアント側でOAUTH認証を完了させ、トークンが取得できれば、そのトークンをgRPC側でも利用ができ、設定を大幅に楽にできます。

というわけで今回は、GrpcClientがまずHTTP経由で認証し、そのトークンをそのままgRPCの呼び出しに伝搬させます。
また、ここまでの主題が「パスワードレス」だったので、ログイン自体もパスワードを使わないOne-Time Token（マジックリンク形式）でやることにしました。

## コード

コードはここです。

[https://github.com/mhoshi-vm/play-w-gprc-mtls](https://github.com/mhoshi-vm/play-w-gprc-mtls)

## ログイントークンをgRPCに伝搬させる

- 登場人物は増えて、GrpcServer, GrpcClient, OauthServerに加えBrowserの4つです。
- BrowserでGrpcClient を呼び出すHTTPエンドポイントを開くと、OauthServerにリダイレクトされ、ワンタイムトークンでサインインします。
- OauthServerが認可コードを発行し、GrpcClientがそれをトークンに交換すると、Spring SecurityがID Tokenを持つ `OidcUser` をセッションに保存します
- gRPC用に別途トークン交換をするのではなく、GrpcClientはその `OidcUser` からID Tokenをそのまま取り出し、gRPC呼び出しのBearer資格情報として送ります
- GrpcServerは前々回・前回から1ミリも変わっていません。相変わらず `issuer-uri` を信頼するだけのリソースサーバーなので、ブラウザログイン用に発行されたトークンだとは知らずに、普通に検証してしまいます

```text
Browser                     GrpcClient                    OauthServer                   GrpcServer
   |--(1) GET /-------------------->|                             |                             |
   |                                 |--(2) redirect to /oauth2/authorize---------------------->|
   |<--(3) redirect to /login------------------------------------|                              |
   |--(4) passkey / one-time-token login-------------------------------------------------------->|
   |<--(5) redirect back with authorization code-------------------------------------------------|
   |--(6) follow redirect---------->|                             |                             |
   |                                 |--(7) exchange code for tokens (incl. ID Token)----------->|
   |                                 |<--(8) ID Token-------------|                              |
   |                                 |--(9) gRPC request + ID Token as Bearer------------------------------------->|
   |                                 |                             |<--(10) validate w/ issuer-uri (self-check)---|
   |<--(11) gRPC response-----------|<-----------------------------------------------------------------------------|
```

### OauthServer の起動

```
cd oauth-server
./mvnw spring-boot:run -Dspring-boot.run.profiles=webauthn
```

このProfileによって `WebAuthConfig.java` が読み込まれます。今まであった素のAuthorization Server設定に加えて、`spring-security-webauthn`（パスキー）と `oneTimeTokenLogin()`（マジックリンク形式のログイン）を追加しています。

```java
@Configuration
@Profile("webauthn")
class WebAuthConfig {

    private final OneTimeTokenGenerationSuccessHandler redirectHandler = new RedirectOneTimeTokenGenerationSuccessHandler(
            "/login/ott");

    @Bean
    Customizer<HttpSecurity> httpSecurityCustomizer() {
        return http -> http
                .oneTimeTokenLogin(ott -> ott
                        .tokenGenerationSuccessHandler(
                                (request, response, oneTimeToken) -> {
                                    System.out.printf("please go to http://localhost:9000/login/ott?token=" + oneTimeToken.getTokenValue() + "\n");
                                    this.redirectHandler.handle(request, response, oneTimeToken);
                                }
                        )
                )
                .logout(Customizer.withDefaults())
                .webAuthn(a -> a
                        .allowedOrigins("http://localhost:9000/")
                        .rpId("localhost")
                        .rpName("webauthn")
                );
    }

    @Bean
    InMemoryUserDetailsManager userDetailsService(SecurityProperties securityProperties) {
        SecurityProperties.User user = securityProperties.getUser();
        return new InMemoryUserDetailsManager(User.withUsername(user.getName())
                .password("{noop}" + user.getPassword())
                .roles(user.getRoles().toArray(String[]::new))
                .build());
    }

    @Bean
    OneTimeTokenService oneTimeTokenService() {
        return new InMemoryOneTimeTokenService();
    }
}
```

メール送信の仕組みまでは用意していないので、`tokenGenerationSuccessHandler` はマジックリンクをメールで送る代わりに標準出力にそのまま出しています。実際のアプリならここでメールやプッシュ通知を送ることになります。

OauthServer側のGrpcClient用クライアント登録はこうなっています。ここでも秘密情報は一切なく、`require-proof-key=false` なのは、これがパブリックなSPAではなく、サーバーサイドの（ある程度）confidentialなクライアントだからです。

```
spring.security.oauth2.authorizationserver.client.frontend.registration.client-id=webauthn
spring.security.oauth2.authorizationserver.client.frontend.registration.client-authentication-methods=none
spring.security.oauth2.authorizationserver.client.frontend.registration.authorization-grant-types=authorization_code,refresh_token
spring.security.oauth2.authorizationserver.client.frontend.registration.redirect-uris=http://localhost:8081/login/oauth2/code/oauthserver
spring.security.oauth2.authorizationserver.client.frontend.registration.post-logout-redirect-uris=http://localhost:8081
spring.security.oauth2.authorizationserver.client.frontend.registration.scopes=openid,profile
spring.security.oauth2.authorizationserver.client.frontend.require-proof-key=false

spring.security.user.name=admin
```

### GrpcServer の起動

```
cd server
./mvnw compile
./mvnw spring-boot:run -Dspring-boot.run.profiles=oauth
```

前回・前々回から1文字も変わっていません。

```
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:9000
```

### GrpcClient の起動

```
cd client
./mvnw compile
./mvnw spring-boot:run -Dspring-boot.run.profiles=oauth_webauthn
```

このProfileによって `WebAuthnClientConfiguration.java` と以下のpropertiesが読み込まれます。
ここでも client_secret は含まれません。

```
spring.security.oauth2.client.provider.oauthserver.issuer-uri=${AUTH_SERVER_ISSUER:http://localhost:9000}

spring.security.oauth2.client.registration.oauthserver.provider=oauthserver
spring.security.oauth2.client.registration.oauthserver.client-id=webauthn
spring.security.oauth2.client.registration.oauthserver.client-authentication-method=none
spring.security.oauth2.client.registration.oauthserver.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.oauthserver.scope=openid
spring.security.oauth2.client.registration.oauthserver.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
```

gRPC stubのインターセプターは、OauthServerに何かを新しく依頼するのではなく、その時点で認証済みの `OidcUser` からID Tokenをそのまま取り出します。

```java
@Configuration
@Profile("oauth_webauthn")
class WebAuthnClientConfiguration {

    @Bean
    @Lazy
    SimpleGrpc.SimpleBlockingStub basic(GrpcChannelFactory channels) {
        return SimpleGrpc.newBlockingStub(channels.createChannel("0.0.0.0:9090", ChannelBuilderOptions.defaults()))
                .withInterceptors(new BearerTokenAuthenticationInterceptor(WebAuthnClientConfiguration::currentJwtTokenValue));
    }

    /**
     * Returns the ID token of the currently logged-in browser user so it can be
     * propagated to the gRPC server, which validates it against the same
     * OauthServer issuer. oauth2Login() authenticates as an OAuth2AuthenticationToken
     * wrapping an OidcUser, never a JwtAuthenticationToken (that type only shows up
     * behind an oauth2ResourceServer() filter validating an incoming bearer token).
     */
    static String currentJwtTokenValue() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication instanceof OAuth2AuthenticationToken oauth2Authentication
                && oauth2Authentication.getPrincipal() instanceof OidcUser oidcUser) {
            return oidcUser.getIdToken().getTokenValue();
        }
        throw new IllegalStateException("The gRPC call requires an OIDC-authenticated caller");
    }

}
```

client_secretのやりとりも、秘密鍵での署名も、前回みたいなトークンのキャッシュ・更新ロジックも一切ありません。
`currentJwtTokenValue()` はgRPC呼び出しのたびに動きますが、やっていることは `SecurityContextHolder` に既にあるものを読むだけです。

### gRPC実行

今回は `curl localhost:8081` だけでは完結しません。ログインに実際のブラウザが必要だからです。手順はこうです。

1. ブラウザで `http://localhost:8081` を開きます。`http://localhost:9000/login` に飛ばされます。
2. このページにはパスワードのフォームと、「Request a One-Time Token」フォーム、「Sign in with a passkey」ボタンがあります。パスワードフォームは無視して（そもそもパスワードは設定していません）、One-Time Tokenのフォームに `admin` と入力します。
3. メール送信の仕組みは用意していないので、OauthServerのコンソールにマジックリンクがそのまま出力されます。
   ```
   please go to http://localhost:9000/login/ott?token=c63eed63-bce6-4988-8ab5-d894cbf9a3c7
   ```
4. そのリンクを開き、「Sign in」を押します。
5. OauthServerの認可エンドポイント、GrpcClientのコールバックを経由して、最終的に `http://localhost:8081` に戻ってきます。その時点ですでにgRPC呼び出しが実行済みで、以下が出力されています。
   ```
   message: "Hello ==> hello"
   ```

実際に、この一連の流れを最新のコミットに対して curl とクッキージャーでブラウザの代わりにスクリプト化し、最初から最後まで通してみました（手を抜かず本当に動くことを確認したかったので）。gRPCサーバー側のリクエストログには、実際に流れたものがそのまま出ています。

```
authorization: Bearer eyJraWQiOiJmMzJmYmQ1ZS1hMTU1LTQ5YWUtODA2YS0wYTliYTkxMzdlNTciLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiIsImF1ZCI6IndlYmF1dGhuIiwiYXpwIjoid2ViYXV0aG4iLCJhdXRoX3RpbWUiOjE3ODQzODI4NDEsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6OTAwMCIsImV4cCI6MTc4NDM4NDY0MSwiaWF0IjoxNzg0MzgyODQxLCJub25jZSI6InBSQy1QdUNFeThUbXd6TDNEdGd2VVIxb3NyTGdhVzJUTFgyeXJGZGJTMGciLCJqdGkiOiJjOTg5ZTIxOC05NWQ2LTQ1NzUtOTNiZS0wMGM2MzIzMzM4ZWQiLCJzaWQiOiJEeUQ1Q0ZqWjFpYVh5blAxRHVUVENRR2I0d1d5TTQ5dzdYSG9PdlFoeUEwIn0....
```

デコードすると中身はこうです。

```json
{
  "sub": "admin",
  "aud": "webauthn",
  "azp": "webauthn",
  "auth_time": 1784382841,
  "iss": "http://localhost:9000",
  "exp": 1784384641,
  "iat": 1784382841,
  "nonce": "pRC-PuCEy8TmwzL3DtgvUR1osrLgaW2TLX2yrFdbS0g",
  "jti": "c989e218-95d6-4575-93be-00c6323338ed",
  "sid": "DyD5CFjZ1iaXynP1DuTTCQGb4wWyM49w7XHoOvQhyA0"
}
```

サーバー側では `GrpcService` が普通に `Hello hello` とログを出して返答しています。前々回・前回の client_credentials や private_key_jwt のトークンのときと全く同じように、issuerと署名だけを見て検証しています。
なおGrpcServerは `aud` を一切見ておらず、`issuer-uri` しか見ていないので、この使い回しが成立してしまいます。
本番でやるなら、ログイントークンをそのまま流すのではなく、gRPC用にスコープされたアクセストークンへ交換するトークン交換（RFC 8693）を一段挟むべきでしょう。

## おわりに

認証の起点をWebにしてしまえば、gRPCの認証はもう自前の資格情報を持つ必要がなくなり、ログインの結果をそのまま受け継げばよくなります。
ログインそのものをWebAuthnとOne-Time Tokenでやることで、ブラウザからgRPCサーバーまで、一連の流れの中でパスワードには一度も触れずに済みました。
