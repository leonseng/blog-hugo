---
title: "F5 Nginx Ingress Controller as Ingress Gateway for Istio Service Mesh"
date: 2022-05-24T15:21:10+10:00
draft: false
tags: ['nginx', 'f5', 'ingress', 'istio', 'kubernetes']
categories: ['tech']
---

---

> Disclaimer:
> The NGINX Ingress Controller referenced in this post is the [F5 NGINX Ingress Controller](https://www.nginx.com/products/nginx-ingress-controller/), not [the one by the Kubernetes community](https://kubernetes.github.io/ingress-nginx/).

Using F5 NGINX Ingress Controller (henceforth known as **NGINX IC** for brevity) as ingress to a Kubernetes cluster secured by Istio service mesh, with strict mTLS policy configured, presents a hurdle - how does **NGINX IC** participate in the mTLS certificate exchange with services/applications in the Istio service mesh?

In this post, let's go through two methods of integrating **NGINX IC** with Istio service mesh:
1. [**NGINX IC** with Istio sidecar](#nginx-ic-with-istio-sidecar)
1. [**NGINX IC** with client TLS keypair issued by Istio CA](#nginx-ic-with-client-cert-issued-by-istio-ca)

# NGINX IC with Istio sidecar

The first method injects an Istio sidecar into the **NGINX IC** deployment, offloading the mTLS exchange and certificate rotation to Istio. Per [official documentation](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#controlling-the-injection-policy), this can be done by labelling the **NGINX IC** pods with `sidecar.istio.io/inject=true`.

By default, the Istio sidecar will intercept all inbound and outbound traffic from the **NGINX IC** container. However, as an ingress gateway handling inbound traffic, it is preferable to not have Istio proxy intercepting and modifying inbound traffic. This can be configured by adding the `traffic.sidecar.istio.io/excludeInboundPorts` annotation to the **NGINX IC** pods. The manifest below shows a configuration where inbound traffic destined for port `80` and `443` will bypass the Istio sidecar and reach the **NGINX IC** directly instead.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
spec:
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
        sidecar.istio.io/inject: true  # inject Istio sidecar
      annotations:
        traffic.sidecar.istio.io/excludeInboundPorts: "80,443"  # bypass Istio proxy for 80 and 443 traffic
    spec:
      # ...
```

For the full deployment manifest, refer to the [official installation guide](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/).

With inbound traffic sorted, let's turn our attention to the backend. The following manifests will be used as the Kubernetes applications in the service mesh:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-meshed
  labels:
    istio-injection: enabled
spec: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: httpbin
  name: httpbin
  namespace: demo-meshed
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
  namespace: demo-meshed
spec:
  ports:
    - name: http
      port: 8081
      protocol: TCP
      targetPort: 80
  selector:
    app: httpbin
  type: ClusterIP
```

Starting with a simple [VirtualServer](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#virtualserver-specification) resource to configure **NGINX IC** to discover the backend:
```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: nic.example.com
  namespace: demo-meshed
spec:
  host: nic.example.com
  upstreams:
    - name: httpbin
      service: httpbin
      port: 8081
  routes:
    - path: /httpbin/
      action:
        proxy:
          upstream: httpbin
          rewritePath: /
```

This results in an NGINX configuration that proxies inbound requests with:
1. the backend target being the application pod IP and port, as **NGINX IC** by default discovers the Kubernetes service endpoints and populates the upstream with them.
1. the host header being `nic.example.com`.

However, with the service mesh in place, traffic from **NGINX IC** to the backend is intercepted by the sidecar. For the sidecar to route to the correct endpoint and perform mTLS certificate exchange with the backend's sidecar, a request from **NGINX IC** arriving at the sidecar must have:
1. a backend target port matching the Kubernetes service port for the application: `8081`, which can be done by setting `use-cluster-ip: true` to the upstream in the [VirtualServer](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#virtualserver-specification) resource. It updates the backend target to the Cluster IP and port of the service, but does change some NGINX behaviour that relates to multiple backends, as documented [here](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#upstream).
1. a host header containing DNS name of the Kubernetes service: `httpbin.demo-meshed`, which is achieved with the [requestHeaders](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#actionproxyrequestheaderssetheader) attribute under the proxy actions.

If you are interested in how I arrive at the two values above, have a look at the [Appendix](#appendix). The two changes above gives us a [VirtualServer](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#virtualserver-specification) resource which looks like this:
```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: nic-meshed.example.com
  namespace: demo-meshed
spec:
  host: nic-meshed.example.com
  upstreams:
    - name: httpbin
      service: httpbin
      port: 8081
      use-cluster-ip: true  # set Cluster IP of service as backend
  routes:
    - path: /httpbin/
      action:
        proxy:
          upstream: httpbin
          requestHeaders:  # rewrite host header
            set:
              - name: Host
                value: httpbin.demo-meshed
          rewritePath: /
```

resulting in the following NGINX configuration on **NGINX IC**:
```nginx
upstream vs_demo-meshed_nic-meshed.example.com_httpbin {
    zone vs_demo-meshed_nic-meshed.example.com_httpbin 512k;
    random two least_conn;
    server 172.20.173.145:8081 max_fails=1 fail_timeout=10s max_conns=0;
}

server {
    listen 80;
    listen [::]:80;
    server_name nic-meshed.example.com;
    location /httpbin/ {
        proxy_set_header Host "httpbin.demo-meshed";
        proxy_pass http://vs_demo-meshed_nic-meshed.example.com_httpbin/;
        # ...
    }
}
```

To verify the integration works, we first check that the application sidecar has strict mTLS policy configured.
```sh
$ APP_POD=$(kubectl -n demo-meshed get pod -l app=httpbin -o jsonpath='{.items[0].metadata.name}')
$
$ istioctl x describe pod $APP_POD.demo-meshed
Pod: httpbin-849ccf48fc-dmcfk.demo-meshed
   Pod Ports: 15090 (istio-proxy)
Suggestion: add 'version' label to pod for Istio telemetry.
--------------------
Service: httpbin.demo-meshed
   Port: http 8081/HTTP targets pod port 80
--------------------
Effectve PeerAuthentication:
   Workload mTLS: STRICT
Applied PeerAuthentication:
   default.istio-system
Skipping Gateway information (no ingress gateway pods)
```

Next, we send a request to **NGINX IC**:
```sh
$ INGRESS_IP=$(kubectl -n nginx-ingress get svc -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
$
$ curl -vvv --connect-to nic-meshed.example.com::$INGRESS_IP http://nic-meshed.example.com/httpbin/get
* Connecting to hostname: a5912c2c5b08d4bed85dcac014d4e008-5c4a497f2c2ea1f6.elb.ap-southeast-2.amazonaws.com
*   Trying 54.253.97.116:80...
* TCP_NODELAY set
* Connected to a5912c2c5b08d4bed85dcac014d4e008-5c4a497f2c2ea1f6.elb.ap-southeast-2.amazonaws.com (54.253.97.116) port 80 (#0)
> GET /httpbin/get HTTP/1.1
> Host: nic-meshed.example.com
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.21.5
< Date: Tue, 24 May 2022 10:56:05 GMT
< Content-Type: application/json
< Content-Length: 698
< Connection: keep-alive
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 58
<
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.demo-meshed",
    "User-Agent": "curl/7.68.0",
    "X-B3-Parentspanid": "a01501caee791f43",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "426d757f49b5917b",
    "X-B3-Traceid": "2bdbc306558cf2f4a01501caee791f43",
    "X-Envoy-Attempt-Count": "1",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/demo-meshed/sa/default;Hash=30e68c7cde63689d985db9b671183e554eb57ec6c003ff59765c3258ff75942e;Subject=\"\";URI=spiffe://cluster.local/ns/nginx-ingress/sa/nginx-ingress",
    "X-Forwarded-Host": "nic-meshed.example.com"
  },
  "origin": "101.161.74.87",
  "url": "http://nic-meshed.example.com/get"
}
```

The log entries from the following sources also shows the traffic flow from **NGINX IC** to the application through the service mesh sidecars:
- **NGINX IC**
    ```sh
    $ INGRESS_POD=$(kubectl -n nginx-ingress get pod -l app=nginx-ingress -o jsonpath='{.items[0].metadata.name}')
    $
    $ kubectl -n nginx-ingress logs $INGRESS_POD --tail 1
    101.161.74.87 - - [24/May/2022:10:57:33 +0000] "GET /httpbin/get HTTP/1.1" 200 698 "-" "curl/7.68.0" "-"
    ```
- **NGINX IC** sidecar
    ```sh
    $ kubectl -n nginx-ingress logs $INGRESS_POD -c istio-proxy --tail 1
    [2022-05-24T10:57:33.312Z] "GET /get HTTP/1.1" 200 - via_upstream - "-" 0 698 15 14 "101.161.74.87" "curl/7.68.0" "fc119dcf-e7d7-4ccd-ad49-3f82060e875a" "httpbin.demo-meshed" "10.0.2.153:80" outbound|8081||httpbin.demo-meshed.svc.cluster.local 10.0.1.121:33676 172.20.197.199:8081 101.161.74.87:0 - default
    ```
- Application sidecar
    ```sh
    $ kubectl -n demo-meshed logs $APP_POD -c istio-proxy --tail 1
    [2022-05-24T10:57:33.314Z] "GET /get HTTP/1.1" 200 - via_upstream - "-" 0 698 2 1 "101.161.74.87" "curl/7.68.0" "fc119dcf-e7d7-4ccd-ad49-3f82060e875a" "httpbin.demo-meshed" "10.0.2.153:80" inbound|80|| 127.0.0.6:51515 10.0.2.153:80 101.161.74.87:0 outbound_.8081_._.httpbin.demo-meshed.svc.cluster.local default
    ```

# NGINX IC with client cert issued by Istio CA

Another approach of integrating **NGINX IC** into Istio service mesh is to have **NGINX IC** perform the mTLS certificate exchange itself. This method does not require a sidecar for **NGINX IC**, instead we directly provide **NGINX IC** with a client TLS key pair that is trusted by Istio service mesh, to establish the connection to applications in the service mesh. The deployment of **NGINX IC** without sidecar is identical to the previous section, except without the Istio `sidecar.istio.io/inject` label and `traffic.sidecar.istio.io/excludeInboundPorts` annotation.

For **NGINX IC** to perform mTLS certificate exchange towards the backend, we need to create an [EgressMTLS](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/#egressmtls) policy, which references the client TLS key pair and the CA certificate, and [attach it to the VirtualServer](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#virtualserverpolicy). The creation of the client TLS key pair for use in an Istio service mesh has been covered in an earlier post - [Istio Authorization With Client Certificates](../istio-authorization-with-client-certificates/#client-certificate-setup). They are then stored in two Kubernetes secrets:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: istio-client-tls
  namespace: demo-meshed
type: kubernetes.io/tls
data:
  tls.crt: # <base64 encoded client certificate>
  tls.key: # <base64 encoded client key>
---
apiVersion: v1
kind: Secret
metadata:
  name: istio-client-trusted-ca
  namespace: demo-meshed
type: kubernetes.io/tls
data:
  ca.crt: # <base64 encoded Istio CA certificate>
```

Next, we create an [EgressMTLS](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/#egressmtls) policy, referencing the two secrets defined above:
```yaml
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: istio-client-access
  namespace: demo-meshed
spec:
  egressMTLS:
    tlsSecret: istio-client-tls
    trustedCertSecret: istio-client-trusted-ca
```

Finally, we create a [VirtualServer](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#virtualserver-specification) resource to route traffic to the application in the service mesh, referencing the [EgressMTLS](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/#egressmtls) policy:
```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: nic.example.com
spec:
  ingressClassName: nginx
  host: nic.example.com
  policies:
    - name: istio-client-access
  upstreams:
    - name: httpbin
      service: httpbin
      port: 8081
      tls:
        enable: true
  routes:
    - path: /httpbin/
      action:
        proxy:
          upstream: httpbin
          rewritePath: /
```

There are some differences between this [VirtualServer](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#virtualserver-specification) resource and the one in the previous section with the sidecar:
1. [tls](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#upstreamtls) is now enabled on the upstream, and along with the [EgressMTLS](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/#egressmtls) policy, will have **NGINX IC** perform the mTLS certificate exchange towards the application in the service mesh.
1. [use-cluster-ip](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/#upstream) is no longer required, allowing **NGINX IC** to discover all endpoints for the Kubernetes service, and directly interface with the application pods. This also means **NGINX IC** now has control of the load balancing algorithms towards the application pods.

The resulting configuration on **NGINX IC** looks like this:
```nginx
upstream vs_demo-meshed_nic.example.com_httpbin {
    zone vs_demo-meshed_nic.example.com_httpbin 512k;
    random two least_conn;
    server 10.0.1.235:80 max_fails=1 fail_timeout=10s max_conns=0;
    server 10.0.1.235:80 max_fails=1 fail_timeout=10s max_conns=0;
}

server {
    listen 80;
    listen [::]:80;
    server_name nic.example.com;
    proxy_ssl_certificate /etc/nginx/secrets/demo-meshed-istio-client-tls;
    proxy_ssl_certificate_key /etc/nginx/secrets/demo-meshed-istio-client-tls;
    proxy_ssl_trusted_certificate /etc/nginx/secrets/demo-meshed-istio-client-trusted-ca;
    proxy_ssl_verify off;
    proxy_ssl_verify_depth 3;
    proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    proxy_ssl_ciphers DEFAULT;
    proxy_ssl_session_reuse on;
    proxy_ssl_server_name off;
    proxy_ssl_name $proxy_host;

    location /httpbin/ {
        proxy_set_header Host "$host";
        proxy_pass https://vs_demo-meshed_nic.example.com_httpbin/;
        # ...
    }
}
```

We test the configuration by sending a request to the **NGINX IC**:
```sh
$ INGRESS_IP=$(kubectl -n nginx-ingress get svc -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
$
$ curl -vvv --connect-to nic.example.com::$INGRESS_IP http://nic.example.com/httpbin/get
* Connecting to hostname: a0dff85c8d7a143b689689b7a80c0290-a0d79f4e4510ab9f.elb.ap-southeast-2.amazonaws.com
*   Trying 3.104.230.13:80...
* TCP_NODELAY set
* Connected to a0dff85c8d7a143b689689b7a80c0290-a0d79f4e4510ab9f.elb.ap-southeast-2.amazonaws.com (3.104.230.13) port 80 (#0)
> GET /httpbin/get HTTP/1.1
> Host: nic.example.com
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.21.5
< Date: Fri, 27 May 2022 05:25:41 GMT
< Content-Type: application/json
< Content-Length: 624
< Connection: keep-alive
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 4
< x-envoy-decorator-operation: httpbin.demo-meshed.svc.cluster.local:8081/*
<
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "nic.example.com",
    "User-Agent": "curl/7.68.0",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "44dd3ba25a13e591",
    "X-B3-Traceid": "07f97ab871c35c5d44dd3ba25a13e591",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/demo-meshed/sa/default;Hash=df444c78ba818ae3a2769b7e5d851ad3fb6aaceec28786d69c88d4ac008443eb;Subject=\"O=test,CN=**NGINX IC**-client.example.com\";URI=spiffe://cluster.local/ns/nginx-ingress/sa/nginx-ingress",
    "X-Forwarded-Host": "nic.example.com"
  },
  "origin": "101.161.74.87",
  "url": "http://nic.example.com/get"
}
* Connection #0 to host a0dff85c8d7a143b689689b7a80c0290-a0d79f4e4510ab9f.elb.ap-southeast-2.amazonaws.com left intact
```

Checking the logs on **NGINX IC** and the application sidecar verifies the expected traffic flow:
- **NGINX IC**
    ```sh
    $ INGRESS_POD=$(kubectl -n nginx-ingress get pod -l app=nginx-ingress -o jsonpath='{.items[0].metadata.name}')
    $
    $ kubectl -n nginx-ingress logs $INGRESS_POD --tail 1
    101.161.74.87 - - [27/May/2022:05:25:41 +0000] "GET /httpbin/get HTTP/1.1" 200 624 "-" "curl/7.68.0" "-"
    ```
- Application sidecar
    ```sh
    $ APP_POD=$(kubectl -n demo-meshed get pod -l app=httpbin -o jsonpath='{.items[0].metadata.name}')
    $
    $ kubectl -n demo-meshed logs $APP_POD -c istio-proxy --tail 1
    [2022-05-27T05:25:41.448Z] "GET /get HTTP/1.1" 200 - via_upstream - "-" 0 624 4 4 "101.161.74.87" "curl/7.68.0" "2b96181a-1a4e-4edc-ab17-e7feef0f2f8d" "nic.example.com" "10.0.1.235:80" inbound|80|| 127.0.0.6:36503 10.0.1.235:80 101.161.74.87:0 - default
    ```

# Conclusion

We went through two ways of integrating **NGINX IC** into Istio service mesh.

The first method of [deploying **NGINX IC** with the Istio sidecar](#nginx-ic-with-istio-sidecar) provides:
- the convenience of having Istio manage the mTLS certificate exchange with the applications in the service mesh, as well as
- automatically rotating the short lived certs on **NGINX IC** sidecar for better security.

However, the downsides are that:
- it requires traffic to go through two proxies - NGINX and Istio proxy (Envoy), and
- we lose some flexibility in configuring some load balancing algorithms on the **NGINX IC** as it only sees the Kubernetes service for the application as an upstream, and leaves the load balancing to Istio proxy instead.

The second method involves [having **NGINX IC** itself perform the mTLS certificate exchange](#nginx-ic-with-client-cert-issued-by-istio-ca). This gives more control to **NGINX IC**, but presents a security risk if the client TLS keypair loaded on **NGINX IC** gets compromised.

# Appendix

The following section shows some commands executed to arrive at the values configured on **NGINX IC** for traffic to be successfully proxied through the Istio sidecar towards the backend application:
1. A listener on port `8081` which matches the Kubernetes service port, with a destination to the route `8081`
    ```
    $ INGRESS_POD=$(kubectl -n nginx-ingress get pod -l app=nginx-ingress -o jsonpath='{.items[0].metadata.name}')
    $
    $ istioctl pc listener $INGRESS_POD.nginx-ingress --port 8081
    ADDRESS        PORT MATCH                                DESTINATION
    0.0.0.0        8081 Trans: raw_buffer; App: http/1.1,h2c Route: 8081
    0.0.0.0        8081 ALL                                  PassthroughCluster
    ```
1. The route object `8081` which matches the host header against a list of domain names, or the DNS name of the Kubernetes service `httpbin.demo-meshed` here, and routes to the `outbound|8081||httpbin.demo-meshed.svc.cluster.local` cluster
    ```
    $ istioctl pc route $INGRESS_POD.nginx-ingress --name 8081 -o json
    [
      {
        "name": "8081",
        "virtualHosts": [
          ...
          {
            "name": "httpbin.demo-meshed.svc.cluster.local:8081",
            "domains": [
              "httpbin.demo-meshed.svc.cluster.local",
              "httpbin.demo-meshed.svc.cluster.local:8081",
              "httpbin.demo-meshed",
              "httpbin.demo-meshed:8081",
              "httpbin.demo-meshed.svc",
              "httpbin.demo-meshed.svc:8081",
              "172.20.197.199",
              "172.20.197.199:8081"
            ],
            "routes": [
              {
                "name": "default",
                "match": {
                  "prefix": "/"
                },
                "route": {
                  "cluster": "outbound|8081||httpbin.demo-meshed.svc.cluster.local",
                  ...
    ```
1. An endpoint in the cluster, resolved from the Kubernetes service
    ```
    $ istioctl pc endpoints $INGRESS_POD.nginx-ingress --cluster "outbound|8081||httpbin.demo-meshed.svc.cluster.local"
    ENDPOINT          STATUS      OUTLIER CHECK     CLUSTER
    10.0.2.153:80     HEALTHY     OK                outbound|8081||httpbin.demo-meshed.svc.cluster.local
    $
    $ kubectl -n demo-meshed describe svc httpbin
    Name:              httpbin
    Namespace:         demo-meshed
    Labels:            app=httpbin
                      app.kubernetes.io/instance=demo-meshed
    Annotations:       <none>
    Selector:          app=httpbin
    Type:              ClusterIP
    IP Family Policy:  SingleStack
    IP Families:       IPv4
    IP:                172.20.197.199
    IPs:               172.20.197.199
    Port:              http  8081/TCP
    TargetPort:        80/TCP
    Endpoints:         10.0.2.153:80
    Session Affinity:  None
    Events:            <none>
    ```

The key things to note are the port and domain name which Istio sidecar is matching in order to route to the application in the service mesh, which are `8081` and `httpbin.demo-meshed` as shown in the printout of the listener and route objects on the sidecar.
