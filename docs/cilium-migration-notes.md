# Cilium Migration Attempt — Notes and Post-Mortem

## Motivation

Calico in VXLAN mode does the job, but it's iptables-based. Cilium uses eBPF to implement networking directly in the kernel without iptables chains. The two main benefits that made this worth testing:

1. **Hubble** — Cilium's built-in observability layer. Real-time network flow visibility without any additional tooling. You can see exactly which pod communicated with which service, at what rate, with latency metrics. This is significantly better than what you get with Calico's basic logging.

2. **eBPF NetworkPolicy** — Cilium's eBPF dataplane can enforce policies at the socket level, before packets even hit the network stack. Lower overhead than iptables on clusters with many policies.

## What Happened

Installed Cilium using the official Helm chart with kube-proxy replacement disabled (Cilium replaces kube-proxy as well as the CNI). The install itself went smoothly. Hubble was enabled and the flow observability was immediately impressive — exactly what I wanted.

The problem showed up about 20 minutes after install. The k8s node (control plane, which also runs workloads on this homelab) started showing memory pressure. Then k8s1 did the same. By the time I noticed, the kubelet on k8s was being OOMKilled.

**Root cause:** Cilium's eBPF maps and the Hubble ring buffer consumed significantly more RAM than Calico. On this cluster, nodes have 16GB RAM and run both cluster workloads and the system. Calico uses approximately 100-150MB at steady state. Cilium with Hubble enabled was using closer to 800MB-1.2GB per node just for the CNI components.

With Prometheus (monitoring-stack), ArgoCD, and production workloads already resident in memory, there was simply not enough headroom. The kernel's OOM killer started evicting kubelet before I could reduce Cilium's resource limits.

## Recovery

Recovery required direct node access because kubelet was down:

```bash
# On the control plane node (192.168.1.110)
# Remove Cilium manifests to stop the OOM cycle
helm uninstall cilium -n kube-system

# Cilium replaces kube-proxy, so restore it
kubectl apply -f /etc/kubernetes/manifests/ # re-apply kube-proxy DaemonSet

# Reinstall Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

The cluster was down for approximately 25 minutes.

## Lessons Learned

**Resource planning before migrating CNI.** CNI components run on every node as DaemonSets. A CNI that uses 2x the memory means every node needs 2x the memory budget for networking. On a homelab with tight RAM, this matters.

**Cilium's kube-proxy replacement is all-or-nothing.** You can run Cilium alongside kube-proxy (without the kube-proxy replacement), which uses less memory. The full replacement (which I enabled) also takes over load balancing from iptables entirely, which adds to the eBPF map memory footprint.

**Hubble is expensive relative to its benefit on a small cluster.** The ring buffer alone consumes significant memory (configurable, but defaults are large). On a production cluster where the observability ROI is high, this is worth it. On a 3-node homelab, the Calico + Prometheus combination provides sufficient visibility.

## Would I Use Cilium in Production?

Yes — on cloud instances with adequate RAM (m5.xlarge and above), Cilium is the better choice. The eBPF dataplane, Hubble observability, and L7 policy enforcement (HTTP-level NetworkPolicies) are genuinely useful features that Calico doesn't have. The memory overhead that killed this homelab is negligible on a cloud node with 16GB+ allocated to Kubernetes.

## Current Setup

Calico v3.28.x, VXLAN mode, iptables backend. No Hubble. Network flow visibility comes from Prometheus metrics (CNI stats, kube-state-metrics) and Falco for security events. Good enough for a homelab.
