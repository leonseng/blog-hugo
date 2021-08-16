---
title: "Learning Istio | Accessing external TCP services using ServiceEntry"
date: 2021-08-16T11:31:40+10:00
draft: false
series: ["Learning Istio"]
tags: ['kubernetes', 'istio']
categories: ['tech']
---

In this post, we will be testing Istio's [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/) by accessing a PostgreDB database hosted externally from the Kubernetes cluster.

# Setup

## "External" PostgresDB service

Since we are running the Kubernetes cluster locally in Docker containers using `k3d`, we can create an "external" service by running a `PostgresDB` Docker container on the same host and expose its ports to localhost.

Create a local PostgresDB container database using Docker
```
docker run --name postgres --restart always -e POSTGRES_PASSWORD=password -d -p 5432:5432 postgres
```

Create a test database `app_db`
```
docker exec -u postgres -it postgres createdb app_db
```

This service should be accessible within the cluster at `host.k3d.internal:5432` (See [k3d FAQ](https://k3d.io/faq/faq/#how-to-access-services-like-a-database-running-on-my-docker-host-machine) for more information on `host.k3d.internal`)

## Postgres client

To test the externally hosted service, we will use [pgcli](https://www.pgcli.com/) to open a connection towards the database. I have published an image [leonseng/pgcli-docker](https://hub.docker.com/r/leonseng/pgcli-docker) on [Dockerhub](https://hub.docker.com/), which contains the `pgcli` binary for the purpose of this test.

Create a deployment with the image
```
kubectl create deployment pgcli --image leonseng/pgcli-docker:3.1.0 -- sleep 36000
```

Assuming the namespace has been labelled with `istio-injection=enabled`, the pod should come up with 2 containers - one for `pgcli-docker`, another for `istio-proxy`
```
$ kubectl get pods pgcli-6d678b54fb-v8fpp
NAME                     READY   STATUS    RESTARTS   AGE
pgcli-6d678b54fb-v8fpp   2/2     Running   0          30m
```

Try initial connection to the PostgresDB external to the Kubernetes cluster
```
$ kubectl exec <pgcli_pod> -it -- pgcli postgres://postgres:password@host.k3d.internal:5432/app_db
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.

command terminated with exit code 1
$
```

Connection fails as expected due to missing entry in the registry for the external service. Looking at logs of `istio-proxy` confirms that traffic is being sent to the `BlackHoleCluster`
```
$ kubectl logs <pgcli_pod> -c istio-proxy --tail 50 -f
[2021-08-16T00:47:49.898Z] "- - -" 0 UH - - "-" 0 0 0 - "-" "-" "-" "-" "-" BlackHoleCluster - 172.17.0.1:5432 10.42.0.10:35742 - -
```

# Service Entry

Create a `ServiceEntry` which registers the PostgresDB service at `host.k3d.internal:5432`

```
kubectl create -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: postgresdb
spec:
  hosts:
  - host.k3d.internal
  location: MESH_EXTERNAL
  ports:
  - number: 5432
    name: postgres
    protocol: TCP
  resolution: DNS
EOF

```

Once the `ServiceEntry` has been created, the `pgcli` client is now able to connect to the PostgresDB
```
$ kubectl exec <pgcli_pod> -it -- pgcli postgres://postgres:password@host.k3d.internal:5432/app_db
Server: PostgreSQL 13.3 (Debian 13.3-1.pgdg100+1)
Version: 3.1.0
Chat: https://gitter.im/dbcli/pgcli
Home: http://pgcli.com
postgres@host:app_db> quit
Goodbye!
$
```

This successful connection is also logged on the `istio-proxy`
```
[2021-08-16T00:57:32.037Z] "- - -" 0 - - - "-" 452 880 30823 - "-" "-" "-" "-" "172.17.0.1:5432" outbound|5432||host.k3d.internal 10.42.0.10:40108 172.17.0.1:5432 10.42.0.10:40106 - -
```

Digging deeper into the `istio-proxy` configuration will show the relevant `Envoy` objects created by this `ServiceEntry`

```
$ istioctl proxy-config listeners pgcli-6d678b54fb-v8fpp | grep host.k3d.internal
0.0.0.0       5432  ALL                                                                      Cluster: outbound|5432||host.k3d.internal

$ istioctl proxy-config clusters pgcli-6d678b54fb-v8fpp | grep host.k3d.internal
host.k3d.internal                                       5432      -          outbound      STRICT_DNS

$ istioctl proxy-config endpoints pgcli-6d678b54fb-v8fpp | grep host.k3d.internal
172.17.0.1:5432                  HEALTHY     OK                outbound|5432||host.k3d.internal
```
