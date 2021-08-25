---
title: "Setting up Multus for multi-interface pods"
date: 2021-08-25T16:47:00+10:00
draft: true
---

Coming from the telecommunications industry, there's a push to containerized some of the packet pushing functions. Conventional networking setup for applications just wouldn't cut it for reasons such as:

- ability to connect to multiple networks (VLANs, VPNs etc...) so that containerized routing functions can route traffic between networks
- direct access to the hardware network interface, often achieved with SR-IOV

Enough with the preface, I just want to see if I can setup my Kubernetes cluster to support applications that requires access to multiple network interfaces.

This is for the odd bunch of people who need to attach multiple interfaces to their Kubernetes pods.

---


## Setup

No SRIOV, just macvlan

## Deploy Kubernetes cluter with Kubespray

## Install Multus



```
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-internal
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "ens6",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "10.1.10.0/24",
        "rangeStart": "10.1.10.200",
        "rangeEnd": "10.1.10.250",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "10.1.10.1"
      }
    }'
EOF

cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-external
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "ens7",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "10.1.20.0/24",
        "rangeStart": "10.1.20.200",
        "rangeEnd": "10.1.20.250",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "10.1.20.1"
      }
    }'
EOF
```


## Where's my macvlan CNI plugin

```
ubuntu@ubuntu:~/multus-cni$ k describe pods
Name:         busybox-5f4cf4b5f6-r7xjx
Namespace:    default
Priority:     0
Node:         node3/10.1.1.7
Start Time:   Wed, 25 Aug 2021 05:17:57 +0000
Labels:       app=busybox
              pod-template-hash=5f4cf4b5f6
Annotations:  cni.projectcalico.org/podIP: 10.233.92.2/32
              cni.projectcalico.org/podIPs: 10.233.92.2/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "cni0",
                    "ips": [
                        "10.233.92.2"
                    ],
                    "default": true,
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks-status:
                [{
                    "name": "cni0",
                    "ips": [
                        "10.233.92.2"
                    ],
                    "default": true,
                    "dns": {}
                }]
Status:       Running
IP:           10.233.92.2
IPs:
  IP:           10.233.92.2
Controlled By:  ReplicaSet/busybox-5f4cf4b5f6
Containers:
  busybox:
    Container ID:  docker://abe0ef847d7997c21a9315c3288c54177b9185c49ce8d15f4a8fd0ccf2836eda
    Image:         busybox:1.24
    Image ID:      docker-pullable://busybox@sha256:8ea3273d79b47a8b6d018be398c17590a4b5ec604515f416c5b797db9dde3ad8
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      3600
    State:          Running
      Started:      Wed, 25 Aug 2021 05:17:59 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-dnbnb (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-dnbnb:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason          Age   From               Message
  ----    ------          ----  ----               -------
  Normal  Scheduled       111s  default-scheduler  Successfully assigned default/busybox-5f4cf4b5f6-r7xjx to node3
  Normal  AddedInterface  110s  multus             Add eth0 [10.233.92.2/32] from cni0
  Normal  Pulled          110s  kubelet            Container image "busybox:1.24" already present on machine
  Normal  Created         110s  kubelet            Created container busybox
  Normal  Started         109s  kubelet            Started container busybox


Name:           busybox-9d4f77bc6-fg28r
Namespace:      default
Priority:       0
Node:           node3/10.1.1.7
Start Time:     Wed, 25 Aug 2021 05:19:06 +0000
Labels:         app=busybox
                pod-template-hash=9d4f77bc6
Annotations:    cni.projectcalico.org/podIP:
                cni.projectcalico.org/podIPs:
                k8s.v1.cni.cncf.io/networks: macvlan-internal, macvlan-external
Status:         Pending
IP:
IPs:            <none>
Controlled By:  ReplicaSet/busybox-9d4f77bc6
Containers:
  busybox:
    Container ID:
    Image:         busybox:1.24
    Image ID:
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      3600
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-b9b6x (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  kube-api-access-b9b6x:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason                  Age               From               Message
  ----     ------                  ----              ----               -------
  Normal   Scheduled               41s               default-scheduler  Successfully assigned default/busybox-9d4f77bc6-fg28r to node3
  Normal   AddedInterface          41s               multus             Add eth0 [10.233.92.3/32] from cni0
  Warning  FailedCreatePodSandBox  40s               kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "5278abc55f294af7d2ba3dc285c0bdbc317aeffb99fad9d0fa0ff7f295904605" network for pod "busybox-9d4f77bc6-fg28r": networkPlugin cni failed to set up pod "busybox-9d4f77bc6-fg28r_default" network: [default/busybox-9d4f77bc6-fg28r:macvlan-internal]: error adding container to network "macvlan-internal": failed to find plugin "macvlan" in path [/opt/cni/bin /opt/cni/bin], failed to clean up sandbox container "5278abc55f294af7d2ba3dc285c0bdbc317aeffb99fad9d0fa0ff7f295904605" network for pod "busybox-9d4f77bc6-fg28r": networkPlugin cni failed to teardown pod "busybox-9d4f77bc6-fg28r_default" network: delegateDel: error invoking DelegateDel - "macvlan": error in getting result from DelNetwork: failed to find plugin "macvlan" in path [/opt/cni/bin /opt/cni/bin] / delegateDel: error invoking DelegateDel - "macvlan": error in getting result from DelNetwork: failed to find plugin "macvlan" in path [/opt/cni/bin /opt/cni/bin]]
  Normal   SandboxChanged          7s (x4 over 39s)  kubelet            Pod sandbox changed, it will be killed and re-created.
ubuntu@ubuntu:~/multus-cni$
```


Verify
```
ubuntu@ubuntu:~$ k exec busybox-9d4f77bc6-fg28r ip a
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1480 qdisc noqueue
    link/ether ee:32:83:6f:64:93 brd ff:ff:ff:ff:ff:ff
    inet 10.233.92.4/32 brd 10.233.92.4 scope global eth0
       valid_lft forever preferred_lft forever
5: net1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 52:0d:f8:61:ce:4d brd ff:ff:ff:ff:ff:ff
    inet 10.1.10.200/24 brd 10.1.10.255 scope global net1
       valid_lft forever preferred_lft forever
6: net2@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether 2a:df:58:ea:e8:f1 brd ff:ff:ff:ff:ff:ff
    inet 10.1.20.200/24 brd 10.1.20.255 scope global net2
       valid_lft forever preferred_lft forever
ubuntu@ubuntu:~$ k exec busybox-9d4f77bc6-fg28r ping 10.1.20.1
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
PING 10.1.20.1 (10.1.20.1): 56 data bytes
64 bytes from 10.1.20.1: seq=0 ttl=64 time=2.885 ms
64 bytes from 10.1.20.1: seq=1 ttl=64 time=0.316 ms
^C
ubuntu@ubuntu:~$
```




---

That was fun. Now, going back to the opening paragraph regarding containerizing routing functions - I do think there are downsides:

- routers are expected to be highly performant, therefore consumes a lot of resources
- direct access to hardware resources (NIC) means the router applications are less portable (you can't just redeploy the application on a different cluster). IP addresses are also often statically assigned to said router functions to form BGP peering

Sometimes I even wonder if it's worth the effort... That's a question for the industry as a whole to ponder on and improve upon.
