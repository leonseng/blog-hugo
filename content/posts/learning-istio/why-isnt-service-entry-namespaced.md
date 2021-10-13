---
title: "Learning Istio | Why Isn't Service Entry Namespaced!?"
date: 2021-09-04T23:36:07+10:00
series: ["Learning Istio"]
tags: ['kubernetes', 'istio']
categories: ['tech']
---

I got a question on how we can restrict access to certain external endpoints on a per namespace basis. There was an idea to use Istio's `egress gateway` to control access to external endpoints, though I'm not convinced that's a valid [use case for an `egress gateway`](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/#use-case) today. So I went off to do some investigation, and found some options:

1. [Specifying which namespaces can access certain hosts defined in the `ServiceEntry`](#exportto)
1. [Specifying which endpoints can be accessed from a namespace](#sidecar-resource)

But before that, a bit of back story of how we got here...

## A naive beginning

First, I updated Istio `outboundTrafficPolicy` to `REGISTRY_ONLY` so that we need to EXPLICITLY allow connectivity to external endpoints

```
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY -y
```

To test the access restriction, I deployed a `debian` pod in the `default` namespace

```
kubectl create deployment debian --image debian --replicas=1 -- tail -f /dev/null
```

and tried running `apt update`

```
$ kubectl exec -it debian-8484c5df49-9x7lt -- bash
Defaulting container name to debian.
Use 'kubectl describe pod/debian-8484c5df49-9x7lt -n default' to see all of the containers in this pod.
root@debian-8484c5df49-9x7lt:/#
root@debian-8484c5df49-9x7lt:/# apt-get update
Err:1 http://security.debian.org/debian-security bullseye-security InRelease
  502  Bad Gateway [IP: 151.101.130.132 80]
Err:2 http://deb.debian.org/debian bullseye InRelease
  502  Bad Gateway [IP: 151.101.30.132 80]
Err:3 http://deb.debian.org/debian bullseye-updates InRelease
  502  Bad Gateway [IP: 151.101.30.132 80]
Reading package lists... Done
N: See apt-secure(8) manpage for repository creation and user configuration details.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
E: The repository 'http://security.debian.org/debian-security bullseye-security InRelease' is not signed.
E: Failed to fetch http://security.debian.org/debian-security/dists/bullseye-security/InRelease  502  Bad Gateway [IP: 151.101.130.132 80]
E: Failed to fetch http://deb.debian.org/debian/dists/bullseye/InRelease  502  Bad Gateway [IP: 151.101.30.132 80]
E: The repository 'http://deb.debian.org/debian bullseye InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: Failed to fetch http://deb.debian.org/debian/dists/bullseye-updates/InRelease  502  Bad Gateway [IP: 151.101.30.132 80]
E: The repository 'http://deb.debian.org/debian bullseye-updates InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
root@debian-8484c5df49-9x7lt:/#
```

As expected the command fails when `apt update` tries to reach external endpoints `security.debian.org` and `deb.debian.org`, due to the endpoints not being defined in the registry.

I then created a `ServiceEntry` matching the two hostnames

```
kubectl create -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: debian-org
spec:
  hosts:
  - security.debian.org
  - deb.debian.org
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
EOF
```

and re-ran `apt update`. This time it ran successfully as expected.

```
$ kubectl exec -it debian-8484c5df49-9x7lt -- bash
Defaulting container name to debian.
Use 'kubectl describe pod/debian-8484c5df49-9x7lt -n default' to see all of the containers in this pod.
root@debian-8484c5df49-9x7lt:/# apt update
Hit:1 http://security.debian.org/debian-security bullseye-security InRelease
Get:2 http://deb.debian.org/debian bullseye InRelease [113 kB]
Get:3 http://deb.debian.org/debian bullseye-updates InRelease [36.8 kB]
Get:4 http://deb.debian.org/debian bullseye/main amd64 Packages [8178 kB]
Fetched 8327 kB in 3s (2812 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
root@debian-8484c5df49-9x7lt:/#
```

Now, I thought this would be the end of the story. All I have to do is to only allow cluster admins the permissions for creating `ServiceEntry` resources, and developers would have to engage cluster admins to get access to external resources!

## Plot twist

But I figured this came to me too easily... they should have figured it out without asking for help. So, I tried repeating the test from another namespace `newguy`

```
kubectl create ns newguy
kubectl label namespace newguy istio-injection=enabled
kubectl -n newguy create deployment debian --image debian --replicas=1 -- tail -f /dev/null
```

Lo and behold, when I run `apt update` in this container, it worked... when I thought it shouldn't...

```
$ kubectl -n newguy exec -it debian-8484c5df49-dpf2h -- bash
Defaulting container name to debian.
Use 'kubectl describe pod/debian-8484c5df49-dpf2h -n newguy' to see all of the containers in this pod.
root@debian-8484c5df49-dpf2h:/# apt update
Get:1 http://deb.debian.org/debian bullseye InRelease [113 kB]
Get:2 http://deb.debian.org/debian bullseye-updates InRelease [36.8 kB]
Get:3 http://deb.debian.org/debian bullseye/main amd64 Packages [8178 kB]
Get:4 http://security.debian.org/debian-security bullseye-security InRelease [44.1 kB]
Get:5 http://security.debian.org/debian-security bullseye-security/main amd64 Packages [29.4 kB]
Fetched 8401 kB in 3s (3267 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
root@debian-8484c5df49-dpf2h:/#
```

Turns out that by default, many of Istio's resources are translated to configurations which are applied to the sidecar proxies in all namespaces. Here's one from Istio's [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/) documentation:

> The 'exportTo' field allows for control over the visibility of a service declaration to other namespaces in the mesh. By default, a service is exported to all namespaces.

So, that means by default, my `newguy:debian` pod contains configuration to get to the `*.debian.org` hosts from the `ServiceEntry` definition in the default namespace. I want the opposite of that - I don't want the `istio-proxy` in the `newguy` namespace to pick up configuration defined in other namespaces.

Now that we've established our problem, let's set out to explore the options:
1. [Specifying which namespaces can access certain hosts defined in the `ServiceEntry`](#exportto)
1. [Specifying which endpoints can be accessed from a namespace](#sidecar-resource)

## exportTo

The most direct way is to use the `exportTo` attribute in `ServiceEntry` to specify the namespaces in which pods are allowed to access the external endpoints. To achieve that, I updated the `ServiceEntry` to only export to the namespace in which it's defined, or just `"."`:
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: debian-org
spec:
  exportTo:
  - "."
  hosts:
  - security.debian.org
  - deb.debian.org
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
EOF
```

The container in the `newguy` namespace is no longer able to access the endpoints:
```
$ kubectl -n newguy exec debian-8484c5df49-dpf2h -- apt update
Defaulted container "debian" out of: debian, istio-proxy, istio-init (init)

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Err:1 http://deb.debian.org/debian bullseye InRelease
  502  Bad Gateway [IP: 151.101.30.132 80]
Err:2 http://deb.debian.org/debian bullseye-updates InRelease
  502  Bad Gateway [IP: 151.101.30.132 80]
Err:3 http://security.debian.org/debian-security bullseye-security InRelease
  502  Bad Gateway [IP: 151.101.2.132 80]
Reading package lists...
E: The repository 'http://deb.debian.org/debian bullseye InRelease' is no longer signed.
E: Failed to fetch http://deb.debian.org/debian/dists/bullseye/InRelease  502  Bad Gateway [IP: 151.101.30.132 80]
E: Failed to fetch http://deb.debian.org/debian/dists/bullseye-updates/InRelease  502  Bad Gateway [IP: 151.101.30.132 80]
E: The repository 'http://deb.debian.org/debian bullseye-updates InRelease' is no longer signed.
E: Failed to fetch http://security.debian.org/debian-security/dists/bullseye-security/InRelease  502  Bad Gateway [IP: 151.101.2.132 80]
E: The repository 'http://security.debian.org/debian-security bullseye-security InRelease' is no longer signed.
command terminated with exit code 100
```

## Sidecar resource

The `exportTo` attribute presents a per `ServiceEntry` way of controlling access. To achieve a per namespace way of access control, we can turn to the `Sidecar`. No, not the `istio-proxy` sidecars that come with pods. I'm talking about the [Sidecar](https://istio.io/latest/docs/reference/config/networking/sidecar/) custom resource.

> By default, Istio will program all sidecar proxies in the mesh with the necessary configuration required to reach every workload instance in the mesh, as well as accept traffic on all the ports associated with the workload. The Sidecar configuration provides a way to fine tune the set of ports, protocols that the proxy will accept when forwarding traffic to and from the workload. In addition, it is possible to restrict the set of services that the proxy can reach when forwarding outbound traffic from workload instances.

To demonstrate, I created the following `Sidecar` definition in the `newguy` namespace which only allows egress traffic only to other workloads in the same namespace as well as to services in the `istio-system` namespace.

```
kubectl -n newguy create -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
EOF
```

When I try running `apt update` in the `newguy` namespace again:

```
$ kubectl -n newguy exec debian-8484c5df49-dpf2h -- apt update
Defaulting container name to debian.
Use 'kubectl describe pod/debian-8484c5df49-dpf2h -n newguy' to see all of the containers in this pod.

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Err:1 http://security.debian.org/debian-security bullseye-security InRelease
  502  Bad Gateway [IP: 151.101.66.132 80]
Err:2 http://deb.debian.org/debian bullseye InRelease
  502  Bad Gateway [IP: 151.101.30.132 80]
Err:3 http://deb.debian.org/debian bullseye-updates InRelease
  502  Bad Gateway [IP: 151.101.30.132 80]
Reading package lists...
E: The repository 'http://security.debian.org/debian-security bullseye-security InRelease' is no longer signed.
E: Failed to fetch http://security.debian.org/debian-security/dists/bullseye-security/InRelease  502  Bad Gateway [IP: 151.101.66.132 80]
E: Failed to fetch http://deb.debian.org/debian/dists/bullseye/InRelease  502  Bad Gateway [IP: 151.101.30.132 80]
E: The repository 'http://deb.debian.org/debian bullseye InRelease' is no longer signed.
E: Failed to fetch http://deb.debian.org/debian/dists/bullseye-updates/InRelease  502  Bad Gateway [IP: 151.101.30.132 80]
E: The repository 'http://deb.debian.org/debian bullseye-updates InRelease' is no longer signed.
command terminated with exit code 100
```

It fails as expected! And when I add the two hostnames in the egress hosts list of the `Sidecar` resource:

```
kubectl -n newguy apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
    - "default/security.debian.org"
    - "default/deb.debian.org"
EOF
```

`apt update` now runs successfully!

```
$ kubectl -n newguy exec debian-8484c5df49-dpf2h -- apt update
Defaulting container name to debian.
Use 'kubectl describe pod/debian-8484c5df49-dpf2h -n newguy' to see all of the containers in this pod.

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Hit:1 http://security.debian.org/debian-security bullseye-security InRelease
Hit:2 http://deb.debian.org/debian bullseye InRelease
Hit:3 http://deb.debian.org/debian bullseye-updates InRelease
Reading package lists...
Building dependency tree...
Reading state information...
All packages are up to date.
$
```
