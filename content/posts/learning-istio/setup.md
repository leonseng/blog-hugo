---
title: "Learning Istio | Setup"
date: 2021-08-02T14:20:35+10:00
draft: false
series: ["Learning Istio"]
tags: ['kubernetes', 'istio']
categories: ['tech']
---

In this series, we will be testing out several features in Istio with a local Kubernetes (k3s) cluster.

# Deploy k3s cluster

First step is to deploy the k8s cluster with [k3d](https://k3d.io/) - a wrapper to run k3s in docker. Start by creating a k3d config file:

```yaml
# k3d-istio.yaml
apiVersion: k3d.io/v1alpha2
kind: Simple
name: istio
servers: 1
agents: 2

ports:
  # for exposing Istio ingress on localhost
  - port: 8080:80
    nodeFilters:
      - loadbalancer
  - port: 8443:443
    nodeFilters:
      - loadbalancer
options:
  k3s:
    extraServerArgs:
      - --no-deploy=traefik  # we will be using Istio ingress instead
```

Deploy the cluster with k3d

```sh
k3d cluster create --config k3d-istio.yaml
```

Once the cluster has been deployed, configure `kubectl` to use the newly created context `k3d-istio`

```sh
kubectl config use-context k3d-istio
```

Verify cluster creation by running

```sh
kubectl cluster-info
```
---

# Deploy sample application

Deploy the sample [Bookinfo](https://istio.io/latest/docs/examples/bookinfo/#deploying-the-application) application so that we can observe the difference Istio brings


```sh
kubectl create ns bookinfo
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/bookinfo/platform/kube/bookinfo.yaml
```

You should see some pods created in the namespace

```sh
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79f774bdb9-swd8j       1/1     Running   0          42m
ratings-v1-b6994bb9-vgx86         1/1     Running   0          42m
productpage-v1-6b746f74dc-wkd2r   1/1     Running   0          42m
reviews-v1-545db77b95-vcps9       1/1     Running   0          42m
reviews-v2-7bf8c9648f-4tscf       1/1     Running   0          42m
reviews-v3-84779c7bbc-tgxx7       1/1     Running   0          42m
```

To access the Bookinfo application, port forward the `productpage` service to localhost:

```sh
kubectl port-forward svc/productpage 9080
```

You should now be able to see the product page by browsing to [http://localhost:9080/productpage](http://localhost:9080/productpage).

---

# Install Istio

> This section is largely based on Istio's [quick start guide](https://istio.io/latest/docs/setup/getting-started/) with some minor differences to how the application is accessed (due to k3s networking)

We will be using `istioctl` to install Istio in the cluster. See [Istio Installation Guides](https://istio.io/latest/docs/setup/install/) for alternative installation methods.

Follow the instructions [here](https://istio.io/latest/docs/setup/getting-started/#download) to install the `istioctl` binary. Verify that `istioctl` has been installed by running `istioctl version`:

```sh
istioctl version
```

Next, install Istio with the `demo` profile in the cluster

```sh
istioctl install --set profile=demo -y
```

The `demo` profile comes with all the Istio core components, and enables high levels of tracing and access logging. See [Istio Installation Configuration Profiles](https://istio.io/latest/docs/setup/additional-setup/config-profiles/) for more information on all supported configuration profiles.

After a while, you should be able to see all Istio core components deployed in the `istio-system` namespace

```sh
$ kubectl -n istio-system get all
NAME                                       READY   STATUS    RESTARTS   AGE
pod/istiod-568d797f55-j2z9p                1/1     Running   0          15m
pod/svclb-istio-ingressgateway-nvbs8       5/5     Running   0          14m
pod/svclb-istio-ingressgateway-2r9xx       5/5     Running   0          14m
pod/svclb-istio-ingressgateway-c62h4       5/5     Running   0          14m
pod/istio-egressgateway-5547fcc8fc-cqnzs   1/1     Running   0          14m
pod/istio-ingressgateway-8f568d595-q4g96   1/1     Running   0          14m

NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP                        PORT(S)
                                                      AGE
service/istiod                 ClusterIP      10.43.189.242   <none>                             15010/TCP,15012/TCP,443/TCP,15014/TCP                                        15m
service/istio-egressgateway    ClusterIP      10.43.162.21    <none>                             80/TCP,443/TCP
                                                      14m
service/istio-ingressgateway   LoadBalancer   10.43.59.75     172.22.0.2,172.22.0.3,172.22.0.4   15021:31379/TCP,80:30381/TCP,443:32247/TCP,31400:31774/TCP,15443:32131/TCP   14m

NAME                                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/svclb-istio-ingressgateway   3         3         3       3            3           <none>          14m

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istiod                 1/1     1            1           15m
deployment.apps/istio-egressgateway    1/1     1            1           14m
deployment.apps/istio-ingressgateway   1/1     1            1           14m

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/istiod-568d797f55                1         1         1       15m
replicaset.apps/istio-egressgateway-5547fcc8fc   1         1         1       14m
replicaset.apps/istio-ingressgateway-8f568d595   1         1         1       14m
```

---

# Enabling Istio sidecar injection

Now that we have everything set up, let's begin by enabling Istio in our application namespace `default` by adding the label `istio-injection=enabled` to the namespace:

```sh
kubectl label namespace default istio-injection=enabled
```

At this point, there should be no changes to the bookinfo application (note the container count is still `1/1` for all pods):

```sh
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79f774bdb9-swd8j       1/1     Running   0          56m
ratings-v1-b6994bb9-vgx86         1/1     Running   0          56m
productpage-v1-6b746f74dc-wkd2r   1/1     Running   0          56m
reviews-v1-545db77b95-vcps9       1/1     Running   0          56m
reviews-v2-7bf8c9648f-4tscf       1/1     Running   0          56m
reviews-v3-84779c7bbc-tgxx7       1/1     Running   0          56m
```

To get the sidecar injected, we need to get the pods redeployed by deleting the existing ones with the command

```sh
kubectl get pods --no-headers | awk '{print $1}' | xargs kubectl delete pod
```

The pods should be redeployed with 2 containers in each of them

```sh
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
reviews-v1-545db77b95-5c5z4       2/2     Running   0          91s
ratings-v1-b6994bb9-v2mc4         2/2     Running   0          91s
productpage-v1-6b746f74dc-wjkr5   2/2     Running   0          91s
details-v1-79f774bdb9-fm7jd       2/2     Running   0          91s
reviews-v2-7bf8c9648f-b9mrh       2/2     Running   0          91s
reviews-v3-84779c7bbc-lvrwv       2/2     Running   0          91s
```

with the new container being the Istio sidecar proxy, as demonstrated in the following command

```sh
$ kubectl get pods productpage-v1-6b746f74dc-wjkr5 -o jsonpath='{range .spec.containers[*]}{.image}{"\n"}{end}'
docker.io/istio/examples-bookinfo-productpage-v1:1.16.2
docker.io/istio/proxyv2:1.10.3
```

---
# Security Hardening

## Mutual TLS

By default, Istio configures the destination workloads using `PERMISSIVE` mode, where a service can accept both plain text and mutual TLS traffic. To ensure all our cluster traffic is encrypted, we will change this to `STRICT` mode.

There are two spots to enforce this
1. By namespace
```
$ kubectl apply -n <namespace> -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```
2. Globally - note the resource is applied in the `istio-system` namespace.
```
$ kubectl apply -n istio-system -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

## Outbound Traffic Policy

To ensure we have better control of traffic exiting the cluster to reach external services, we will be configuring the Istio `meshConfig.outboundTrafficPolicy.mode` option to `REGISTRY_ONLY`. This means that pods/sidecars in the cluster are only able to reach external services if they are first defined in Istio's internal service registry (via [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/) definitions).


Use `istioctl` to enforce the policy
```
istioctl install --set profile=demo -y --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
```

Verify the configuration is applied correctly
```
$ kubectl get istiooperator installed-state -n istio-system -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'
REGISTRY_ONLY
```

---

# Next steps

Now that we have a Kubernetes cluster with Istio installed, and a sample application with Istio sidecar injected, we should be ready to test out some Istio features in future articles.
