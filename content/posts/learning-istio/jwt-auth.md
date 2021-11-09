---
title: "Learning Istio | JWT Auth"
date: 2021-11-06T21:32:16+11:00
draft: false
series: ["Learning Istio"]
tags: ['kubernetes', 'istio', 'jwt']
categories: ['tech']
---

In this post, we will be looking at how Istio handles end user authentication/authorization based on JSON Web Tokens (JWT). JWT is commonly used in OAuth2.0 flows to specify the resources a client has access to, but there are a couple of things to verify before the client is given access:

1. Is the JWT issued by the right party
1. Is the client who they claim to be

The logic for the checks above are usually coded into the application.

Alternatively, as we will discover in this post, we can simplify the application by offloading this to Istio using the [RequestAuthentication](https://istio.io/latest/docs/reference/config/security/request_authentication/) and [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) resources.

# Setup

## Test application

For the test application, I will be using the [httpbin](https://hub.docker.com/r/kennethreitz/httpbin/) image as it exposes a `/headers` endpoint which prints out the headers as seen by the application, allowing us to see the changes done by the sidecar proxy.
```sh
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: httpbin
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
      - image: kennethreitz/httpbin
        name: httpbin
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: httpbin
  name: httpbin
spec:
  ports:
  - name: http  # this is important. See Additional Learnings below
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: httpbin
EOF
```

## JWT provider

The JWT used in most of the examples in this post is obtained from Auth0 via the [client credentials flow](https://auth0.com/docs/authorization/flows/call-your-api-using-the-client-credentials-flow):
```sh
# Get access token
$ BASE_URL=https://leonseng.au.auth0.com
$ APP_CLIENT_ID=<Auth0 app client ID>
$ APP_CLIENT_SECRET=<Auth0 app client secret>
$ API_IDENTIFIER=<Auth0 API identifier>
$ ACCESS_TOKEN=$(curl -s --request POST \
  --url "$BASE_URL/oauth/token" \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=client_credentials \
  --data client_id=$APP_CLIENT_ID \
  --data client_secret=$APP_CLIENT_SECRET \
  --data audience=$API_IDENTIFIER | jq -r .access_token)

# Check content of JWT
$ jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(echo $ACCESS_TOKEN)
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "qv9xb5h9OYy-uJgVyDEyx"
}
{
  "iss": "https://leonseng.au.auth0.com/",
  "sub": "QWPjwvmVTLVJiQejcPJim0CKR3pxtgd3@clients",
  "aud": "istio-jwt-test",
  "iat": 1636331240,
  "exp": 1636331300,
  "azp": "QWPjwvmVTLVJiQejcPJim0CKR3pxtgd3",
  "scope": "read:database write:database",
  "gty": "client-credentials"
}
```

# RequestAuthentication

Istio's [RequestAuthentication](https://istio.io/latest/docs/reference/config/security/request_authentication/) is responsible for validating the JWT in a request is signed by the expected issuer, and that the payload has not been tampered with.

Below is an example where we specify the JWT issuer and the JSON Web Key Set (JWKS) for JWT validation. The decoded JWT payload can be passed onto the application in a HTTP header via the `outputPayloadToHeader` field, allowing application to access the trusted claim without having to perform token validation itself.

```sh
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: "httpbin-jwt-req-auth"
spec:
  selector:
    matchLabels:
      app: httpbin
  jwtRules:
  - issuer: "https://leonseng.au.auth0.com/"
    jwksUri: "https://leonseng.au.auth0.com/.well-known/jwks.json"
    outputPayloadToHeader: x-jwt-payload
EOF
```

Here's a test to show that our request with the right JWT can go through
```sh
$ RESPONSE=$(curl -s -H "Authorization: Bearer $ACCESS_TOKEN" httpbin/headers)
$ echo $RESPONSE | jq .
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin",
    "User-Agent": "curl/7.79.1-DEV",
    "X-B3-Parentspanid": "40f941f2847fb38d",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "7c93024a6eb09cf5",
    "X-B3-Traceid": "38ab74cb85a129d240f941f2847fb38d",
    "X-Envoy-Attempt-Count": "1",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=dc1fd96b48b91947ef1bfeeb6a9755164343eb982eeb2d29373e3521a90350dc;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default",
    "X-Jwt-Payload": "eyJpc3MiOiJodHRwczovL2xlb25zZW5nLmF1LmF1dGgwLmNvbS8iLCJzdWIiOiJRV1Bqd3ZtVlRMVkppUWVqY1BKaW0wQ0tSM3B4dGdkM0BjbGllbnRzIiwiYXVkIjoiaXN0aW8tand0LXRlc3QiLCJpYXQiOjE2MzY0MjkwODUsImV4cCI6MTYzNjUxNTQ4NSwiYXpwIjoiUVdQand2bVZUTFZKaVFlamNQSmltMENLUjNweHRnZDMiLCJzY29wZSI6InJlYWQ6ZGF0YWJhc2Ugd3JpdGU6ZGF0YWJhc2UiLCJndHkiOiJjbGllbnQtY3JlZGVudGlhbHMifQ"
  }
}
```

We can see that the decoded payload is accessible by the application in the `X-Jwt-Payload` header after performing a base64 decode
```sh
$ FORWARDED_PAYLOAD=$(echo $RESPONSE | jq -r '.headers."X-Jwt-Payload"')
$ echo $FORWARDED_PAYLOAD | base64 -d | jq .
base64: invalid input
{
  "iss": "https://leonseng.au.auth0.com/",
  "sub": "QWPjwvmVTLVJiQejcPJim0CKR3pxtgd3@clients",
  "aud": "istio-jwt-test",
  "iat": 1636429085,
  "exp": 1636515485,
  "azp": "QWPjwvmVTLVJiQejcPJim0CKR3pxtgd3",
  "scope": "read:database write:database",
  "gty": "client-credentials"
}
```

Malformed, expired and JWT issued by other issuers will be rejected:
```sh
# Malformed token
$ curl -s -H "Authorization: Bearer bad" httpbin/headers
Jwt is not in the form of Header.Payload.Signature with two dots and 3 sections

# Expired token
$ curl -s -H "Authorization: Bearer $ACCESS_TOKEN" httpbin/headers
Jwt is expired

# Valid JWT from another issuer - jwt.io
INVALID_TOKEN=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.NHVaYe26MbtOYhSKkoKYdFVomg4i8ZJd8_-RU8VNbftc4TSMb4bXP3l3YlNWACwyXPGffz5aXHc6lty1Y2t4SWRqGteragsVdZufDn5BlnJl9pdR_kdVFUsra2rWKEofkZeIC4yWytE58sMIihvo9H1ScmmVwBcQP6XETqYd0aSHp1gOa9RdUPDvoXQ5oqygTqVtxaDr6wUFKrKItgBMzWIdNZ6y7O9E0DhEPTbE9rfBo6KTFsHAZnMg4k68CDp2woYIaXbmYTWcvbzIuHO7_37GT79XdIwkm95QJ7hYC9RiwrV7mesbY4PAahERJawntho0my942XheVLmGwLMBkQ
$ curl -s -H "Authorization: Bearer $INVALID_TOKEN" httpbin/headers
Jwks doesn't have key to match kid or alg from Jwt
```

Requests without JWT is expected to fail, but **it didn't**
```sh
$ curl -s httpbin/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin",
    "User-Agent": "curl/7.79.1-DEV",
    "X-B3-Parentspanid": "de8e7a515c92e784",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "a2994910ea2fd7f2",
    "X-B3-Traceid": "11f49e2d3230c711de8e7a515c92e784",
    "X-Envoy-Attempt-Count": "1",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=b01278f0dac370955d49f07b1484118c6581fea591a3843d5ccd341ef7b872e6;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default"
  }
}
```

This was unexpected to me, but it is a [documented behaviour](https://istio.io/latest/docs/reference/config/security/request_authentication/):

> A request that does not contain any authentication credentials will be accepted but will not have any authenticated identity. To restrict access to authenticated requests only, this should be accompanied by an authorization rule.

---

# AuthorizationPolicy

[AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) further extends RBAC through the configuration of more granular rules, covering:
1. [who is the requester](#who-is-the-requester)
1. [what the requester is trying to do](#what-is-the-requester-trying-to-do)
1. [additional conditions in the JWT](#additional-conditions-in-the-jwt)

Below is an example which we will be using
```sh
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin-authz-policy
spec:
  selector:
    matchLabels:
      app: httpbin
  rules:
  - from:
    - source:
        requestPrincipals: ["https://leonseng.au.auth0.com//QWPjwvmVTLVJiQejcPJim0CKR3pxtgd3@clients"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/headers"]
    when:
    - key: request.auth.claims[aud]
      values: ["httpbin"]
    - key: request.auth.claims[scope]
      values: ["write:database, fetch:email"]
EOF
```

A quick test to verify that it hasn't broken our setup
```sh
$ curl -s -H "Authorization: Bearer $ACCESS_TOKEN" httpbin/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin",
    "User-Agent": "curl/7.79.1-DEV",
    "X-B3-Parentspanid": "60c70087740ac4fa",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "a4f01ac9e90d5ac5",
    "X-B3-Traceid": "afcacf8ed3355d7160c70087740ac4fa",
    "X-Envoy-Attempt-Count": "1",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=72d075873d9fb6b17553f5b428fac8ad0168162e917b74f44c42b7e145267507;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default",
    "X-Jwt-Payload": "eyJpc3MiOiJodHRwczovL2xlb25zZW5nLmF1LmF1dGgwLmNvbS8iLCJzdWIiOiJRV1Bqd3ZtVlRMVkppUWVqY1BKaW0wQ0tSM3B4dGdkM0BjbGllbnRzIiwiYXVkIjoiaXN0aW8tand0LXRlc3QiLCJpYXQiOjE2MzYzNzUxNDMsImV4cCI6MTYzNjQ2MTU0MywiYXpwIjoiUVdQand2bVZUTFZKaVFlamNQSmltMENLUjNweHRnZDMiLCJzY29wZSI6InJlYWQ6ZGF0YWJhc2Ugd3JpdGU6ZGF0YWJhc2UiLCJndHkiOiJjbGllbnQtY3JlZGVudGlhbHMifQ"
  }
}
```

## Who is the requester

The identity of the requester using JWT can be specified in the `requestPrincipals` field as documented [here](https://istio.io/latest/docs/reference/config/security/authorization-policy/#Source). Istio constructs the identity from the JWT payload values in the format of `<iss>/<sub>`. However, if you just want to enforce the presence of a valid JWT (regardless of the identity), `requestPrincipals` can be set to `[*]`.
```sh
from:
- source:
    requestPrincipals: ["https://leonseng.au.auth0.com//QWPjwvmVTLVJiQejcPJim0CKR3pxtgd3@clients"]  # iss/sub
```

Requests without a JWT or with a different user/subject will be denied
```sh
# No JWT provided
$ curl -s httpbin/headers
RBAC: access denied

# JWT with different user/subject
$ jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(echo $ACCESS_TOKEN)
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "qv9xb5h9OYy-uJgVyDEyx"
}
{
  "iss": "https://leonseng.au.auth0.com/",
  "sub": "uoPlJVCJID9UxJS1jdOMPcr9Gmz2TGgP@clients",
  "aud": "istio-jwt-test",
  "iat": 1636375387,
  "exp": 1636461787,
  "azp": "uoPlJVCJID9UxJS1jdOMPcr9Gmz2TGgP",
  "scope": "read:database write:database",
  "gty": "client-credentials"
}
$ curl -s -H "Authorization: Bearer $ACCESS_TOKEN" httpbin/headers
RBAC: access denied
```

## What is the requester trying to do

Next, we can [restrict which HTTP verbs and path a requester has access to](https://istio.io/latest/docs/reference/config/security/authorization-policy/#Operation). In our example, we are only allowing `GET` requests to `/headers` path
```sh
to:
- operation:
    methods: ["GET"]
    paths: ["/headers"]
```

Request using another HTTP verb and/or accessing another path will not be allowed
```sh
# POST to /post
$ curl -s -X POST -H "Authorization: Bearer $ACCESS_TOKEN" httpbin/post
RBAC: access denied
```

## Additional conditions in the JWT

Lastly, Istio also enables the evaluation of additional [conditions against the JWT claims](https://istio.io/latest/docs/reference/config/security/conditions/). The `AuthorizationPolicy` applied is checking against the claims in the JWT payload
```sh
when:
- key: request.auth.claims[aud]
  values: ["istio-jwt-test"]
- key: request.auth.claims[scope]
  values: ["write:database"]
```

In this example, we send a request with the scope `read:database`, which will be rejected as the `AuthorizationPolicy` is expecting a `write:database` scope
```sh
$ jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(echo $ACCESS_TOKEN)
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "qv9xb5h9OYy-uJgVyDEyx"
}
{
  "iss": "https://leonseng.au.auth0.com/",
  "sub": "xYvKrg4QHLA5bsna0Hg1MYiD3itPY1gC@clients",
  "aud": "istio-jwt-test",
  "iat": 1636376574,
  "exp": 1636462974,
  "azp": "xYvKrg4QHLA5bsna0Hg1MYiD3itPY1gC",
  "scope": "read:database",
  "gty": "client-credentials"
}
$ curl -s -X POST -H "Authorization: Bearer $ACCESS_TOKEN" httpbin/post
RBAC: access denied
```

---

# Additional learnings

Here's a list of things that were picked up during my tests that weren't immediately intuitive:

1. Istio requires the port name in the `Service` to be prefixed with the protocol as described [here](https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/). Failing to adhere to the naming convention will break the authentication feature provided by the [RequestAuthentication](https://istio.io/latest/docs/reference/config/security/request_authentication/) resource. This can be caught by running `istioctl analyze` in the application namespace, which reveals a [PortNameIsNotUnderNamingConvention](https://istio.io/latest/docs/reference/config/analysis/ist0118/) message:
    ```sh
    $ istioctl analyze
    <snipped>
    Info [IST0118] (Service httpbin.default) Port name (port: 80, targetPort: 80) doesn't follow the naming convention of Istio port.
    ```
1. Defining a [RequestAuthentication](https://istio.io/latest/docs/reference/config/security/request_authentication/) alone does not stop requests without JWT, the requests just won't have identities tied to them. Augment it with a [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) to enforce the presence of JWT.
1. The `scope` field in a JWT can contain multiple scopes in a space delimited format,
    ```
    scope: read:database write:database
    ```
    Fortunately, Istio recognizes that and separates the string into multiple scopes. This allows us to match individual scopes from the `scope` field without having to do string manipulations.
1. To assist with troubleshooting, set the log level for the `jwt` and `rbac` loggers to `debug`, which will produce more logs on JWT validation and RBAC enforcement in the application sidecar proxy.
    ```sh
    istioctl proxy-config log $(k get pods -l app=httpbin -o jsonpath='{.items[*].metadata.name}') --level jwt:debug rbac:debug
    ```
