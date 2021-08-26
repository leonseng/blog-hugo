---
title: "Setting up Multus for multi-interface pods"
date: 2021-08-25T16:47:00+10:00
draft: true
---

This is for the odd bunch of people who need to attach multiple interfaces to their Kubernetes pods. Why?

Coming from the telecommunications industry, there's a push to containerized some of the packet pushing functions on platforms such as Kubernetes to promote DevOps/NetOps practices (think infrastructure/configuration as code). Conventional networking setup for applications just wouldn't cut it for reasons such as:

- ability to connect to multiple networks (VLANs, VPNs etc...) so that containerized routing functions can route traffic between networks
- direct access to the hardware network interface, often achieved with macvlan and SR-IOV to ensure high throughput

Some challenges I've seen so far include:

- routers (especially in the core) are expected to be highly performant, therefore consumes a lot of resources and physical NICs, meaning entire racks of computes are often dedicated to them
- the nature of exchanging routing information using protocols like BGP means IP addresses are often statically assigned to the containerized routing functions, so don't expect to lift and shift the container into another subnet/availability zone using different subnet ranges without some configuration changes on the other side of the connection

But, I'm not here to talk about use cases. Today, I just want to say I can do it 🙃

---


## Setup

I'll be testing this out with a Kubernetes cluster created with Kubespray. The secret sauce here is the [Multus](https://github.com/k8snetworkplumbingwg/multus-cni) CNI which enables attaching multiple network interfaces to pods. On top of that, I'll be using macvlan to give the pods "direct" access to the Kubernetes nodes' network interfaces so that my pods look more like routers with real interfaces.

< insert network diagram >

### Deploy Kubernetes cluster with Kubespray

For deploying the Kubernetes cluster, I just followed the [Quick Start](https://github.com/kubernetes-sigs/kubespray#quick-start) guide, replacing the IPS with the IP addresses of my VMs
```
sudo pip3 install -r requirements.txt
cp -rfp inventory/sample inventory/mycluster
declare -a IPS=(10.1.1.5 10.1.1.6 10.1.1.7)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

### Install Multus

Once the cluster is up and running, I installed the Multus plugin following the Multus' [Installation](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md#installation) instructions:
```
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git && cd multus-cni
cat ./images/multus-daemonset.yml | kubectl apply -f -
```

## Defining Network Attachments

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
        "gateway": "10.1.20.1"
      }
    }'
EOF
```

Create deployment
```
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: busybox
  name: busybox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: macvlan-internal, macvlan-external
      labels:
        app: busybox
    spec:
      containers:
      - image: busybox:1.24
        name: busybox
        command: ["tail"]
        args:
        - "-f"
        - "/dev/null"
EOF
```

## Where's my macvlan CNI plugin

```
ubuntu@ubuntu:~/multus-cni$ k describe pods
Name:         busybox-5f4cf4b5f6-r7xjx
...
Events:
  Type     Reason                  Age               From               Message
  ----     ------                  ----              ----               -------
  Warning  FailedCreatePodSandBox  40s               kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "5278abc55f294af7d2ba3dc285c0bdbc317aeffb99fad9d0fa0ff7f295904605" network for pod "busybox-9d4f77bc6-fg28r": networkPlugin cni failed to set up pod "busybox-9d4f77bc6-fg28r_default" network: [default/busybox-9d4f77bc6-fg28r:macvlan-internal]: error adding container to network "macvlan-internal": failed to find plugin "macvlan" in path [/opt/cni/bin /opt/cni/bin], failed to clean up sandbox container "5278abc55f294af7d2ba3dc285c0bdbc317aeffb99fad9d0fa0ff7f295904605" network for pod "busybox-9d4f77bc6-fg28r": networkPlugin cni failed to teardown pod "busybox-9d4f77bc6-fg28r_default" network: delegateDel: error invoking DelegateDel - "macvlan": error in getting result from DelNetwork: failed to find plugin "macvlan" in path [/opt/cni/bin /opt/cni/bin] / delegateDel: error invoking DelegateDel - "macvlan": error in getting result from DelNetwork: failed to find plugin "macvlan" in path [/opt/cni/bin /opt/cni/bin]]
```

The message tells ms the macvlan plugin can't be found. Checking on the Kubernetes node tells the same story:
```
ubuntu@node1$ ls -la /opt/cni/bin/ | grep macvlan
ubuntu@node1$
```

I don't know why the macvlan plugin isn't provided by Kubespray, but that's an investigation for another time. For now, off I go to build the macvlan plugin from [source](https://github.com/containernetworking/plugins/tree/master/plugins/main/macvlan). The procedure is pretty simple:

1. Make sure I have `go` installed
1. Clone the [containernetworking/plugins](https://github.com/containernetworking/plugins) repository
1. `cd` into plugins/plugins/main/macvlan
1. Run `go build`
1. Copy the resulting macvlan binary into `/opt/cni/bin/` of all my Kubernetes nodes

Once the macvlan plugin is made available, the pod comes up fine
```
ubuntu@ubuntu:~$ k get pods
NAME                      READY   STATUS    RESTARTS   AGE
busybox-9d4f77bc6-fg28r   1/1     Running   2          2h
ubuntu@ubuntu:~$
```

Verify
```
ubuntu@ubuntu:~$ k exec busybox-56588d745b-nk4dn -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1480 qdisc noqueue
    link/ether 82:ea:54:30:dd:a5 brd ff:ff:ff:ff:ff:ff
    inet 10.233.92.11/32 brd 10.233.92.11 scope global eth0
       valid_lft forever preferred_lft forever
5: net1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:09:61:4f:15:f0 brd ff:ff:ff:ff:ff:ff
    inet 10.1.10.203/24 brd 10.1.10.255 scope global net1
       valid_lft forever preferred_lft forever
6: net2@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether a6:3f:9f:cb:46:88 brd ff:ff:ff:ff:ff:ff
    inet 10.1.20.203/24 brd 10.1.20.255 scope global net2
       valid_lft forever preferred_lft forever
ubuntu@ubuntu:~$
ubuntu@ubuntu:~$ k exec busybox-56588d745b-nk4dn -- ping 10.1.20.1
PING 10.1.20.1 (10.1.20.1): 56 data bytes
64 bytes from 10.1.20.1: seq=0 ttl=64 time=0.759 ms
64 bytes from 10.1.20.1: seq=1 ttl=64 time=0.353 ms
^C
ubuntu@ubuntu:~$
```


## SPEEEEEEED

---

That was fun.