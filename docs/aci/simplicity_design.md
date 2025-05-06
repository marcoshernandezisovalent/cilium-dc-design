---
title: Basic Design
layout: default
parent: ACI Designs
nav_order: 1
---

# Basic Design

{: .note }
This design is primarily intended for testing or lab environments. For full production deployments, reviewing and potentially adopting the [Advanced Design](../../advanced_design) is recommended.

The basic network infrastructure for this Basic Design is composed of the following required ACI components:

* **Tenant:** The Kubernetes cluster resides within a dedicated or shared ACI tenant.
* **Primary Bridge Domain (BD):** A central BD provides L2 connectivity and L3 gateways for the Kubernetes nodes.
    * Configuration includes a **primary subnet** matching the Kubernetes node IP range. The BD Subnet Virtual IP (SVI) serves as the default gateway for the nodes.
    * Configuration includes **one or more secondary subnets**. These IP ranges are designated for use by the Cilium Egress Gateway feature as source IPs for egress traffic.
* **Primary Endpoint Group (EPG):** An EPG linked to the primary BD is created. The primary network interfaces of all Kubernetes nodes are associated with this EPG (e.g., via static path bindings or VMM integration).
* **Node Endpoint Security Group (ESG):** A `node` ESG is created. An IP-based selector matching the primary node subnet groups all nodes within this ESG, allowing nodes within the ESG to communicate without contracts.
* **Egress Endpoint Security Groups (ESGs):** Additional `egress` ESGs are created. Each uses an IP-based selector matching specific `Egress IP` addresses (derived from the egress gateway configuration). These ESGs classify egress traffic based on Pod/Namespace identity for granular ACI contract enforcement.
* **Dedicated L3Out for Ingress:** A separate Floating SVI L3Out is configured. This L3Out establishes BGP peering exclusively with a designated subset of Kubernetes nodes (`ingress nodes`) for advertising external Kubernetes Services.

This Basic Design provides the following capabilities:

* Secures traffic initiated *by* Kubernetes nodes (e.g., accessing external services) using ACI contracts applied via the `node` ESG.

    {: .note }
    Internal cluster micro-segmentation using [CiliumNetworkPolicies](https://docs.cilium.io/en/stable/security/policy/index.html) is a key feature of Cilium itself, but detailed configuration is beyond the scope of this ACI integration guide.

* Secures traffic initiated *from* Pods that is NATted using specific Egress IPs, leveraging ACI contracts applied via the dedicated `egress` ESGs.
* Supports DHCP Relay: Enables Kubernetes nodes to obtain IP addresses automatically during bootstrap via the ACI fabric, simplifying deployment and scaling.

* Provides direct visibility of node IPs within the ACI fabric via the EPG/ESG association.
* Supports heterogeneous node types: The cluster can comprise a mix of bare-metal servers and virtual machines on various hypervisors, provided they connect to the designated ACI EPG.
* Offers routing simplicity: The default gateway for all nodes is the SVI IP address of the primary BD subnet.
* Enables BGP-based ECMP for load balancing external Kubernetes service traffic across the designated `ingress nodes`.

## Cluster EPG Physical Connectivity

No strict requirements dictate the physical connectivity method for attaching Kubernetes nodes to the cluster EPG, provided the necessary level of redundancy is met. Designs often utilize vPC for host connectivity. Implementing L2 redundancy generally improves failover performance compared to relying solely on L3 convergence mechanisms.

## Selective BGP Peering for Service Advertisement

This design employs selective BGP peering. A dedicated L3Out (typically using Floating SVI) is created in ACI solely for establishing BGP sessions with a designated subset of Kubernetes nodes, referred to as `ingress nodes`. These nodes advertise external Kubernetes Service IPs (`/32` routes) to the ACI fabric via this L3Out.

The `ingress nodes` require configuration with two logical network interfaces:

1.  **Primary Interface:** Connects to the main cluster EPG/BD (same as non-ingress nodes). This handles standard node-to-node communication and uses the BD SVI as its default gateway. This interface's IP should be the primary `node-ip` for kubelet. Keeping `ingress nodes` within the main EPG simplifies node provisioning (e.g., DHCP) and internal cluster communication.
2.  **Secondary Interface (for BGP):** Connects (logically or physically) to the network segment defined by the dedicated L3Out. This interface is used for the BGP peering session with the ACI anchor leaves associated with the L3Out.

![Selective BGP peering design](../images/selective-bgp.png)
*Selective BGP Peering Design*

**Routing Considerations for Ingress Nodes:**

Traffic arriving at an `ingress node` via the L3Out (destined for a Kubernetes Service IP) must have its return traffic sent back out through the same L3Out path to maintain symmetry and avoid potential RPF issues or policy drops on the primary interface path. Linux Policy-Based Routing (PBR) is configured on the `ingress nodes` to achieve this:

* **Create a separate routing table:** (e.g., table ID `100`).
* **Add default route in the new table:** In table `100`, configure a default route pointing to the appropriate gateway IP address within the L3Out's network segment (e.g., the Floating SVI IP if used).
* **Create IP routing rules:** Use `ip rule add` commands to direct traffic *sourced from* the Kubernetes Service IP range(s) handled by this node to use routing table `100`.

This ensures that replies to client requests arriving via the L3Out are routed back out the secondary interface towards the L3Out gateway.

![Cilium BGP Control Plane traffic flows](../images/BGP-Control-Plane-flow.png)
*Cilium BGP Control Plane Traffic Flows*

{: .note }
For nodes configured with multiple network interfaces, it is fundamental to ensure that [kubelet's `node-ip` is set correctly on each node](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/#create-the-config-file). In the Basic Design, this **must** be the IP address of the primary interface connected to the main ACI EPG/BD. Cilium relies on this primary node IP for internal pod-to-pod east-west routing and does not provide mechanisms to choose alternative interfaces for this traffic.

Refer to the [ACI BGP Design](/cilium-dc-design/docs/aci/aci_bgp_design/) section for general ACI BGP configuration details applicable here.

## Cilium BGP Design Considerations

From the Cilium perspective, the main consideration in this design is determining the number of `ingress nodes` to deploy and deciding whether to dedicate these nodes solely to the ingress function.

**Deployment Recommendations:**

* **Redundancy:** Deploy a minimum of two `ingress nodes`, preferably connected to different ACI leaf switches handling the L3Out peering. This provides resilience against `ingress node` or leaf failures and allows for hitless maintenance or upgrades.
* **Dedicated vs. Shared:** Depending on cluster scale, ingress traffic volume, and performance requirements, dedicating specific nodes solely for the ingress function can offer advantages:
    * **Performance Predictability:** If regular application Pods are not scheduled onto `ingress nodes`, ingress traffic typically traverses a predictable path (Client -> ACI -> Ingress Node -> Worker Node), potentially leading to more consistent latency.
    * **Resource Isolation:** Dedicated nodes ensure that their CPU, memory, and network bandwidth are fully available for handling ingress traffic and BGP processing, without contention from application workloads.
    * **Reduced Peering Scale:** Since only a subset of nodes establishes BGP peering with the fabric, the overall BGP management overhead on both ACI and Cilium is reduced compared to peering with all nodes.
    * **Specialized Hardware:** Not all nodes in a cluster need identical hardware. High-performance hardware (e.g., faster CPUs, high-throughput NICs) can be specifically utilized for the `ingress nodes`. For example, using [CiliumNodeConfig](https://docs.cilium.io/en/stable/configuration/per-node-config/#per-node-configuration), bare-metal nodes equipped with Mellanox or Intel NICs supporting features like [Cilium's Big TCP](https://docs.cilium.io/en/stable/operations/performance/tuning/#ipv4-big-tcp) can be deployed, potentially achieving very high throughput per ingress node.

Refer to the [Example configuration](../examples/examples/) section of this document for implementation details.

## Cilium Egress Design Considerations

For handling egress traffic in this Basic Design, Cilium's Egress Gateway feature utilizes IP addresses sourced from the secondary subnets configured on the primary ACI BD. These Egress IPs are then classified using ACI ESGs for policy enforcement.

**Deployment Recommendations:**

The primary consideration is the number of nodes designated to handle egress traffic (`egress nodes`) and whether these should be dedicated:

* **Redundancy:** Deploy a minimum of two `egress nodes`, physically connected across different ACI leaf switches or pairs for redundancy against node/leaf failures and during maintenance.
* **Dedicated vs. Shared:** Similar to ingress, dedicating nodes for egress can be beneficial depending on scale and requirements, offering resource isolation and potentially specialized hardware advantages. The rationale mirrors the points discussed for dedicated `ingress nodes`.

{: .note }
A single node designated as an `egress node` can be assigned multiple Egress IP addresses (drawn from the secondary BD subnets). This allows one physical node to represent multiple distinct egress identities, which can be mapped via Cilium Egress Gateway policies to different Kubernetes Namespaces or Pod selectors. For example, Egress IP-A could be used by Namespace A, while Egress IP-B on the same node serves Namespace B.

### Egress Gateway and ACI Integration
{: .no_toc }

In this Basic Design, the integration between Cilium Egress Gateway and ACI policy relies on ESGs applied within the primary cluster BD:

* **Egress IPs from Secondary Subnets:** Cilium Egress Gateway is configured to use IP addresses from the secondary subnet(s) defined on the main ACI BD. These become the fixed source IPs for designated egress traffic.
* **ESG Classification:** ACI ESGs are created with IP-based selectors matching these specific Egress IPs (or ranges).
* **Cilium Policy Mapping:** Standard Cilium Egress Gateway policies map Kubernetes Pods/Namespaces to specific Egress IPs.
* **ACI Contract Enforcement:** Contracts are applied between the `egress` ESGs (representing specific application egress identities) and external destinations (represented by other ESGs or ExtEPGs), allowing granular control within ACI.

This approach controls traffic leaving the cluster via the Egress Gateway mechanism, while internal cluster traffic remains unaffected by these specific ESG policies.

![Egress Gateway and ESGs](../images/egress.png)
*Egress Gateway Traffic Flow using ESGs*

{: .note}
While Cilium *can* advertise Egress Gateway IPs via BGP (as described in the Advanced Design Option 1), this Basic Design favors ESG classification. Using ESGs avoids consuming ExtEPG resources on the L3Out (which might be limited, e.g., ~250 per L3Out instance) for egress classification, preserving those ExtEPGs primarily for classifying external Kubernetes Services advertised via BGP.

## Design Trade-offs

This Basic Design aims to provide a relatively straightforward and scalable integration; however, it involves the following trade-offs:

1.  **`externalTrafficPolicy: Cluster` Likely Required:** Because incoming traffic for external services first lands on a potentially small subset of `ingress nodes` (which may not host the target backend Pod), using `externalTrafficPolicy: Cluster` is often necessary. This allows the `ingress node` (via kube-proxy or Cilium's service handling) to perform a second layer of load balancing to forward the traffic to a node actually running the backend Pod. This differs from the Advanced design where DSR can optimize the return path.
2.  **Potential Ingress/Egress Bottlenecks:** Concentrating all external service traffic (ingress) through the `ingress nodes` and potentially egress traffic through specific `egress nodes` can create bottlenecks if these node pools are not adequately scaled (horizontally or vertically) for the required traffic load.
3.  **No Pod IP Advertisement:** This design does not involve advertising Kubernetes Pod IP subnets into the ACI fabric via BGP. Direct reachability *to* Pod IPs from outside the cluster is generally not a goal here; access is primarily via Kubernetes Services (ingress) or controlled via Egress Gateway (egress).
4.  **Ingress Node Complexity:** The requirement for `ingress nodes` to have two interfaces (one in the main EPG/BD, one for the L3Out) and associated Policy-Based Routing adds configuration complexity specific to those nodes.

Trade-off (1) is inherent in designs separating ingress peering from the main node pool without advanced features like DSR fully mitigating the extra hop. Issue (2) is manageable with proper capacity planning and scaling of the ingress/egress node pools. Issue (3) is a fundamental aspect of the design, relying on Services and Egress Gateways for external interaction rather than direct Pod routing. Issue (4) introduces operational overhead for the designated ingress nodes.

[Next](/cilium-dc-design/docs/aci/advanced_design/){: .btn }
{: .text-right }
