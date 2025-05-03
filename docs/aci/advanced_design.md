---
title: Advanced Design
layout: default
parent: ACI Designs
nav_order: 2
---

# Advanced Design

The basic network infrastructure for this design is composed of the following core ACI components:

* **Tenant:** The Kubernetes cluster can reside within a dedicated tenant or a shared tenant, particularly in environments hosting multiple clusters.
* **Floating SVI L3Out:** A single L3Out utilizing the Floating SVI feature serves several purposes:
  * Provides the primary L3 gateway and L2 broadcast domain for node-to-node communication for most, if not all, Kubernetes nodes.
  * Facilitates traffic classification and security policy enforcement via ACI Contracts applied to dedicated External EPGs (ExtEPGs) for nodes and services.
  * Establishes BGP peering with all or a subset of Kubernetes nodes for advertising external Kubernetes Services (`/32` routes) and enabling load balancing.

* **Optional Bridge Domain (BD) for Dedicated Egress:** Depending on the chosen egress strategy (detailed later), a separate BD might be configured specifically for nodes handling egress traffic:
  * An EPG within this BD connects the secondary interfaces of designated egress nodes.
  * Multiple Endpoint Security Groups (ESGs) within this EPG can use IP selectors to classify traffic based on specific Kubernetes `Egress IPs`, allowing distinct policies for different Pods or Namespaces.

This design provides the following capabilities:

* Secures traffic entering and leaving the cluster using ACI contracts.
* Supports DHCP Relay: This enables Kubernetes nodes to be bootstrapped without requiring manual IP address configuration, simplifying cluster deployment and horizontal scaling.

  {: .warning }
  Be aware of the specific guidelines and limitations associated with DHCP Relay policies configured on L3Outs. Refer to the official Cisco documentation, such as: [DHCP Limitations](https://www.cisco.com/c/en/us/td/docs/dcn/aci/apic/6x/basic-configuration/cisco-apic-basic-configuration-guide-61x/provisioning-core-aci-fabric-services-61x.html#guidelines-and-limitations-for-a-dhcp-relay-policy).

* Supports heterogeneous node types: The cluster can comprise a mix of bare-metal servers and virtual machines running on various hypervisors, provided network connectivity is established.
* Offers routing simplicity: The default gateway for the nodes is typically the ACI Floating SVI IP address associated with the main L3Out.
* Eliminates the need to advertise the Kubernetes Pod CIDR block into ACI.
* Leverages BGP-based ECMP for external Kubernetes service load balancing, potentially enhanced with resilient hashing techniques (like Maglev, implemented by Cilium).
* Achieves near-optimal traffic flows for external services through [Direct Server Return (DSR)](/cilium-dc-design/docs/fabric_agnostic_features/#direct-server-return).

## Cluster L3Out Physical Connectivity

No strict requirement dictates the physical connectivity method for the nodes associated with the L3Out, as long as the desired level of redundancy is achieved. Many deployments will likely utilize vPC for connecting the hosts. Implementing L2 redundancy at the host connection level generally improves failover performance by reducing reliance solely on BGP convergence times.

## BGP Design
**Centralized BGP Peering for Service Advertisement**

In this model, all Kubernetes nodes participating in BGP peering establish sessions with a designated pair of ACI leaf switches, often referred to as "anchor" leaves. This peering occurs regardless of the nodes' physical connectivity points within the fabric. This approach simplifies both the physical network configuration and the Cilium BGP setup. As of ACI release 6.1(2) (current as of May 2, 2025), ACI [supports](https://www.cisco.com/c/en/us/td/cilium-dc-design/docs/dcn/aci/apic/6x/verified-scalability/cisco-aci-verified-scalability-guide-612.html) up to 2,000 BGP neighbors per leaf switch, a limit unlikely to pose a constraint for a single large Kubernetes cluster.
When multiple Kubernetes clusters are connected to the same ACI fabric, different pairs of anchor leaves can be designated for each cluster to distribute peering load and resources effectively.

![Centralized Routing](../images/centralized-routing.png)
*Centralized Routing Model*

### ECMP Considerations

* **ACI ECMP Path Limit:** By default, ACI installs up to 16 equal-cost paths for eBGP/iBGP routes. If more than 16 Kubernetes `nodes` are configured for BGP peering, ACI's maximum ECMP path limit can be increased (typically up to 64, check specific hardware/software limits).
* **`externalTrafficPolicy: Cluster` Requirement:** Features like Cilium's Maglev resilient hashing and Direct Server Return (DSR) necessitate setting the `externalTrafficPolicy` to `Cluster` on Kubernetes Services. This setting means every node peering with ACI via BGP advertises itself as a potential next hop for every externally exposed Service, regardless of whether it currently hosts a backend Pod for that Service.
* **Path Selection with > Max ECMP Nodes:** The ACI ECMP selection algorithm installs routes up to the configured maximum number of paths per destination prefix (i.e., per exposed Service `/32`). If more equal-cost paths are available than the configured limit (e.g., 20 nodes peering but max ECMP is 16), ACI uses a hash-based mechanism to select a stable subset of next hops from the available pool. While not all peering nodes might be simultaneously active in the forwarding path for a *single* service prefix on a *single* leaf, traffic distribution across the peering nodes remains generally fair system-wide. Furthermore, because DSR ensures the return traffic bypasses the initial ingress node, the potential impact of uneven ingress forwarding is mitigated.

{: .note }
This Advanced Design **requires ACI version 6.1(2) or later**. This prerequisite stems from the dependency on both the *Propagate Next-Hop* and *Ignore IGP Metric* BGP features, which are essential for optimal routing and path selection in this topology.

## Cilium Egress Design Options

Regarding the design for handling egress traffic originating from Kubernetes Pods, two primary options can be evaluated based on specific requirements and ACI scale considerations.

### Option 1: Egress IP Advertisement via BGP (Preferred)

Cilium can advertise the specific IP addresses used by its Egress Gateway feature directly via BGP. This is typically configured within the Cilium BGP policy resources by specifying the Egress Gateway IP pool for advertisement.
ACI External EPGs (ExtEPGs) can then be used to classify this egress traffic based on the advertised Egress IP prefixes, allowing contract-based policies to be applied.

If the number of unique Egress IPs leads to concerns about ACI ExtEPG scale limits, *and* if egress traffic inspection via a firewall is required anyway, an alternative approach involves using a single ExtEPG that matches the entire Egress Gateway IP subnet. Service Graph redirection can then be employed to forward all traffic matching this ExtEPG to the firewall for policy enforcement.

This BGP-based option maintains design simplicity, as all nodes (including those potentially acting as egress gateways) can remain topologically identical, connecting primarily through the main L3Out.

### Option 2: Egress IP Classification with ESGs

Alternatively, the capabilities of ACI Endpoint Security Groups (ESGs) can be harnessed to create a distinct structure for managing egress traffic:

* **Dedicated Egress Network Segment:** Nodes designated to perform egress functions are configured with a secondary network interface. This interface connects to a separate ACI Bridge Domain (BD) and EPG, distinct from the main L3Out used for node-to-node and service traffic.
* **ESG Classification within Egress EPG:** ESGs are defined within this dedicated egress EPG. IP-based selectors within these ESGs are used to classify traffic based on the specific, static `Egress IP` addresses assigned by the Cilium Egress Gateway feature.
* **Cilium Egress Gateway Policies:** Standard Cilium Egress Gateway policies are implemented within Kubernetes to associate specific Pods or Namespaces with designated egress nodes and their corresponding fixed egress IP addresses. This ensures predictable source IPs for outbound traffic from different application groups.
* **Policy Enforcement via ESGs:** By mapping specific Egress IPs to distinct ESGs, ACI contracts can be applied between these ESGs and external destinations (represented typically by ExtEPGs or other ESGs), enabling granular control over outbound traffic flows at a Namespace or application level directly within ACI policy.

It is important to emphasize that this design specifically targets traffic *leaving* the cluster via the designated egress interfaces. Internal cluster traffic (pod-to-pod) remains unaffected by these ESG configurations and continues to flow over the primary node network.

![Egress Gateway and ESGs](../images/egress.png)
*Egress Gateway Traffic Flow using ESGs*

#### Egress Node Requirements (for ESG Option)

When implementing the ESG-based egress design (Option 2), the designated Egress nodes require configuration with two network interfaces:

* **Primary Node Interface:** Connects to the main ACI L3Out (shared with other nodes). This interface handles standard node-to-node communication and potentially Kubernetes service traffic if the node also participates in BGP. Its IP address should be configured as the primary `node-ip` for kubelet.
    * It is not strictly necessary for nodes dedicated *solely* to egress traffic to establish BGP peering via this interface, although they can if also serving ingress traffic.
* **Dedicated Egress Interface:** Connects to the separate ACI BD and EPG designated for egress traffic, where ESGs are configured. This interface carries the actual Pod-initiated egress traffic sourced from the `Egress IP` addresses.

{: .note }
For nodes configured with multiple interfaces, it is fundamental to ensure that kubelet is configured to use the correct primary `node-ip` – in this design, that must be the IP address of the interface connected to the main ACI L3Out. Cilium relies on this primary node IP for its internal pod-to-pod east-west routing decisions and typically does not provide mechanisms to selectively route pod traffic over alternative interfaces based on destination.

##### Routing Considerations (for ESG Option)

A potential challenge with the dual-interface egress node setup (Option 2) is ensuring symmetric routing. Traffic initiated by Pods via the dedicated egress interface (using an `Egress IP` as the source) must receive replies back through the same egress interface. If return traffic mistakenly attempts to exit via the primary node interface (connected to the L3Out), it might be dropped due to RPF checks or firewall rules.

To enforce symmetric routing for egress flows, Linux Policy-Based Routing (PBR) is typically configured on the egress nodes:

* **Create a separate routing table:** For example, table ID `100`.
* **Add default route in the new table:** In table `100`, configure a default route pointing to the ACI gateway IP address residing on the dedicated *egress* BD/subnet.
* **Create IP routing rules:** Use `ip rule add` commands to direct traffic *sourced from* the specific `Egress IP` addresses assigned to this node to use routing table `100`.

This ensures that any traffic originating from the node's `Egress IP`(s) uses the dedicated egress path and its associated gateway, preserving routing symmetry for the duration of the connection.

**General Egress Node Deployment Considerations:**

Regardless of the chosen egress design (Option 1 or 2), careful consideration should be given to the number and placement of `egress nodes`:

* **Redundancy:** Deploy a minimum of two `egress nodes`, preferably connected to different ACI leaf switches (or leaf pairs) to provide resilience against node or leaf failures and facilitate hitless upgrades.
* **Dedicated vs. Shared:** Depending on cluster scale, egress traffic volume, and policy requirements, dedicating specific nodes solely for the egress function might be beneficial. This isolates egress workloads and simplifies resource management compared to having nodes perform ingress, egress, and regular workload functions simultaneously.

{: .note }
A single egress node can be configured with multiple `Egress IP` addresses. This allows one node to serve as the egress point for several distinct egress identities (e.g., different Namespaces or application tiers), improving resource utilization. For instance, Egress IP-A could be mapped via Cilium policy to Namespace A, while Egress IP-B on the same node is mapped to Namespace B.

## Design Trade-offs

This Advanced Design aims to provide an easily managed and highly scalable solution; however, it presents certain trade-offs compared to simpler alternatives:

1.  **`externalTrafficPolicy: Cluster` Requirement:** External Kubernetes Services must use `externalTrafficPolicy: Cluster` due to the reliance on Cilium's Maglev hashing and DSR. This means BGP advertises all peering nodes as potential next hops for every service. The potential downsides (like traffic hitting a node without a local backend Pod) are largely mitigated by DSR ensuring optimal return paths.
2.  **Potential Egress Bottlenecks:** Concentrating egress traffic through a potentially small set of dedicated egress nodes (especially in Option 2) could create performance bottlenecks if not sized appropriately.
3.  **Reduced Node IP Visibility in L3Out:** Nodes primarily interact with ACI via the Floating SVI L3Out. While ExtEPGs can classify nodes, direct visibility and policy based on individual node IPs within ACI might be less straightforward compared to the Basic Design where nodes might be in direct EPGs.
4.  **Increased Complexity (Egress Option 2):** If the ESG-based egress approach (Option 2) is chosen, the configuration becomes more complex due to the need for secondary interfaces, separate BD/EPG/ESGs, and Policy-Based Routing on the egress nodes.

Issue (1) is an inherent aspect of using Maglev/DSR for optimal load balancing and resilience. Issue (2) can generally be addressed through appropriate horizontal scaling (adding more egress nodes) or vertical scaling (using more powerful nodes). Issue (4) is specific to the chosen egress method.

[Next](/cilium-dc-design/docs/aci/aci_bgp_design/){: .btn }
{: .text-right }
