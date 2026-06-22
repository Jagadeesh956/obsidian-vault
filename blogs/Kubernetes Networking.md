---
title: "How Kubernetes Networking Works"
date: 2026-06-21
tags: [Kubernetes, Networking, DevOps, SRE]
summary: "A high-level walkthrough of K8s networking — from components and ingress routing to egress ACLs and service mesh."
---

## Overview

Kubernetes networking is one of the most misunderstood parts of the platform. Unlike traditional VMs, every Pod gets its own IP, and the network model is built around a flat, routable address space. This post walks through how traffic actually flows — in, out, and between services.

---

## 1. Components Involved in Kubernetes Networking

| Component | Role |
|---|---|
| **Pod Network (CNI)** | Assigns IPs to Pods and enables Pod-to-Pod communication |
| **kube-proxy** | Programs iptables/IPVS rules on each node to route Service traffic |
| **CoreDNS** | Resolves Service names to ClusterIPs inside the cluster |
| **Service** | Stable virtual IP (ClusterIP) that load-balances to matching Pods |
| **Ingress Controller** | Reverse proxy that routes external HTTP/S traffic to Services |
| **Cloud Load Balancer** | Entry point from the internet; created when a Service type is `LoadBalancer` |
| **Network Policy** | Firewall rules controlling Pod-to-Pod and Pod-to-external traffic |

The **CNI plugin** (Calico, Flannel, Cilium, etc.) is the foundation — it gives every Pod a routable IP and ensures Pods on different nodes can reach each other without NAT.

---

## 2. Ingress — How External Traffic Reaches a Pod

### The path of an inbound request

```
Internet
  └─▶ Cloud Load Balancer  (L4 — TCP/UDP)
        └─▶ Ingress Controller Pod  (nginx / Traefik / Envoy)
              └─▶ Evaluates Ingress rules  (host + path matching)
                    └─▶ Service  (ClusterIP)
                          └─▶ kube-proxy  (iptables / IPVS)
                                └─▶ Pod
```

### What each component does

**Cloud Load Balancer**
Created automatically when you apply a `Service` of type `LoadBalancer`. It is a Layer 4 (TCP) load balancer — it forwards raw packets to a node's NodePort without inspecting HTTP headers.

**Ingress Controller**
A Pod running a reverse proxy (nginx, Traefik, Envoy, etc.) that watches Kubernetes `Ingress` resources via the API server. When you define an Ingress rule:

```yaml
rules:
  - host: app.example.com
    http:
      paths:
        - path: /api
          backend:
            service:
              name: api-service
              port:
                number: 80
```

…the controller dynamically updates its proxy config to route `app.example.com/api` → `api-service:80`. No restart needed — it watches the API server for changes in real time.

**kube-proxy**
Runs on every node as a DaemonSet. It watches the API server for `Service` and `Endpoints` objects and programs iptables (or IPVS) rules so that traffic destined for a ClusterIP gets DNAT'd to one of the backing Pod IPs. This is how a Service's stable virtual IP maps to real, ephemeral Pod IPs.

**CoreDNS**
When a Pod calls `http://api-service`, CoreDNS resolves `api-service.default.svc.cluster.local` to its ClusterIP. kube-proxy then takes over from there.

---

## 3. Egress — How Traffic Leaves the Cluster

### The path of an outbound request from a Pod

```
Pod  (10.0.1.5)
  └─▶ Node kernel  (iptables MASQUERADE / SNAT)
        └─▶ Node's primary NIC  (node IP)
              └─▶ Cloud VPC routing
                    └─▶ Internet / external service
```

### How cloud providers apply ACL rules

**SNAT / Masquerade**
By default, outbound Pod traffic is Source NAT'd to the node's IP at the iptables level before leaving the node. The destination sees the node IP, not the Pod IP.

**Cloud Security Groups / Firewall Rules**
Cloud providers attach security rules to the **node's network interface**. Because egress traffic appears to come from the node IP after SNAT, rules are evaluated at the node level:

- **AWS** — Security Group egress rules on the EC2 instance
- **GCP** — VPC Firewall egress rules matched against the instance's network tag
- **Azure** — Network Security Group (NSG) rules on the VM's NIC

**Kubernetes Network Policy** (enforced by the CNI plugin) adds a second layer — it can drop egress traffic from a Pod *before* it even reaches the node's NIC, using eBPF (Cilium) or iptables (Calico).

**NAT Gateway / Cloud NAT**
For Pods that need a stable outbound IP (e.g. to whitelist in a third-party firewall), traffic is routed through a managed NAT Gateway. This gives all egress traffic a predictable external IP regardless of which node the Pod is scheduled on.

---

## 4. Service Mesh — What It Adds

A service mesh (Istio, Linkerd, Cilium Service Mesh) injects a **sidecar proxy** (Envoy or similar) into every Pod. All inbound and outbound traffic passes through this proxy transparently — the application code is completely unaware of it.

### How the sidecar intercepts traffic

```
Inbound request
  └─▶ iptables rules (injected by mesh) redirect port → sidecar
        └─▶ Envoy sidecar  (mTLS termination, policy check, tracing)
              └─▶ Application container  (localhost)

Outbound request
  └─▶ iptables redirect → sidecar
        └─▶ Envoy sidecar  (load balancing, retries, circuit breaker)
              └─▶ Destination Pod's sidecar  (mTLS handshake)
                    └─▶ Destination application
```

### What a service mesh provides that plain Kubernetes does not

| Capability | Plain K8s | Service Mesh |
|---|---|---|
| mTLS between services | ✗ manual | ✓ automatic |
| Traffic splitting (canary, A/B) | Limited | ✓ fine-grained |
| Retries / timeouts / circuit breaking | Must code in app | ✓ proxy-level |
| Distributed tracing | Manual instrumentation | ✓ automatic |
| Observability (golden signals per service) | Manual | ✓ built-in |
| L7 authorization policies | Network Policy only (L3/L4) | ✓ per-route, per-method |

### When to use a service mesh

A service mesh adds operational overhead — more moving parts and higher resource usage. It pays off when you need:

- **Zero-trust security** between microservices (mTLS everywhere, no implicit trust)
- **Progressive delivery** — canary releases, traffic shifting without redeployment
- **Deep observability** without touching application code
- **Multi-cluster or hybrid-cloud** routing and failover

For simpler setups, Kubernetes Network Policies combined with a capable CNI plugin like Cilium cover most security needs without the full mesh complexity.
