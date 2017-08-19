---
title: Enforcing Kubernetes network policies with iptables
date: 2017-05-03
---

Network policies in Kubernetes provides primary means to secure a pod by exerting control over who can connect to pod. Intent of this blog post is not to describe what network policies are but to show how iptables on the the cluster nodes can be used to build a distributed firewall solution that enforces network policies in Kubernetes clusters. This write up draws up from the insights of implementing a network policy controller in [Kube-router](https://github.com/cloudnativelabs/kube-router). But concepts described can be used to build your own version of network policy enforcer with iptables.

Kubernetes networking has following security model:

* for every pod by default ingress is allowed, so a pod can receive traffic from any one
* default allow behaviour can be changed to default deny on per namespace basis. When a namespace is configured with isolation tye of **DefaultDeny** no traffic is allowed to the pods in that namespace
* when a namespace is configured with **DefaultDeny** isolation type, network policies can be configured in the namespace to whitelist the traffic to the pods in that namespace

### Demo

Lets see quick demo of network policies in action

[![asciicast](https://asciinema.org/a/120735.png)](https://asciinema.org/a/120735)

Lets quickly recap what network policies are then we will dive in to the iptables based solution aspects.

### network policy

Kubernetes network policies are application centric compared to infrastructure/network centric standard firewalls. There are no explicit CIDR or IP used for matching source or destination IP's. Network policies build up on labels and selectors which are key concepts of Kubernetes that are used to organize (for e.g all DB tier pods of app) and select subsets of objects. A typical network policy looks like below:
    
``` :text  
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
   - from:
     - namespaceSelector:
       matchLabels:
         project: myproject
     - podSelector:
       matchLabels:
       role: frontend
   ports:
     - protocol: tcp
     port: 6379
```    

In the network policy **matchLabels** in **podSelector** section is used to identify set of destination pods for which a network policy applies. Similarly ingress section of network policy has a **matchLabels** in **podSelector** section which identifies the set of pods that will become source pods set. Alternatively **namespaceSelector** in the ingress section can be used to select all pods in a namespace. Ports section of network policy spec defines one or more port and protocol combinations that are to be opened up.

### evaluating networking policies

Network policies are declarative in nature to express application intent. So its implementation of network policies responsibility to translate the intent to configuration. Kubernetes provides a convenient way to retrieve the set of pods matching a label selector through API or kubectl as shown below.
    
``` :text
curl 192.168.1.99:8080/api/v1/namespaces/testns1/pods?labelSelector=app%3Dguestbook,tier%3Dfrontend
kubectl get pods -o json --selector=app=guestbook,tier=frontend --namespace=testns1 | jq '.items[] | .status | .podIP'
```    

So its easy to get the set of source pod IP's and set of destination pod IP's based on the **matchLables** in network policy spec. Essentially a network policy implicitly evaluates to be definition of set of source pods that can talk to set of destination pod over specified port and protocol. Pods being ephermal in nature, set of pods matching the label selector is a dynamic set. In later section we will see how this poses challenges in how delta changes to iptable configuration applied with minimal disruption to datapath. But first lets see how we can configure iptables at a given time to reflect current state of network policies.

### representing network policy in iptables

Since we are only concerned about the filtering of the packets, we will use only iptable filter functionality and all the examples below refer to the use of **filter** table.

We have seen how we can evaluate a network policy to a whitelist rules expressing set of source pod IP's that can reach port and protocol on set of destination pod IP's. A natural choice is to represent each network policy as user defined chain. A network policy evaluates to single destination pod set. But a network policy specification can have multiple ingress rules. Each ingress rule evaluates to set of pods that are source pods, port and protocol combination. Below is pseudo code how user chain for the network policy can be populated with rules to accept the traffic.
    
``` :text   
for each dstIP in set of destination pod IP as per the podSelector in the network policy {
    for each ingressRule in ingress rules of network policy {
        for each srcIP in set of source pod IP as per the podSelector in $ingressRule {
            for each port, protocol combination in ports section of ingress rule of the network policy {
                 iptables -A KUBE-NWPLCY-CHAIN -s $srcIP -d $dstIP -p $protocol -m $protocol --dport $port -j ACCEPT 
            }
        }
    }
}
``` 

Since we have a rule for each source pod ip and destination pod IP combination, as you may have suspected this will blow up the chain with large number of rules. Fortunatley we can use **ipset** to better represent the rules and more importantly a data path that scales well. A slightly modified pseudo code to populate the rules in the chain.
    
    
``` :text   
form ipset dstIpset with all destination pod ip evaluated as per the podSelector in the policy
for each ingressRule in ingressRules of network policy {
    form ipset srcIpset wth all source pod IP as evaluated as per the podSelector in ingress rule of the network policy 
    for port, protocol in ports of the network policy {
         iptables -A KUBE-NWPLCY-CHAIN -p $protocol -m set --match-set $srcIpset src -m set --match-set dstIpset dst -m $protocol --dport $port -j ACCEPT
    }
}
```    

Each cluster node can configured with representation of network policies with iptables as above. Destination ipset can be further optimized to represent pod IP of the pods running on specific node. Now lets see how inbound traffic to pod can be run through the network policy chains.

### Pod ingress traffic paths

There are multiple data paths through which packet is received by a pod as shown in below diagram.

![Pod Ingress traffic](/img/pod-ingress-traffic.jpg)

1. Pod on a node reaching to pod on same node through bridge (pod2->pod3)
2. Pod on a node reaching to pod on same node but through the service proxy (pod1->pod2)
3. Pod on a node reaching to pod on other node (pod3->pod4)
4. Pod on a node reaching to pod on other node through service proxy (pod5->pod6)
5. external client reaching to pod (external client->pod7)

### pod specific firewall chain

Ingress traffic to a pod can be whitelisted through one or more network policies. So its important that packet destined to a pod is run through the necessary network policy chains. Again its easier to have iptable chain per pod. Now we need to run through the traffic destined for a pod through rules in the pod specific firewall chain. We only have to consider the pods whose namespace has ingress set to DefaultDeny configuration. Also on a any cluster node, we are only concerned about the pods running on the node.

For the case #1 data path above we will need to use **physdev** module to match the traffic that is getting bridged locally. From iptables perspective it will still go through the FORWARD chain. For the case #2 from iptables perspective it is output packet, so it will only hit OUTPUT chain. Rest of the cases #3 to #5 all fall under the same category. Traffic will hit the FORWARD chain. Following is the pseudo code to populate FORWARD and OUTPUT chains to jump the traffic destined for a pod to pod specific firewall chain.
    
``` :text
for each pod running on the node {
    if pod namespace has 'DefautDeny' as ingress type {
        iptables -A FORWARD -d $podIP -m physdev --physdev-is-bridged -j KUBE-POD-SPECIFIC-FW-CHAIN
        iptables -A FORWARD -d $podIP -j KUBE-POD-SPECIFIC-FW-CHAIN
        iptables '-A OUTPUT -d $podIP  -j KUBE-POD-SPECIFIC-FW-CHAIN
    }
}
```    

### rules to run through network policies

We will need rules in the pod specific firewall chain to take appropriate actions. First we need to have default rule as a last rule in the chain to REJECT the traffic in case traffic destined to pod does not match any white list rules in the network policy chains. We will need a rule to permit the return traffic to the pod (for the connections which originated from the pod) using conntrack. A sample set of rules in the pod specific firewall chain would look like below. Number of rules to jump to the network policy chains depends on the how many network policies are applicable to a pod.
    
``` :text   
iptables -A KUBE-POD-SPECIFIC-FW-CHAIN -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A KUBE-POD-SPECIFIC-FW-CHAIN -j KUBE-NWPLCY-CHAIN1
iptables -A KUBE-POD-SPECIFIC-FW-CHAIN -j KUBE-NWPLCY-CHAIN2
iptables -A KUBE-POD-SPECIFIC-FW-CHAIN -j KUBE-NWPLCY-CHAIN3
iptables -A KUBE-POD-SPECIFIC-FW-CHAIN -j REJECT --reject-with icmp-port-unreachable
```    

### walkthrough how iptables are setup

Quick walkthrough of how iptables are setup on the node

[![asciicast](https://asciinema.org/a/120868.png)](https://asciinema.org/a/120868)

### packet flow

Packet flow through the cluster node goes through following path with pod specific firewall and network policy chains in place:

* destination IP is matched in FORWARD and OUTPUT chains of filter table, if destination ip corresponds to a pod running on the node, that needs ingress blocked by default and only permit the traffic as per the network policies, then jump to pod specific firewall chain.
* in the pod specific firewall chain, if the packet belongs to established or related session then allow the traffic, else
* in the pod specific firewall chain run through all network policy chain, if there is any matching rule (matching source ip, port and protocol) in any of the network policy chains then packet is accepted
* if there is no match of rule in any of the network policy chains then packets get dropped.

### state synchronization

Like mentioned earlier network policy only express the intent. So its implementations responsibility to translate the intent to desired configuration. There are number of events that need reconfiguring the iptable so that configuration reflects the desired intent. For e.g

* add/delete of pods
* modification to namespace default ingress behaviour
* add/delete of network policies

All these events will cause the add/delete of rules in the chains, or add/deletion of network policy and pod specific firewall chain. Implementation of network policies have to watch for the events from Kubernetes API server and have to reflect the changes in the iptables configuration.

### conclusion

We went over how iptables can be used to implement a solution that can enforce network policies on each cluster node. An implementation of approach discussed in the article is implemented in [Kube-router](https://github.com/cloudnativelabs/kube-router). You can deploy Kube-router and see how iptables are used as effective solution to enforce network policies.


