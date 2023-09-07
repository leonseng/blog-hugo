---
title: "Generate Self-Signed JSON Web Keys And JSON Web Tokens"
date: 2023-09-07T14:17:35+10:00
draft: false
tags: ['auth', 'jwt']
categories: ['tech']
---

When testing JSON Web Token-based authentication and authorization, relying on third party identity/authorization servers introduces an unnecessary dependency and a potential point of failure, especially for automated tests or continuous integration (CI) pipelines.

Instead, we can speed up the process by generating the data ourselves with the instructions below, creating the following:
- [**JSON Web Keys (JWK)**/**JSON Web Key Sets (JWKS)**](#json-web-key--key-set), and
- [**JSON Web Tokens (JWT)**](#json-web-token)

The following command-line tools will be required:
- [openssl](https://www.openssl.org/)
- [jwker](https://github.com/jphastings/jwker/releases)
- [jq](https://jqlang.github.io/jq/)
- [jwt command-line tool](https://pkg.go.dev/github.com/golang-jwt/jwt/cmd/jwt)

# tl;dr

```
# Create RSA key pair
openssl genrsa -out private.key 2048
openssl rsa -in private.key -pubout -out public.key

# Generate JWK and store it in jwk.json
jwker public.key | jq -S ". += {\"kid\":\"$RANDOM\"}" | tee jwk.json

# Generate JWKS and store it in jwks.json
cat jwk.json | jq '{"keys": [.]}' | tee jwks.json

# Generate JWT and store it in jwt.json
# Add/remove headers and claims as necessary
jwt -key private.key -alg RS256 -sign + \
  -header kid=17334 \
  -claim iss=leonseng.com \
  -claim sub=me \
  -claim aud=you \
  -claim iat=$(date +%s) \
  -claim nbf=$(date +%s) \
  -claim exp=$(date -d "+24hours" +%s) \
  -claim jti=$RANDOM \
  -claim custom=foobar \
  | tee jwt.json
```

# JSON Web Key & JSON Web Key Set

A JWK or JWKS is used to ensure a JWT presented by the client is issued by a trusted authority (authentication). To generate a JWK and JWKS:

1. Create an RSA key pair
    ```
    openssl genrsa -out private.key 2048
    openssl rsa -in private.key -pubout -out public.key
    ```
1. Using [jwker](https://github.com/jphastings/jwker/releases), create a JWK from the public key
    ```
    jwker public.key | jq -S ". += {\"kid\":\"$RANDOM\"}" | tee jwk.json
    ```
    Note the optional addition of the `kid` attribute using [jq](https://jqlang.github.io/jq/). It can be of any value, but the same value must referenced in the JWT header
1. Create the JWKS by placing the JWK in an array under the `keys` attribute
    ```
    cat jwk.json | jq '{"keys": [.]}' | tee jwks.json
    ```

The JWKS should look something like this:
```
$ cat jwks.json
{
  "keys": [
    {
      "e": "AQAB",
      "kid": "17334",
      "kty": "RSA",
      "n": "xNhypnjqaKSEu8rBEuxghS..."
    }
  ]
}
```

# JSON Web Token

A JWT is a token issued by a trusted source, and contains some information on the client/subscriber in the form of `claims` within the payload. JWT is usually embedded in the request as a bearer token in the `Authorization` header, e.g.
```
Authorization: Bearer eyJhbGciOiJSUzI1NiI...
```
An application or layer 7 proxy then parses the JWT to authenticate the client - ensuring it is signed by a trusted authority, as well as authorizing the client by checking the claim values. As the JWT is cryptograhically signed, any attempts by the client to tamper with the JWT claims will invalidate the signature.

To generate a JWT, use the [jwt command-line tool](https://pkg.go.dev/github.com/golang-jwt/jwt/cmd/jwt), referencing the private key created earlier. Headers and claims can be added as required via the `-header` and `-claim` options.
```
jwt -key private.key -alg RS256 -sign + \
  -header kid=17334 \
  -claim iss=leonseng.com \
  -claim sub=me \
  -claim aud=you \
  -claim iat=$(date +%s) \
  -claim nbf=$(date +%s) \
  -claim exp=$(date -d "+24hours" +%s) \
  -claim jti=$RANDOM \
  -claim custom=foobar \
  | tee jwt.json
```

The JWT should look something like this:
```
$ cat jwt.json
eyJhbGciOiJSUzI1NiIsImtpZ...
```

And when decoded:
```
$ cat jwt.json | jwt -show -
Header:
{
    "alg": "RS256",
    "kid": "17334",
    "typ": "JWT"
}
Claims:
{
    "aud": "you",
    "custom": "foobar",
    "exp": "1694174012",
    "iat": "1694087612",
    "iss": "leonseng.com",
    "jti": "33a81d22-f6e2-4a9d-8dc9-9f75c2bfa512",
    "nbf": "1694087612",
    "sub": "me"
}
```
