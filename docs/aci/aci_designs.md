---
title: ACI Designs
layout: default
nav_order: 2
---

# Executive Summary

Application modernization represents a significant trend, with numerous organizations refactoring or re-architecting existing applications to capitalize on cloud-native technologies for enhanced agility and scalability. Nevertheless, a substantial portion of applications often remains in a legacy state due to factors such as complexity, cost considerations, and risk aversion. This situation commonly results in a hybrid environment where modernized and traditional applications must coexist.

Effectively managing traffic between modernized, containerized workloads and legacy applications necessitates a solution capable of bridging these distinct environments. Cisco ACI can supply the network policy framework and connectivity for legacy workloads, while Cilium manages network policies within the Kubernetes clusters hosting the modernized applications. By utilizing the designs presented for ACI and Cilium, organizations can attain consistent security and observability across both legacy and cloud-native segments, facilitating seamless communication and control.

Within Kubernetes environments, Cilium specializes in enforcing east-west network policies, governing communication between microservices and pods internal to the cluster. By leveraging eBPF technology, Cilium delivers efficient and granular control over this internal traffic. This allows administrators to define precise policies that restrict inter-service communication based on identity, labels, or other criteria. Such east-west policy enforcement is fundamental for securing Kubernetes environments, mitigating the potential impact (blast radius) of security breaches, and fulfilling regulatory compliance mandates. Furthermore, Cilium improves observability by providing real-time insights into network traffic flows, equipping administrators to monitor and troubleshoot network behavior with greater effectiveness.

The designs detailed on this site aim to strike a balance between implementing robust security policies, ensuring high service availability, and facilitating seamless, optimal communication across diverse application architectures.

At a high level, the following traffic patterns are considered:

- **Node-to-Node Communication:** This communication pattern forms the bedrock for much of the networking functionality within a Kubernetes cluster, enabling effective interaction between pods, services, and control plane components. Cilium Native Routing represents a networking mode within Cilium where network packets are routed directly between nodes by leveraging the Linux kernel's native routing capabilities, rather than depending on overlay networks such as VXLAN. This approach retains the benefits offered by eBPF while eliminating encapsulation overhead, leading to improved performance. Connectivity for this pattern is provided by an L3Out or Bridge Domain (BD) within ACI.

- **External Service Access:** Within Kubernetes, a Service acts as an abstraction layer defining a logical group of Pods and the policy for accessing them. It furnishes a stable IP address for reaching applications running in Pods, irrespective of Pod scaling or rescheduling events. The IP addresses allocated to these Services are deterministic. Each service is advertised towards ACI via BGP as a `/32` host route. These routes can be readily classified using ACI External End Point Groups (ExtEPGs). This classification enables administrators to apply contracts governing communication to the service and, where necessary, implement service graph redirection (e.g., to a firewall). BGP, being a well-established and highly scalable routing protocol, facilitates the advertisement of Kubernetes services, allowing services to be easily scaled and adapted to evolving network conditions.

- **Egress Communication:** Also referred to as Pod-initiated traffic, this pattern includes any network connection originating from a Kubernetes pod destined for external services, APIs, databases, or other resources. Kubernetes Pods are inherently ephemeral; they can be created, destroyed, and rescheduled frequently, resulting in dynamically assigned IP addresses. Consequently, relying on these transient Pod IPs for security policies or auditing purposes proves unreliable due to their frequent changes. Typically, Kubernetes employs Source Network Address Translation (SNAT) on worker nodes to enable Pod access to external networks. SNAT replaces the source Pod IP address with the worker node's IP address. As a result, external services perceive traffic as originating from the node's IP, obscuring the identity and context of the initiating Pod. Egress Gateways in Kubernetes offer controlled exit points for traffic leaving the cluster. Implementing High Availability (HA) for these Egress Gateways significantly boosts the reliability and resilience of egress traffic paths. Similar to Service IP allocation, the IP addresses assigned to Egress Gateways are deterministic. Labels within Kubernetes can be leveraged to specify how individual Pods, or entire Namespaces, egress the cluster. By integrating the deterministic nature of Egress Gateways with ACI External EPGs or External Security Groups (ESGs) using IP Selectors, administrators can enforce granular control over the external services accessible by Pods.

![High level design overview](../images/exec-summary.png)

The illustration provides a simplified overview of how policy can be applied to manage north-south communication flows to and from a Kubernetes cluster. In the depicted scenario, HR users can access the HR application, but their traffic is first redirected through a firewall for inspection. Conversely, users accessing the Finance application bypass the firewall and are restricted solely to Finance services. Pods within the Corporate Namespace possess the ability to access other applications within the Datacenter, while Pods in the Marketing namespace are limited to accessing the internet, potentially routed via a firewall.

In summary, the integration of Cisco Application Centric Infrastructure (ACI) and Cilium presents a unified solution for managing hybrid environments, effectively bridging legacy and cloud-native application landscapes:

- **Security and Observability:** ACI delivers consistent security policy enforcement for legacy applications, complemented by Cilium's granular network policy enforcement within Kubernetes clusters.

- **Traffic Control:** Cilium adeptly manages east-west traffic, optimizing internal cluster communications, while ACI provides robust capabilities for north-south traffic management, ensuring precise control and facilitating service insertion.

- **Service and Egress Management:** The combination of deterministic external service access through BGP and controlled egress communication via Egress Gateways and ACI ESGs/ExtEPGs enhances both scalability and security posture.

[Next](/cilium-dc-design/docs/aci/basic_design/){: .btn }
{: .text-right }
