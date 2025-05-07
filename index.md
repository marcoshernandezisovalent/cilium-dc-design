---
title: Isovalent and Cisco DC Fabrics
layout: default
nav_order: 1
---

{: .warning }
Disclaimer:
This design document is currently under review and is subject to change. The information contained herein is preliminary and may be updated or modified as the review process progresses.

# Introduction
{: .no_toc }


This Best Practices for the Modern Datacenter Documentation Site covers Kubernetes deployments with Cisco ACI and Isovalent Networking for Kubernetes. This site provides guidance on networking best practices for deploying Kubernetes Clusters using Isovalent Networking for Kubernetes (based on Cilium) as the CNI. While it aims to cover a wide range of scenarios and design considerations, please note that not all options are covered. Alternative designs exist, and users are encouraged to conduct their own research and adjust designs to meet specific needs and requirements.

This Best Practices site serves as a guide to help navigate various concepts and options to find a balance between features and complexity when deploying Kubernetes Clusters in the Datacenter. The goal is to provide users with the knowledge and tools needed to implement effective and efficient solutions.

Contributions from the community are welcome. Users with insights or improvements can share them by submitting a pull request or opening an issue on the project's GitHub repository.

## Document Structure

This design document includes the following chapters/sections:

- [Fabric Agnostic Features](docs/fabric_agnostic_features/): This chapter describes the Cilium features common across all designs.
- **ACI Designs:**
  - [Basic Design](docs/aci/simplicity_design/): The most straightforward design to help users get started, ideal for testing basic functionalities.
  - [Advanced Design](docs/aci/advanced_design/): An optimized, high-scale design that trades some configuration simplicity for increased scalability. (Recommended)
  - [ACI BGP Design](docs/aci/aci_bgp_design/): This section provides details on the ACI BGP configuration, which is consistent for both the Basic and Advanced designs.
  - [Scale Testing](docs/aci/aci_tested_scale/): A collection of scale metrics and tests performed for the Advanced design.
  - [Config Examples](docs/aci/examples/examples/): Complete configuration examples for both ACI and Kubernetes/OpenShift.


## Kubernetes

Kubernetes has become the de facto standard for container orchestration in today's cloud-native ecosystem, providing a robust framework for deploying, scaling, and managing containerized applications. As enterprises adopt Kubernetes, they often need to ensure smooth network connectivity and service discovery across diverse and dynamic environments.
Traditionally, implementing BGP peering to all nodes in a Kubernetes cluster has been a common strategy to provide security and visibility into the workload running on the cluster, as well as handling external services routing and load balancing requirements.

BGP peering enables Kubernetes nodes to directly exchange routing information, improving network traffic flow and reducing latency.
However, when designing BGP peering for Kubernetes clusters, particularly as they grow in size and complexity, several key considerations help ensure efficient, scalable, and manageable network configurations.

Two design solutions are proposed:

- A Basic Design trading some features and performance for ease of configuration and operations.
- A slightly more Advanced Design that trades some simplicity for better performance and resilient ECMP hashing towards services.

At present, these designs do not require additional components from Cisco or other third-party vendors to meet their objectives. Users are encouraged to deploy programmatically or even automate the configuration on ACI. These efforts can aid in repeatability, reduce risk, and accelerate deployments. However, such efforts are optional and are beyond the scope of this design document.

This document will examine the technical foundations, implementation considerations, and operational benefits of this approach. It guides readers through the process of implementing this combined model, ensuring that network reliability, performance, and security are maintained and significantly improved.

## Isovalent Networking for Kubernetes (Cilium)

Isovalent Networking for Kubernetes provides networking, security, and observability capabilities for containerized applications, particularly in Kubernetes environments. It uses eBPF (extended Berkeley Packet Filter), a powerful Linux kernel technology, to manage network policies, service connectivity, and security efficiently at the application layer. It is the enterprise offering of the Cilium open source project, created by Isovalent.

Key features of Isovalent Networking for Kubernetes include:
* **Advanced Network Security**: Enables fine-grained security policies defined based on application-layer identities rather than just IP addresses. This allows for more precise control over communication between microservices.
* **Scalability and Performance**: By using eBPF, Isovalent Networking for Kubernetes achieves high performance and scalability, running network policies directly in the Linux kernel without needing complex user-space processing.
* **Service Mesh Integration**: Can be integrated with service mesh architectures to provide transparent, high-performance networking and security features that complement service mesh features.
* **Observability**: Offers robust observability tools, allowing users to monitor and troubleshoot network traffic and security policies with detailed metrics and visibility into application communication patterns.
* **Kubernetes Native**: Designed for Kubernetes, it seamlessly integrates to provide native support for network policies and service discovery.

Isovalent Networking for Kubernetes is widely adopted in cloud-native environments for its ability to enhance security and network performance while providing comprehensive observability and integration with existing Kubernetes infrastructure.

## Isovalent

As the creator of eBPF, Cilium, and Tetragon, Isovalent is a key part of innovation in cloud-native ecosystems. All major cloud providers have adopted Isovalent’s technologies, making it the de-facto standard for cloud-native networking and security.

The Isovalent Enterprise Platform provides advanced Networking, Security, and Observability solutions for Kubernetes and cloud-native environments. This is combined with a [global customer support offering](https://isovalent.com/blog/post/isovalent-enterprise-cilium-support-customer-environments/).

Isovalent joined Cisco via acquisition in April 2024.

The Isovalent Platform is used by enterprises such as Goldman Sachs, J.P. Morgan Chase, UBS, and S&P Global, as well as technology leaders like Databricks, Adobe, and Confluent, and some leading AI Large Language Model (LLM) providers.


{: .note }
Disclaimer: This document has been developed specifically for the **Isovalent Networking for Kubernetes** version. No support or guidance is provided for the Cilium open-source version.
Cisco TAC does not provide support for Cilium, and Isovalent Networking for Kubernetes support can be purchased separately.

## How to Request Design Assistance

It is understood that every network environment is unique. Assistance is available to help adjust solutions to meet specific needs. For design assistance, experts are available to help through the process. Contact information:

### Contact Information

* [Contact Isovalent](mailto:isovalentsales@cisco.com)
* Contact the Cisco account manager ([Global Contact Page](https://www.cisco.com/site/us/en/about/contact-cisco/index.html))
* [Request a demo](https://isovalent.com/request-demo/)

Support aims to provide personalized guidance to ensure the network design aligns with operational requirements and future growth plans, helping to achieve project goals.

## Goals

The design goals for this Kubernetes networking solution focus on addressing key challenges administrators face when managing traffic flow and ensuring secure connectivity within and outside the Kubernetes cluster. The approach implements a robust and scalable load-balancing strategy combined with a secure networking model that connects containerized applications and external non-containerized systems.

{: .note }
With both designs discussed in this document, the role of the network fabric (ACI in this case) is to provide **North-South** traffic filtering. ACI can selectively permit or deny traffic ingressing or egressing the Kubernetes or Openshift cluster using a combination of security contracts and [ESGs](https://www.cisco.com/c/en/us/td/docs/dcn/whitepapers/cisco-aci-esg-design-guide.html). ACI **cannot** control intra-cluster (East-West) traffic using the approaches proposed here. This role belongs to [Cilium's rich security feature set](https://docs.cilium.io/en/stable/overview/intro/). Isovalent Networking for Kubernetes includes a powerful yet easy to use [GUI tool](https://docs.cilium.io/en/stable/observability/hubble/#hubble-intro) that lets security administrators create, test, validate and monitor intra-cluster security policies. Using ACI and Cilium together provides a very complementary layered approach to securing the data center.


## Two Designs
This design document outlines two approaches to achieving the networking objectives. Each option is designed for different priorities and operational considerations.

Regardless of the chosen option, **both** designs provide the following outcomes:

* High Performance
  * By using [Native Routing](https://docs.cilium.io/en/stable/network/concepts/routing/#native-routing) it is possible to increase network stack performance by removing the Node-to-Node overlay as well as enabling advanced features like [BigTCP](https://docs.cilium.io/en/stable/operations/performance/tuning/#ipv4-big-tcp).

  {: .note }
  BigTCP is currently in beta and should not be used for production workloads. However, this design is ready for its future stable release, allowing it to be enabled easily with a simple configuration flag once it becomes available.

* Efficient load balancing:
  * Load-balancing that effectively distributes external client traffic to services within the Kubernetes cluster.
  * Utilize cluster nodes to perform BGP peering, enabling Layer-3/Layer-4 Equal-Cost Multi-Path (ECMP)-based load balancing.

* External service security:
    * The load balancer IP is mapped to an ACI external Endpoint Group (external EPG) to help enforce security policies through ACI contracts.

* Pod identity
  * With the [Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway/egress-gateway/) feature, it's possible to masquerade Pod traffic leaving the cluster with predictable IPs associated with the gateway nodes. As an example, this feature can be used in combination with legacy firewalls to allow traffic to legacy infrastructure only from specific pods within a given namespace.

* DHCP relay support:
  * This makes it easy to bootstrap and horizontally scale the cluster.

  {: .warning }
  Note the DHCP relay limitations for L3outs. See: [DHCP Limitations](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/6x/basic-configuration/cisco-apic-basic-configuration-guide-61x/provisioning-core-aci-fabric-services-61x.html#guidelines-and-limitations-for-a-dhcp-relay-policy)


{: .note }
To maintain clarity and ease of reading, all configuration snippets and examples have been grouped into dedicated example sections within this document. This structure ensures that relevant information and guidance on configurations are easily accessible and presented in a clear manner. Readers can refer to these sections for detailed configurations and examples.

### Basic Design

This design serves as an excellent starting point for quickly getting started, allowing exploration of Cilium's capabilities easily.

* Objectives:
  * Ability to run on any ACI Software version.
  * Provide a straightforward and easily maintainable solution.
  * Utilize ACI for External Service load-balancing.
  * Utilize ACI Contracts to control North-South traffic.
* Approach:
  * All Cluster Nodes are deployed in an EPG and linked to an ESG.
  * Native routing mode: Since all nodes share the same L2 Domain (the Node BD), Pod-to-Pod traffic can be directly routed.
  * Dedicated Egress nodes, with the Egress IP(s) linked to an ESG.
  * Dedicated Ingress nodes peering with BGP for External Services.
* Benefits: Quick deployment, reduced complexity, and lower operational overhead.
* Limitations:
  * Cilium selects service backends randomly and ensures that traffic remains sticky to the backend. However, this method can cause issues if a node fails or if there is a change in the ECMP Next Hops. This can result in traffic being sent to a different Kubernetes node, which lacks context about which backend was handling the connection. This can lead to unexpected disruptions on connection-oriented protocols like TCP, as client connections get reset by the newly selected backends.
  * May not fully optimize network throughput in scenarios with complex traffic patterns.

### Advanced Design with Maglev for ECMP Flow Consistency (Recommended)

This design offers a more comprehensive solution, ideal for production environments. It includes advanced features and optimizations recommended for using the full potential of Cilium in a stable and scalable manner. While it may require more initial setup and configuration, this design ensures robust performance and resilience, making it well-suited for enterprise-level deployments.

* Objectives:
  * Maximize network performance and ensure consistent flow distribution in ECMP scenarios with [Maglev](docs/fabric_agnostic_features/#maglev).
  * Utilize ACI Contracts to control North-South traffic.
* Approach:
    * All Nodes are deployed behind a L3Out.
      * Use a `Node` External EPG to secure Kubernetes Node traffic.
      * Use one External EPG per exposed External Service for North-South traffic accessing Kubernetes services.
      * Use [Maglev hashing](docs/fabric_agnostic_features/#maglev) to maintain ECMP flow consistency. It is suited for environments needing optimal load distribution and performance. Maglev requires the following:
        * Direct Server Return: the response is sent directly back to the client from the pod, skipping the original node that received the request.
        * Native routing mode: The native routing mode uses the routing capabilities of the network Cilium runs on instead of performing encapsulation for Pod-to-Pod traffic.
    * Dedicated Egress nodes, with the Egress IP(s) linked to an ESG.
* Benefits: Improved flow consistency, enhanced network utilization, and better handling of dynamic traffic patterns.
* Limitations:
  * ACI 6.1(2) or later is required.
  * Node IP visibility is reduced as they are placed behind an L3Out.


[Next](/cilium-dc-design/docs/fabric_agnostic_features/){: .btn }
{: .text-right }
