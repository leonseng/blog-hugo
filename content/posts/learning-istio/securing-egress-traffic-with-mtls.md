---
title: "Learning Istio | Securing Egress Traffic With mTLS"
date: 2021-10-12T15:59:45+11:00
series: ["Learning Istio"]
tags: ['kubernetes', 'istio']
categories: ['tech']
---

There are times when applications deployed in Kubernetes need to communicate with external services that requires mTLS authentication, where the applications have to present client certificates signed by a common root/intermediate CA when accessing the service. This can lead to unpleasant scenarios where

1. application owners have to keep track of certificates for each of their applications
1. applications written in different language/libraries have different ways of implementing mTLS connections

As an application owner, I would prefer to just deal with plain ol' HTTP on port `80`, and not have to modify the application to handle HTTPS or mTLS. Fortunately, Istio has some in-built capabilities to alleviate the pain points. In this post, I will be covering two scenarios:
1. [Istio cluster has the same root CA as the external service](#common-ca)
1. [Istio cluster has a different root CA from the external service](#different-ca)

# Common CA

Many enterprises have root CAs they use to sign and verify all internal services. To ensure compliance, a good practice is to create an intermediate CA from the root CA, and plug that into the cluster when deploying Istio, as detailed [here](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/). For such scenarios, Istio supports [TLS origination for egress traffic](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-tls-origination/#tls-origination-for-egress-traffic), and we can enable mTLS by setting the TLS mode in the `DestinationRule` to `ISTIO_MUTUAL` as documented [here](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ClientTLSSettings). This tells the sidecar proxy to use a client certificate generated automatically by Istio (signed using the intermediate CA, hence the enterprise root CA) when calling the external service for mTLS authentication.

To demonstrate this, I have deployed an external service `nginx-mtls.common-ca.local:8443` using an NGINX container running on a remote host `10.1.1.4`. mTLS authentication is enabled by configuring it to perform client SSL verification. The root CA specified for client SSL verification is also used to generated the server certificate for the NGINX server.

> See [leonseng/nginx-mtls](https://github.com/leonseng/nginx-mtls) for more information on the external service

I deployed a `curl` pod to mimic an application performing a `GET` request to the external service:
```
kubectl apply -f - <<EOF
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
      labels:
        app: curl
    spec:
      containers:
      - command:
        - tail
        args:
        - -f
        - /dev/null
        image: curlimages/curl
        name: curl
EOF
```

As Istio's `outboundTrafficPolicy` is set to `REGISTRY_ONLY`, a `ServiceEntry` is required to allow any applications in the cluster to reach the external service `nginx-mtls.common-ca.local`:
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: nginx-mtls-common-ca
spec:
  hosts:
  - nginx-mtls.common-ca.local
  location: MESH_EXTERNAL
  ports:
  - number: 8443
    name: https
    protocol: HTTPS
  resolution: STATIC
  endpoints:
  - address: 10.1.1.4
EOF
```

As it is, the application is expected to supply the client certificate for the mTLS connection. Attempting to call the external service without the client certificate would result in a failed request:
```
$ kubectl exec curl-5fd94f6d69-526vq -c curl -- \
  curl -s --resolve nginx-mtls.common-ca.local:8443:10.1.1.4 \
  https://nginx-mtls.common-ca.local:8443
command terminated with exit code 35
```

To enable mTLS, we need the following resources:

1. A `DestinationRule` to initiate the mTLS connection on port `80`
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: nginx-mtls-common-ca
    spec:
      host: nginx-mtls.common-ca.local
      trafficPolicy:
        portLevelSettings:
        - port:
            number: 80
          tls:
            mode: ISTIO_MUTUAL
    EOF
    ```
1. Update the `ServiceEntry` with a new port entry for the HTTP port `80`, and a `targetPort` attribute set to the HTTPS port `8443`:
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1beta1
    kind: ServiceEntry
    metadata:
      name: nginx-mtls-common-ca
    spec:
      hosts:
      - nginx-mtls.common-ca.local
      location: MESH_EXTERNAL
      ports:
      - number: 80
        name: http-port
        protocol: HTTP
        targetPort: 8443
      - number: 8443
        name: https
        protocol: TLS
      resolution: STATIC
      endpoints:
      - address: 10.1.1.4
    EOF
    ```

The application will now be able to target the HTTP endpoint, leaving it to Istio to set up the mTLS connection on its behalf towards the external service:
```
$ kubectl exec curl-5fd94f6d69-526vq -c curl -- \
  curl -s --resolve nginx-mtls.common-ca.local:80:10.1.1.4 \
  http://nginx-mtls.common-ca.local \
  | grep title
<title>Welcome to nginx!</title>
```

# Different CA

There are cases where Istio is deployed with a CA certificate issued by a root CA different from the one used by the external service for client verification, or if Istio generated its own self-signed certificate. For mTLS to work in such scenarios, we would have to obtain client certificates signed by the enterprise's root CA, and configure Istio to use these client certificates when setting up the mTLS the connections.

Istio provides at least two ways of handling the client certificates:
1. [Common client certificate for all applications](#common-client-certificate-for-all-applications)
1. [Unique client certificate for each application](#unique-client-certificate-for-each-application)

## Common client certificate for all applications

If the external service provider trusts the cluster, and thereby all applications hosted within the cluster, we would only need one client certificate and key pair for an egress gateway perform the mTLS connection on behalf of all applications within the cluster. This does require the deployment of an egress gateway (which is outside the scope of this post), and have all traffic to the external service routed via the egress gateway. Istio has a handy page on [Perform mutual TLS origination with an egress gateway](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway-tls-origination/#perform-mutual-tls-origination-with-an-egress-gateway), but there's quite a bit to unpack there.

For this example use case, I have deployed another external service `nginx-mtls.diff-ca.local:9443` running on an NGINX container on the remote host `10.1.1.4`. The certificates for the server and for client verification are signed with a root CA different from the one used to create the intermediate CA for Istio.

We first need to handle the connection between the application and the egress gateway, by directing traffic to the external server `nginx-mtls.diff-ca.local` via the egress gateway with:

1. A `Gateway` on the egress gateway to listen for traffic to the external service (on port 443 because of the mTLS connection between the application and the egress gateway)
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: istio-egressgateway
    spec:
      selector:
        istio: egressgateway
      servers:
      - port:
          number: 443
          name: https
          protocol: HTTPS
        hosts:
        - nginx-mtls.diff-ca.local
        tls:
            mode: ISTIO_MUTUAL
    EOF
    ```
1. A `VirtualService` to direct traffic to the external service via the egress gateway
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: direct-nginx-mtls-through-egress-gateway
    spec:
      hosts:
      - nginx-mtls.diff-ca.local
      gateways:
      - mesh
      http:
      - match:
        - gateways:
          - mesh
          port: 80
        route:
        - destination:
            host: istio-egressgateway.istio-system.svc.cluster.local
            subset: nginx-mtls
            port:
              number: 443
    EOF
    ```
1. A `DestinationRule` to perform mTLS origination from application to the egress gateway, whilst preserving the SNI string towards the external service `nginx-mtls.diff-ca.local`
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: egressgateway-for-nginx-mtls
    spec:
      host: istio-egressgateway.istio-system.svc.cluster.local
      subsets:
      - name: nginx-mtls
        trafficPolicy:
          portLevelSettings:
          - port:
              number: 443
            tls:
              mode: ISTIO_MUTUAL
              sni: nginx-mtls.diff-ca.local
    EOF
    ```


For the second half of the connection, we need the egress gateway to route the traffic to the external service over an mTLS connection. First, we create a generic `Secret` to store the enterprise root CA, client certificate and key:

> Note that I've created the `Secret` in the `istio-system` because that's where my egress gateway is deployed
```
kubectl -n istio-system create secret generic nginx-mtls-external \
  --from-file=tls.key=client.key \
  --from-file=tls.crt=client.crt \
  --from-file=ca.crt=enterpriseRootCA.pem
```

Next, we define the following:

1. A `ServiceEntry` for the external service to allow traffic to leave the cluster
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1beta1
    kind: ServiceEntry
    metadata:
      name: nginx-mtls-diff-ca
    spec:
      hosts:
      - nginx-mtls.diff-ca.local
      location: MESH_EXTERNAL
      ports:
      - number: 9443
        name: https
        protocol: TLS
      resolution: STATIC
      endpoints:
      - address: 10.1.1.4
    EOF
    ```
1. Update the `VirtualService` defined earlier to redirect traffic hitting the egress gateway to now leave the cluster towards the external service (note the new `HTTPMatchRequest`)
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: direct-nginx-mtls-through-egress-gateway
    spec:
      hosts:
      - nginx-mtls.diff-ca.local
      gateways:
      - istio-egressgateway
      - mesh
      http:
      - match:
        - gateways:
          - mesh
          port: 80
        route:
        - destination:
            host: istio-egressgateway.istio-system.svc.cluster.local
            subset: nginx-mtls
            port:
              number: 443
      - match:
        - gateways:
          - istio-egressgateway
          port: 443
        route:
        - destination:
            host: nginx-mtls.diff-ca.local
            port:
              number: 9443
    EOF
    ```
1. A `DestinationRule` for the external service, with the client TLS mode set to `MUTUAL` for mTLS. The `Secret` containing the certificates and key is also referenced here to provide Istio sidecars with right files for setting up the mTLS connection.
    > Note that I've defined the `DestinationRule` in the same namespace as where the `Secret` is defined in this example
    ```
    kubectl -n istio-system apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: originate-tls-for-nginx-mtls
    spec:
      host: nginx-mtls.diff-ca.local
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
        portLevelSettings:
        - port:
            number: 9443
          tls:
            mode: MUTUAL
            credentialName: nginx-mtls-external
            sni: nginx-mtls.diff-ca.local
    EOF
    ```

With all that in place, the application should now be able to access the external service that is expecting a client certificate signed with a different root CA from the cluster's CA
```
$ kubectl exec curl-5fd94f6d69-526vq -c curl -- \
  curl -s --resolve nginx-mtls.diff-ca.local:80:10.1.1.4 \
  http://nginx-mtls.diff-ca.local \
  | grep title
<title>Welcome to nginx!</title>
```

## Unique client certificate for each application

If there is a requirement for each application to have unique client certificates, or managing an egress gateway sounds like a chore, one can leave the task of managing client certificates to the application owners.

First, the cluster admin has to define the following:

1. A `ServiceEntry` for the external service to allow traffic to leave the cluster
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1beta1
    kind: ServiceEntry
    metadata:
      name: nginx-mtls-diff-ca
    spec:
      hosts:
      - nginx-mtls.diff-ca.local
      location: MESH_EXTERNAL
      ports:
      - number: 9443
        name: https
        protocol: HTTPS
      resolution: STATIC
      endpoints:
      - address: 10.1.1.4
    EOF
    ```
1. A `VirtualService` to route traffic destined for the external service, and converting the HTTP port (`80`) to the HTTPS port (`9443`). Note that this is just a port number change, the protocol is still HTTP at this point.
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: nginx-mtls-diff-ca
    spec:
      hosts:
      - nginx-mtls.diff-ca.local
      http:
      - match:
        - port: 80
        route:
        - destination:
            host: nginx-mtls.diff-ca.local
            port:
              number: 9443
    EOF
    ```
1. A `DestinationRule` to perform mTLS connection, referencing the CA certificate, client certificate and key files in particular locations in the sidecar proxy. These files will be loaded into the sidecar proxy by the application owners in the following section.
    ```
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: nginx-mtls-diff-ca
    spec:
      host: nginx-mtls.diff-ca.local
      trafficPolicy:
        portLevelSettings:
        - port:
            number: 9443
          tls:
            mode: MUTUAL
            clientCertificate: /etc/certs/tls.crt
            privateKey: /etc/certs/tls.key
            caCertificates: /etc/certs/ca.crt
    EOF
    ```

With the above set up, the application owners then have to provide the CA certificate, client certificate and key files for their applications:

1. Create a generic `Secret` to store the enterprise root CA, client certificate and key for the mTLS connection towards
    ```
    kubectl create secret generic nginx-mtls-external \
      --from-file=tls.key=app-client.key \
      --from-file=tls.crt=app-client.crt \
      --from-file=ca.crt=enterpriseCA.pem
    ```
1. Add the following annotations to the `Pod` template in the `Deployment` to load the certs and key from the `Secret` into the pod's sidecar proxy in the directory specified by the `DestinationRule` from before (`/etc/certs/` in this case)
    ```
    sidecar.istio.io/userVolume: '[{"name":"client-certs", "secret":{"secretName":"nginx-mtls-external"}}]'
    sidecar.istio.io/userVolumeMount: '[{"name":"client-certs", "mountPath":"/etc/certs", "readonly":true}]'
    ```

   The `Deployment` manifest will look something like this:

    ```
    kubectl apply -f - <<EOF
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
          annotations:
            sidecar.istio.io/userVolume: '[{"name":"client-certs", "secret":{"secretName":"nginx-mtls-external"}}]'
            sidecar.istio.io/userVolumeMount: '[{"name":"client-certs", "mountPath":"/etc/certs", "readonly":true}]'
          labels:
            app: curl
        spec:
          containers:
          - command:
            - tail
            args:
            - -f
            - /dev/null
            image: curlimages/curl
            name: curl
    EOF
    ```

    You can verify that the certificates and key are mounted correctly using `istioctl pc secrets`

    ```
    $ istioctl pc secrets curl-55b48d797c-6f5h6 | grep /etc/certs
    file-cert:/etc/certs/tls.crt~/etc/certs/tls.key     Cert Chain     ACTIVE     true           344012585647005735528648296646953979292086906406     2022-09-27T11:16:30Z     2021-09-27T11:16:30Z
    file-root:/etc/certs/ca.crt                         CA             ACTIVE     true           720903288241772125710852709688782830101643184205     2026-09-26T11:03:03Z     2021-09-27T11:03:03Z
    ```

Now, the application should be able to access the external service via HTTP on port `80`, and the sidecar proxy should initiate the mTLS connection on its behalf on the HTTPS port `9443`:
```
$ kubectl exec curl-55b48d797c-6f5h6 -c curl -- \
  curl -s --resolve nginx-mtls.diff-ca.local:80:10.1.1.4 \
  http://nginx-mtls.diff-ca.local \
  | grep title
<title>Welcome to nginx!</title>
```
