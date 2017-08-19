---
title: Kubernetes pod networking and beyond with BGP
date: 2017-05-22
---
In earlier blog on [Kubernetes networking](https://cloudnativelabs.github.io/post/2017-04-18-kubernetes-networking/) we have seen how Kubernetes is non-prescriptive of how the network should be designed for running pods. There can be multiple way to design the network that meets Kubernetes networking requirements with varying degree of complexity, flexibility. In this blog we will see how [Kube-router](https://github.com/cloudnativelabs/kube-router) implements a pure L3 solution for cross node pod-to-pod networking using BGP and see how use of BGP gives unique advantage which enables pod IP and Kubernetes service cluster IP to be routable from out side of the cluster. 

## host routing based L3 solution

As we have seen previous blog [Kubernetes networking](https://cloudnativelabs.github.io/post/2017-04-18-kubernetes-networking/), we can have solution based on either L2 or L3 approach or based on the overly solutions. Kube-router choose to implement a solution based on L3 routing solution. Kube-router does not perform subnet management and relies upon Kubernetes controll manager to do the subnet allocation for each node. Each node in the cluster is allocated unique subnet from the cluster CIDR range for allocation of IP's to the pods running on the node. For cross node pod-to-pod networking we can build a L3 solution either by gateway based routing or host based routing as seen in the [blog](https://cloudnativelabs.github.io/post/2017-04-18-kubernetes-networking/).  

Lets see how host based routing achives cross-node pod-to-pod networking with below example. Lets assume Node1 and Node2 are allocated subnets 10.1.1.0/24 and 10.1.2.0/24 respectivley. So its just matter of installing route on each node to route the subnet to the node to which subnet is allocated as shown below. 

![Host Routing](/img/l3-host-routing.jpg)

A simplest approach Kube-router could have taken is to learn the subnets assinged to each node and install the routes accordingly on each of the nodes. There are two reasons this approach was not taken.
First Kubernetes has a clean approach w.r.t networking for the green field cloud native applications. But its possible in the near future (there is huge community [push](https://groups.google.com/forum/#!topic/kubernetes-sig-network/8nDSQ2hF40w) for the multi-networks ) Kubernetes might have multiple networks and isolated network spanning multiple nodes much like network in IAAS clouds. 
Second if we want pod ip's routable from outside the cluster this approach wont work.

Kube-router has taken the approach of running a standard BGP routing protocol on each node which can advertise pod CIDR's to the peers and configured BGP peers. There are other solutions like Calico, Contiv which similarly use BGP for Kubernetes networking. But Kube-router with ability to act as service proxy as well puts in unique situation to enable new use cases. Kube-router along with advertising pod CIDR of each nodes to peers, can also advertise service cluster IP's to external routing infrastructure.
We will go over the deails how BGP enables these scenarios in below section.

### Why BGP?

First natural question is why use BGP? Is it not overkill for the protocol that runs the internet to be used for simple use-cases? As it turns out BGP core is a relatively simple protocol for routes exchange. As we will see in next section use of BGP is really transparent and no configuration is required for basic pod-to-pod connectivity. In case where you want pod IP's to be routable we just need to peer with BGP router outside the cluster. Use of BGP in the datacenters is quite common and is in fact an approach taken by SDN controllers like Contrail, Nuage for overlay networks in IAAS clouds.

### Pod-to-pod cross node networking with BGP

Kube-router runs iBGP on each cluster nodes, and automatically peers with all the nodes in the cluster in full-mesh at this point (can be easily changed to use route reflector for scaling). All the nodes in the cluster will be put in configurable priavate ASN. Kube-router will advertise the pod CIDR allocated for the node to the peers and install the learned routes from the peers in the node host namespace.

Please see the demo to get the feel for how Kube-router runs iBGP on each node to advertise the route and pod-to-pod networking established.

[![asciicast](https://asciinema.org/a/120885.png)](https://asciinema.org/a/120885)

In this simple setup each node is running BGP is in isolation to the cluster and there are no concerns of routes getting leaked or interfering with underlay so pod IP's remains routable only inside the cluster.

### Routable pods and service IP

We can also peer with existing routers outside the cluster so that pod IP's will be routable from outside the cluster. Also, Kube-router advertises the cluster IP when service is created. A service can be accessed from outside the cluster using single cluster IP and standard ports.  

Please see below demo how Kube-router advertises pod CIDR and service Cluster IP to the external routers.

[![asciicast](https://asciinema.org/a/121635.png)](https://asciinema.org/a/121635)

### conclusion

A pure L3 routing based solution is possibly a best solution in terms of performance, simplicity. We can use standard tools like traceroute, ip route to troubleshoot connectivity issues. Combined with BGP ability to advertise routes to external BGP peers we can have a flexible solution with Kube-router. 

