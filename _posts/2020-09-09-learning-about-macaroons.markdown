---
layout: post
title: "Learning about Macaroons"
date: 2020-09-09 15:00:00 +0000
categories: oauth2
---

Today I was trying to learn a bit more about one of the new features in AM 7: Macaroon token support. The official
[documentation] gives a basic understanding of what this token format is all about, but if that wasn't enough, I've
found [Neil's blog] post about it a bit more enlightening (still waiting on the second part of the series though 😀).

## The gist

The basic use-case would look something like this:
* The OAuth2 client obtains a long lived access token that has loads of permissions associated with it. This token is
in the Macaroon token format.
* The OAuth2 client either updates the token itself, or asks AM via REST to add some _caveats_ to the access token.
* The outcome of the previous operation is a new access token in the same Macaroon token format, but with the caveats
included.
* The client attempts to access resources with the new token, and the caveats embedded into the token will restrict the
overall access the client can obtain.

## So what are these caveats?

 Think of them as additional restrictions that can be placed on the original access token.
All caveats must be satisfied at the time of use, otherwise the token won't be considered valid. There are two main
types of caveats, so let's go through them now.

### First party caveats

These can be satisfied locally. AM by default supports the following caveats:
* **aud**: If the access token was originally issued for more than one audience, this caveat can select a subset of
them.
* **cnf**: The access token can be bound to a particular client certificate. [RFC 8705][rfc8705] describes a mechanism
to bind a certificate to the access token, so that only the holder of the client certificate is able to use the
access token. This particular caveat allows obtaining an access token without binding it to a client certificate at
the time of the token issuance, and bind it later to a certificate at the time of use.
* **exp**: The expiration time of the original access token can be reduced to a really short amount of time.
* **scope**: Reduce the number of scopes the access token has.

As you can see caveats can help to harden your OAuth2 deployments quite significantly. With Macaroon tokens the main
responsibility of the client is to keep the original Macaroon token safe. Whenever the client needs to actually use
the access token, it can add new caveats to limit the exposure of the access tokens. Even if the access tokens are
captured and replayed by a malicious party, they will struggle to use them for unintended purposes.

### Third party caveats

These caveats cannot be satisfied by the authorization server, instead the authorization server will expect a thing
called a _discharge Macaroon_. The discharge Macaroon is essentially a Macaroon token issued by the third party that
will certify that the caveat has been indeed satisfied.

When a third party caveat is added to the Macaroon, the third party's signing key will be encrypted and put into the
updated Macaroon token. The discharge Macaroon issued by the third party then gets signed with that key. When the
authorization server receives the Macaroon with the third party caveat, it will verify the signature of the discharge
Macaroon and ensure that the signing key corresponds to the encrypted key stored in the Macaroon access token.

The discharge Macaroon itself is provided to the authorization server in the `X-Discharge-Macaroon` request header.

### Example usage

The Macaroon token format can be enabled quite easily on the OAuth2 Provider service by simply changing the `Use
Macaroon Access and Refresh Tokens` setting. Once that's done you just have to obtain an access token for your OAuth2
client:

```shell
curl --request POST \
  --url https://openam.dev:443/oauth2/access_token \
  --header 'authorization: Basic bXljbGllbnQ6cGFzc3dvcmQ=' \
  --data username=demo \
  --data password=Ch4ng31t \
  --data grant_type=password \
  --data 'scope=uid givenName openid profile'
```

The caveats can be added to the access token either by using the REST API, or by using a Macaroon library.

#### REST API

Sadly the REST API is not part of the API explorer, so we can only rely on the [endpoint documentation], but that one
does not provide an example for the `restrict` action... After some reverse engineering I've found that first party
caveats can be added like this for example:

```shell
curl --request POST \
  --url 'https://openam.dev:443/json/token/macaroon?_action=restrict' \
  --header 'content-type: application/json' \
  --data '{
    "macaroon": "<macaroon-to-update>",
    "caveat": {
        "type": "first-party",
        "identifier": {
            "scope": "givenName uid",
            "exp": 1599666108,
            "aud": "https://openam.dev:443",
            "cnf": {
                "x5t#S256": "<ca-certificate-sha256-hash>"
            }
        }
    }
}'
```

To add a third party caveat, this command does the trick:
```shell
curl --request POST \
  --url 'https://openam.dev:443/json/token/macaroon?_action=restrict' \
  --header 'content-type: application/json' \
  --data '{
    "macaroon": "<macaroon-to-update>",
    "caveat": {
        "type": "third-party",
        "location": "https://third.party.restriction/",
        "key": "<base64url-encoded-256-bit-hmac-key>",
        "identifier": "terms-accepted"
    }
}'
```

#### Macaroon library

For sake of interoperability and simplicity one should be able to use the Macaroon library developed by ForgeRock.
The Maven POM snippet required for AM 7.0.0 is the following:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.forgerock.commons</groupId>
            <artifactId>commons-bom</artifactId>
            <version>26.0.0-20200727141957-8a360d1</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
    </dependencies>
</dependencyManagement>
<dependencies>
    <dependency>
        <groupId>org.forgerock.commons</groupId>
        <artifactId>forgerock-macaroons</artifactId>
    </dependency>
</dependencies>
```

Adding first party caveats boils down to something as simple as:

```java
Macaroon macaroon = Macaroon.deserialize("<oauth2-access-token-macaroon-that-has-uid-scope>");
macaroon.addFirstPartyCaveat(json(object(field("scope", "uid"))));
System.out.println(macaroon.serialize()); // prints the updated Macaroon token that will only have access to uid scope
```

Adding third party caveats are just a little bit more complicated:
```java
SecretKey keyToUseWhenCreatingDischargeMacaroon = Macaroon.generateKey();
macaroon.addThirdPartyCaveat("https://third.party.restriction/",
        keyToUseWhenCreatingDischargeMacaroon.getEncoded(), "terms-accepted".getBytes(UTF_8));
System.out.println(macaroon.serialize());
```

Creating the discharge Macaroon then can be accomplished by calling:
```java
Macaroon.create(keyToUseWhenCreatingDischargeMacaroon,
        "https://third.party.restriction/", "terms-accepted".getBytes(UTF_8)).serialize());
```

### Macaroon introspection

Once you have all the necessary caveats added to your Macaroon, getting an introspection response from AM is pretty
simple, you just have to add the discharge Macaroon as a request header (multiple discharge Macaroons are also
supported if you need more than one third party caveats):

```shell
curl --request POST \
  --url https://openam.dev:443/oauth2/introspect \
  --header 'content-type: application/x-www-form-urlencoded' \
  --header 'x-discharge-macaroon: <discharge-macaroon>' \
  --data token=<macaroon-with-third-party-caveat> \
  --data client_id=myclient \
  --data client_secret=password
```

### Fun facts

* The `restrict` action currently only supports adding one type of caveat at the time, so if you need to add both first
party and third party caveats, you will need to make multiple REST calls.
* The /json/token/macaroon endpoint doesn't actually validate the OAuth2 access tokens whatsoever. You can inspect and
restrict macaroon tokens using these endpoints all day long, but whether they actually correspond to valid and active
access tokens in AM, you will only figure out when you make the introspection/tokeninfo/userinfo/etc calls.

[documentation]: https://backstage.forgerock.com/docs/am/7/oauth2-guide/oauth2-macaroons.html
[Neil's blog]: https://neilmadden.blog/2020/07/29/least-privilege-with-less-effort-macaroon-access-tokens-in-am-7-0/
[rfc8705]: https://tools.ietf.org/html/rfc8705
[endpoint documentation]: https://backstage.forgerock.com/docs/am/7/oauth2-guide/varlist-oauth2-introspect-macaroon-endpoint.html
