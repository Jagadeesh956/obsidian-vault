# Linux Networking Fundamentals and Kubernetes Networking Internals

> A ground-up explanation of how Linux networking primitives work, and how Kubernetes (via kubelet + CNI plugins) composes them into pod networking — including the actual files and system state that get touched at each stage of a cluster's life.

## Table of Contents
- [[#Part 1 — Linux Networking Fundamentals]]
  - [[#1.1 Network Namespaces]]
  - [[#1.2 veth Pairs]]
  - [[#1.3 Linux Bridges]]
  - [[#1.4 Routing Tables and iproute2]]
  - [[#1.5 iptables and netfilter]]
  - [[#1.6 IPVS]]
  - [[#1.7 IP Forwarding and sysctl]]
  - [[#1.8 VXLAN and Overlay Networks]]
  - [[#1.9 DNS Resolution Basics]]
- [[#Part 2 — The Kubernetes Networking Model]]
  - [[#2.1 The Four Kubernetes Networking Rules]]
  - [[#2.2 The Pause Container and Pod Network Namespace]]
  - [[#2.3 The CNI Spec]]
  - [[#2.4 Services and kube-proxy]]
  - [[#2.5 CoreDNS]]
  - [[#2.6 NetworkPolicy]]
- [[#Part 3 — File-Level Changes at Each Cluster Stage]]
  - [[#3.1 Stage — Cluster Bootstrap]]
  - [[#3.2 Stage — Node Join]]
  - [[#3.3 Stage — Pod Creation (the full sequence)]]
  - [[#3.4 Stage — Service Creation and kube-proxy Sync]]
  - [[#3.5 Stage — Pod Deletion]]
  - [[#3.6 Stage — Node Drain / Leave / Maintenance]]
- [[#Part 4 — End-to-End Worked Example]]
- [[#Appendix — Useful Inspection Commands]]

---

## Part 1 — Linux Networking Fundamentals

### 1.1 Network Namespaces

A [[Network Namespace]] (`netns`) is a Linux kernel feature (via `CLONE_NEWNET`) that gives a process its own isolated copy of the network stack: its own interfaces, routing table, ARP table, iptables rules, and `/proc/net` state. This is the single most important primitive for containers — a container is "just" a process that runs inside its own namespaces.

Key facts:
- Namespaces live at `/proc/<pid>/ns/net` (a symlink you can bind-mount elsewhere).
- Named namespaces created with `ip netns add <name>` are bind-mounted at `/var/run/netns/<name>`, which is how `ip netns exec` finds them.
- By default a fresh netns has only a `lo` (loopback) interface, and it's down.
- Namespaces persist as long as either a process is inside them or something bind-mounts the netns file — this is exactly how a container's network survives across `docker exec` sessions.

```bash
ip netns add demo
ip netns exec demo ip addr        # only lo, DOWN
ip netns exec demo ip link set lo up
```

See also: [[#1.2 veth Pairs]] for how you actually get connectivity into a namespace.

### 1.2 veth Pairs

A [[veth pair]] (virtual ethernet pair) is two connected virtual NICs that act like a patch cable — anything sent into one end appears on the other. This is how a namespace (which starts totally isolated) gets connected to the outside world.

Typical container pattern:
1. Create a veth pair on the host: `veth0` ↔ `veth1`.
2. Move one end (`veth1`) into the container's netns, usually renamed to `eth0` inside.
3. Leave the other end (`veth0`) on the host, and attach it to a [[#1.3 Linux Bridges|bridge]] or otherwise route to it.

```bash
ip link add veth0 type veth peer name veth1
ip link set veth1 netns demo
ip netns exec demo ip link set veth1 name eth0
ip netns exec demo ip addr add 10.244.1.5/24 dev eth0
ip netns exec demo ip link set eth0 up
ip link set veth0 up
```

This exact sequence — pair creation, namespace move, rename, address assignment — is what every CNI plugin does programmatically for every pod.

### 1.3 Linux Bridges

A [[Linux Bridge]] is a kernel-level software switch (`brctl` historically, now `ip link add type bridge`). Interfaces attached to a bridge behave like ports on a physical L2 switch — frames are forwarded based on learned MAC addresses.

```bash
ip link add cni0 type bridge
ip link set cni0 up
ip link set veth0 master cni0     # attach host-side veth to the bridge
```

This is the classic single-host "bridge model": every pod's host-side veth is plugged into the same bridge (commonly named `cni0` or `cbr0`), giving all pods on that node L2 connectivity to each other, with the bridge itself holding the node's pod-subnet gateway IP.

### 1.4 Routing Tables and iproute2

The kernel forwards packets by consulting [[Routing Table|routing tables]] (`ip route`). Linux supports multiple routing tables (main, local, custom tables 0–252), selected via [[Policy Routing|policy routing rules]] (`ip rule`).

```bash
ip route show                 # main table
ip route add 10.244.2.0/24 via 192.168.1.12 dev eth0   # reach another node's pod subnet
```

In Kubernetes this matters enormously for **routed** CNI plugins (e.g. Calico in BGP mode): each node gets a route for every other node's pod CIDR, pointing at that node's IP. No overlay encapsulation needed — it's pure L3 routing, which is why it's fast, but it requires the underlying network to allow it (or a BGP daemon to distribute the routes, as Calico's `bird`/`calico-node` does).

### 1.5 iptables and netfilter

[[netfilter]] is the in-kernel packet-filtering framework; [[iptables]] (and its successor `nft`/nftables) is the userspace tool that programs it. Packets pass through **chains** (PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING) across **tables** (filter, nat, mangle).

Concepts essential to Kubernetes:
- **DNAT** (Destination NAT) — rewrites a packet's destination IP:port. This is how a ClusterIP gets translated to an actual pod IP:port.
- **SNAT/MASQUERADE** — rewrites the source IP, typically to the host's IP, so return traffic routes back correctly (used for pod → external traffic, and often for pod → Service traffic from a different node).
- **Custom chains** — iptables lets you create your own named chains and jump to them; kube-proxy builds an entire tree of custom chains (`KUBE-SERVICES`, `KUBE-SVC-*`, `KUBE-SEP-*`) rather than dumping thousands of rules into `FORWARD` directly.

```bash
iptables -t nat -L -n -v | head -30
iptables -t nat -S | grep KUBE-SERVICES
```

### 1.6 IPVS

[[IPVS]] (IP Virtual Server) is an in-kernel L4 load balancer (part of the same netfilter/LVS framework) that predates Kubernetes but is used by kube-proxy as an alternative to iptables mode. Rather than one iptables rule per pod endpoint, IPVS keeps endpoints in an efficient hash-table-backed structure, giving O(1) lookup instead of iptables' linear chain traversal — this matters a lot at 10,000+ Services.

```bash
ipvsadm -Ln    # list virtual servers and real servers, analogous to KUBE-SVC / KUBE-SEP
```

### 1.7 IP Forwarding and sysctl

By default a Linux host does **not** forward packets between interfaces — each interface only handles traffic addressed to itself. To act as a router (which every Kubernetes node must, to shuttle pod traffic), you need:

```bash
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.bridge.bridge-nf-call-iptables=1   # bridged traffic also traverses iptables
```

These are written persistently to files under `/etc/sysctl.d/` (e.g. `/etc/sysctl.d/99-kubernetes-cri.conf`, or a CNI-specific file like `/etc/sysctl.d/10-bridge-nf-call-iptables.conf`) — this is one of the first file-level changes any container runtime / CNI setup makes on a fresh node, because without it, bridged pod traffic silently bypasses your iptables/NetworkPolicy rules entirely.

### 1.8 VXLAN and Overlay Networks

[[VXLAN]] (Virtual Extensible LAN) encapsulates L2 Ethernet frames inside UDP packets (port 4789 by default), letting you build a virtual L2/L3 overlay network on top of an underlying L3-only network — critical when the physical network doesn't know about pod subnets.

Flannel's default `vxlan` backend, and Calico's `vxlan` mode, both create a `flannel.1` / `vxlan.calico` interface on each node. A packet destined for a pod on another node is encapsulated (original packet becomes the VXLAN payload), sent host-to-host over the real network, and decapsulated on arrival.

Overlay (VXLAN/IPIP) vs routed (BGP) is one of the fundamental CNI design trade-offs: overlays work anywhere but add encapsulation overhead and MTU complications; routed modes are faster but need L3 reachability or BGP peering with the underlying fabric.

### 1.9 DNS Resolution Basics

Standard Linux name resolution reads `/etc/resolv.conf` (nameservers, search domains) and `/etc/nsswitch.conf` (resolution order). Kubernetes overrides `/etc/resolv.conf` **inside every pod's netns** to point at the cluster DNS service (CoreDNS's ClusterIP) and injects the namespace-aware search domains — this is a per-pod file, written by the [[#2.2 The Pause Container and Pod Network Namespace|kubelet]], not by the CNI plugin itself.

---

## Part 2 — The Kubernetes Networking Model

### 2.1 The Four Kubernetes Networking Rules

Kubernetes doesn't implement networking itself — it defines a **contract** that any conforming network implementation must satisfy:
1. Every pod gets its own IP address ("IP-per-pod").
2. Pods on any node can reach pods on any other node **without NAT**.
3. Agents on a node (e.g. kubelet) can reach all pods on that node.
4. Containers within a pod share one network namespace — they talk over `localhost`.

Everything in Part 1 exists so that some plugin can make these four guarantees true.

### 2.2 The Pause Container and Pod Network Namespace

Every pod's [[Network Namespace]] is actually owned by an infrastructure container called the **pause container** (`sandbox`), not by any of your app containers. Sequence:
1. The container runtime (containerd/CRI-O) creates the sandbox: a new netns, and starts `pause` in it (its only job is to `sleep` forever, holding the namespace open).
2. Every application container in that pod is later started with `--net=container:<pause>` — i.e., it joins the *existing* namespace rather than getting its own.

This is why containers in a pod share an IP and can reach each other over `localhost`, and why a pod's IP survives individual container restarts (as long as the sandbox/pause container itself isn't recreated).

### 2.3 The CNI Spec

The [[CNI (Container Network Interface)]] spec is deliberately tiny: a CNI plugin is just an **executable binary** that the container runtime invokes with a JSON config on stdin and a handful of environment variables (`CNI_COMMAND`, `CNI_CONTAINERID`, `CNI_NETNS`, `CNI_IFNAME`), and it must implement four verbs:
- `ADD` — set up networking for a new sandbox, return the assigned IP/routes/DNS as JSON.
- `DEL` — tear it down.
- `CHECK` — verify existing state is still correct.
- `VERSION` — report supported spec versions.

Plugins are chained: a "main" plugin (bridge, ptp, host-device, ipvlan…) creates the interface, an [[IPAM plugin]] (`host-local`, `dhcp`, or a cloud-specific one) allocates the IP, and optional "meta" plugins (`portmap`, `bandwidth`, `firewall`) layer on extra behavior. This chaining is literally an ordered list in the `.conflist` file described in Part 3.

### 2.4 Services and kube-proxy

A Kubernetes **Service** gives a stable virtual IP (ClusterIP) that load-balances across a dynamic set of pod IPs (Endpoints). `kube-proxy` runs on every node and is the component that actually implements this, in one of a few modes:

- **iptables mode (historically default):** For each Service, kube-proxy writes a `KUBE-SVC-<hash>` chain; traffic to the ClusterIP jumps into it, and it probabilistically jumps into one of several `KUBE-SEP-<hash>` chains (one per backend pod, using iptables' `--probability` statistic match for randomized load-balancing), which perform the DNAT to the real pod IP:port.
- **IPVS mode:** kube-proxy instead programs the Service as an IPVS virtual server and each pod endpoint as a real server — see [[#1.6 IPVS]]. Scales far better with large Service counts.
- **nftables mode (newer):** same idea as iptables mode but on the nftables backend, avoiding iptables' linear-scan performance cliff.

All modes watch the API server (via the `Endpoints`/`EndpointSlice` and `Service` objects) and re-sync the local dataplane whenever backends change — pod restarts, scaling, rolling deploys — with zero application awareness required.

### 2.5 CoreDNS

[[CoreDNS]] runs as pods (usually 2+ replicas) behind its own Service, at a well-known ClusterIP. It watches Services and Endpoints via the API server and serves records like `<service>.<namespace>.svc.cluster.local`. As noted in [[#1.9 DNS Resolution Basics]], kubelet points every pod's `/etc/resolv.conf` at this ClusterIP so name resolution "just works" without any pod-level configuration.

### 2.6 NetworkPolicy

`NetworkPolicy` objects are pure API declarations — Kubernetes core does **not** enforce them. Enforcement is delegated entirely to the CNI plugin (Calico, Cilium, Weave, etc.), which typically translates policies into extra iptables/nftables/eBPF rules attached to each pod's veth or netns, filtering traffic before or after the bridge/routing step described in [[#1.3 Linux Bridges]] and [[#1.5 iptables and netfilter]]. A cluster using a CNI plugin that doesn't implement NetworkPolicy (plain Flannel, for example) will silently accept but never enforce these objects — a very common real-world gotcha.

---

## Part 3 — File-Level Changes at Each Cluster Stage

This section walks through, stage by stage, exactly what gets written to disk and to kernel state on a Linux node.

### 3.1 Stage — Cluster Bootstrap

When `kubeadm init` / node provisioning runs, before any pod ever gets scheduled:

| File / Path | Written by | Purpose |
|---|---|---|
| `/etc/sysctl.d/99-kubernetes-cri.conf` (or similar) | kubelet/kubeadm | Sets `net.bridge.bridge-nf-call-iptables=1`, `net.ipv4.ip_forward=1` — see [[#1.7 IP Forwarding and sysctl]] |
| `/etc/cni/net.d/` | *(empty until CNI installed)* | Directory kubelet's embedded CNI client watches for network config |
| `/opt/cni/bin/` | CNI plugin installer (often a DaemonSet init container) | Drops the actual CNI executable binaries (`bridge`, `host-local`, `loopback`, `flannel`, `calico`, `portmap`, etc.) |
| `/var/lib/cni/` | CNI plugin at runtime | State directory: IPAM allocations, per-container cache |
| `/etc/kubernetes/kubelet.conf`, kubelet's `--network-plugin`/`--cni-*` flags (or `KubeletConfiguration` `cniConfDir`/`cniBinDir`) | kubeadm | Tells kubelet where to look for the above two directories |

When you `kubectl apply -f <cni-manifest>.yaml` (e.g. Flannel, Calico), a DaemonSet runs on every node whose sole job is to **write files** into `/etc/cni/net.d/` and `/opt/cni/bin/` — this is the moment the node actually becomes CNI-capable. Example, Flannel writes:

```
/etc/cni/net.d/10-flannel.conflist     # the CNI conflist — plugin chain: flannel -> portmap
/etc/kube-flannel/net-conf.json        # flannel's own daemon config (backend type, pod CIDR)
/run/flannel/subnet.env                # this node's allocated subnet, written after flanneld leases it from etcd/API
```

and creates a `flannel.1` VXLAN interface (see [[#1.8 VXLAN and Overlay Networks]]) plus host routes for every other node's subnet.

Calico instead writes `/etc/cni/net.d/10-calico.conflist`, `/etc/cni/net.d/calico-kubeconfig`, and runs `bird`/`felix` as the dataplane agent, programming routes and iptables/ipset (`cali*` chains, `cali40*` ipsets) directly rather than relying on a static bridge.

### 3.2 Stage — Node Join

When a new worker joins:
1. kubelet starts, registers the Node object with the API server.
2. The CNI DaemonSet's pod gets scheduled onto the new node (via its own tolerations, bypassing the chicken-and-egg problem of "no network yet") and performs the same file drops as [[#3.1 Stage — Cluster Bootstrap]] locally.
3. The overlay/routing mesh converges: e.g. Flannel's daemon requests a subnet allocation, writes `/run/flannel/subnet.env`, and every *other* node's flanneld notices (via watching the API/etcd) and adds a route + VXLAN FDB entry for the new subnet. In BGP-mode Calico, `bird` peers with the new node and routes propagate automatically.

### 3.3 Stage — Pod Creation (the full sequence)

This is the core of "how K8s leverages Linux networking," end to end, for a single pod scheduled to a node:

1. **Scheduler** binds the pod to a node (API object update only — no networking yet).
2. **kubelet** on that node sees the binding, asks the **CRI runtime** (containerd/CRI-O) to create a pod sandbox.
3. **Runtime** creates the [[#2.2 The Pause Container and Pod Network Namespace|pause container]] with a fresh network namespace, e.g. bind-mounted at `/var/run/netns/cni-<uuid>`.
4. **Runtime invokes the CNI plugin's `ADD`** (the binary in `/opt/cni/bin/`, per the ordered chain in `/etc/cni/net.d/*.conflist`), passing `CNI_NETNS=/var/run/netns/cni-<uuid>` and `CNI_IFNAME=eth0` on the environment, and the conflist JSON on stdin.
5. **Main plugin** (e.g. `bridge`) does the [[#1.2 veth Pairs]] dance:
   - Creates `veth<hostside>` / `eth0` pair.
   - Moves `eth0` end into the pod's netns.
   - Attaches the host-side end to `cni0`/`cbr0` (see [[#1.3 Linux Bridges]]) — or for routed plugins, skips the bridge and instead adds a point-to-point route + a `/32` host route on the node.
6. **IPAM plugin** (`host-local`) allocates the next free IP from the node's pod subnet by writing a **lock file and lease file** under `/var/lib/cni/networks/<network-name>/<allocated-ip>` containing the container ID — this on-disk file *is* the IP allocation record; deleting it (which `DEL` does) is what frees the address for reuse.
7. IPAM's result flows back up the plugin chain; the `bridge` plugin assigns the IP to `eth0` inside the netns (`ip addr add`) and adds the default route inside the namespace pointing at the bridge/gateway IP.
8. **CNI writes a cache file** at `/var/lib/cni/results/<network>-<container-id>-<ifname>.json` — the full result JSON, used later so `DEL` knows exactly what to tear down without re-deriving it.
9. Control returns to the runtime, which reports the pod IP back to **kubelet**, which writes it into the Pod's `status.podIP` via the API server.
10. **kubelet** also writes the pod's `/etc/resolv.conf`, `/etc/hosts`, and `/etc/hostname` into files under `/var/lib/kubelet/pods/<pod-uid>/etc-hosts` etc., bind-mounted into the container — this is where the CoreDNS ClusterIP from [[#2.5 CoreDNS]] gets injected.
11. If a NetworkPolicy selects this pod, the CNI's policy engine (Calico's Felix, Cilium's agent) reacts to the new pod/endpoint and adds the relevant iptables/ipset/eBPF rules referencing the new veth or IP — see [[#2.6 NetworkPolicy]].
12. If any Service's `EndpointSlice` now includes this pod (because its labels match a Service selector), **kube-proxy** on every node updates its dataplane — new `KUBE-SEP-*` chain or new IPVS real-server entry — see [[#2.4 Services and kube-proxy]].

### 3.4 Stage — Service Creation and kube-proxy Sync

When a `Service` object is created/updated:
1. API server persists it; `EndpointSlice` controller computes matching pod IPs.
2. kube-proxy (watching both objects) recomputes the desired dataplane state and does an **atomic rule replace**:
   - iptables mode: writes a full new ruleset to a temp file and calls `iptables-restore` (not incremental `iptables -A` calls — this atomicity matters for avoiding half-applied states under high Service churn).
   - IPVS mode: uses the `ipvsadm`/netlink interface directly to add/remove virtual and real servers, and also manages a dummy interface (`kube-ipvs0`) that all ClusterIPs are bound to as a routing target.
3. This happens independently on **every node** in the cluster — there's no central load balancer; each node's kube-proxy is eventually consistent with the API server's view of Services/Endpoints.

### 3.5 Stage — Pod Deletion

Reverse of [[#3.3 Stage — Pod Creation (the full sequence)]]:
1. kubelet tells the runtime to stop the pod's containers, then tear down the sandbox.
2. Runtime invokes CNI plugin's `DEL`, passing the same `CNI_CONTAINERID`/`CNI_NETNS`.
3. Plugin reads back the cache file from `/var/lib/cni/results/`, removes the veth (deleting the host side auto-deletes the pair), removes it from the bridge/route table.
4. IPAM plugin deletes the lease file under `/var/lib/cni/networks/<network>/<ip>`, freeing that IP for the next pod.
5. Netns is destroyed once the last process (the pause container) exits and the bind-mount is removed.
6. EndpointSlice controller removes the pod's IP; kube-proxy on every node resyncs and removes the corresponding `KUBE-SEP-*` chain / IPVS real server, per [[#3.4 Stage — Service Creation and kube-proxy Sync]].

### 3.6 Stage — Node Drain / Leave / Maintenance

- **CNI plugin upgrades:** updating the DaemonSet rewrites `/etc/cni/net.d/*.conflist` and swaps binaries in `/opt/cni/bin/`; kubelet's CNI client picks up the new conflist on the *next* sandbox creation (existing pods' networking is untouched until they're recreated).
- **Node removal:** the node's flannel/Calico daemon deregisters, and every other node's route table / VXLAN FDB / BGP peers drop the routes pointing at that node's subnet — same convergence mechanism as join, in reverse.
- **iptables/IPVS garbage collection:** kube-proxy periodically does a full resync (not just event-driven), reconciling any drift — e.g. stale `KUBE-SEP-*` chains left behind by a kube-proxy crash mid-update.
- **Orphaned CNI state:** if a node crashes mid-`ADD`/`DEL`, stale files can be left in `/var/lib/cni/networks/` or `/var/lib/cni/results/`; most CNI plugins reconcile this against the container runtime's actual sandbox list on their next start.

---

## Part 4 — End-to-End Worked Example

Pod `web-7d4f` scheduled to `node-2`, part of Deployment `web`, exposed by Service `web-svc` (ClusterIP `10.96.5.20`), cluster running Flannel (VXLAN) + iptables-mode kube-proxy, pod subnet for `node-2` is `10.244.2.0/24`:

1. Sandbox created, netns `/var/run/netns/cni-8f3a...`.
2. Flannel CNI plugin (chained with `host-local` IPAM and `portmap`) runs `ADD`:
   - `veth7a2b` ↔ `eth0` pair created; `eth0` moved into netns.
   - `veth7a2b` attached to `cni0` bridge.
   - `host-local` allocates `10.244.2.14`, writes `/var/lib/cni/networks/cbr0/10.244.2.14`.
   - `eth0` inside netns gets `10.244.2.14/24`, default route via `10.244.2.1` (the bridge's gateway IP).
3. Result cached at `/var/lib/cni/results/cbr0-8f3a...-eth0.json`; `podIP` reported to API server.
4. `web-7d4f`'s IP joins `web-svc`'s `EndpointSlice`.
5. Every node's kube-proxy rewrites its `KUBE-SVC-WEB...` chain to include a new `KUBE-SEP-...` targeting `DNAT --to-destination 10.244.2.14:8080`.
6. A pod on `node-1` calling `http://web-svc` → DNS resolves via CoreDNS to `10.96.5.20` → local iptables DNATs to `10.244.2.14:8080` → since destination subnet belongs to `node-2`, node-1's flanneld-installed route sends it out via `flannel.1` → VXLAN-encapsulated over the real network to `node-2` → decapsulated → delivered to `cni0` → forwarded to `veth7a2b` → arrives at `web-7d4f`'s `eth0`.
7. Response retraces the same path; if `node-1` and `node-2` are on different L3 segments, MASQUERADE (SNAT) rules kube-proxy also installs ensure the return path routes correctly back through the DNAT-ing node rather than directly to the original client.

---

## Appendix — Useful Inspection Commands

```bash
# Namespaces
lsns -t net
ip netns list

# A running pod's network namespace (via crictl / containerd)
crictl inspectp <sandbox-id> | grep -i netns

# Interfaces & bridge membership
ip -d link show
bridge link show

# Routes
ip route show table all
ip rule list

# iptables — Kubernetes Service chains
iptables -t nat -L KUBE-SERVICES -n
iptables -t nat -L KUBE-SVC-<hash> -n

# IPVS mode equivalent
ipvsadm -Ln

# CNI on-disk state
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conflist
ls /opt/cni/bin/
ls /var/lib/cni/networks/*/
ls /var/lib/cni/results/

# CoreDNS resolution from inside a pod
kubectl exec -it <pod> -- cat /etc/resolv.conf
```

---

### Related concepts to expand into their own notes
[[Network Namespace]] · [[veth pair]] · [[Linux Bridge]] · [[Routing Table]] · [[Policy Routing]] · [[iptables]] · [[netfilter]] · [[IPVS]] · [[VXLAN]] · [[CNI (Container Network Interface)]] · [[IPAM plugin]] · [[CoreDNS]]
