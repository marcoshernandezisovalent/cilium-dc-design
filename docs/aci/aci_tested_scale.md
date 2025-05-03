---
title: ACI Scale Testing
layout: default
parent: ACI Designs
nav_order: 4
---

# Scale Testing - Work In Progress

The [Advanced Design](../../advanced_design) detailed within this document has undergone scalability testing aimed at verifying its effectiveness and reliability across diverse conditions. However, it is crucial to recognize that these tests were performed based on generalized scenarios and assumptions. Given that each organization possesses unique requirements and architectural nuances, conducting independent testing is strongly recommended to validate the design's performance and suitability within the specific deployment environment.

Customized testing facilitates the identification of potential issues stemming from unique architectural components or specific use cases relevant to the organization. This validation process helps ensure that the proposed solution meets performance expectations and integrates seamlessly with existing systems.

The Advanced design has currently been tested under the following conditions:

- A 350-node Kubernetes Cluster.
- Each node advertising 250 BGP `/32` Service Routes.
- All nodes peering with 2 ACI Border Leaves.
- An ACI Fabric composed of 4 leaf switches.
- Clients accessing services via:
  - An L3Out.
  - An EPG/ESG.

## Generic ACI Scale Considerations

Within the context of the Cilium and Cisco ACI integration design, the following ACI scale metrics should be considered:

{: .note }
These metrics correspond to ACI version 6.1(2). If a different ACI version is deployed, consult the official [Verified Scalability Guide](https://www.cisco.com/c/en/us/support/cloud-systems-management/application-policy-infrastructure-controller-apic/tsd-products-support-series-home.html) for that specific version, as limits may vary. The current date is May 2, 2025.

- **Floating L3Out:** Maximum of 6 anchor leaves and 32 non-anchor leaves.
- **IPs per MAC:** 4096.
- **BFD Neighbors:** 2,000 sessions per leaf using minimum BFD timers: minTx: 300ms, minRx: 300ms, multiplier: 3.
- **BGP Neighbors:** Up to 2,000 per leaf. Fabric-wide scale depends on prefix count and path diversity (e.g., ~70,000 external prefixes with a single path per neighbor might reduce the total neighbor count below the theoretical 20k fabric maximum).
- **Shared L3Out (Inter-VRF Leaking):** 2,000 IPv4 prefixes per L3Out instance used for leaking.
- **External EPGs per L3Out:** 250 per L3Out instance, with a fabric-wide limit of 600 ExtEPGs across all L3Outs.
- **ESGs per Fabric:** 10,000.
- **ESGs per VRF:** 4,000.
- **ESGs per Tenant:** 4,000.
- **L3 IP Selectors per Leaf:** 5,000.
- **IP Longest Prefix Match (LPM) Entries:** Approximately 20,000 IPv4 entries with the default dual-stack hardware profile. Note that altering the hardware profile can impact other scale limits, including the number of supported ECMP paths.

## Conducted Tests:

### Adding/Removing BGP Peers

**Test Scenario:**
A Kubernetes node's participation in BGP peering was toggled by removing and subsequently reapplying the specific Kubernetes label that enables Cilium BGP on that node.

**Observed Impact:**
No discernible traffic impact was observed. This behavior is expected due to the nature of Kubernetes service routing (often using mechanisms like Maglev hashing for backend selection). Even if a node is temporarily removed from BGP peering but continues to receive traffic destined for a service IP, Kube-proxy or Cilium's eBPF datapath on that node can typically still forward the traffic directly to an appropriate backend Pod running locally or tunnel/route it to another node hosting a valid backend Pod.

### Reloading a Kubernetes Node

**Test Scenario:**
A Kubernetes node participating in BGP peering was gracefully reloaded.

**Observed Impact:**
Minimal traffic disruption was observed. Potential packet drops can occur during the BGP routing table reconvergence period – specifically, if traffic is forwarded to the reloading node *after* it has started shutting down but *before* ACI (or other peers) have removed it as a valid next hop via BFD/BGP updates.
This potential impact can be further minimized by preemptively removing the node from the BGP peering configuration (e.g., by removing the relevant label) before initiating the reload, allowing routes to withdraw gracefully.

### Cilium Agent Upgrade/Restart

The BFD and BGP processes related to external peering typically run within the Cilium agent pod on each Kubernetes node. Restarting the Cilium agent pod (e.g., during a Cilium upgrade or for other maintenance reasons) will cause the BFD session with ACI to drop. However, due to the BGP Graceful Restart capability configured on both ACI and Cilium, the overall impact is minimized.

**Test Scenario:**
The Cilium agent pod on a node participating in BGP peering was restarted.

**Observed Impact:**
Minimal to none. Although the BFD session drops, the ACI leaf maintains the BGP routes learned from that node due to BGP Graceful Restart being triggered. Forwarding continues using the existing routes while the Cilium agent restarts and re-establishes the BGP/BFD sessions.

### Reloading an ACI Anchor Node

**Test Scenario:**
An ACI anchor leaf switch, participating in BGP peering with Kubernetes nodes, was reloaded non-gracefully (simulating a crash or power loss, where BFD "down" messages might not be sent reliably before failure).

**Observed Impact:**
Minimal impact, primarily limited to potential loss of in-flight packets traversing the specific anchor leaf at the moment of failure.
The overall routing stability for Kubernetes services remained high. Because the design utilizes Next-Hop Propagation (when running ACI 6.1(2)+ and configured appropriately), the ultimate next-hop IP address for the `/32` service routes installed on other ACI leaves is the IP address of the originating Kubernetes node, not the anchor leaf itself. Therefore, the failure of a single anchor leaf does not invalidate the primary routes used by other leaves to reach the services via the Kubernetes nodes. Traffic simply avoids the failed anchor and utilizes paths through remaining anchor/non-anchor leaves connected to the Kubernetes nodes.

[Next](/cilium-dc-design/docs/aci/examples/examples/){: .btn }
{: .text-right }
