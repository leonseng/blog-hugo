---
title: "Why Isn't Service Entry Namespaced!?"
date: 2021-09-04T23:36:07+10:00
series: ["Learning Istio"]
tags: ['kubernetes', 'istio']
categories: ['tech']
---

I got a question on how we can restrict access to certain external endpoints on a per namespace basis. There was a proposed solution to use Istio's `egress gateway` to possibly control access to particular external endpoints, though I'm not convinced that's a valid [use case of an `egress gateway`](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/#use-case) today. So off I go to do some investigation...

## A naive beginning

First, I updated Istio `outboundTrafficPolicy` to `REGISTRY_ONLY` so that we need to EXPLICITLY allow connectivity to external endpoints

```
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY -y
```

To test the access restriction, I deployed a `debian` pod in the `default` namespace

```
k create deployment debian --image debian --replicas=1 -- tail -f /dev/null
```

and tried running `apt update`

```
$ k exec -it debian-8484c5df49-9x7lt -- bash
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
k create -f - <<EOF
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
$ k exec -it debian-8484c5df49-9x7lt -- bash
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
k create ns newguy
kubectl label namespace newguy istio-injection=enabled
k -n newguy create deployment debian --image debian --replicas=1 -- tail -f /dev/null
```

Lo and behold, when I run `apt update` in this container, it worked... when I thought it shouldn't...

```
k -n newguy exec -it debian-8484c5df49-dpf2h -- bash
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

I went to do a bit of digging, and came across...

## Sidecar resource

No, not the `istio-proxy` sidecars that come with pods. I'm talking about the [Sidecar](https://istio.io/latest/docs/reference/config/networking/sidecar/) custom resource.

> By default, Istio will program all sidecar proxies in the mesh with the necessary configuration required to reach every workload instance in the mesh, as well as accept traffic on all the ports associated with the workload. The Sidecar configuration provides a way to fine tune the set of ports, protocols that the proxy will accept when forwarding traffic to and from the workload. In addition, it is possible to restrict the set of services that the proxy can reach when forwarding outbound traffic from workload instances.

So, that means my `newguy:debian` pod by default contains configuration to get to the `*.debian.org` hosts from the `ServiceEntry` definition in the default namespace. All I have to do is to restrict the sidecar configuration for the `newguy` namespace, and not pick up the configuration from the `ServiceEntry` definition in the default namespace.

The configuration below configures sidecars in `newguy` to allow egress traffic only to other workloads in the same namespace as well as to services in the `istio-system` namespace.

```
k -n newguy create -f - <<EOF
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
$ k -n newguy exec debian-8484c5df49-dpf2h -- apt update
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
k -n newguy apply -f - <<EOF
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
$ k -n newguy exec debian-8484c5df49-dpf2h -- apt update
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
leon@metabox:~$
```

## Conclusion

To restrict access to certain external endpoints on a per namespace basis, one can define an "allow list" within the `Sidecar` resource in the namespace. Note that creation of `Sidecar` resource should also be restricted to admins to ensure users/developers are unable to get around this restriction.
