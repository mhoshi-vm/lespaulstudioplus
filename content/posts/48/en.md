---
title: "Adding Authentication to Spring gRPC with Spring Authorization Server (oauth2Login Edition)"
date: "2026-07-18T14:00:00+09:00"
tags: ["Spring", "gRPC", "OIDC", "OAUTH2"]
thumbnail: "img.png"
---

Up through the [previous post](../47), I'd been writing as if gRPC communication only ever involved a client and a server.
But authentication doesn't have to start at gRPC at all — if you can start from a web browser instead, gRPC authentication gets a lot easier.
This post tries exactly that.
<!--more--> 

## Introduction

The client_credentials and private_key_jwt approaches from the [two](../46) [previous](../47) posts were both written as if the only characters on stage were a gRPC client and server (plus the auth server).
That's what's usually called Machine-to-Machine (M2M) authentication.

But just like in those earlier posts, I always kicked things off by running `curl` myself. In other words, if we let a web client be the starting point of authentication instead, the whole picture changes.

Once the web client finishes OAuth authentication and gets hold of a token, that same token can be reused on the gRPC side too, which makes the setup considerably simpler.

So this time, GrpcClient authenticates over HTTP first, and then propagates that same token straight through to the gRPC call.
And since the theme so far has been "passwordless," I made the login itself use One-Time Token (a magic-link style flow) rather than a password.

## Code

The code is here.

[https://github.com/mhoshi-vm/play-w-gprc-mtls](https://github.com/mhoshi-vm/play-w-gprc-mtls)

## Propagating the Login Token to gRPC

- The cast has grown: GrpcServer, GrpcClient, OauthServer, plus a Browser — four now
- Opening the HTTP endpoint that calls GrpcClient in the Browser redirects you to OauthServer, where you sign in with a one-time token
- OauthServer issues an authorization code, and once GrpcClient exchanges it for tokens, Spring Security stores an `OidcUser` holding the ID Token in the session
- Rather than doing a separate token exchange for gRPC, GrpcClient just pulls the ID Token straight out of that `OidcUser` and sends it as the gRPC call's Bearer credential
- GrpcServer hasn't changed by so much as a millimeter from the last two posts. It's still just a resource server trusting `issuer-uri`, so it happily validates a token without knowing it was actually minted for a browser login

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

### Starting the OauthServer

```
cd oauth-server
./mvnw spring-boot:run -Dspring-boot.run.profiles=webauthn
```

This profile loads `WebAuthConfig.java`. It adds `spring-security-webauthn` (passkeys) plus a `oneTimeTokenLogin()` (a magic-link style login), on top of the plain Authorization Server config we already had.

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

I didn't have a real mail sender wired up, so instead of emailing the magic link, `tokenGenerationSuccessHandler` just prints it to stdout. In a real app this is where you'd send an email or a push notification instead.

The OauthServer-side client registration for GrpcClient looks like this — again, no secret anywhere, and `require-proof-key=false` because this is a confidential-ish server-side client, not a public SPA.

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

### Starting the GrpcServer

```
cd server
./mvnw compile
./mvnw spring-boot:run -Dspring-boot.run.profiles=oauth
```

Not a single character different from the last two posts.

```
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:9000
```

### Starting the GrpcClient

```
cd client
./mvnw compile
./mvnw spring-boot:run -Dspring-boot.run.profiles=oauth_webauthn
```

This profile loads `WebAuthnClientConfiguration.java` along with the following properties. There's no client_secret here either.

```
spring.security.oauth2.client.provider.oauthserver.issuer-uri=${AUTH_SERVER_ISSUER:http://localhost:9000}

spring.security.oauth2.client.registration.oauthserver.provider=oauthserver
spring.security.oauth2.client.registration.oauthserver.client-id=webauthn
spring.security.oauth2.client.registration.oauthserver.client-authentication-method=none
spring.security.oauth2.client.registration.oauthserver.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.oauthserver.scope=openid
spring.security.oauth2.client.registration.oauthserver.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
```

The gRPC stub's interceptor doesn't ask OauthServer for anything new — it just pulls the ID Token straight out of the `OidcUser` that's already authenticated at that point:

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

No client_secret exchange, no private key signing, no cached token refresh logic like the last post. `currentJwtTokenValue()` runs on every gRPC call and just reads whatever is already sitting in `SecurityContextHolder`.

### Running gRPC

This time `curl localhost:8081` alone won't get you there, since logging in needs an actual browser. So:

1. Open `http://localhost:8081` in a browser. You'll get bounced to `http://localhost:9000/login`.
2. That page offers a password form, a "Request a One-Time Token" form, and a "Sign in with a passkey" button. Skip the password form (there's no configured password anyway) and type `admin` into the One-Time Token form.
3. Since there's no mail sender configured, the OauthServer console prints the magic link instead of emailing it:
   ```
   please go to http://localhost:9000/login/ott?token=c63eed63-bce6-4988-8ab5-d894cbf9a3c7
   ```
4. Open that link, click "Sign in".
5. You're bounced back through OauthServer's authorization endpoint, then to GrpcClient's callback, and finally land back on `http://localhost:8081`, which by then has already made the gRPC call and printed:
   ```
   message: "Hello ==> hello"
   ```

I actually drove this whole sequence end-to-end (scripted with curl and a cookie jar standing in for the browser, to prove it works without hand-waving) against the latest commit, and the gRPC server's own request log shows exactly what went out on the wire:

```
authorization: Bearer eyJraWQiOiJmMzJmYmQ1ZS1hMTU1LTQ5YWUtODA2YS0wYTliYTkxMzdlNTciLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiIsImF1ZCI6IndlYmF1dGhuIiwiYXpwIjoid2ViYXV0aG4iLCJhdXRoX3RpbWUiOjE3ODQzODI4NDEsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6OTAwMCIsImV4cCI6MTc4NDM4NDY0MSwiaWF0IjoxNzg0MzgyODQxLCJub25jZSI6InBSQy1QdUNFeThUbXd6TDNEdGd2VVIxb3NyTGdhVzJUTFgyeXJGZGJTMGciLCJqdGkiOiJjOTg5ZTIxOC05NWQ2LTQ1NzUtOTNiZS0wMGM2MzIzMzM4ZWQiLCJzaWQiOiJEeUQ1Q0ZqWjFpYVh5blAxRHVUVENRR2I0d1d5TTQ5dzdYSG9PdlFoeUEwIn0....
```

Decoded, the payload is:

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

On the server side, `GrpcService` logs `Hello hello` as usual and replies — validated purely by checking the issuer and signature, exactly like the client_credentials and private_key_jwt tokens in the earlier two posts.
GrpcServer never checks `aud` at all — only `issuer-uri` — which is exactly what makes this reuse work.
For anything real, rather than passing the login token straight through, you'd probably want to add a token exchange (RFC 8693) step to swap it for an access token properly scoped for gRPC.

## Conclusion

Once you make the web the starting point of authentication, gRPC no longer needs its own credential at all — it can just inherit whatever came out of the login.
By doing the login itself with WebAuthn and One-Time Token, the whole chain — browser all the way to the gRPC server — never touched a password once.
