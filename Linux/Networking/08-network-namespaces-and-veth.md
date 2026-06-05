# 08 · Network Namespaces, veth, Bridges, and Container Networking

The kernel feature that makes containers possible. Every Docker container, every Kubernetes pod, every netns-isolated CI job uses this.

## 8.1 What is a network namespace

A **netns** is a kernel namespace (one of seven: mount, pid, net, ipc, uts, user, cgroup) that isolates network resources:

- Interfaces (`eth0`, `lo`, etc.)
- Routing tables
- ARP/neighbor caches
- Conntrack tables
- iptables/nftables rules
- Sockets

Each netns has its own everything-network. The host's `lo` is *not* the container's `lo`.

```
                Host netns
                ┌──────────┐
                │ eth0     │
                │ lo       │
                │ docker0  │
                │ veth-A   │ ── veth pair ──┐
                └──────────┘                │
                                            ▼
                                     Container netns
                                     ┌──────────┐
                                     │ lo       │
                                     │ eth0 (= veth-B) │
                                     └──────────┘
```

Commands:
```bash
ip netns add ns1                # create
ip netns exec ns1 ip link       # run command in ns
ip link set dev veth0 netns ns1 # move interface
ip netns delete ns1             # destroy
```

A process enters a netns via `setns(fd, CLONE_NEWNET)` or starts in one via `unshare(CLONE_NEWNET)`. The `/proc/<pid>/ns/net` is a fd you can bind-mount or pass around (`nsenter`).

## 8.2 veth — virtual ethernet pair

A `veth` is a kernel-only ethernet pair. Two interfaces; whatever you TX into one, the other RX-es.

```bash
ip link add veth0 type veth peer name veth1
ip link set veth1 netns ns1   # move one side to ns1
```

veth is the bridge between netns. Container has one end (`eth0` inside); host has the other.

Key property: TX skb on veth A becomes RX skb on veth B *with no copy* — pointer flip. But:
- The skb traverses both netns' filter chains (netfilter, tc).
- GRO/GSO can apply; the skb is a normal one.

Performance: a tuned veth pair does ~10Gbps on a single CPU. Slower than direct NIC by a constant factor (sk_buff allocation, two-side processing).

## 8.3 Bridges — Linux software switch

`bridge` is the L2 switch. Add interfaces with `bridge link add` or `ip link set master`. Frames switch by destination MAC.

```
        Host
   ┌────────────────┐
   │ bridge: br0    │ ── eth0 (uplink)
   │   │            │ 
   │   ├ veth-A     │ ── (paired veth-A' in container-1)
   │   ├ veth-B     │ ── (paired veth-B' in container-2)
   │   └ veth-C     │ ── (paired veth-C' in container-3)
   └────────────────┘
```

Properties:
- IPv4/v6 forwarding: if `net.ipv4.ip_forward=1`, bridge acts like a hub+router combo.
- STP optional (`bridge stp on/off`).
- VLAN-aware (`bridge vlan add ...`).
- Performance: handles many flows; CPU-bound at high pps.

This is Docker's default network model (the `docker0` bridge).

## 8.4 macvlan / ipvlan — direct sub-interface

veth + bridge has overhead. **macvlan** creates child interfaces with their own MAC, sharing the parent NIC.

```bash
ip link add link eth0 mvlan0 type macvlan mode bridge
ip link set mvlan0 netns ns1
```

Each macvlan has its own L2 identity. The NIC sees multiple MACs. Modes:
- `private`: child interfaces can't talk to each other.
- `vepa`: traffic between children goes through external switch (802.1Qbg).
- `bridge`: children can talk; no host involvement.
- `passthru`: one child takes over the parent.

**ipvlan** is the L3 version: children share the parent's MAC but have unique IPs.

Why use these over veth+bridge:
- Lower CPU cost (one veth-equivalent saved).
- Direct external visibility — packets arrive with the container's own MAC/IP.

Why not:
- macvlan can't reach the host's address (kernel quirk — "no hairpin").
- ipvlan can't do source-NAT through iptables MASQUERADE the way bridges can.
- More complex DHCP / address management.

## 8.5 Container networking — CNI taxonomy

Kubernetes uses CNI (Container Network Interface) plugins. Each is a binary called by kubelet to attach a pod to the network.

| Plugin | Mode | Forwarding | NetworkPolicy |
|--------|------|-----------|---------------|
| **bridge** (reference) | veth + linux bridge | Linux routing | None |
| **flannel** | veth + bridge, overlay (vxlan/host-gw) | Linux routing | Limited |
| **Calico** | veth, no bridge (route-based, BGP) | Linux IP routing | iptables/eBPF |
| **Cilium** | veth, no bridge | eBPF (in-kernel datapath) | eBPF |
| **AWS VPC CNI** | ENI per pod (ipvlan or AWS-specific) | AWS VPC | none/AWS SG |
| **OVN-Kubernetes** | Open vSwitch overlay | OpenFlow | OVN |
| **kube-router** | veth + BGP | Linux | iptables/eBPF |
| **Multus** | meta-plugin, attaches multiple | varies | varies |
| **Antrea** | veth + OVS | OpenFlow | OVS flows |
| **Weave** | veth + libnetwork overlay | UDP+fastdp | iptables |

Key concepts for the interview:

- **Overlay** (flannel-vxlan, weave): pod IP space is independent of node IP space; encapsulated over UDP/VXLAN. Pros: works anywhere, simple. Cons: 50-byte VXLAN overhead, NIC offload required, MTU lowered.
- **Underlay / routed** (Calico-BGP, kube-router): each node advertises its pod CIDR via BGP; routers in the underlay learn it. Pros: native MTU, simpler debugging. Cons: requires BGP-speaking underlay.
- **Cloud-provider-specific** (AWS VPC CNI, Azure CNI): each pod gets a real VPC IP. Pros: no overlay, IAM/SG works per pod. Cons: tied to provider, IP exhaustion.
- **eBPF-native** (Cilium): replaces kube-proxy with eBPF programs; service / network policy in eBPF.

## 8.6 kube-proxy modes

Kube-proxy implements Service VIPs (cluster IP). Three modes:

| Mode | How | Pros | Cons |
|------|-----|------|------|
| **iptables** | DNAT rules per service | Default, well-tested | O(n) match cost; slow with many services |
| **IPVS** | Linux IPVS in kernel | Fast, many algorithms | Conntrack quirks; more setup |
| **eBPF (Cilium)** | eBPF programs at tc/socket hook | Fast, custom, policy-aware | Cilium-specific |

A cluster with 5000 services using iptables: 50000+ rules; kube-proxy reload time ~30s. IPVS or Cilium is the upgrade path.

## 8.7 conntrack and netns

Each netns has its own conntrack table. Important for:
- A container's flows don't fill the host's conntrack.
- Conntrack tracks NAT state per-netns.

Sysctl: `net.netfilter.nf_conntrack_max` is per-netns since 4.10+ (with `nf_conntrack_default_on`).

## 8.8 Pid 1 and networking

Common Docker gotcha: shell script as pid 1 doesn't reap zombies → zombie processes accumulate. Use `tini` or `dumb-init` for pid 1 in containers.

Network-related: pid 1 owns the netns. When pid 1 exits, the namespace is torn down (unless there's another bind-mount via `/proc/<pid>/ns/net` or `ip netns add` which pins it).

## 8.9 ARP/ND in netns

Each netns has its own neighbor cache. `ip neigh show` differs between netns.

`gc_thresh*` sysctls control table size — under-tuned for big clusters (thousands of pods per node).

## 8.10 Common interview probes

- **"Draw a docker container's network."** Bridge `docker0`, veth pair, container's `eth0`, NAT via iptables MASQUERADE for outbound.
- **"How does kubernetes pod-to-pod networking work?"** Depends on CNI; describe overlay vs routed; mention CNI spec is "give pod an IP that's reachable from any other pod, no NAT required."
- **"What if conntrack overflows?"** Sysctl `nf_conntrack_max`, table full → drops, log "nf_conntrack: table full." Mitigation: raise max, eBPF-based conntrack bypass (Cilium), or no-conntrack (notrack target).
- **"How is service VIP implemented?"** iptables DNAT to a randomly selected endpoint (probability-based); IPVS does the same with a real LB algorithm; eBPF maps at the socket layer.
- **"How do you debug a packet drop in a container?"** Enter the netns with `nsenter -n -t $pid`, then `tcpdump`, `ip route`, `iptables -nvL`, `conntrack -L`.

## 8.11 Corner cases

- **`net.ipv4.ip_forward=0` in container.** Pod doesn't forward; usually desired but breaks transparent proxies in sidecars.
- **veth checksum offload broken.** Some kernels disabled checksum offload on veth for safety; impacts perf inside container. `ethtool -K veth0 tx off rx off`.
- **iptables in pod sees only its own netns.** Sidecar (Envoy) iptables rules don't affect main container traffic unless they share netns (which Kubernetes pods do!).
- **`docker0` MTU=1500, encapsulated VXLAN can't fit.** MTU mismatch → fragmentation. Lower container MTU to 1450.
- **Pod IP duplication after restart.** CNI IPAM races; some CNIs leak. Manual cleanup needed.
- **PMTUD broken across overlay.** Encapsulated ICMP Frag-Needed may not deliver. Use `tcp_mtu_probing=1`.
- **GRO on veth.** veth supports GRO since 5.x; before that, intra-container TCP was slow.
- **DNS in containers.** Kubernetes uses CoreDNS; pod resolv.conf points to a cluster IP that's actually iptables/IPVS-redirected to a CoreDNS pod. ndots:5 setting causes slow resolution for external names; common interview "why is DNS slow."

## 8.12 Live-migration of network state

If you snapshot+restore a container (CRIU), TCP state is lost — unless `TCP_REPAIR` (3.8+):

1. Pause socket. `setsockopt(TCP_REPAIR, 1)` — sender doesn't send segments, receiver doesn't ACK.
2. Read state: seq, sndbuf, recvbuf, opts.
3. On new host: create socket, `TCP_REPAIR` mode, set state, `TCP_REPAIR` off.

This is exotic but real (CRIU, OpenVZ).

## 8.13 Performance numbers (cheat sheet)

| Path | Throughput per CPU | Latency |
|------|-------------------|---------|
| Host loopback (lo) | 30+ Gbps | ~1 µs |
| veth pair, tuned | 10 Gbps | ~5 µs |
| veth+bridge | 5 Gbps | ~10 µs |
| macvlan | 10 Gbps | ~6 µs |
| ipvlan | 12 Gbps | ~5 µs |
| VXLAN overlay (sw) | 2 Gbps | ~30 µs |
| VXLAN overlay (NIC offload) | 8 Gbps | ~8 µs |
| Cilium eBPF (no bridge) | 12 Gbps | ~3 µs |

Order of magnitude; tune your own.

## 8.14 Alternative implementations

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Container internet | bridge + iptables MASQ | macvlan + direct routing | AWS VPC CNI (ENI per pod) |
| Pod-to-pod | overlay (flannel-vxlan) | BGP (Calico) | eBPF (Cilium) |
| Service VIP | iptables DNAT | IPVS | eBPF (Cilium) |
| Network policy | iptables | nftables | eBPF (Cilium NPL) |
| Host firewall on top | iptables-INPUT | nftables-input | XDP_DROP |
| Multi-tenant isolation | separate netns + bridges | VRF | netns + VLAN |

## Must-internalize

- netns isolates: interfaces, routes, ARP, iptables, conntrack, sockets.
- veth = kernel ethernet pair; one end in each netns. No copy on pair traversal.
- bridge = L2 switch; macvlan = sub-interface with own MAC; ipvlan = sub-interface with shared MAC.
- CNI plugins choose: overlay, routed, or cloud-native. Cilium is the eBPF future.
- kube-proxy: iptables (default, O(n)), IPVS (fast), eBPF (Cilium replaces it).
- conntrack is per-netns since 4.10+; can fill up; mitigation = bigger table or bypass.
- Debug: `nsenter -n -t $pid`, then `tcpdump`/`ip route`/`iptables`/`conntrack`.
- TCP_REPAIR enables container TCP live migration.