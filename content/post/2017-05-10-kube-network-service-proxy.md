---
title: Kubernetes network services proxy with IPVS/LVS
date: 2017-05-12
---

A Kubernetes Service is an abstraction which groups a logical set of Pods that provide the same functionality. A service in Kubernetes can be of different types, of which 'ClusterIP' and 'NodePort' types forms basis for service discovery and load balancing. Both of the service types requires a service proxy running on each of the cluster node. Kubernetes has an implementation of service proxy 'Kube-proxy' based on iptables. While Kube-proxy provides out-of-box solution its not necessarily an optimal solution for all users.
In this blog we will go over an implementation of Kubernetes network services proxy built on top of Linux IPVS in [Kube-router](https://github.com/cloudnativelabs/kube-router).
We will also go over pros and cons of Kube-proxy which is based on iptables and compare it against in general any implementation of IPVS/LVS based network service proxy for Kubernetes.

### TL;DR

Please see the demo to get the feel for IPVS in action as service proxy for Kubernetes as implemented in Kube-router.

[![asciicast](https://asciinema.org/a/120312.png)](https://asciinema.org/a/120312)

Lets delve into the details now.

### ClusterIP and NodePort services

Each Service of type 'ClusterIP' and 'NodePort'is assigned a unique IP address (called as clusterIP) throught the lifespan of the Service. Pods can be configured to talk to the Service through cluster IP and know that communication to the Service will be automatically load-balanced out to some pod that is a member of the Service.
Additionally for the Service of "NodePort" type, the Kubernetes master will allocate a port from a flag-configured range (default: 30000-32767), and each Node will proxy that port (the same port number on every Node) into your Service.

### IPVS

[IPVS (IP Virtual Server)](http://kb.linuxvirtualserver.org/wiki/IPVS) implements transport-layer (L4) load balancing inside the Linux kernel. IPVS running on a host acts as a load balancer at the front of a cluster of real servers, it can direct requests for TCP/UDP based services to the real servers, and makes services of the real servers to appear as a virtual service on a single IP address. IPVS support rich set of connection scheduling algorithms (Round-Robin, Weighted Round-Robin, Least-Connection etc.) inside the Linux kernel.
IPVS is known to be fast as its implemented in Kernel and provides different IP load balancing techniques (NAT, TUN and LVS/DR).

### Kubernetes service proxy with IPVS

As we learned earlier each of the Kubernetes service of type ClusterIP and NodePort needs L4 load balancer and service proxy on each of the cluster node. Kubernetes notion of Service and Endpoints logically can be mapped cleanly to IPVS virtual service and server notions. With IPVS doing heavy lifting its fairly easy to implement a solution for Kubernetes that provides service proxy for ClusterIP and NodePort services. We just need a agent running on each of cluster node that watches
Kubernetes API server for changes to Services and Endpoints and reflect desired state in IPVS. 

### Kube-router implementation of service proxy with IPVS

[Kube-router](https://github.com/cloudnativelabs/kube-router) has an implementation of IPVS based service proxy for ClusterIP and NodePort services. Kube-router watches Kubernetes API server for changes (add/delete/updates) to Services and Endpoints objects and configures IPVS virtual service and servers. While mapping of Service and Endpoints to IPVS virtual service and server is straight forward, there are some challenges. 

#### cluster ip

Typically Cluster IP'a are not routable (not a hard restriction from Kubernetes and one can hook up these IP in to the their non-cluster routing environment). A Cluster IP will be shared by all the nodes in the cluster. At the bare minimum the pods on the node need to be able to access the cluster IP. Kube-router solves the problem by having dummy interface on each node which will be exclusively used to assign cluster IP's. Once IP is assigned to the node, pods on the node can
connect to the cluster IP's. Traffic to cluster IP originating external to the nodes are not dropped by default. 

#### reverse traffic

Kube-router uses IPVS NAT mode for load balancing technique. Like any load balancing mechanism based on DNAT, reverse path traffic need to go through the load balancer for end-to-end functioning. There are two data paths thats needs to meet this requirement. For the traffic originating from the pods on the node, only destination NAT is performed. Since Kube-router uses host-based routing for pod-to-pod networking, reverse path traffic goes through the same node from which traffic originated. Traffic originating external to
cluster and accessing node ports needs masqurading traffic so that SNAT is performed. On the reverse path, traffic goes through the same node as due to SNAT. 

#### ipvs conntrack

IPVS uses its own simple and fast connection tracking for performance reasons, instead of using netfilter connection tracking. Kube-router enables IPVS connection tracking.

### Kube-proxy

Kube-proxy is a network proxy service that runs on each cluster node and proxies L4 sessions to service VIP (service cluster ip) or node port to the back end pods in round robin fashion. Kube-proxy is built on top of iptables functionality in Linux. 

#### pros

- leverages iptables, so pretty much any functionality (SNAT, DNAT, port translation etc) can be achieved. Maniputlate packet at different stages (pre-routing, post routing, input, forward, output etc) can be done
- supports port ranges

#### cons

- arcane implementation of load balancer, hard to troubleshoot anything with out understanding the how Kube-proxy uses iptables to implement load balancer
- not a true load balancer but a simple round robin forwarder 
- no load balancing mechanisms
- iptable performance degrades as number of services increases. More number of services means long list of iptable rules in a chain to match a packet against in the data path, and latency in insert/delete rules in control path


### IPVS/LVS based service proxy

#### pros

- hash based matching vs list based iptables rules in the chains. As number of services and endpoints increases, hash based matching of IPVS will be consistent, where as iptables based Kube-proxy will have performance degradation. 
- biggest advantage is easy to verify configuration with ipvsadm
- been around for long time in linux kernel and widely used.

#### cons

- still need iptable tweeks to achieve masqurading etc
- direct routing does not handle port remapping. So direct routing which is the fastest of load balancing algorithm can not be used

### conclusion

An IPVS based service proxy for Kubernetes deployment is a practical option. Its fast, scales well, battle tested also easier to verify and troubleshoot. Unless there are special needs (like support of port ranges) which can be met only with iptables based Kube-proxy, an IPVS based service proxy is better option. Since we are building on top of solid foundation which does heavy lifting a solution can implemented few hundred lines of code and less error prone.

