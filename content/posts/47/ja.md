---
title: "Spring gRPC を Spring Authorization Server で認証機能をつける（private_key_jwt編）"
date: "2026-07-16T13:00:00+09:00"
tags: ["Spring", "gRPC", "OIDC", "OAUTH2"]
thumbnail: "img_2.png"
---

[前回](../46)で [Spring gRPC](https://spring.io/projects/spring-grpc) と [Spring Authorization Server](https://spring.io/projects/spring-authorization-server) を client_credentials で連携する方法を紹介しました。
ただし、この方法だと client_secret が実質パスワード扱いでした。そこで今回は、`private_key_jwt` を使い完全にパスワードレスな認証をやる方法を紹介します。
<!--more--> 

## はじめに

[前回](../46)と一緒ですが、gRPCの通信を安全にするための認証、それを完全にパスワードレスにやる方法を紹介します。
なお、ここで紹介する方法は、あくまで1パターンであり、仕組みさえわかれば、組み方は複数通り可能です。
（しかも試行錯誤の結果なのでコードが汚いです。）

## コード

コードはここです。

[https://github.com/mhoshi-vm/play-w-gprc-mtls](https://github.com/mhoshi-vm/play-w-gprc-mtls)

## private_key_jwt 方式で

今回は、private_key_jwt 方式でやります。

- 前回と同じく、用意するアプリは、GrpcServer, GrpcClient, OauthServer
- private_key_jwt は、RSA秘密鍵で署名されたJWTを発行しつつ、その公開鍵をJWKS Endpointでホスティングする必要があります。
これ自体はどこでもいいのですが今回はOauthServerに同居させる
- OauthServerには、RSA鍵で署名されたJWTを返す /token をホスティングし、その公開鍵は /oauth2/jwks でホスティング
- GrpcClient は/tokenからJWTを入手して、OauthServerにアクセストークンを依頼する
- OauthServer は、/oauth2/jwksにアクセスし、公開鍵からJWTが正しいと確認し、アクセストークンを発行
- GrpcClientはそのアクセストークンを使って、GrpcServerへアクセス
- GrpcServer はそのアクセストークンが正しいものか OauthServer に確認後、GrpcClientに返答

```text
GrpcClient                      OauthServer                     GrpcServer
    |                               |                               |
    |--(1) GET /token-------------->|                               |
    |                               |                               |
    |<--(2) signed JWT--------------|                               |
    |                               |                               |
    |--(3) POST /oauth2/token------>|                               |
    |    (client_assertion = JWT)   |                               |
    |                               |                               |
    |                               |--(4) verify JWT with          |
    |                               |<---- /oauth2/jwks (self)      |
    |                               |                               |
    |<--(5) access token------------|                               |
    |                               |                               |
    |--(6) gRPC request + access token----------------------------->|
    |                               |                               |
    |                               |<--(7) validate access token---|
    |                               |                               |
    |                               |--(8) token is valid---------->|
    |                               |                               |
    |<--(9) gRPC response-------------------------------------------|
```

### OauthServer の起動

まず、OauthServerが使うRSA鍵ペアを作ります。

````
bash gen_rsa.sh
````

これを使いOauthServerを起動します。

```
cd oauth-server
./mvnw spring-boot:run -Dspring-boot.run.profiles=private_key_jwt
```

このProfileによる`TokenConfig.java` と `TokenController.java` が読み込まれます。
抜粋して解説します。

`JWKSource` Bean をつくることで、OauthServerの"/oauth2/jwks"が有効になります。
前手順のRSA鍵をベースに登録していきます。
```java
    @Bean
    public JWKSource<SecurityContext> jwkSource(JwtProperties jwtProperties) {

        RSAPublicKey publicKey = jwtProperties.publicKey();
        RSAPrivateKey privateKey = jwtProperties.privateKey();

        // 2. Build the Nimbus RSAKey representation
        RSAKey rsaKey = new RSAKey.Builder(publicKey)
                .privateKey(privateKey)
                .keyID(jwtProperties.kid())
                .build();

        // 3. Wrap it in a JWKSet and return
        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }
```

あとでControllerで使うjwtEncoderもBeanにします。ここでは、`JwkSource` で使ったRSAの秘密鍵を使います。
```java
    @Bean
    JwtEncoder jwtEncoder(JwtProperties jwtProperties) {
        JWK jwk = new RSAKey.Builder(jwtProperties.publicKey()).privateKey(jwtProperties.privateKey()).build();
        JWKSource<SecurityContext> jwks = new ImmutableJWKSet<>(new JWKSet(jwk));
        return new NimbusJwtEncoder(jwks);
    }
```
Oauthクライアントを登録します。client_credentialsとは違い、propertiesだけではできないので、コードで書きます。
また、ポイントとして、`clientSecret`を登録していない点です。
```java
    @Bean
    RegisteredClientRepository registeredClientRepository() {
        RegisteredClient grpcClient = RegisteredClient.withId(UUID.randomUUID().toString())
                .clientId("grpc")
                // Note: No clientSecret is configured because we are using JWT assertions
                .clientAuthenticationMethod(ClientAuthenticationMethod.PRIVATE_KEY_JWT)
                .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
                .scope("grpc:invoke") // replace with your actual scopes
                .clientSettings(ClientSettings.builder()
                        // 1. Tell the Auth Server which algorithm the client used to sign the JWT
                        .tokenEndpointAuthenticationSigningAlgorithm(SignatureAlgorithm.RS256)
                        // 2. Tell the Auth Server where to download the client's public keys
                        .jwkSetUrl("http://localhost:9000/oauth2/jwks")
                        .build())
                .build();
        return new InMemoryRegisteredClientRepository(grpcClient);
    }
```
`/token` はこんな感じで実装します。一旦は（もちろん非推奨ですが）特に認証もなく、ハードコードされたJWTを返すようにします。

```java
    @GetMapping("/token")
    public String token() {
        Instant now = Instant.now();
        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("grpc")
                .issuedAt(now)
                .expiresAt(now.plusSeconds(30))
                .subject("grpc")
                .audience(new ArrayList<>(List.of("http://localhost:9000")))
                .claim("scope", "grpc:invoke")
                .build();

        return this.encoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
    }
```

### GrpcServer の起動

gRPCサーバーを起動します。(compileは1度は必須)

```
cd server
./mvnw compile
./mvnw spring-boot:run -Dspring-boot.run.profiles=oauth
```

これは前回と何も変わらないです。

```
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:9000
```

### GrpcClient の起動

gRPCクライアントを起動します。(compileは1度は必須)

```
cd client
./mvnw compile
./mvnw spring-boot:run -Dspring-boot.run.profiles=oauth_private_key_jwt
```

このProfileによって、`PrivJwtKeyClientConfiguration.java` が読み込まれます。
変わっているのが以下の箇所であり
- /tokenからJWTを入手
- アクセストークンをOauth経由で取得
- アクセストークンが有効期限切れになるまで再利用

```java
    @Bean
    @Lazy
    SimpleGrpc.SimpleBlockingStub basic(GrpcChannelFactory channels, ClientRegistrationRepository registry, RestClient restClient) {
        ClientRegistration reg = registry.findByRegistrationId("grpc-client");

        return SimpleGrpc.newBlockingStub(channels.createChannel("0.0.0.0:9090", ChannelBuilderOptions.defaults().withInterceptors(List.of(new BearerTokenAuthenticationInterceptor(() -> token(reg, restClient))))));
    }

    private String token(ClientRegistration reg, RestClient restClient) {
        // 2. Read the current token
        OAuth2AccessToken currentToken = this.cachedAccessToken;

        // 3. Check if we have a token that is valid for at least 30 more seconds
        if (isTokenValid(currentToken)) {
            return currentToken.getTokenValue();
        }

        // 4. Token is missing or expired, sync up to fetch a new one
        synchronized (this) {
            // Double-check locking in case another thread already refreshed it while we were waiting
            currentToken = this.cachedAccessToken;
            if (isTokenValid(currentToken)) {
                return currentToken.getTokenValue();
            }

            RestClientClientCredentialsTokenResponseClient creds = new RestClientClientCredentialsTokenResponseClient();

            String preSignedJwt = fetchToken(restClient);

            creds.addParametersConverter(request -> {
                MultiValueMap<String, String> parameters = new LinkedMultiValueMap<>();
                parameters.add("client_assertion_type", "urn:ietf:params:oauth:client-assertion-type:jwt-bearer");
                parameters.add("client_assertion", preSignedJwt);
                return parameters;
            });

            // 5. Update the cache with the new token
            this.cachedAccessToken = creds.getTokenResponse(new OAuth2ClientCredentialsGrantRequest(reg)).getAccessToken();
            return this.cachedAccessToken.getTokenValue();
        }
    }

    // Helper method to keep the if-statements clean
    private boolean isTokenValid(OAuth2AccessToken token) {
        if (token == null || token.getExpiresAt() == null) {
            return false;
        }
        // Add a 30-second buffer to prevent the token from expiring mid-flight during the gRPC call
        return token.getExpiresAt().isAfter(Instant.now().plusSeconds(10));
    }
```
### gRPC実行

その後、以下のコマンドを打ちます。

```
curl localhost:8081
```

client側のプロンプトで以下が出力されれば、gRPC成功

```
message: "Hello ==> hello"
```

たとえば意図的に不正なJWTトークンを返すようにしてみると(issuerを適当にかえてみる)

```java
    @GetMapping("/token")
    public String token() {
        Instant now = Instant.now();
        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("aaaaa")   <<<<<<<<<<< ここ
                .issuedAt(now)
                .expiresAt(now.plusSeconds(30))
                .subject("grpc")
                .audience(new ArrayList<>(List.of("http://localhost:9000")))
                .claim("scope", "grpc:invoke")
                .build();

        return this.encoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
    }
```


OauthServerのログをみると以下のようなものがみれるかと思います。つまり正しく認証ができています。

```
2026-07-16T15:10:35.120+09:00 DEBUG 27914 --- [oauth-server] [nio-9000-exec-9] m.m.a.RequestResponseBodyMethodProcessor : Using 'text/plain', given [*/*] and supported [text/plain, */*, application/json, application/*+json]
2026-07-16T15:10:35.120+09:00 DEBUG 27914 --- [oauth-server] [nio-9000-exec-9] m.m.a.RequestResponseBodyMethodProcessor : Writing ["eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJhYWFhYSIsInN1YiI6ImdycGMiLCJhdWQiOiJodHRwOi8vbG9jYWxob3N0OjkwMDAiLCJ (truncated)..."]
2026-07-16T15:10:35.213+09:00 DEBUG 27914 --- [oauth-server] [nio-9000-exec-9] o.s.web.servlet.DispatcherServlet        : Completed 200 OK
2026-07-16T15:10:35.220+09:00 DEBUG 27914 --- [oauth-server] [io-9000-exec-10] o.s.security.web.FilterChainProxy        : Securing POST /oauth2/token
2026-07-16T15:10:35.222+09:00 DEBUG 27914 --- [oauth-server] [io-9000-exec-10] o.s.s.oauth2.jwt.JwtClaimValidator       : The iss claim is not valid
2026-07-16T15:10:35.223+09:00 DEBUG 27914 --- [oauth-server] [io-9000-exec-10] o.s.s.authentication.ProviderManager     : Authentication failed with provider JwtClientAssertionAuthenticationProvider since [invalid_client] Client authentication failed: client_assertion
2026-07-16T15:10:35.223+09:00 DEBUG 27914 --- [oauth-server] [io-9000-exec-10] .s.a.DefaultAuthenticationEventPublisher : No event was found for the exception org.springframework.security.oauth2.core.OAuth2AuthenticationException
```

この通りパスワードなしで、oauthを使いgrpcを認証できました。
なお、このやり方だと /token にアクセスできる全てのデバイスがJWTを取れてしまい認証できてしまいます。
え、セキュアじゃないじゃん、とは思うかもしれませんが、/tokenをセキュアに保つ手段は複数とれます。
アクセスしてきたデバイスが自分自身を証明する情報をもとにtoken発行を制御すればいいとなります。
一般的にはTLS証明書（mTLSっぽくなってしまいますが）、他にも接続元IPやFQDN（改ざんできない前提ですが）といった手段がとれるかと思います。
いずれにせよ、パスワードでの認証方法に依存せずに構築することができます。

なお、ここでのやり方は、完全なM2Mのケースであり、まったくユーザーによる操作が介在しない、という前提での話です。
しかし、今回やった方法ですが、以下のコマンドを打って、gRPCをしていますよね？

```
curl localhost:8081
```

つまりユーザー操作が介在しているわけです。このユーザー操作が介在している前提があるのであれば、「gRPC間の認証をパスワードレスでやる」は実はもっと簡単な方法があります。
それは次のエントリで紹介します。

## おわりに

Spring gRPC と Spring Authorization Server を合体させパスワードレス(private_key_jwt)ができました。