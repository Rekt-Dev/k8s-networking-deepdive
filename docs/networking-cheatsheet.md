# Kubernetes Networking Cheatsheet

Quick reference for networking diagnostics on this kubeadm cluster.

## Cluster Network Config

```
Pod CIDR:     10.244.0.0/16
Service CIDR: 10.96.0.0/12
DNS service:  10.96.0.10 (kube-dns)
CNI:          Calico v3.28.x (VXLAN)
Runtime:      containerd
```

## Pod Networking Diagnostics

```bash
# Check pod IP and node assignment
kubectl get pod <pod> -n <ns> -o wide

# Get all pod IPs in a namespace
kubectl get pods -n <ns> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Test pod-to-pod connectivity
kubectl exec -n <ns> <pod-a> -- curl -s http://<pod-b-ip>:<port>

# Test pod-to-service connectivity
kubectl exec -n <ns> <pod> -- curl -s http://<service-name>.<ns>.svc.cluster.local:<port>

# Debug from a tooling pod
kubectl run debug --image=nicolaka/netshoot --restart=Never --rm -it -- /bin/bash
```

## DNS Diagnostics

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Test DNS resolution from within a pod
kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec <pod> -- nslookup <service>.<namespace>.svc.cluster.local

# DNS lookup with dig
kubectl exec <pod> -- dig @10.96.0.10 kubernetes.default.svc.cluster.local

# Check /etc/resolv.conf in a pod
kubectl exec <pod> -- cat /etc/resolv.conf
# Expected: search <namespace>.svc.cluster.local svc.cluster.local cluster.local
```

## NetworkPolicy Diagnostics

```bash
# List all NetworkPolicies
kubectl get networkpolicies --all-namespaces

# Describe a policy
kubectl describe networkpolicy <name> -n <ns>

# Check Calico policy enforcement on a node (calicoctrl)
# On a node with calicoctl installed:
calicoctl get networkpolicy --all-namespaces

# View iptables rules created by Calico
iptables -L | grep cali
iptables -t nat -L | grep cali | head -30

# Count iptables rules (high count = many policies)
iptables -L | wc -l
```

## Calico Diagnostics

```bash
# Check Calico node status
kubectl get pods -n kube-system -l k8s-app=calico-node

# View Calico node logs (Felix)
kubectl logs -n kube-system -l k8s-app=calico-node -c calico-node --tail=30

# Check Calico IPAM pool
kubectl get ippool -o yaml

# View Felix config
kubectl get felixconfiguration default -o yaml

# Check Calico BGP (if enabled)
calicoctl node status

# Calico route info on a node
ip route | grep cali
ip route | grep tunl  # VXLAN tunnels
```

## Service Diagnostics

```bash
# Check service endpoints
kubectl get endpoints <service> -n <ns>
kubectl describe endpoints <service> -n <ns>

# Debug a service with no endpoints
kubectl get pods -n <ns> --show-labels  # Verify labels match service selector
kubectl describe service <service> -n <ns>  # Check selector

# Check kube-proxy rules (iptables mode)
iptables -t nat -L KUBE-SERVICES | grep <service-ip>

# Test service from outside cluster (NodePort)
curl http://192.168.1.110:30080
```

## Ingress Diagnostics

```bash
# Check ingress controller pods
kubectl get pods -n ingress-nginx

# View ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50

# List all ingress resources
kubectl get ingress --all-namespaces

# Check ingress controller's generated nginx config
kubectl exec -n ingress-nginx <controller-pod> -- nginx -T | grep -A10 "server_name <host>"

# Test ingress from inside cluster
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- \
  curl -H "Host: app.homelab.local" http://ingress-nginx-controller.ingress-nginx.svc.cluster.local
```

## VXLAN / Overlay Diagnostics

```bash
# Check VXLAN interfaces
ip link show | grep vxlan
ip link show vxlan.calico

# Check FDB (forwarding database) entries for VXLAN
bridge fdb show dev vxlan.calico

# Capture VXLAN traffic (port 4789)
tcpdump -i any -n 'udp port 4789' -c 20

# Decode VXLAN to see encapsulated pod traffic
tcpdump -i vxlan.calico -n -c 20
```

## Common Issues

**Pod gets IP but cannot reach other pods:**
1. Check NetworkPolicy — default-deny-all blocks everything
2. Check if CNI pods are running: `kubectl get pods -n kube-system | grep calico`
3. Check Felix logs for policy drop messages

**DNS not resolving:**
1. Check CoreDNS pods are running
2. Check `/etc/resolv.conf` in the pod — namespace matters
3. Check NetworkPolicy allows egress to kube-system on port 53

**Service has no endpoints:**
1. Pod labels must exactly match service selector
2. Pod must be in Ready state
3. Check pod readiness probe is passing

**Ingress not routing:**
1. Check ingress class annotation matches installed controller
2. Check backend service and port names/numbers are correct
3. Check ingress controller logs for 404/502 details
