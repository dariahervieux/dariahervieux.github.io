---
title: "Basics of OpenID Connect (OIDC) explained"
categories:
  - Back-end
  - Security
tags:
  - security
  - authentication
---

> **TL;DR** An overview of Open ID connect: what it is, how it works, authentication flow examples.

## Overview

Have you heard of OAuth 2.0? 
Not yet? Aouch! This article assumes that you are already familiar with the basics of OAuth 2.0.
To get more information about OAuth 2.0, you can check out [OAuth 2 Simplified article](https://aaronparecki.com/oauth-2-simplified/) by Aaron Parecki. 

OpenID Connect 1.0 (OIDC) is a simple identity layer on top of the OAuth 2.0 [RFC6749](https://tools.ietf.org/html/rfc6749) protocol.

OIDC specification consists of several parts, some of them are optional. In this article we will focus only of the mandatory parts:

 * [Core](https://openid.net/specs/openid-connect-core-1_0.html) and 
 * [OAuth 2.0 Multiple Response Type Encoding Practices](https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html)

## Participants and typical workflow

OIDC introduces new terms for the client and the authorization server participants of the authentication workflow.
Here are the participants:

* End-User - a human, the owner of the protected resources.
* Relying Party (RP) - this is a **OAuth Client requesting End-User Authentication** from an OpenID Provider, it **relies** on the information returned by the server in a secure manner.
* OpenID Provider (OP) - this is an OAuth 2.0 Authorization Server with the capability to **Authenticate the End-User** and to provide information (Claims) to a Relying Party about the Authentication event and the End-User.

The simplified workflow of the authentication process is the following:
1. The first step is exactly the same as for Authorization workflow using OAuth2 - the client application (RP) makes a request to the Authorization end point of the Authorization server (OP). 
2. The OP server first authenticates the user (redirection to login) and then obtains an authorization grant from the user (redirection to authorization page).
3. The OP server issues an ID token(🗎) and, if requested by the client application, an access token(🔑) at the same time.
4. The RP uses access token to retrieve additional information about the authenticated user through UserInfo endpoint.

{% include figure image_path="assets/images/articles/OIDC/OIDC-simplified-wokflow.svg" alt="OIDC User Authentication simplified flow" caption="OIDC User Authentication simplified flow" %}

As you can see, the difference between OAuth authorization flow and the OIDC authentication flow is that the Authorization server returns a new type of a token - ID Token (🗎) - along with the Access token(🔑).

To distinguish between the two types of tokens: **Access token** is the user's **authorization to use protected resources**, **ID Token** is a **certificate of the End-User Authentication event authenticity**.

## ID token

ID token allows Clients to **Authenticate** the End-User.
The ID Token is a data structure that contains data (Claims) about the Authentication of an End-User and the End-User herself.

Here is a list of **mandatory** claims which are present in the ID token:

* **iss** - response issuer identifier, basically the URL of the OP. For example `https://op.my-organization.com`.
* **sub** - subject identifier, a unique identifier of the End-User within OP. For example `56355235245` or `797766d4-4475-460f-a14d-da3e8fa6229e`.
* **aud** - audience(s), the list of `client_id` of the RP (OAuth2 Client) to whom this ID Token is intended for.
* **exp** - expiration time, the expiration date (in a form of epoch timestamp) of the ID token. For example `1554241195`.
* **iat** - issue time, the moment(in a form of epoch timestamp) at which this token was issued. For example `1554241255`.

So the minimalist ID token consists of the End-User identifier, the OP identifier, the list of RP ids this token is intended for and the token issue and expiration dates:

```json
  {
   "iss": "https://op.my-organization.com",
   "sub": "797766d4-4475-460f-a14d-da3e8fa6229e",
   "aud": "13716fb0-3873-419c-972c-9cb333907b76",
   "nonce": "n-0S6_WzA2Mj",
   "exp": 1554241195,
   "iat": 1554241255
  }
```

ID token can contain some standard optional [claims](https://openid.net/specs/openid-connect-core-1_0.html#IDToken), any [claims related to the user profile](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims) as well as any custom claims recognized by the Client.

ID token is encoded as [JSON Web Token (JWT) [RFC7519]](https://tools.ietf.org/html/rfc7519) and signed using [JSON Web Signature (JWS) [RFC7515]](https://tools.ietf.org/html/rfc7515). ID token can be optionally encrypted using [JSON Web Encryption (JWE) [RFC7516]](https://tools.ietf.org/html/rfc7516).

### New response type

[OAuth 2.0 Multiple Response Type Encoding Practices](https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html) specification registers a new response type of the OAuth [Authorization end point]() - `id_token` which is used to get the ID token from the Authorization serer:

> When supplied as the **response_type** parameter in an OAuth 2.0 Authorization Request, a successful response MUST include the parameter **id_token**. 

## Getting authenticated End-User information

OIDC proposes two ways to get information about the Authenticated End-User:

* directly in the ID token within optional claims
* via [UserInfo](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo) endpoint

## Authentication process

The user can be authenticated using one of the exchange flows based on OAuth2 protocol's flows:

1. Authorization Code Flow (`response_type=code`)
2. Implicit Flow (`response_type=id_token token` or `response_type=id_token`)
3. [Hybrid Flow](https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html#Combinations) ( `response_type=code id_token token`, `response_type=code id_token`  , `response_type=code token`)

OIDC specification introduces a new type of flow - **Hybrid Flow**, which is, no surprise, the hybrid between Authorization and Implicit flows.
The flow to use is determined by the value(s) of `response_type` parameter sent to authorization endpoint (`/authorize` in this article).

Here is a comparison table for flows. 

| Property      | Authorization Code  | Implicit Flow  | Hybrid Flow  |
| ------------- |:-------------------:|:--------------:|:------------:|
| Communication in one round trip | NO (2 endpoints are involved) | YES (`/authorize`) | NO (2 endpoints are involved) |
| All tokens returned from Authorization Endpoint(`/authorize`) | NO  |   YES | NO |
| All tokens returned from Token Endpoint (`/token` in this article) | YES  |  NO | NO |
| Client can be authenticated (using `client_id` and `client_secret`) | YES  |  NO (`client_secret` is never sent) | YES |
| Tokens not revealed to User Agent | 
| Refresh Token can be issued  | YES  |  NO (`client_secret` is never sent) | YES |

The use of **TLS is mandatory** for all communications with the Authorization server.

### Authorization Code Flow

Authorization Code Flow is mostly used for server-to-server communication. In this case a OAuth client application is a WEB server application ([confidential client](https://tools.ietf.org/html/rfc6749#section-2.1)), where OAuth client credentials can be kept securely.
In this flow a temporary code issued to the used is exchanged for tokens by the RP. This way issued tokens are not revealed  to User Agent (a browser), i.e. tokens are never stored/accessed by the User Agent. When performing a request to the token end point of the OP, the client (RP)  must authenticate with its credentials.

It is recommended to use Authorization Code flow whenever possible.

Here is an example of Authorization Code flow for a confidential client with the JS front-end running a browser.

{% include figure image_path="assets/images/articles/OIDC/OIDC-Authorization-Code-Flow.svg" alt="Authorization Code Flow sequence" caption="Authorization Code Flow sequence" %}

Authorization Code Flow can be used for public clients also with additional security mesures ([PKCE](https://tools.ietf.org/html/rfc7636)) to prevent the authorization code interception attack.

### Implicit Flow

Implicit flow is for communication between the user application (native application or an application running in a browser) and the server. In this case we are dealing with the public client where `client_secret` can not be stored within the user application securely. Consequently, the client(RP) can not be authenticated by the Authorization server (OP) using OAuth client credentials. Moreover token values are stored in a browser memory and thus are more vulnerable to security threats.

This flow is less secure and should be used when Authorization Code flow is impossible.
However it is faster and much simpler since tokens are obtained in a single round.

The example of the Implicit flow for the public client (JS application) running in a browser.

{% include figure image_path="assets/images/articles/OIDC/OIDC-Implicit-Flow.svg" alt="Implicit Flow sequence" caption="Implicit Flow sequence" %}

## Resources

1. [Diagrams of All The OpenID Connect Flows](https://medium.com/@darutk/diagrams-of-all-the-openid-connect-flows-6968e3990660) by Takahiko Kawasaki
2. OAuth 2.0 for Native Apps - [RFC8252](https://tools.ietf.org/html/rfc8252)
3. Proof Key for Code Exchange by OAuth Public Clients - [RFC7636](https://tools.ietf.org/html/rfc7636)
4. The OAuth 2.0 Authorization Framework - [RFC6749](https://tools.ietf.org/html/rfc6749)
5. JSON Web Token (JWT) - [RFC7519](https://tools.ietf.org/html/rfc7519)
6. JSON Web Encryption (JWE) - [RFC7516](https://tools.ietf.org/html/rfc7516)
7. JSON Web Signature (JWS) - [RFC7515](https://tools.ietf.org/html/rfc7515)

