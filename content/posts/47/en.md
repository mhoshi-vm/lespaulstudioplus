---
title: "Adding Authentication to Spring gRPC with Spring Authorization Server (private_key_jwt Edition)"
date: "2026-07-16T13:00:00+09:00"
tags: ["Spring", "gRPC", "OIDC", "OAUTH2"]
thumbnail: "img_2.png"
---

In the [previous post](../46) I introduced how to integrate [Spring gRPC](https://spring.io/projects/spring-grpc) with [Spring Authorization Server](https://spring.io/projects/spring-authorization-server) using client_credentials.
However, with that method the client_secret was effectively treated as a password. So this time, I'll introduce how to achieve fully passwordless authentication using `private_key_jwt`.
<!--more--> 

## Introduction

Same as the [previous post](../46): this is about authentication to secure gRPC communication, but done in a completely passwordless way.
Note that the method introduced here is just one pattern; once you understand the mechanism, there are multiple ways to put it together.
(Also, since this is the result of trial and error, the code is messy.)

## Code

The code is here.

[https://github.com/mhoshi-vm/play-w-gprc-mtls](https://github.com/mhoshi-vm/play-w-gprc-mtls)

## With the private_key_jwt Method

This time, we use the private_key_jwt method.

- Same as last time, the apps we prepare are GrpcServer, GrpcClient, and OauthServer
- private_key_jwt requires issuing a JWT signed with an RSA private key while hosting the corresponding public key on a JWKS Endpoint.
This can live anywhere, but this time we co-locate it with the OauthServer
- The OauthServer hosts /token, which returns a JWT signed with the RSA key, and the corresponding public key is hosted at /oauth2/jwks
- GrpcClient obtains the JWT from /token and requests an access token from the OauthServer
- The OauthServer accesses /oauth2/jwks, verifies the JWT is valid using the public key, and issues an access token
- GrpcClient uses that access token to access GrpcServer
- GrpcServer checks with the OauthServer that the access token is valid, then responds to GrpcClient

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

### Starting the OauthServer

First, create the RSA key pair used by the OauthServer.

````
bash gen_rsa.sh
````

Then start the OauthServer using it.

```
cd oauth-server
./mvnw spring-boot:run -Dspring-boot.run.profiles=private_key_jwt
```

This profile loads `TokenConfig.java` and `TokenController.java`.
Let me explain some excerpts.

Creating a `JWKSource` Bean enables the OauthServer's "/oauth2/jwks".
We register it based on the RSA keys from the previous step.
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

We also make the jwtEncoder, used later in the controller, a Bean. Here we use the RSA private key that was used for the `JwkSource`.
```java
    @Bean
    JwtEncoder jwtEncoder(JwtProperties jwtProperties) {
        JWK jwk = new RSAKey.Builder(jwtProperties.publicKey()).privateKey(jwtProperties.privateKey()).build();
        JWKSource<SecurityContext> jwks = new ImmutableJWKSet<>(new JWKSet(jwk));
        return new NimbusJwtEncoder(jwks);
    }
```
We register the OAuth client. Unlike client_credentials, this can't be done with properties alone, so we write it in code.
Also, the key point is that no `clientSecret` is registered.
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
`/token` is implemented like this. For now (although of course not recommended), it returns a hard-coded JWT without any authentication.

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

### Starting the GrpcServer

Start the gRPC server. (Compiling at least once is required.)

```
cd server
./mvnw compile
./mvnw spring-boot:run -Dspring-boot.run.profiles=oauth
```

Nothing has changed from last time.

```
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:9000
```

### Starting the GrpcClient

Start the gRPC client. (Compiling at least once is required.)

```
cd client
./mvnw compile
./mvnw spring-boot:run -Dspring-boot.run.profiles=oauth_private_key_jwt
```

This profile loads `PrivJwtKeyClientConfiguration.java`.
What has changed is the following part:
- Obtain the JWT from /token
- Obtain the access token via OAuth
- Reuse the access token until it expires

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
### Running gRPC

After that, run the following command.

```
curl localhost:8081
```

If the following is printed in the client's prompt, the gRPC call succeeded.

```
message: "Hello ==> hello"
```

For example, if you intentionally make it return an invalid JWT token (change the issuer to something random):

```java
    @GetMapping("/token")
    public String token() {
        Instant now = Instant.now();
        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("aaaaa")   <<<<<<<<<<< here
                .issuedAt(now)
                .expiresAt(now.plusSeconds(30))
                .subject("grpc")
                .audience(new ArrayList<>(List.of("http://localhost:9000")))
                .claim("scope", "grpc:invoke")
                .build();

        return this.encoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
    }
```


If you look at the OauthServer's logs, you should see something like the following. In other words, authentication is working correctly.

```
2026-07-16T15:10:35.120+09:00 DEBUG 27914 --- [oauth-server] [nio-9000-exec-9] m.m.a.RequestResponseBodyMethodProcessor : Using 'text/plain', given [*/*] and supported [text/plain, */*, application/json, application/*+json]
2026-07-16T15:10:35.120+09:00 DEBUG 27914 --- [oauth-server] [nio-9000-exec-9] m.m.a.RequestResponseBodyMethodProcessor : Writing ["eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJhYWFhYSIsInN1YiI6ImdycGMiLCJhdWQiOiJodHRwOi8vbG9jYWxob3N0OjkwMDAiLCJ (truncated)..."]
2026-07-16T15:10:35.213+09:00 DEBUG 27914 --- [oauth-server] [nio-9000-exec-9] o.s.web.servlet.DispatcherServlet        : Completed 200 OK
2026-07-16T15:10:35.220+09:00 DEBUG 27914 --- [oauth-server] [io-9000-exec-10] o.s.security.web.FilterChainProxy        : Securing POST /oauth2/token
2026-07-16T15:10:35.222+09:00 DEBUG 27914 --- [oauth-server] [io-9000-exec-10] o.s.s.oauth2.jwt.JwtClaimValidator       : The iss claim is not valid
2026-07-16T15:10:35.223+09:00 DEBUG 27914 --- [oauth-server] [io-9000-exec-10] o.s.s.authentication.ProviderManager     : Authentication failed with provider JwtClientAssertionAuthenticationProvider since [invalid_client] Client authentication failed: client_assertion
2026-07-16T15:10:35.223+09:00 DEBUG 27914 --- [oauth-server] [io-9000-exec-10] .s.a.DefaultAuthenticationEventPublisher : No event was found for the exception org.springframework.security.oauth2.core.OAuth2AuthenticationException
```

As shown, we were able to authenticate gRPC using OAuth without any passwords.
However, with this approach, every device that can access /token can obtain a JWT and get authenticated.
You might think "wait, that's not secure at all", but there are multiple ways to keep /token secure.
The idea is to control token issuance based on information that proves the identity of the device making the request.
Typically that would be a TLS certificate (which makes it look somewhat like mTLS), and other options include the source IP or FQDN (assuming they can't be tampered with).
Either way, you can build this without depending on password-based authentication.

Note that everything above assumes a pure M2M scenario, with no human involved at all.
But look at what we actually did to trigger the call: we ran `curl localhost:8081` ourselves. That's already a human pressing enter.
If we accept that a human is present right at that first HTTP hit, passwordless gRPC authentication turns out to have an even simpler answer.
I'll cover that in the next entry.

## Conclusion

By combining Spring gRPC and Spring Authorization Server, we achieved passwordless authentication (private_key_jwt).