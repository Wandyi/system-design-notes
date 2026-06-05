# 08 · Networking — CNI and Services

The networking model is one of K8s's most asked-about topics, especially for staff candidates with networking background.

## 8.1 The K8s network model — three rules

1. **Every pod gets a unique cluster-wide IP.**
2. **Pods can talk to all other pods (and nodes) without NAT.**
3. **Agents on a node (kubelet, kube-proxy, daemons) can talk to all pods on that node.**

The CNI plugin's job is to make rule 1 + 2 true. Everything else (Services, policies, mesh) is layered on top.

## 8.2 The CNI spec

CNI ("Container Network Interface") is a small spec. Each plugin:
- Reads JSON config from stdin.
- Outputs JSON result.
- Has commands: `ADD`, `DEL`, `CHECK`.

```json
{
  "cniVersion": "1.0.0",
  "name": "mynet",
  "type": "myplugin",
  "ipam": {"type": "host-local", "subnet": "10.0.0.0/24"}
}
```

When kubelet creates a sandbox, it calls:
```
ADD with netns=/proc/<pid>/ns/net
       containerid=<id>
       ifname=eth0
```

The plugin:
1. Creates a veth pair.
2. Moves one end into the pod's netns.
3. Allocates an IP (IPAM).
4. Configures the interface (IP, MTU, routes).
5. Returns the IP + interface details.

## 8.3 CNI taxonomy

| Plugin | Model | Forwarding |
|--------|-------|-----------|
| **bridge** (reference) | veth + Linux bridge | Linux IP routing |
| **flannel** (vxlan / host-gw) | veth + bridge, overlay or routed | UDP encap (vxlan) or routed |
| **Calico** | veth, no bridge (route-based) | Linux IP routing + BGP |
| **Cilium** | veth, no bridge | eBPF datapath |
| **AWS VPC CNI** | ENI per pod (or branched ENIs) | AWS VPC |
| **Azure CNI** | bridge + Azure VNET | Azure VNET routing |
| **GKE Dataplane V2** | Cilium-based eBPF | eBPF |
| **OVN-Kubernetes** | OVS overlay | OpenFlow |
| **Antrea** | OVS overlay or routed | OVS |
| **Weave** | overlay with UDP | Userspace/fastdp |
| **Multus** | meta plugin, multiple interfaces per pod | varies |

### Overlay vs routed

- **Overlay** (flannel-vxlan, weave): pod CIDR is independent of node network. Packets are encapsulated (VXLAN/Geneve) into UDP packets between nodes. Pros: works anywhere. Cons: MTU overhead (~50 bytes), NIC offload required for performance.
- **Routed** (Calico-BGP, kube-router): each node announces its pod subnet via BGP; the underlay routes natively. Pros: native MTU, easier debug. Cons: requires BGP-speaking underlay (or supports route reflectors).
- **Cloud-native** (AWS VPC CNI, GKE-Calico): each pod gets a real VPC IP. Pros: no overlay, IAM/Security Groups per pod, native VPC features. Cons: cloud-provider lock-in, IP exhaustion at scale.

## 8.4 Pod-to-pod traffic (the data plane)

Same-node:

```
   pod A (10.244.1.5)
      │
      ▼ veth-A in pod ns
      │
      └─ veth-A' in host ns
                │
                ▼ Linux bridge / Linux routing / Cilium eBPF
                │
              veth-B' (host)
                │
              veth-B (pod B ns)
                │
                ▼
            pod B (10.244.1.6)
```

Cross-node:

```
   pod A (10.244.1.5)   pod B (10.244.2.7)
         │                    │
         ▼                    ▼
    host eth0 ───── network ───── host eth0
    [encap with VXLAN if overlay]
    [BGP-route lookup if routed]
    [Linux routing or eBPF map lookup either way]
```

## 8.5 Services — the cluster's load balancer

A `Service` is a stable virtual IP fronting a set of pods (selected by label).

```yaml
apiVersion: v1
kind: Service
metadata: {name: my-svc}
spec:
  selector: {app: my-app}
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP   # ClusterIP | NodePort | LoadBalancer | ExternalName
```

The Service controller (in kube-controller-manager) maintains:
- **Endpoints** (legacy) — the pod IPs + ports.
- **EndpointSlices** (1.21+ default) — sharded; each slice has ≤100 endpoints.

### Service types

| Type | What |
|------|------|
| **ClusterIP** | Internal VIP only; default |
| **NodePort** | + opens a port on every node (range 30000-32767) |
| **LoadBalancer** | + cloud LB provisioned via cloud-controller-manager |
| **ExternalName** | DNS CNAME alias (no proxying) |
| **Headless** (`clusterIP: None`) | No VIP; DNS returns A records of pod IPs directly |

### EndpointSlices

For a Service with 1000 endpoints, the old Endpoints resource was one 1000-element object — every change rewrote it, every watcher saw the full list.

EndpointSlices shard: 10 EndpointSlices × 100 endpoints. A single endpoint change updates one slice. The watch traffic scales as O(changes), not O(endpoints).

## 8.6 kube-proxy — the Service implementation

Each node runs kube-proxy (or a replacement like Cilium). It watches Services + EndpointSlices and programs the data plane.

### iptables mode (default)

For each Service:
```
-A KUBE-SERVICES -d 10.96.0.10 -p tcp --dport 80 -j KUBE-SVC-XXX
-A KUBE-SVC-XXX -m statistic --mode random --probability 0.500 -j KUBE-SEP-A
-A KUBE-SVC-XXX -j KUBE-SEP-B
-A KUBE-SEP-A -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-B -j DNAT --to-destination 10.244.1.6:8080
```

Random per-packet selection (no consistent hashing). Once selected, conntrack remembers the DNAT for the duration of the flow.

**Scaling problem**: each Service adds ~3 rules per endpoint. 5000 services × 10 endpoints = 150K rules. iptables walks linearly → kube-proxy reload time grows. By 1000+ services, scale-test pain is real.

### IPVS mode

Kernel IPVS table — O(1) lookup. Supports algorithms: round-robin, weighted, least-connection, source-hash, destination-hash. Faster than iptables at scale.

```
ipvsadm -L -n
TCP 10.96.0.10:80 rr
  -> 10.244.1.5:8080  Masq  1  0  0
  -> 10.244.1.6:8080  Masq  1  0  0
```

### eBPF mode (Cilium)

No kube-proxy at all. eBPF programs at socket connect (or tc-ingress) replace the VIP with a real backend address. The cluster IP is never put on the wire — it's resolved at the socket level.

Benefits:
- O(1) lookup via BPF map.
- Maglev consistent hashing.
- No conntrack table fill from kube-proxy.
- DSR mode for high-throughput.

Cilium does this; KubeProxy-replacement mode in EKS / GKE Dataplane V2.

## 8.7 DNS — CoreDNS

CoreDNS runs as Pods, fronted by a Service (typically `kube-dns` Service IP `10.96.0.10`).

Pod's `/etc/resolv.conf`:
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**ndots:5** is the bite: a name with fewer than 5 dots is tried with all `search` suffixes first. So `google.com` is tried as:
- `google.com.default.svc.cluster.local`  ← NXDOMAIN
- `google.com.svc.cluster.local`           ← NXDOMAIN
- `google.com.cluster.local`               ← NXDOMAIN
- `google.com`                             ← finally resolves

5× the DNS traffic. Solutions:
- Use FQDNs with trailing dot (`google.com.`) — skips search.
- Lower `ndots` per pod (`spec.dnsConfig.options`).
- Run NodeLocalDNS — per-node caching daemon.

### NodeLocalDNS

A DaemonSet runs a DNS cache on each node, listening on a fixed link-local IP (`169.254.20.10`). Pods are configured to use that IP. Saves the round-trip to CoreDNS for cache hits.

### Service DNS records

For a Service `foo` in namespace `bar`:
- `foo.bar.svc.cluster.local` — ClusterIP A record.
- For headless (clusterIP: None): A records per pod IP.
- `<pod-name>.foo.bar.svc.cluster.local` for StatefulSet-managed pods.
- SRV records for ports.

## 8.8 Network MTU and the gotcha

Standard Ethernet MTU: 1500.
VXLAN overhead: 50 bytes (8 VXLAN + 8 UDP + 20 IP + 14 outer Ether). Pod MTU on VXLAN: 1450.

Failing to lower the pod MTU = fragmentation = perf cliff. Most CNI plugins auto-detect; verify with `ip link show eth0` inside a pod.

For underlay encrypted overlays (WireGuard between nodes): even more overhead.

## 8.9 Cluster IP allocation

The control plane reserves a CIDR for Service IPs: `--service-cluster-ip-range=10.96.0.0/12`. The kube-apiserver allocates from this range when ClusterIP is unset.

Pod CIDR is allocated per-node by the NodeIPAM controller (or by CNI itself). `--pod-cidr-allocation-mode` controls.

IP exhaustion is a real failure mode at scale; plan CIDRs.

## 8.10 IPv4 / IPv6 / dual stack

Since 1.21+ dual-stack is GA. Services can have both addresses:

```yaml
spec:
  ipFamilies: [IPv4, IPv6]
  ipFamilyPolicy: PreferDualStack   # SingleStack | PreferDualStack | RequireDualStack
  clusterIPs: [10.96.0.10, fd00::1]
```

CNI must support it; not all do well.

## 8.11 Ingress and Gateway API

**Ingress** (legacy): a resource pointing at an Ingress controller (NGINX, HAProxy, Traefik, GCE-LB). Controller installs frontend (LB) and routes by host/path.

**Gateway API** (1.27+ stable): a more flexible model with three resources:
- `GatewayClass` — kind of gateway (NGINX, Envoy, cloud-LB).
- `Gateway` — instance + listeners (ports).
- `HTTPRoute` (or GRPCRoute, TLSRoute) — routing rules.

Gateway API addresses Ingress's limitations: cross-namespace routes, modular controllers, role separation (operator manages Gateway, dev manages HTTPRoute).

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: my-route, namespace: my-app}
spec:
  parentRefs:
  - name: my-gateway
    namespace: gateway-ns
  hostnames: [api.example.com]
  rules:
  - matches:
    - path: {type: PathPrefix, value: /v1}
    backendRefs:
    - name: my-svc-v1
      port: 80
```

## 8.12 Service types in cloud — LoadBalancer

When `type: LoadBalancer`, cloud-controller-manager:
- Reads the Service.
- Provisions a cloud LB (AWS NLB/ALB, GCE LB, Azure LB).
- Configures health checks (uses readinessProbe).
- Writes the LB's external IP to `status.loadBalancer.ingress`.

NodePort is opened on every node; the cloud LB targets NodePort on all nodes; kube-proxy routes from NodePort → ClusterIP → pod. The cloud LB doesn't need to know about pods.

**externalTrafficPolicy**:
- `Cluster` (default): node receives traffic on NodePort, may forward to another node's pod. Loses client source IP (SNAT'd).
- `Local`: node only forwards to local pods. Preserves source IP. May cause uneven load (nodes without pods drop traffic; LB health-checks NodePort to find which nodes have pods).

## 8.13 EndpointSlices in detail

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-svc-abc
  labels:
    kubernetes.io/service-name: my-svc
endpoints:
- addresses: ["10.244.1.5"]
  conditions: {ready: true, serving: true, terminating: false}
  nodeName: node-1
  zone: us-east-1a
  hints:
    forZones: [{name: us-east-1a}]
ports:
- port: 8080
  protocol: TCP
```

The `hints.forZones` enables **Topology Aware Routing** (KEP-2433): kube-proxy/CNI prefers endpoints with hints matching the consumer's zone. Reduces cross-zone traffic costs.

## 8.14 Headless Services and StatefulSet

```yaml
apiVersion: v1
kind: Service
spec:
  clusterIP: None
  selector: {app: my-stateful}
```

No VIP; DNS returns A records for each pod IP. StatefulSet uses headless Services for stable pod DNS names (`my-stateful-0.my-svc.ns.svc.cluster.local`).

## 8.15 Common interview probes

- **"How does a pod find a Service?"** Pod calls DNS → CoreDNS returns ClusterIP → kube-proxy/eBPF resolves ClusterIP to backend → packet flows.
- **"What's the difference between kube-proxy iptables and IPVS?"** iptables linear scan; IPVS kernel hash table O(1). Both still use conntrack.
- **"What's an EndpointSlice?"** Sharded list of endpoints for a Service. Replaces Endpoints. Each slice ≤100 endpoints.
- **"What's `externalTrafficPolicy: Local`?"** Cloud LB only sends to nodes with local pods. Preserves source IP.
- **"How does CNI work?"** kubelet calls CNI binary with `ADD` + netns path. Plugin creates veth, allocates IP, configures.
- **"Why is DNS sometimes slow?"** `ndots:5` causes many NXDOMAIN lookups; use FQDN with trailing dot or NodeLocalDNS.

## 8.16 Corner cases

- **Pod IP reused after pod deletion** — CNI may return the IP to IPAM; new pod gets old IP. Conntrack from old connections may misroute briefly.
- **Network policy with no egress allowed** — DNS to CoreDNS fails; almost everything breaks. Always allow DNS egress.
- **Pod stuck in `ContainerCreating` with `NetworkPluginNotReady`** — CNI plugin not installed; kubelet won't create pods.
- **Service IP not reachable from outside** — by design; ClusterIP is internal-only.
- **kube-proxy iptables flush during restart** — brief window of broken Services. Cilium eBPF has atomic map updates and avoids this.
- **PodIP missing from EndpointSlice** — readiness probe failing; Pod is Running but not Ready.

## 8.17 Alternative approaches

| Need | Default | Alternative 1 | Alternative 2 |
|------|---------|---------------|---------------|
| Pod networking | overlay (flannel-vxlan) | routed (Calico-BGP) | cloud-native (AWS VPC CNI) |
| Service VIP | iptables | IPVS | eBPF (Cilium) |
| External traffic | NodePort + cloud LB | Ingress | Gateway API |
| DNS perf | CoreDNS only | + NodeLocalDNS | FQDN + ndots:1 |
| Source IP preservation | externalTrafficPolicy: Local | Proxy protocol | NLB instead of ALB |
| Topology-aware routing | hints.forZones | service mesh locality | manual zonal services |

## Must-internalize

- Network model: every pod has unique IP; no NAT inside cluster.
- CNI: ADD with netns path → veth + IP allocated.
- Overlay vs routed vs cloud-native; pick by ops appetite.
- Services: ClusterIP > NodePort > LoadBalancer > ExternalName.
- EndpointSlices replace Endpoints; shard at 100.
- kube-proxy modes: iptables (default, slow at scale) > IPVS (kernel) > eBPF (Cilium).
- DNS: CoreDNS + ndots:5 = many lookups; NodeLocalDNS caches.
- externalTrafficPolicy: Cluster (SNAT, even load) vs Local (preserve IP, uneven).
- Gateway API is the future of Ingress.
