---
title: "Istio Authorization With Client Certificates"
date: 2022-02-09T14:34:59+11:00
draft: false
tags: ['kubernetes', 'istio', 'mtls', 'auth', 'spiffe']
categories: ['tech']
---

Istio has inbuilt capabilities to perform authentication/authorization against external requests using JWT, as explored in a previous [post](/posts/learning-istio/jwt-auth/), but not everyone uses JWT. So here's an idea of performing authentication/authorization for external requests with just the client certificates presented in the mTLS handshake.

**tl;dr**:

> It's possible to use SPIFFE ID presented in the client certificate, in conjunction with Istio [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) to control access to applications in the service mesh.
>
> However, it opens up to masquerade attacks if someone is able to gain access to the client key pair. So, additional processes may be required if this method is used, e.g. frequent rotation of SPIFFE ID.

# Request Authentication

> The following section is heavily borrowed from this article [Istio proxy SPIFFE validation for inbound connection](https://support.f5.com/csp/article/K65743455), which details generating client certificates with the cluster CA used by Istio.

Every Istio deployment has a cluster Certificate Authority (CA), which is used by `istiod` to sign and issue certificates to all `istio-proxy` sidecars for pod-to-pod mTLS connections. For an application within an Istio service mesh to receive a request from a client external to the service mesh, the client needs to present a certificate signed by the same cluster CA, and contains a SPIFFE identity within the trust domain.

Before we proceed further, here are a couple of things to note:

1. SPIFFE identity within Istio usually has the format `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`, and is presented in the subjectAltName (SAN) field of the client certificate:
    ```
    ...
    subjectAltName = URI:spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>
    ```
1. The trust domain defaults to `cluster.local` on Istio. You can verify this by running
    ```sh
    $ kubectl -n istio-system get configmap istio -o yaml | grep trustDomain
    trustDomain: cluster.local
    ```

## Client Certificate Setup

First, we need the cluster CA key pair, and the root CA certificate if the cluster is using an intermediate CA. These may already exists in the cluster as a Kubernetes Secret `cacerts`, appearing as something like `ca-cert.pem`, `ca-key.pem` and `root-cert.pem` in the `data` field.
```sh
kubectl -n istio-system get secret cacerts -o jsonpath='{.data.ca-cert\.pem}' | base64 -d > ca-cert.pem
kubectl -n istio-system get secret cacerts -o jsonpath='{.data.ca-key\.pem}' | base64 -d > ca-key.pem
kubectl -n istio-system get secret cacerts -o jsonpath='{.data.root-cert\.pem}' | base64 -d > root-cert.pem
```

Next, we create the client key pair:
```sh
# Create client key and CSR
openssl req -new -out curl-spiffe.csr -newkey rsa:2048 -nodes -keyout curl-spiffe.key \
  -subj "/CN=curl.example.com/O=test" -addext "subjectAltName = URI:spiffe://cluster.local/external-client"

# Generate and sign client certificate with cluster CA
openssl x509 -req -in curl-spiffe.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out curl-spiffe.crt -days 365 -sha256 \
  -extensions v3_ca -extfile <(printf "[ v3_ca ]\nsubjectAltName = URI:spiffe://cluster.local/external-client")
```

Note the addition of the SAN extension with the SPIFFE ID `spiffe://cluster.local/external-client`, indicating the client has a workload ID `external-client` and is in the same trust domain `cluster.local` as Istio.
```sh
$ openssl x509 -text -in curl-spiffe.crt
...
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                URI:spiffe://cluster.local/external-client
...
```

The final thing we need is to create a certificate chain containing the client certificate, cluster CA certificate and root CA certificate (if present). `istio-proxy` or `Envoy` requires a full certificate chain to be presented by the client, else the mTLS connection will be rejected.
```sh
cp curl-spiffe.crt curl-spiffe-chain.crt
cat ca-cert.pem >>curl-spiffe-chain.crt
cat root-cert.pem >>curl-spiffe-chain.crt
```

We now have the client key pair `curl-spiffe.key` and `curl-spiffe-chain.crt` ready to go.

## Verifying Client Key Pair

To validate the client key pair, we will deploy an application within the service mesh:
```sh
kubectl create namespace foo

kubectl label ns foo istio-injection=enabled

kubectl -n foo create deployment httpbin --image kennethreitz/httpbin

kubectl -n foo apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
    service: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
EOF
```

and a client external to the service mesh, i.e. without a sidecar, using the `sidecar.istio.io/inject: "false"` label.
```sh
kubectl -n foo apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: curl
  name: curl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: curl
        sidecar.istio.io/inject: "false"
    spec:
      containers:
      - command:
        - tail
        args:
        - "-f"
        - "/dev/null"
        image: curlimages/curl
        name: curl
EOF

# Copy client key pair and CA cert into the client container
CURL_POD=$(kubectl -n foo get pods --selector app=curl -o jsonpath='{.items[0].metadata.name}')
kubectl -n foo cp curl-spiffe.key $CURL_POD:/tmp/
kubectl -n foo cp curl-spiffe-chain.crt $CURL_POD:/tmp/
kubectl -n foo cp ca-cert.pem $CURL_POD:/tmp/
```

Attempting to send a request to the `httpbin` service without providing the key pair will fail as expected:
```sh
$ kubectl -n foo exec $CURL_POD -- curl -sk https://httpbin.foo:8000/get
command terminated with exit code 16
```

Retrying the request with the certificates and key works:
```sh
$ kubectl -n foo exec $CURL_POD -- curl -sk https://httpbin.foo:8000/headers \
  --cacert /tmp/ca-cert.pem \
  --cert /tmp/curl-spiffe-chain.crt \
  --key /tmp/curl-spiffe.key
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.foo:8000",
    "User-Agent": "curl/7.81.0-DEV",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "e54f837029b984fb",
    "X-B3-Traceid": "1d46a423e3fd66a6e54f837029b984fb",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/foo/sa/default;Hash=8d5816f39f390cb936cabe2151aee79d751830ea07dde2deb2900d8134edf7a7;Subject=\"O=test,CN=curl.example.com\";URI=spiffe://cluster.local/external-client"
  }
}
```

Awesome. With authentication out of the way, let's move on to authorizing requests from this external client.

# Request Authorization

We've seen Istio's [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) in action [using information in JWT](/posts/learning-istio/jwt-auth/), and the good news is we can use it here too! The reason we included the SPIFFE ID in the client certificate is because its value gets extracted and can be used for matching in the [source.principals](https://istio.io/latest/docs/reference/config/security/authorization-policy/#Source) field.

We begin by creating an [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) that only allows some other workload (workload ID `someone-else`) to access our application
```sh
kubectl -n foo apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin
spec:
  selector:
    matchLabels:
      app: httpbin
  rules:
  - from:
    - source:
        principals: ["cluster.local/someone-else"]
EOF
```

Attempting to access the application from the external client will fail:
```sh
$ kubectl -n foo exec $CURL_POD -- curl -sk https://httpbin.foo:8000/headers \
  --cacert /tmp/ca-cert.pem \
  --cert /tmp/curl-spiffe-chain.crt \
  --key /tmp/curl-spiffe.key
RBAC: access denied
```

Now, we update the [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) to allow our client's workdload ID `external-client`
```sh
kubectl -n foo apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin
spec:
  selector:
    matchLabels:
      app: httpbin
  rules:
  - from:
    - source:
        principals: ["cluster.local/external-client"]
EOF
```

And the request succeeds as expected:
```sh
$ kubectl -n foo exec $CURL_POD -- curl -sk https://httpbin.foo:8000/headers \
  --cacert /tmp/ca-cert.pem \
  --cert /tmp/curl-spiffe-chain.crt \
  --key /tmp/curl-spiffe.key
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.foo:8000",
    "User-Agent": "curl/7.81.0-DEV",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "5184d2557d92d7c2",
    "X-B3-Traceid": "3d54c4b557de8ac05184d2557d92d7c2",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/foo/sa/default;Hash=8d5816f39f390cb936cabe2151aee79d751830ea07dde2deb2900d8134edf7a7;Subject=\"O=test,CN=curl.example.com\";URI=spiffe://cluster.local/external-client"
  }
}
```

Point proven! That is, if you expose your application directly out of the cluster...

# Now Do It On The Ingress Gateway

Most clusters have their north south traffic through an ingress gateway instead. And applications/clients external to to the cluster will likely have a different CA. Let's try and replicate the above with the traffic going through the ingress gateway.

> Before proceeding, we need to perform some cleanup
> ```
> kubectl -n foo delete authorizationpolicies.security.istio.io httpbin
> ```

## Configuring mTLS On Ingress Gateway

We need the ingress gateway to now handle the mTLS connection with the client, which is well documented [here](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-mutual-tls-ingress-gateway). Start by generating the key pairs for the application and client:
```sh
# Generate new CA key pair
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=org' -keyout org.key -out org.crt

# Generate new key pair for server, signed by CA
openssl req -out httpbin.foo.csr -newkey rsa:2048 -nodes -keyout httpbin.foo.key \
  -subj "/CN=httpbin.foo/O=httpbin organization"
openssl x509 -req -days 365 -CA org.crt -CAkey org.key -set_serial 0 -in httpbin.foo.csr -out httpbin.foo.crt

# Generate new key pair for client with SPIFFE ID, signed by CA
openssl req -new -out curl-spiffe.csr -newkey rsa:2048 -nodes -keyout curl-spiffe.key \
  -subj "/CN=curl.example.com/O=test" -addext "subjectAltName = URI:spiffe://cluster.local/external-client"
openssl x509 -req -in curl-spiffe.csr -CA org.crt -CAkey org.key -CAcreateserial -out curl-spiffe.crt -days 365 -sha256 \
  -extensions v3_ca -extfile <(printf "[ v3_ca ]\nsubjectAltName = URI:spiffe://cluster.local/external-client")

# Create full certificate chain containing client and CA certificate
cp curl-spiffe.crt curl-spiffe-chain.crt
cat org.crt >>curl-spiffe-chain.crt
```

Then, configure the ingress gateway to listen on `httpbin.foo.com:443`, with mTLS enabled:
```sh
kubectl -n istio-system create secret generic httpbin-foo-credential \
  --from-file=tls.key=httpbin.foo.key \
  --from-file=tls.crt=httpbin.foo.crt \
  --from-file=ca.crt=org.crt

kubectl -n istio-system apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: MUTUAL
      credentialName: httpbin-foo-credential
    hosts:
    - httpbin.foo.com
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.foo.com"
  gateways:
  - httpbin-gateway
  http:
  - route:
    - destination:
        port:
          number: 8000
        host: httpbin.foo.svc.cluster.local
EOF
```

We can verify mTLS configuration with a request using the client key pair:
```sh
INGRESS_IP=$(kubectl -n istio-system get svc --selector app=istio-ingressgateway -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
$ curl --resolve httpbin.foo.com:443:$INGRESS_IP -sk https://httpbin.foo.com/headers \
  --cacert org.crt \
  --cert curl-spiffe-chain.crt \
  --key curl-spiffe.key
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.foo.com",
    "User-Agent": "curl/7.68.0",
    "X-B3-Parentspanid": "ac16bebc94fa3dbc",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "47428f1bbb8a30eb",
    "X-B3-Traceid": "76b9206d3d55a16bac16bebc94fa3dbc",
    "X-Envoy-Attempt-Count": "1",
    "X-Envoy-Internal": "true",
    "X-Forwarded-Client-Cert": "Hash=e91a9ae1bf68ea3506ee5926a18ce759a0b34a996d5a8fcfe814828ab261e56b;Cert=\"-----BEGIN%20CERTIFICATE-----%0AMIIDEjCCAfqgAwIBAgIUbF88dqNfNl2KJyyd5JuZZaYpL50wDQYJKoZIhvcNAQEL%0ABQAwJTEVMBMGA1UECgwMZXhhbXBsZSBJbmMuMQwwCgYDVQQDDANvcmcwHhcNMjIw%0AMjA5MjMzOTI1WhcNMjMwMjA5MjMzOTI1WjAqMRkwFwYDVQQDDBBjdXJsLmV4YW1w%0AbGUuY29tMQ0wCwYDVQQKDAR0ZXN0MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIB%0ACgKCAQEA35feAhVqTDnNDRKPfM0a6Vh4Vy79FzPl55m0e1Q%2Fl3xgvkIj%2BAfa788B%0At%2BBpAX4OEFHVYrSGmfjakdJVPP%2BNCHhtcRRl9ndyFLeexN5OxbCOaz8er4FklcJt%0A8VAYt%2BYcb6Yd8sEoYnKPYQSemVH9DQDwlusXA%2Bg61ulF%2BNr0vIq%2Bik26W7ZtaRsH%0A%2BOV6X4QrlBxAps5K8LUxzTz4ROqVjOklHf4AVM2V4ubt7w6qEIBDnZ3VGLUOqT1%2B%0AT59s%2B1U%2BEmI1UTBpDcFlNdWTQ4UXXMVZ5iYyxx317HRt0Boa7LhaP7oC1AUdnhHH%0AqTcB0WMWDF3G0S80nB7M6fK2BKG1PQIDAQABozUwMzAxBgNVHREEKjAohiZzcGlm%0AZmU6Ly9jbHVzdGVyLmxvY2FsL2V4dGVybmFsLWNsaWVudDANBgkqhkiG9w0BAQsF%0AAAOCAQEAXxHhroOtPfaSqu%2ByOTkJg3jWlHQrwYQLOVU7cM1oPFnK%2FXi%2BVgDgqGGe%0ASr80OhHJs%2BkF%2BhSTJOOiUq%2B%2BSUt4ClZo7%2FLPTlK5HvYrVzzgICbzvgdFFEflgd4R%0AincZ2vbODACtcaoXzOkNGzuuScqYm7kFe90iy1XInKw9FhtfE8L7rxMCYzuxKpng%0AgEzFxlt1SWKz%2B%2Fcb%2FfePLiAqqRUpRYLY7SXjlIpgkvW%2F4xQ9nVPtrag%2Fn%2Fl1QMy%2B%0Actq2X46wyuVmN4GY9pLULyjH%2FMhiQDmdJ%2ByU1Sc1%2BXf6QeCpmCb53%2Fi%2BJM1N%2Fkj0%0Azd%2BNeJtM0KIfuR8386r3GDCgXV1wrw%3D%3D%0A-----END%20CERTIFICATE-----%0A\";Subject=\"O=test,CN=curl.example.com\";URI=spiffe://cluster.local/external-client,By=spiffe://cluster.local/ns/foo/sa/default;Hash=3c64fe468fe970fc4c68b4a4dc63a89caf041e2b1db36a27ce4730c7955125a7;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  }
}
```

> Take note of the `X-Forwarded-Client-Cert` header, as it will be important in the next bit.

## Authorization Based On Principal (Spoiler: This Didn't Work)

If we apply the same [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) used in the previous section:
```sh
kubectl -n foo apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin
spec:
  selector:
    matchLabels:
      app: httpbin
  rules:
  - from:
    - source:
        principals: ["cluster.local/external-client"]
EOF
```

```sh
$ curl --resolve httpbin.foo.com:443:$INGRESS_IP -sk https://httpbin.foo.com/headers \
  --cacert org.crt \
  --cert curl-spiffe-chain.crt \
  --key curl-spiffe.key
RBAC: access denied
```

We see the request fail even though our client certificate contains the expected SPIFFE ID `spiffe://cluster.local/external-client`!

This is because the request that reaches our application does not come directly from the external client, but through the ingress gateway. As a separate mTLS handshake occurs between the ingress gateway and the application, the principal of the request as seen by the `istio-sidecar` of our application will actually be the ingress gateway, or `spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account`. This was seen in the `X-Forwarded-Client-Cert` header earlier. What we should do is move the [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) to the ingress gateway instead:
```sh
kubectl -n foo delete authorizationpolicies.security.istio.io httpbin

kubectl -n istio-system apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  rules:
  - from:
    - source:
        principals: ["cluster.local/external-client"]
    to:
    - operation:
        hosts: ["httpbin.foo.com"]
EOF
```

And... that didn't work:
```sh
$ curl --resolve httpbin.foo.com:443:$INGRESS_IP -sk https://httpbin.foo.com/headers \
  --cacert org.crt \
  --cert curl-spiffe-chain.crt \
  --key curl-spiffe.key
RBAC: access denied
```

A closer look at the ingress gateway logs (debug level) reveals that request principals are not available for matching in [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) rules. I'm not sure why, please let me know if you do.

## Authorization Based On XFCC header

For now, let's try another way - by checking the value of `x-forwarded-client-cert` (XFCC) header ends with the client SPIFFE ID (the sidecar proxy should append the last client ID to the header):
```sh
kubectl -n istio-system apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  rules:
  - to:
    - operation:
        hosts: ["httpbin.foo.com"]
    when:
    - key: request.headers[x-forwarded-client-cert]
      values: ["*URI=spiffe://cluster.local/external-client"]
EOF
```

```sh
$ curl --resolve httpbin.foo.com:443:$INGRESS_IP -sk https://httpbin.foo.com/headers \
  --cacert org.crt \
  --cert curl-spiffe-chain.crt \
  --key curl-spiffe.key
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.foo.com",
    "User-Agent": "curl/7.68.0",
    "X-B3-Parentspanid": "e70694db7976f375",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "cd4549198a32d9bc",
    "X-B3-Traceid": "fd48181fb82a2185e70694db7976f375",
    "X-Envoy-Attempt-Count": "1",
    "X-Envoy-Internal": "true",
    "X-Forwarded-Client-Cert": "Hash=e91a9ae1bf68ea3506ee5926a18ce759a0b34a996d5a8fcfe814828ab261e56b;Cert=\"-----BEGIN%20CERTIFICATE-----%0AMIIDEjCCAfqgAwIBAgIUbF88dqNfNl2KJyyd5JuZZaYpL50wDQYJKoZIhvcNAQEL%0ABQAwJTEVMBMGA1UECgwMZXhhbXBsZSBJbmMuMQwwCgYDVQQDDANvcmcwHhcNMjIw%0AMjA5MjMzOTI1WhcNMjMwMjA5MjMzOTI1WjAqMRkwFwYDVQQDDBBjdXJsLmV4YW1w%0AbGUuY29tMQ0wCwYDVQQKDAR0ZXN0MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIB%0ACgKCAQEA35feAhVqTDnNDRKPfM0a6Vh4Vy79FzPl55m0e1Q%2Fl3xgvkIj%2BAfa788B%0At%2BBpAX4OEFHVYrSGmfjakdJVPP%2BNCHhtcRRl9ndyFLeexN5OxbCOaz8er4FklcJt%0A8VAYt%2BYcb6Yd8sEoYnKPYQSemVH9DQDwlusXA%2Bg61ulF%2BNr0vIq%2Bik26W7ZtaRsH%0A%2BOV6X4QrlBxAps5K8LUxzTz4ROqVjOklHf4AVM2V4ubt7w6qEIBDnZ3VGLUOqT1%2B%0AT59s%2B1U%2BEmI1UTBpDcFlNdWTQ4UXXMVZ5iYyxx317HRt0Boa7LhaP7oC1AUdnhHH%0AqTcB0WMWDF3G0S80nB7M6fK2BKG1PQIDAQABozUwMzAxBgNVHREEKjAohiZzcGlm%0AZmU6Ly9jbHVzdGVyLmxvY2FsL2V4dGVybmFsLWNsaWVudDANBgkqhkiG9w0BAQsF%0AAAOCAQEAXxHhroOtPfaSqu%2ByOTkJg3jWlHQrwYQLOVU7cM1oPFnK%2FXi%2BVgDgqGGe%0ASr80OhHJs%2BkF%2BhSTJOOiUq%2B%2BSUt4ClZo7%2FLPTlK5HvYrVzzgICbzvgdFFEflgd4R%0AincZ2vbODACtcaoXzOkNGzuuScqYm7kFe90iy1XInKw9FhtfE8L7rxMCYzuxKpng%0AgEzFxlt1SWKz%2B%2Fcb%2FfePLiAqqRUpRYLY7SXjlIpgkvW%2F4xQ9nVPtrag%2Fn%2Fl1QMy%2B%0Actq2X46wyuVmN4GY9pLULyjH%2FMhiQDmdJ%2ByU1Sc1%2BXf6QeCpmCb53%2Fi%2BJM1N%2Fkj0%0Azd%2BNeJtM0KIfuR8386r3GDCgXV1wrw%3D%3D%0A-----END%20CERTIFICATE-----%0A\";Subject=\"O=test,CN=curl.example.com\";URI=spiffe://cluster.local/external-client,By=spiffe://cluster.local/ns/foo/sa/default;Hash=3c64fe468fe970fc4c68b4a4dc63a89caf041e2b1db36a27ce4730c7955125a7;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  }
}
```

The request got through! **But is it safe?!**

I updated the [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) to accept another workload ID `someone-else`:
```sh
kubectl -n istio-system apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  rules:
  - to:
    - operation:
        hosts: ["httpbin.foo.com"]
    when:
    - key: request.headers[x-forwarded-client-cert]
      values: ["*URI=spiffe://cluster.local/someone-else"]
EOF
```

and attempted to perform a requests from my client with the `x-forwarded-client-cert` header set to `URI=spiffe://cluster.local/someone-else`
```sh
$ curl --resolve httpbin.foo.com:443:$INGRESS_IP -sk https://httpbin.foo.com/headers \
  -H x-forwarded-client-cert:URI=spiffe://cluster.local/someone-else \
  --cacert org.crt \
  --cert curl-spiffe-chain.crt \
  --key curl-spiffe.key
RBAC: access denied
```

Good. Looking at the ingress gateway logs, it seems like the ingress gateway sanitizes the `x-forwarded-client-cert` header from the client.
```log
2022-02-10T00:59:22.599433Z     debug   envoy rbac      checking request: requestedServerName: httpbin.foo.com, sourceIP: 10.233.90.0:16569, directRemoteIP: 10.233.90.0:16569, remoteIP: 10.233.90.0:16569,localAddress: 10.233.92.41:8443, ssl: uriSanPeerCertificate: spiffe://cluster.local/external-client, dnsSanPeerCertificate: , subjectPeerCertificate: O=test,CN=curl.example.com, headers: ':method', 'GET'
':path', '/headers'
':scheme', 'https'
':authority', 'httpbin.foo.com'
'user-agent', 'curl/7.68.0'
'accept', '*/*'
'x-forwarded-client-cert', 'Hash=e91a9ae1bf68ea3506ee5926a18ce759a0b34a996d5a8fcfe814828ab261e56b;Cert="-----BEGIN%20CERTIFICATE-----%0AMIIDEjCCAfqgAwIBAgIUbF88dqNfNl2KJyyd5JuZZaYpL50wDQYJKoZIhvcNAQEL%0ABQAwJTEVMBMGA1UECgwMZXhhbXBsZSBJbmMuMQwwCgYDVQQDDANvcmcwHhcNMjIw%0AMjA5MjMzOTI1WhcNMjMwMjA5MjMzOTI1WjAqMRkwFwYDVQQDDBBjdXJsLmV4YW1w%0AbGUuY29tMQ0wCwYDVQQKDAR0ZXN0MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIB%0ACgKCAQEA35feAhVqTDnNDRKPfM0a6Vh4Vy79FzPl55m0e1Q%2Fl3xgvkIj%2BAfa788B%0At%2BBpAX4OEFHVYrSGmfjakdJVPP%2BNCHhtcRRl9ndyFLeexN5OxbCOaz8er4FklcJt%0A8VAYt%2BYcb6Yd8sEoYnKPYQSemVH9DQDwlusXA%2Bg61ulF%2BNr0vIq%2Bik26W7ZtaRsH%0A%2BOV6X4QrlBxAps5K8LUxzTz4ROqVjOklHf4AVM2V4ubt7w6qEIBDnZ3VGLUOqT1%2B%0AT59s%2B1U%2BEmI1UTBpDcFlNdWTQ4UXXMVZ5iYyxx317HRt0Boa7LhaP7oC1AUdnhHH%0AqTcB0WMWDF3G0S80nB7M6fK2BKG1PQIDAQABozUwMzAxBgNVHREEKjAohiZzcGlm%0AZmU6Ly9jbHVzdGVyLmxvY2FsL2V4dGVybmFsLWNsaWVudDANBgkqhkiG9w0BAQsF%0AAAOCAQEAXxHhroOtPfaSqu%2ByOTkJg3jWlHQrwYQLOVU7cM1oPFnK%2FXi%2BVgDgqGGe%0ASr80OhHJs%2BkF%2BhSTJOOiUq%2B%2BSUt4ClZo7%2FLPTlK5HvYrVzzgICbzvgdFFEflgd4R%0AincZ2vbODACtcaoXzOkNGzuuScqYm7kFe90iy1XInKw9FhtfE8L7rxMCYzuxKpng%0AgEzFxlt1SWKz%2B%2Fcb%2FfePLiAqqRUpRYLY7SXjlIpgkvW%2F4xQ9nVPtrag%2Fn%2Fl1QMy%2B%0Actq2X46wyuVmN4GY9pLULyjH%2FMhiQDmdJ%2ByU1Sc1%2BXf6QeCpmCb53%2Fi%2BJM1N%2Fkj0%0Azd%2BNeJtM0KIfuR8386r3GDCgXV1wrw%3D%3D%0A-----END%20CERTIFICATE-----%0A";Subject="O=test,CN=curl.example.com";URI=spiffe://cluster.local/external-client'
'x-forwarded-for', '10.233.90.0'
'x-forwarded-proto', 'https'
'x-envoy-internal', 'true'
'x-request-id', '5cd3255c-618a-4db5-aae6-3ddfb24c9631'
'x-envoy-decorator-operation', 'httpbin.foo.svc.cluster.local:8000/*'
'x-envoy-peer-metadata', 'ChQKDkFQUF9DT05UQUlORVJTEgIaAAoaCgpDTFVTVEVSX0lEEgwaCkt1YmVybmV0ZXMKHAoNSVNUSU9fVkVSU0lPThILGgkxLjkuOC1hbTEKvwMKBkxBQkVMUxK0AyqxAwodCgNhcHASFhoUaXN0aW8taW5ncmVzc2dhdGV3YXkKEwoFY2hhcnQSChoIZ2F0ZXdheXMKFAoIaGVyaXRhZ2USCBoGVGlsbGVyCjYKKWluc3RhbGwub3BlcmF0b3IuaXN0aW8uaW8vb3duaW5nLXJlc291cmNlEgkaB3Vua25vd24KGQoFaXN0aW8SEBoOaW5ncmVzc2dhdGV3YXkKGQoMaXN0aW8uaW8vcmV2EgkaB2RlZmF1bHQKMAobb3BlcmF0b3IuaXN0aW8uaW8vY29tcG9uZW50EhEaD0luZ3Jlc3NHYXRld2F5cwohChFwb2QtdGVtcGxhdGUtaGFzaBIMGgo2OTQ5Y2RkNjVjChIKB3JlbGVhc2USBxoFaXN0aW8KOQofc2VydmljZS5pc3Rpby5pby9jYW5vbmljYWwtbmFtZRIWGhRpc3Rpby1pbmdyZXNzZ2F0ZXdheQovCiNzZXJ2aWNlLmlzdGlvLmlvL2Nhbm9uaWNhbC1yZXZpc2lvbhIIGgZsYXRlc3QKIgoXc2lkZWNhci5pc3Rpby5pby9pbmplY3QSBxoFZmFsc2UKLwoETkFNRRInGiVpc3Rpby1pbmdyZXNzZ2F0ZXdheS02OTQ5Y2RkNjVjLW00amI2ChsKCU5BTUVTUEFDRRIOGgxpc3Rpby1zeXN0ZW0KXQoFT1dORVISVBpSa3ViZXJuZXRlczovL2FwaXMvYXBwcy92MS9uYW1lc3BhY2VzL2lzdGlvLXN5c3RlbS9kZXBsb3ltZW50cy9pc3Rpby1pbmdyZXNzZ2F0ZXdheQoXChFQTEFURk9STV9NRVRBREFUQRICKgAKJwoNV09SS0xPQURfTkFNRRIWGhRpc3Rpby1pbmdyZXNzZ2F0ZXdheQ=='
'x-envoy-peer-metadata-id', 'router~10.233.92.41~istio-ingressgateway-6949cdd65c-m4jb6.istio-system~istio-system.svc.cluster.local'
, dynamicMetadata:
```

Documentation [here](https://istio.io/latest/docs/ops/configuration/traffic-management/network-topologies/#configuring-x-forwarded-client-cert-headers) further confirms that the default value for `forwardClientCertDetails` is set to `SANITIZE` on gateways. More information on this behaviour can be found in Envoy documentations (see [here](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-forwarded-client-cert) and [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-forward-client-cert-details)).

# Conclusion

I've demonstrated that it is possible to perform some level of authorization based on the SPIFFE ID presented in the client certificate, both at the pod and at the ingress gateway level.

However, it should be used with caution, as someone malicious who has access to the client key pair will be able to pretend to be a legitimate client. There's no inbuilt certificate rotation here, so treat it like a username/password, and rotate often the SPIFFE ID often!
