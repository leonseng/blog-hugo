---
title: "Wsl as K8s Node With Cilium"
date: 2025-03-10T21:46:52+11:00
draft: true
---


Default

Cilium - routing mode - tunnel
WSL - NAT networking

cilium doesn't run - need to compile kernel with loaded modules

node comes up with NATed IP as node IP
works fine, except when you want to view logs, it tagets NAT_IP:10250, unreachable outside of VM

next, try setting up node IP = host external IP
small problem - k8s needs interface IP
manually set additional ip (host IP) on eth0 (just a vanity IP, non-routable)
kubelet comes up
kubectl log works, just need port-forwarding on windows

problem is when pod in another node tries to access pod on WSL. Cilium uses VXLAN encapsulation by default (UDP XXX), port-forward UDP required,\.

found a program, but connection is flaky, not performant

found mirrored mode on WSL
- in theory remove the need to UDP proxy since WSL "can listen on the ports directly"
- packet capture reveals vxlan dropped [insert image] https://github.com/microsoft/WSL/issues/11027

Need to bypass VXLAN

Update pod networking to use hostnetwork. and it worked!

Though that's just a bandaid

switch to native routing

routingMode: native
ipv4NativeRoutingCidr: 172.16.0.0/16
autoDirectNodeRoutes: true

works


tried swithcing pod networing back, but destination IP is pod IP, which windows cant forwad to WSL???