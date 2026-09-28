# k8s-networking-deepdive

Deep-dive into Kubernetes networking on a physical kubeadm cluster. Covers Calico CNI configuration, advanced NetworkPolicy patterns, CoreDNS customization, and ingress. Includes honest notes from a Cilium migration attempt that killed the host due to RAM pressure, and why Calico was the right call for this environment.

## Cluster Context

- **Nodes:** k8s (control plane, 192.168.1.110), k8s1, k8s2
- **CNI:** Calico v3.28.x (VXLAN mode)
- **Pod CIDR:** 10.244.0.0/16
- **Service CIDR:** 10.96.0.0/12
- **Container runtime:** containerd
- **Kubernetes version:** v1.35.x

## Repository Structure

```
calico/
  ip-pool.yaml              IPPool configuration for this cluster's pod CIDR
  bgp-configuration.yaml    BGP config (reference — VXLAN is active, not BGP)
  felix-configuration.yaml  Per-node agent tuning
network-policies/
  advanced-egress.yaml      CIDR-based egress, metadata endpoint block
  multi-tier-app.yaml       Frontend/backend/database tier isolation
dns/
  custom-coredns.yaml       CoreDNS with homelab.local forward zone
  dns-debug.yaml            Tooling pod for DNS troubleshooting
ingress/
  nginx-values.yaml         ingress-nginx Helm values (NodePort, homelab-tuned)
  tls-example.yaml          Ingress with TLS and security headers
docs/
  cilium-migration-notes.md What happened when we tried to migrate to Cilium
  cni-comparison.md         Calico vs Cilium vs Flannel — actual tradeoffs
  networking-cheatsheet.md  kubectl commands, iptables, CNI debugging reference
```

## Why This Repository Exists

Hands-on networking work on a physical kubeadm cluster: CNI, ServiceAccount DNS, NetworkPolicy enforcement, ingress and runtime behavior — running and breaking things, then debugging them.

The most educational event was the Cilium migration attempt. Everything I understood about eBPF and Cilium was correct — the architecture, the features, the performance characteristics. What I didn't model correctly was the memory footprint at steady state with Hubble enabled. That miscalculation killed the host. Reading the docs in detail post-incident: the memory requirements are documented, I just didn't treat them as a constraint. On a homelab with limited RAM, resource planning for DaemonSets matters more than on cloud nodes.

See `docs/cilium-migration-notes.md` for the full account.

## Calico Configuration

This cluster uses Calico in **VXLAN mode**. Each node has a `vxlan.calico` interface that encapsulates pod-to-pod traffic across nodes. The alternative (BGP mode) would be more efficient — no encapsulation overhead — but requires either router BGP peering or direct L2 between all nodes. Since all nodes are on the same switch, BGP would work, but the performance difference at 1Gbps is immaterial.

The IPPool in `calico/ip-pool.yaml` matches the `--pod-network-cidr` passed to `kubeadm init`. Changing this after cluster creation is non-trivial.

Felix (the per-node Calico agent) configuration in `calico/felix-configuration.yaml` enables Prometheus metrics scraping from each node on port 9091.

## NetworkPolicy Patterns

The `network-policies/` directory demonstrates patterns beyond the basics covered in `k8s-security-hardening`:

**Multi-tier isolation** (`multi-tier-app.yaml`): Database pods accept traffic only from backend pods, which accept traffic only from frontend pods. Each tier has explicit DNS egress. This is the most common real-world NetworkPolicy pattern.

**CIDR egress** (`advanced-egress.yaml`): Policies that allow egress to specific homelab IP ranges rather than cluster-internal services. Necessary for pods that need to reach NFS storage, external APIs, or other infrastructure on the same network.

**Metadata endpoint block**: Blocking 169.254.169.254 is a cloud-native security pattern (prevents SSRF → IMDS credential theft). Including it here as reference for when this knowledge is needed on AWS/GCP workloads.

## CoreDNS Customization

The custom Corefile in `dns/custom-coredns.yaml` adds a forward zone for `homelab.local` pointing at the local DNS server (192.168.1.1). This allows pods to resolve NAS hostnames, printer hostnames, and other local infrastructure without hardcoding IPs.

After applying:

```bash
kubectl apply -f dns/custom-coredns.yaml
kubectl rollout restart deployment/coredns -n kube-system

# Verify
kubectl run test --image=busybox --rm -it --restart=Never -- nslookup nas.homelab.local
```

## Ingress

ingress-nginx is deployed via Helm with NodePort 30080/30443. No LoadBalancer service type is available on a bare metal homelab — external traffic hits `192.168.1.110:30443` and gets proxied to the correct backend based on the Host header.

For services that need real external access (not just homelab LAN), the router (192.168.1.1) forwards port 443 to 192.168.1.110:30443. TLS terminates at the nginx controller.

## Lessons Learned

**VXLAN encapsulation is invisible to iptables.** When debugging network connectivity issues between pods on different nodes, remember that iptables rules see the inner pod IPs, not the VXLAN outer IPs. `tcpdump -i vxlan.calico` is what you want, not `tcpdump -i eth0` for pod traffic.

**NetworkPolicy evaluation order doesn't work the way you think.** There is no "first match wins" — all matching policies are evaluated and the result is the union. Two policies that each partially allow different things combine into a broader allow. Design policies to be self-contained per tier/workload, not to be read as a sequence.

**CoreDNS ndots matters.** By default, Kubernetes sets `ndots: 5` in pod resolv.conf. This means `curl api.example.com` makes 5 DNS queries before resolving externally — `api.example.com.production.svc.cluster.local`, `api.example.com.svc.cluster.local`, `api.example.com.cluster.local`, `api.example.com.homelab.local`, then `api.example.com`. On latency-sensitive services, reduce ndots or use fully-qualified domain names in service calls.

**kube-proxy iptables rules are fragile at scale.** On this 3-node cluster with ~50 services, the iptables KUBE-SERVICES chain has ~200 rules and evaluation time is negligible. On clusters with thousands of services, iptables becomes a bottleneck. This is one of the genuine reasons to use Cilium or IPVS-mode kube-proxy in production.

## Related Repositories

- [k8s-production-patterns](https://github.com/Rekt-Dev/k8s-production-patterns) — the cluster where all of this runs
- [k8s-security-hardening](https://github.com/Rekt-Dev/k8s-security-hardening) — security-focused NetworkPolicies that build on these patterns
- [k8s-observability-stack](https://github.com/Rekt-Dev/k8s-observability-stack) — Prometheus scrapes Calico Felix metrics from this config
