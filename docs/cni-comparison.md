# CNI Comparison: Calico vs Cilium vs Flannel

Based on hands-on experience with all three on a 3-node kubeadm homelab cluster.

## Flannel

**Tried it during initial cluster setup (2024).**

Simple VXLAN overlay. Gets pods communicating with minimal configuration. No NetworkPolicy support — any CNI plugin that implements NetworkPolicy (like Calico) needs to be layered on top.

For a cluster that needs NetworkPolicy (which you should if you're doing anything security-related), Flannel is not a complete solution. The common recommendation of "Flannel + Calico network policy only" is unusual and better served by just using Calico.

**When to use:** Lab environments where you want the simplest possible networking and don't need NetworkPolicy.

## Calico

**Current CNI on this cluster.**

Calico supports three data planes:
- **iptables** (this cluster): Translates NetworkPolicy to iptables chains. Mature, well-understood, slightly more CPU overhead on clusters with many policies.
- **eBPF**: Calico's eBPF mode, lighter than Cilium's but without Hubble.
- **VPP** (experimental): High-performance userspace networking, not applicable here.

Two routing modes:
- **VXLAN** (this cluster): Encapsulates pod traffic. Works without any BGP setup. ~50 bytes per packet overhead.
- **BGP**: Advertises pod routes directly via BGP. No encapsulation. Requires network support or direct L2 connectivity.

For a homelab on a single switch, VXLAN is simpler and the performance difference is immaterial.

**Memory footprint:** ~100-150MB per node at steady state.

**NetworkPolicy:** Full Kubernetes NetworkPolicy spec plus Calico-specific GlobalNetworkPolicy CRD for cluster-wide policies.

## Cilium

**Tested and reverted — see `cilium-migration-notes.md` for the full story.**

Cilium uses eBPF for everything: routing, load balancing, NetworkPolicy enforcement, and observability (Hubble). This gives it capabilities that Calico doesn't have:

- **L7 NetworkPolicy**: Policies based on HTTP methods, paths, headers — not just IP/port.
- **Hubble**: Real-time network flow observability. Think of it as Wireshark for your cluster, but aggregated and queryable.
- **kube-proxy replacement**: Cilium can replace kube-proxy entirely, removing another iptables dependency.

The tradeoffs:
- Higher memory footprint (~800MB-1.2GB per node with Hubble enabled)
- More complex to operate (CiliumNetworkPolicy vs standard NetworkPolicy — migration path if you ever want to switch CNIs)
- eBPF requires kernel 4.9.17+ (all modern Linux distributions qualify)

**When to use Cilium:** Cloud clusters with adequate memory, teams that need L7 policy enforcement, or where Hubble's observability is worth the overhead.

## Summary

| Feature | Flannel | Calico (iptables) | Cilium |
|---------|---------|-------------------|--------|
| NetworkPolicy | No | Yes | Yes + L7 |
| Routing mode | VXLAN | VXLAN or BGP | eBPF or VXLAN |
| Memory (per node) | ~50MB | ~150MB | ~800MB+ |
| Observability | None | Basic metrics | Hubble (excellent) |
| kube-proxy replacement | No | No | Yes |
| Complexity | Low | Medium | High |
| Homelab suitability | Low | High | Medium (RAM dependent) |

For this cluster, Calico is the right choice. The security hardening work (NetworkPolicy, PSA, audit logging) fully utilizes Calico's capabilities. Cilium's additional features would be useful if the cluster had more RAM or was running on cloud instances.
