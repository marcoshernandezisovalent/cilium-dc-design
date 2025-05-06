---
title: Fabric Agnostic Features
layout: default
parent: Isovalent and Cisco DC Fabrics
---
## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Common Features

### Native Routing and Auto Direct Node Routes

Using these two features allows highly efficient packet forwarding between pods without the need for additional encapsulation or overlay networks. 

In a Kubernetes deployment running in direct mode on a network fabric, each pod is assigned an IP address from a routable subnet with direct Layer 2 or Layer 3 connectivity. This allows pods to communicate natively over the physical network infrastructure, eliminating the latency, CPU overhead, and complexity typically associated with tunneling protocols such as VXLAN or GRE. 

By maintaining direct routing to each pod subnet, the solution ensures minimal packet traversal path and maximum throughput delivering the best possible performance for East-West traffic in the cluster. This architecture is especially beneficial in performance-sensitive environments where low latency and deterministic network behavior are critical.

{: .warning}
This design aims to cover the vast majority of customer needs. However, if a cluster grows beyond 1,000 nodes, changes to the design might be needed. In these cases, it is strongly suggested to contact Isovalent for more help and support.

### Isovalent Networking for Kubernetes BGP Control Plane

The Cilium BGP Control Plane [Enterprise feature](https://docs.isovalent.com/configuration-guide/networking/bgpv2/index.html) lets Cilium announce routes to connected routers using the Border Gateway Protocol (BGP). It makes pod networks or services of Type LoadBalancer reachable from outside the cluster in environments that support BGP. In Cilium, the BGP Control Plane doesn't set up the Linux host's data path directly, so it cannot be used by itself to create IP connections within the cluster or to external IPs.

### Cilium Egress Gateway

The [Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway/egress-gateway) feature allows sending traffic that starts from pods and goes to specific network ranges (CIDRs) outside the cluster through certain chosen nodes.

When the egress gateway feature is turned on and egress gateway policies are active, packets leaving the cluster have their source IP address changed to selected, known IPs (`egressIP`) linked to the gateway nodes. This lets network administrators set network rules for pods or namespaces making outbound connections.

{: .warning}
Only Isovalent Networking for Kubernetes supports [Egress Gateway High Availability](https://docs.isovalent.com/configuration-guide/networking/egress-gateway/index.html) (making sure the egress path keeps working even if one gateway node fails).

### XDP Acceleration

[XDP Acceleration](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#loadbalancer-nodeport-xdp-acceleration) helps speed up NodePort, LoadBalancer services, and services with externalIPs when incoming requests need to be forwarded to a backend pod located on a different node. This feature was added in Cilium version 1.8. It works at the XDP (eXpress Data Path) layer, where eBPF runs directly in the network card driver instead of higher up in the system.

Most network drivers that support 10Gbps speeds or higher also support native XDP if using a recent kernel version. For a list of drivers taht support XDP, please refer to the [official Cilium documentation](https://docs.cilium.io/en/stable/reference-guides/bpf/progtypes/#xdp-drivers)

## Advanced Design Only Features

### Maglev

Using advanced load balancing methods in Kubernetes clusters is important for keeping performance high and using resources well. Combining Cilium with Maglev hashing offers a strong solution for stable and efficient load spreading, especially when using Equal-Cost Multi-Path (ECMP) routing.

Maglev consistent hashing reduces problems by making sure each load balancing node has the same view and order for its list of backend servers. This means choosing the backend server based on the packet's details (5-tuple hash) will always send the traffic to the same server without needing to share information with other nodes. This not only makes the system more reliable if failures happen but also spreads the load better because new nodes added to the cluster will choose the same backend server consistently.

![cilium-maglev](../images/cilium-maglev.gif)

Besides that, the Maglev consistent hashing method ensures load is spread evenly among backend servers and causes few problems when backends are added or removed. Specifically, a connection is very likely to pick the same backend server after a server is added or removed as it did before. When a server is removed, the backend lists are updated with few changes for other servers. Usually, this means at most 1% of connections might go to a different server than before.

This is especially important for networks that don't support ECMP Resilient Hashing.

To use Maglev hashing successfully in Kubernetes with Cilium, a main requirement is that Kubernetes nodes can target the Pod IP directly when routing traffic. This ability is needed for keeping load balancing consistent and efficient. Also, for best performance and less delay, the node where the Pod is located should be able to send replies directly back to the network. This direct reply method needs Direct Server Return (DSR).

[This blog post](https://cilium.io/blog/2020/11/10/cilium-19/) explains Maglev in more detail.

### Direct Server Return

When Direct Server Return (DSR) mode is turned on in Cilium, handling traffic from outside the nodes is made much better. In DSR mode, after a request reaches a backend pod on a different node, the reply is sent directly back to the client from that pod. It bypasses the original node that first received the request. This removes the extra step needed for changing the source IP back (reverse SNAT), reducing delay and improving overall network performance.

DSR mode offers two main benefits:

* **Keeps Source IP:** Because the backend pod sends the reply directly to the client, the client's original source IP is kept. This is very helpful for security and monitoring. It lets network rules and logs on the backend node correctly see and match the client's real IP address. This can be vital for setting detailed security rules and for fixing problems.
* **Less Delay and Load:** By removing the need for the reply to go back through the node that first received the request, delay is reduced. This also lowers the processing work on that initial node, as it doesn't have to handle changing IPs back for replies. This allows it to handle incoming requests more efficiently.

Overall, turning on DSR in Cilium improves network traffic flow, boosts security, and uses resources better within the Kubernetes cluster. It's a good choice for setups where performance and knowing the client's true IP are important.

{: .note}
To make sure Maglev and Direct Server Return (DSR) work correctly, the service traffic policy needs to be set to `externalTrafficPolicy: Cluster`. This setup lets all nodes forward traffic to the external service IP, ensuring connections work smoothly. To keep things simple, this design uses BGP peering between all nodes and the fabric. However, if the goal is to reduce the number of BGP neighbors, BGP peering can be set up only with a chosen group of nodes. This can be done using node labels and node selectors in the `IsovalentBGPPeerConfig`, allowing more specific network setups.
If your `egress nodes` are set up to announce IPs over BGP, it is important to ensure they also have BGP connections with the fabric. To keep egress and ingress nodes separate, multiple `IsovalentBGPPeerConfig` settings can be used. This method lets you manage BGP settings separately for different node roles, improving network separation and operational control.
For help with this configuration, please [contact us](../../#how-to-request-design-assistance).


[Next](/cilium-dc-design/docs/aci/aci_designs/){: .btn }
{: .text-right }
