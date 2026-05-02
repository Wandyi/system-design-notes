# AWS VPC at Scale — Realistic Scenarios, Trade-offs, and Alternatives

A staff-level walk through the VPC patterns that come up when an AWS estate grows past one account and one region, when traffic crosses tens of accounts, dozens of regions, hundreds of services, and where tens of thousands of EKS pods exhaust /16s. Each scenario is paired with the realistic alternatives and the trade-offs that drive the choice.

The point of this doc is not "what does a VPC do" — it's *which VPC pattern* you reach for under specific pressures (scale, cost, blast-radius, compliance, latency, ops complexity), and *what you give up* when you do.

---

## 0. Mental Model — VPC Building Blocks (only what's load-bearing)

Just enough vocabulary to make the scenarios crisp:

- **VPC**: an isolated L3 network in one region, one CIDR (or several with secondary CIDRs).
- **Subnet**: an AZ-scoped CIDR slice. Public / private / isolated is a *route table* property, not subnet metadata.
- **Route table**: per-subnet (or per-edge-association). Where most outages start.
- **IGW / Egress-only IGW**: bidirectional v4 / v6-only outbound. NAT gateway lives behind IGW.
- **NAT Gateway**: managed, AZ-scoped, ~45 Gbps per NATGW, ~$0.045/GB processing — the cost lever that matters at scale.
- **VPC Peering**: 1:1, non-transitive, no overlap, free across AZs in same region. Hits a wall fast.
- **Transit Gateway (TGW)**: regional hub; transitive routing; multiple route tables for segmentation. The default backbone past ~10 VPCs.
- **TGW Peering**: cross-region or cross-account inter-TGW.
- **PrivateLink (VPC Endpoint Services + Interface Endpoints)**: 1-way service exposure over ENIs; no CIDR coordination; native to consumers.
- **Gateway Endpoints**: free, for S3/DynamoDB only, route-table based.
- **Gateway Load Balancer (GWLB)**: bump-in-the-wire L3 inspection — hairpins traffic through firewall fleet via GENEVE.
- **VPC Lattice**: app-layer service-to-service mesh across VPCs/accounts; HTTP/gRPC aware.
- **Cloud WAN**: AWS-managed global SD-WAN; replaces TGW-of-TGWs.
- **Direct Connect (DX)**: physical fiber to AWS; private VIF (one VPC), transit VIF (TGW), public VIF (S3/PublicIPs). MACsec optional.
- **Route 53 Resolver Endpoints**: inbound (on-prem → AWS DNS) and outbound (AWS → on-prem DNS).
- **AWS Network Firewall**: managed Suricata-based stateful firewall, deployed via GWLB-style or dedicated subnet.
- **IPAM**: hierarchical IP address manager — the only sane way to plan at scale.
- **RAM (Resource Access Manager)**: shares subnets/TGW/Cloud WAN segments across accounts.
- **Security Group references across VPCs**: works with PrivateLink and same-VPC; *cross-VPC SG references* require TGW + careful design (or VPC peering with cross-VPC SG refs enabled).

Keep these in working memory; the scenarios compose them.

---

## 1. Multi-Account Foundation — Why You Don't Run One Big VPC

**The realistic scenario**: a 500-engineer org running one prod account, one dev, one shared-services. Every team begs for VPC peering, security wants a single egress, finance wants one bill. Within six months the network is a forest of bilateral peerings, route tables drift, and a single CIDR overlap tanks a migration.

### What the staff engineer does instead
- **Account per environment per workload** (Control Tower / OU layout). Account is the strongest blast-radius boundary AWS has — IAM doesn't cross it.
- **One VPC per account** for most workloads (don't shard into many small VPCs unless you have a real isolation requirement — they cost ENI quota and ops attention, not money).
- **Centralized network account** that owns: TGW, central egress VPC, central inspection VPC, central Resolver, IPAM. RAM-shares TGW attachments, subnets, and resolver rules into workload accounts.
- **CIDR allocation governed by IPAM** — no team picks a CIDR; they request a /20 from a pool. Prevents the 10.0.0.0/16 collision that every org hits.

### Trade-offs / alternatives
- **Shared VPC (RAM-shared subnets)** vs **VPC-per-account**: shared VPC reduces NATGW count and simplifies routing but couples lifecycle (a misbehaving team's ENI sprawl impacts neighbors; subnet quota becomes shared). Choose shared VPC when teams are tightly coupled (same prod plane). Choose VPC-per-account for hard isolation (data residency, regulated workloads).
- **Cloud WAN** instead of central-network-account-+-TGW: managed segments and policy as code, but newer (some attachment types not yet supported), region availability still expanding, and harder to migrate off than TGW. Choose if you're a global enterprise starting fresh; stick with TGW if you have legacy.

---

## 2. Hub-and-Spoke with Transit Gateway

**The scenario**: 50+ VPCs across dev/staging/prod accounts. You want any-to-any L3 connectivity for a handful of shared services (auth, observability, secrets), but **most spokes should not see each other**.

### Pattern
- One TGW per region, in the network account, RAM-shared org-wide.
- Multiple **TGW route tables** create logical segments:
  - `prod-rt`: prod VPCs see each other + shared services.
  - `non-prod-rt`: dev/staging see each other + shared services.
  - `shared-services-rt`: sees everyone.
  - `egress-rt`: only routes 0.0.0.0/0 to the egress VPC.
- Each VPC attaches once; its association determines which RT it pulls from; *propagation* into other RTs determines who sees it.
- Inter-region: **TGW Peering** with statically-defined CIDR routes (no propagation across the peer — explicit allow only).

### Why TGW over a mesh of VPC peerings
- VPC peering is **non-transitive** → 50 VPCs = up to 1225 peerings = unmanageable route tables.
- VPC peering doesn't support *segmentation*. Once peered, full L3 reachability subject to RTs and SGs.
- TGW gives transitive routing + per-segment route tables + central place to attach DX/VPN.

### Trade-offs
- TGW costs per-attachment-hour (~$0.05/h) and per-GB ($0.02/GB processed) — at high east-west volume this dwarfs NATGW costs. Quantify against a peering mesh; for two VPCs talking PB/month, a peer is free.
- TGW is regional — global needs Cloud WAN or TGW-peering mesh.
- TGW MTU is 8500 within the same TGW; 1500 across a peer or DX. Plan for path MTU discovery to actually work, or you'll see mysterious tail-latency spikes.

### Alternatives
- **Cloud WAN**: same hub-and-spoke shape, plus *attachment policies as JSON* and global routing in one resource. Migration friction is real (no in-place move from TGW).
- **VPC Peering**: keep for two specific high-bandwidth pairs that are both bilateral and low-cardinality. Don't try to scale this.
- **PrivateLink-only model**: instead of L3 reachability, expose each producer service as an endpoint service. Producers and consumers don't share an L3 plane at all. See §4.

---

## 3. Centralized Egress vs Per-VPC NAT

**The scenario**: 200 VPCs each with their own NAT Gateway in 3 AZs. Each NATGW idles at $32/month and processes maybe 100 GB → another $4.50. The bill shows $20 K/month for *idle NAT*, before any data even moves. The CISO also wants every outbound flow to traverse a chokepoint for inspection and allow-listing.

### Centralized Egress pattern
- One **egress VPC** in the network account, with NATGWs in each AZ + IGW.
- All workload VPCs default-route 0.0.0.0/0 to TGW; TGW's egress route table sends to the egress VPC; egress VPC's NATGW SNATs to the internet.
- Optional: place an **AWS Network Firewall** or third-party (Palo Alto, Fortinet, Check Point) inline via GWLB inside the egress VPC for FQDN allow-listing and IDS/IPS.

### Trade-offs
- **Cost**: massive savings on NATGW idle hours and per-NATGW data processing — but TGW data processing fees apply on the way out (~$0.02/GB) on top of NAT processing. Do the math: centralized wins above ~5 TB/month aggregated egress *or* above ~10 VPCs.
- **Blast radius**: a saturated central egress affects everyone. Per-VPC NAT bulkheads failures.
- **Latency**: extra TGW hop adds < 1 ms typically; rarely matters for outbound-internet traffic.
- **Throughput**: NATGW caps ~45 Gbps per AZ. At 1 M sustained connections / second you'll need to shard the egress VPC across multiple NATGWs (per-AZ scaling) or multiple egress VPCs by traffic class.

### Alternatives
- **Per-VPC NAT**: simpler, no inspection chokepoint, but expensive at scale.
- **No NAT, only VPC endpoints**: for AWS-API-only workloads (Lambdas talking to DynamoDB/S3/SQS), endpoints eliminate egress entirely. Cheap and faster. See §15.
- **Egress via PrivateLink to a partner SaaS** (Datadog, Snowflake, Okta): zero internet, zero NAT for those flows. Pay per-endpoint-hour but win on security posture.
- **Proxy fleet** (Squid / managed Squid / Zscaler ZIA on EC2 behind NLB): preferred when you need URL-level inspection and modern firewalls feel heavy. At hyperscale (1 M+ RPS) a proxy fleet beats Network Firewall on throughput-per-dollar.

---

## 4. Service-to-Service with VPC PrivateLink

**The scenario**: a producer team runs a payments service in account A; consumers in accounts B–Z need it. Three options: VPC peering (CIDR coordination, transitive issues), TGW (full L3 mesh, flat security), PrivateLink (1-way exposure).

### PrivateLink pattern
- Producer puts service behind an NLB; creates an **Endpoint Service**.
- Consumers create **Interface Endpoints** in their own VPCs → ENI in their subnets with their own IPs from their own CIDR.
- Optional **Endpoint Service permissions** + **acceptance required** for explicit allow-listing of consumer accounts.
- Cross-region: PrivateLink is regional; for cross-region, replicate the producer or use Region-to-Region private connectivity via TGW peering.

### Why this is the default for SaaS-style internal services
- **No CIDR coordination**: producer and consumer can both run 10.0.0.0/16 with no NAT acrobatics.
- **One-way** by design: consumers can call producer but producer cannot reach into consumer VPC.
- **Native security group references** from consumer SGs to the endpoint ENI.
- **Service-discovery-friendly**: you get a DNS name that resolves locally to the ENI in each consumer.

### Trade-offs
- Cost: $0.01/endpoint-hour × AZs × consumers + $0.01/GB. With 1000 consumers in 3 AZs that's ~$22 K/month before traffic. Quantify vs TGW data charges.
- L4 only — full TCP flow, no headers-aware routing. For HTTP/gRPC routing across services, PrivateLink under the hood, **VPC Lattice** above it.
- NLB cap on per-target connection rate matters for chatty short-lived clients (use ALB-as-target via PrivateLink for HTTP).
- One Endpoint Service ↔ one NLB. Multiple services per producer = multiple NLB+ES pairs, ops overhead.

### Alternatives
- **TGW + L3 reachability**: simpler when bilateral, breaks down with cardinality and security review.
- **VPC Lattice**: HTTP/gRPC service mesh from AWS; cross-VPC, cross-account, IAM-aware auth, weighted routing. Use for east-west *service mesh* style traffic; PrivateLink for L4 / non-HTTP / strict tenant isolation.
- **App Mesh / Istio / Linkerd**: works inside a flat L3, sidecar-based. Stronger feature set (mTLS, retries, traffic shifting); more ops weight.
- **API Gateway (private)**: if you want REST/HTTP semantics + AWS-native auth + throttling. Higher per-request cost; lower operational burden.

---

## 5. Hybrid — Direct Connect with VPN Failover

**The scenario**: a regulated workload needs <10 ms RTT to on-prem mainframe, with five-nines uptime. Single DX circuit isn't enough; pure VPN won't meet latency.

### Pattern
- **Two DX connections** in two different DX locations (LAGs on each), each terminating on a separate physical router → eliminates single-router and single-fiber failures.
- Connect both DX into a **DX Gateway** → attach to TGW → all VPCs reachable transitively over DX.
- **Site-to-Site VPN** as backup, with BGP, **lower local-pref** so it's only used when DX BGP withdraws.
- ECMP across DX links for primary-primary; auto-failover to VPN within ~30 s on BGP timeout (tune timers).
- **MACsec** on the DX cross-connect for line-rate L2 encryption (regulated traffic).

### Trade-offs
- DX setup time: weeks to months. VPN-only is acceptable for non-latency-critical hybrid.
- DX is region-local. For multi-region, replicate the pattern per region or use a single region as the on-prem entry point + TGW peering inwards (saves circuits, adds 30–80 ms).
- BGP convergence: aggressive timers (3/9 s) save time on failover but introduce flap risk on real link blips. Match what your on-prem network is doing.

### Alternatives
- **AWS Cloud WAN with on-prem CNF (containerized network function)**: vendor-managed SD-WAN tunneled over your existing internet/MPLS. Newer, less mature.
- **SD-WAN appliance (Cisco Viptela / Aruba EdgeConnect / Velocloud) in a Transit VPC**: plays well with on-prem teams who already speak SD-WAN. More moving parts.
- **VPN-only**: only choose if cost/latency budget allows. AWS S2S VPN gives ~1.25 Gbps per tunnel; ECMP over 4–8 tunnels gets to 5–10 Gbps. Internet-path latency is the killer.

---

## 6. Multi-Region Active-Active

**The scenario**: a global product with users in NA, EU, APAC. Service must serve from the closest region; data plane is partitioned by user; cross-region traffic is 5–10 % of total but must be private and predictable.

### Pattern
- **One TGW per region**, peered pairwise in a partial mesh (or a hub region for fewer peers; regret if traffic doesn't actually flow through the hub).
- Inter-TGW routes are **explicit static** — never propagate region A's full routing into region B (blast radius).
- Per-region IPAM scope, with non-overlapping CIDRs (10.0.0.0/9 → NA, 10.128.0.0/9 → EU, etc.). **Plan this up front** — re-CIDR-ing later is the worst migration in AWS.
- **Route 53 Application Recovery Controller** (ARC) for active-active failover routing decisions.
- **Aurora Global / DynamoDB Global Tables** for data-plane region replication; the network layer just makes private inter-region paths possible.

### Trade-offs
- **TGW peering** is bandwidth-priced ($0.02/GB) — at PB/month inter-region, this can dominate the bill. Compare to direct VPC-VPC inter-region peering (free of TGW egress, still $0.02/GB AWS inter-region transfer).
- Latency budget: NA↔EU ~80 ms, NA↔APAC ~150 ms — cache strategically; never put a synchronous cross-region call on a hot path.
- **Cloud WAN** is purpose-built for this and can replace the TGW-peering mesh; consider for a clean-slate global build.

### Alternatives
- **Active-passive with regional failover**: one writer region, others read-only. Simpler, but you pay 100 % capacity in passive region while serving 0 %. Use when consistency wins over cost.
- **Pilot light** in secondary region (minimum infra, scale on failover): cheaper, slower RTO. Acceptable if data plane is the long pole anyway.

---

## 7. EKS at Scale — IP Exhaustion and Secondary CIDRs

**The scenario**: an EKS cluster with the AWS VPC CNI running in a /20 VPC. Every pod gets a real VPC IP. At ~3 000 pods you start hitting IP exhaustion. At ~10 000 pods you've burned three /20s. ENI density is the second wall.

### Solutions — in order of escalation
1. **Secondary CIDRs**: attach 100.64.0.0/16 (CGNAT range — non-routable on the public internet, perfect here) to your VPC; create dedicated pod subnets in that range. EKS CNI's `ENIConfig` puts pods on those subnets while nodes stay on the primary CIDR.
2. **Prefix delegation**: each ENI carries /28 prefixes (16 IPs) instead of single secondary IPs → ~16× more pods per node, fewer ENI calls (which were rate-limited). Default for most node types now.
3. **Custom networking + SNAT-to-node-IP** for outbound, so pods don't need routable IPs on shared services.
4. **Cilium (CNI replacement) with native VPC routing or overlay (VXLAN/Geneve)** — overlay decouples pod IPs from VPC IPs. Trade VPC-level visibility for unlimited IPs. Many large EKS shops do this.
5. **IPv6 pods (dual-stack EKS)**: native v6 from `2600:1f00::/40`-style VPC v6 prefix, no exhaustion, but requires v6-aware everything (RDS, partner APIs, telemetry). See §11.

### Trade-offs
- VPC CNI keeps L3 pod ↔ VPC reachability, security groups for pods, flow logs at pod granularity. Lose this with overlay CNI — gain capacity.
- Prefix delegation reduces EC2 API churn (massive win at scale) but allocates IPs in chunks; sparsely-loaded nodes waste IPs.
- 100.64/10 secondary range: works internally, but becomes opaque to many on-prem firewalls (CGNAT is special-cased everywhere). Test inter-region and hybrid first.

### Alternatives
- **Cluster-per-team** instead of one giant cluster — bypasses IP planning, multiplies control-plane cost.
- **Multi-cluster mesh (Karpenter + Cilium ClusterMesh / Istio multi-cluster)**: pod IPs reused across clusters; more network sophistication.

---

## 8. Centralized Inspection with Gateway Load Balancer

**The scenario**: compliance requires every flow between trust zones (prod ↔ shared, on-prem ↔ AWS, internet ↔ AWS) to traverse a third-party firewall. You don't want the firewall on every VM/ENI path.

### Pattern
- **GWLB** in an **inspection VPC**, fronting an Auto Scaling group of firewall appliances (Palo Alto VM-Series / Fortinet / Check Point / open-source Suricata) speaking GENEVE.
- **GWLB endpoint (GWLBe)** in workload VPCs (or in TGW-attached inspection VPC routing).
- Route tables steer traffic into GWLBe → GENEVE-encapsulates → fires it at the appliance fleet → returned via same flow → original destination.
- Symmetric routing critical: GENEVE encapsulation preserves the 5-tuple so the same appliance sees both directions of a flow.

### Trade-offs
- GENEVE adds 50–100 bytes; PMTU pain if you don't tune jumbo frames (or set DF=0 for v4 fragmentation).
- Single-vendor lock-in to whichever appliance sits behind the GWLB; horizontal scaling is by ASG.
- Appliance throughput per instance is ~5–15 Gbps; for 100 Gbps aggregate you scale wide and pay accordingly.

### Alternatives
- **AWS Network Firewall** (managed Suricata): no appliance fleet to run, cheaper at small scale, less feature-rich. Good for FQDN/TLS-SNI allowlisting; weaker for full IDS/IPS.
- **In-VPC inspection sidecars** (Falco / Cilium L7 policies): push to the workload, no central chokepoint. Higher per-host overhead, but no traffic inversion / hairpin cost.
- **Service-mesh-only** (Istio with mTLS + L7 policy): no L3 firewall at all. Works for HTTP, fails for arbitrary TCP. Common pattern in modern microservice orgs.

---

## 9. Microsegmentation and Zero-Trust on the VPC

**The scenario**: hundreds of services in one VPC. You want "service A can call service B, but not service C" without writing thousand-line NACLs.

### Pattern
- **Security Groups as primary segmentation** — they reference *other security groups* (not CIDRs). Treat each service's task-role SG as the unit; allow flows by `sg-source → sg-dest`.
- Use **Security Group Sharing across VPCs via TGW/PrivateLink** sparingly; SG-references across-VPC require correct TGW config and don't work everywhere — fall back to prefix lists.
- **Managed prefix lists** for set membership (e.g., "list of CI runners IPs", "list of corporate egress IPs"). Update the list, all SGs that reference it auto-update.
- **NACLs** as a coarse safety net only — stateless, ordered, painful. Use for "deny known-bad" / "deny RFC1918 leaks" at subnet boundary; don't use as primary segmentation.
- For L7 zero-trust within a VPC: **service mesh + SPIFFE/SPIRE identities**, or **VPC Lattice + IAM auth** as a managed shortcut.

### Trade-offs
- SG limits: 60 rules per SG (raise to 1000 with per-ENI rule limit decrease). Hit this at large fan-in services; refactor with prefix lists.
- SG references don't work cross-region. For multi-region, use prefix lists synced by automation.
- NACLs deceive — stateless rules + ephemeral source ports = surprise breakage. Document them as if they're explosives.

### Alternatives
- **Pure mesh-policy** (Istio AuthorizationPolicy / Cilium Network Policy): far more expressive (path, header, JWT claim), at the cost of running the mesh.
- **Aviatrix / Tetration-style overlay**: heavyweight, vendor-rich.
- **Network Firewall with stateful rules**: blunt instrument; useful for known-bad CIDR/FQDN.

---

## 10. DNS at Scale — Hybrid + Multi-Region

**The scenario**: an EC2 in account A, region us-east-1 needs to resolve a private record from account B's private hosted zone, while also resolving on-prem records, while not leaking your internal names to public DNS.

### Pattern
- **Route 53 Resolver** as the in-VPC stub resolver (always available at VPC + 2 / `169.254.169.253`).
- **Outbound resolver endpoint** + **resolver rules** for forwarding `*.corp.example.internal` to on-prem DNS.
- **Inbound resolver endpoint** for on-prem → AWS resolution.
- **Private hosted zones** RAM-shared into consumer accounts (or, equivalently, resolver rules — both are valid).
- **Profile** (resolver profile) for org-wide rule sets — share once, apply to many VPCs.

### Trade-offs
- Outbound resolver endpoint costs ~$0.125/h × ENIs × AZs. Cheap, but counts up.
- Private hosted zones associated to many VPCs: ~100 associations supported per zone (raise via support); for thousands of VPCs, prefer **resolver rules** (no association cap).
- Split-horizon DNS: same name resolves differently in AWS vs on-prem — necessary, but eternally confusing during an incident.

### Alternatives
- **Run your own resolver fleet** (CoreDNS / Unbound on EC2 ASG): full control, full ops burden. Pick if you have advanced policy needs (DoT/DoH, GeoDNS internal, query logging into your SIEM).
- **Active Directory DNS as authoritative for internal names**: common in Microsoft-heavy orgs; AWS resolves via outbound endpoint into AD DNS.

---

## 11. IPv6 / Dual-Stack Strategies

**The scenario**: you've consumed every RFC 1918 /8 you'll ever own; new workloads need real addresses; compliance now allows v6.

### Pattern
- **Dual-stack VPC**: VPC has a v4 CIDR + AWS-assigned `/56` (or BYOIP v6). Subnets get both v4 and v6 prefixes.
- **Egress-only IGW** for v6 outbound (no NAT needed; v6 is globally routable).
- **ALB / NLB dual-stack**: clients on either family work.
- **EKS IPv6 mode**: pods get globally-routable v6, no exhaustion.
- **PrivateLink IPv6**: now supported on most service categories.

### Trade-offs
- Many third parties don't speak v6 yet. Outbound-to-internet over v6 may need v4 fallback (NAT64/DNS64) — AWS doesn't natively offer NAT64; build it (Jool on EC2) or accept dual-stack endpoints everywhere.
- Cost: v6 traffic is *not* charged for NAT processing (no NAT). Real win on egress-heavy workloads if peers speak v6.
- Operational: every tool, log parser, security control must understand v6. Audit before flipping.

### Alternative
- **CGNAT (100.64/10) secondary CIDR**: classic dodge. Buys time, doesn't help peering with on-prem (CGNAT collisions are common).

---

## 12. Serverless and VPC

**The scenario**: a Lambda needs to talk to an RDS instance in a private subnet, and to S3, and to Secrets Manager. Naive setup: attach Lambda to VPC, watch cold starts, watch ENI quota, watch S3 egress costs.

### Pattern
- **Lambda + VPC**: Hyperplane ENIs (one per subnet+SG combination, shared across many invocations) → cold start cost is amortized, near-zero per-invocation overhead.
- **Avoid NAT** for Lambda → AWS APIs: instead use **interface endpoints** (Secrets Manager, KMS, Lambda, STS, etc.) and **gateway endpoints** (S3, DynamoDB). Free for gateway endpoints; cheap and faster than NAT for interface endpoints.
- **API Gateway + VPC link** for private API → backend.

### Trade-offs
- Each Hyperplane ENI consumes a private IP per AZ — at scale, plan IPs.
- Lambda VPC adds bootstrap weight; for very latency-sensitive cold paths consider keeping it out of VPC and accessing data plane via PrivateLink-fronted public endpoints (defeats some purists, works fine).
- Interface endpoints proliferate: $7/month per endpoint per AZ; hundreds of endpoints across hundreds of accounts add up. **Centralize**: put endpoints in the network account VPC, share via private hosted zones + resolver rules → workload accounts resolve service names to central ENIs. Saves 90 % on endpoint cost.

### Alternative
- **Out-of-VPC Lambda + IAM-only access** to AWS APIs: simpler, no ENI quota issue. Loses ability to talk to RDS/private resources.

---

## 13. Compliance — PCI/HIPAA Network Segmentation

**The scenario**: PCI scope must be a sealed network with logged ingress/egress, no shared admin paths, and audit-ready proofs of segmentation.

### Pattern
- **Dedicated PCI account + VPC**, never co-mingled with non-scope workloads.
- TGW with **dedicated route table** for PCI; explicit propagation only to PCI-shared services.
- All ingress through ALB/NLB in a DMZ subnet; all egress through a chokepoint with **Network Firewall stateful rules + flow logs to a separate compliance log account**.
- **VPC Reachability Analyzer** runs on schedule; output is the audit artifact proving "no path from non-PCI to PCI." Beats screenshotting RTs.
- **Resource Access Manager** revokes any TGW share that would let a non-PCI account attach.

### Trade-offs
- Duplication of NATGWs, endpoints, observability stacks per scope inflates cost. Worth it; auditors consider blast-radius isolation foundational.
- Operating two networks (PCI and non-PCI) doubles ops cognitive load — invest in IaC parity.

### Alternative
- **Single VPC with subnet-level segmentation**: cheaper, simpler, but auditors will frown. SGs / NACLs / labels all live in the same control plane → one IAM mistake is full collapse.

---

## 14. Disaster Recovery — Blue/Green at the Network Layer

**The scenario**: a regional outage requires shifting 30 % of traffic in 5 minutes to another region. Application can do it; can the network?

### Pattern
- **Route 53 Application Recovery Controller**: routing controls (open/close per region) backed by a multi-region cluster, deliberately decoupled from EC2/Route 53 control planes (which themselves can be impaired in a regional event).
- **Standby-light** in failover region: TGW + key endpoints + minimal data-plane primed; Auto Scaling scales out to replace failed region.
- **DNS-only failover**: Route 53 health-checks + weighted records. Simpler; suffers from client DNS caching (TTL ~60 s ideally; some clients ignore TTL).
- **Anycast via Global Accelerator**: clients hit the nearest healthy region by L4 anycast — failover within seconds, no DNS dependency.

### Trade-offs
- ARC + GA together is the most aggressive RTO posture; costs include $18/h GA accelerator + ARC fees.
- DNS-based failover is cheapest; RTO ~5–10 min realistic.

---

## 15. Cost Optimization — VPC Endpoint Strategy

**The scenario**: bill review surfaces $80 K/month NAT processing charges. 60 % of it is S3, DynamoDB, KMS, and Secrets Manager calls — i.e., AWS APIs reached via NAT-then-internet.

### Pattern
- **Gateway endpoints** for S3 + DynamoDB. Free, route-based, and bypass NAT entirely. Always-on, no-brainer.
- **Interface endpoints** for everything else: STS, KMS, Secrets Manager, ECR, Logs, etc. Keep traffic on the AWS backbone; bypass NAT.
- **Centralize interface endpoints** in network account, share via PHZ + resolver rules — see §12 trade-off note.
- **VPC endpoint policies** to lock down which buckets/keys are reachable.

### Trade-offs
- Each endpoint costs ~$0.01/h × AZs × accounts unless centralized.
- Centralization adds a TGW hop ($0.02/GB) — verify it's still net-cheaper than NAT processing ($0.045/GB) before celebrating.
- Some services don't support endpoints; check coverage.

---

## 16. Observability — Flow Logs, Mirror, Reachability Analyzer

The three tools every staff engineer should use as reflexively as `top`:

| Tool | Use it when | Cost / footprint |
|---|---|---|
| **VPC Flow Logs** | Always. Custom format with `pkt-srcaddr`, `pkt-dstaddr`, `traffic-path`, `flow-direction`. Sink to S3 + Athena, or to CloudWatch + Logs Insights. | ~$0.50/GB ingested into Logs; cheaper into S3 + Athena. Sample 10 % at hyperscale. |
| **Traffic Mirroring** | When packets matter — IDS/IPS deep inspection, debugging weird ALG behavior. | Capacity hit on the source ENI; pricey at sustained throughput. |
| **VPC Reachability Analyzer** | "Why can't A talk to B?" / "Prove that prod cannot reach dev." Static path analysis across SGs/NACLs/RTs/peering/TGW. | Per-analysis fee, cheap. Run on schedule for compliance proofs. |
| **Network Access Analyzer** | Org-wide "what is reachable from the internet?" Finds the one Frankenstein RT that backdoors prod. | Worth it for periodic audits. |
| **CloudWatch Internet Monitor** | User-perceived internet path issues by ASN/region. | Cheap. Useful for the "is it AWS or is it Comcast?" question. |

The key insight: **Flow Logs answer "what happened?", Reachability Analyzer answers "what *could* happen?"** Use both — incident vs audit.

---

## 17. IPAM and CIDR Planning

The boring slide that prevents the most expensive migrations.

- **Hierarchical IPAM scopes**: top-level pool 10.0.0.0/8 → region pools (10.0/12 NA, 10.16/12 EU, …) → environment pools → VPC allocations.
- **No /16 to a single VPC** as the default — too generous. Default to /20 or /22; grow via secondary CIDRs as needed.
- **Reserve 100.64/10 for EKS pod overflow** in every region, before you need it.
- **Reserve a /22 per region for "future shared services"** that you haven't thought of yet.
- **IPAM auto-import** existing resources to find every wild VPC and quietly account for it.
- For BYOIP (bringing public ranges to AWS), use IPAM's BYOIP pool — same UX as managed pools.

Trade-off: tight allocation saves you from collisions, costs you "ugh I need to grow this VPC" support tickets. Loose allocation feels good until two acquired companies both run 10.0.0.0/16.

---

## 18. Anti-Patterns to Recognize

- **Big shared prod VPC owned by no team**: every team adds ENIs, no team cleans up. ENI quota is finite; outage from running out is real.
- **Mesh of VPC peerings past 6 VPCs**: drown in route table maintenance. Move to TGW.
- **Per-VPC NAT in 200 VPCs**: idle cost dwarfs traffic cost. Centralize.
- **All-traffic NACL ALLOW**: stateful illusion. NACLs are stateless; ephemeral ports break things weirdly. Use SGs for primary policy.
- **Internet Gateway with public route in workload subnets**: "private" subnet that isn't. Audit RTs with Network Access Analyzer.
- **DNS for failover** with 24-hour TTLs.
- **No IPAM, no CIDR governance**: deferred pain that compounds with every M&A.
- **Cross-account peering with overlapping CIDRs and `127.0.0.1`-style NAT hacks**: works until it doesn't; collapses trust boundaries; replace with PrivateLink.
- **Single AZ NATGW** for cost reasons: AZ outage = no egress = degraded service. Per-AZ NATGW or accept the risk explicitly.
- **TGW propagation enabled everywhere**: defeats the segmentation reason you bought TGW. Disable propagation; static routes per segment.

---

## 19. Decision Cheat Sheet

| When you have to choose between… | Pick the first if… | Pick the second if… |
|---|---|---|
| **VPC Peering vs TGW** | 2–3 VPCs, bilateral, high BW | ≥ 5 VPCs, segmentation needed, hybrid attached |
| **TGW vs Cloud WAN** | Existing TGW estate, regional focus | Greenfield, global, want policy-as-code |
| **PrivateLink vs TGW for service exposure** | 1-way exposure, multi-tenant SaaS-like, no CIDR coordination | Bidirectional L3, low service count, low cardinality |
| **VPC Lattice vs Service Mesh (Istio)** | HTTP/gRPC, AWS-managed, IAM auth, no sidecars | Full feature set, multi-cloud, ops capacity to run mesh |
| **Centralized Egress vs Per-VPC NAT** | ≥ 5 VPCs or compliance requires chokepoint | Few VPCs, blast-radius isolation > cost |
| **Network Firewall vs GWLB+3rd party** | Modest needs, want managed | Existing PA/Fortinet contract, advanced features |
| **DX + VPN failover vs VPN-only** | Latency or BW SLOs, regulated | Burst hybrid, low traffic |
| **Active-Active Multi-Region vs Active-Passive** | Latency for global users, 5-nines | Cost-sensitive, RTO budget allows |
| **EKS VPC CNI vs Cilium overlay** | < 10 K pods, want native VPC visibility | > 10 K pods, IP scarcity, advanced policy |
| **IPv4 + secondary CIDR vs IPv6** | Internal-only, partners aren't v6 | Greenfield, partners speak v6, want to dodge NAT |
| **Distributed endpoints vs Centralized endpoints** | Few accounts, latency-critical | Many accounts, cost-optimized |

---

## 20. What Makes This Staff-Level

1. **Trade-offs quantified**, not just listed: NATGW idle cost, TGW per-GB processing, endpoint-hour math, where the break-even is.
2. **Explicit alternatives** for every pattern — Cloud WAN vs TGW, Lattice vs PrivateLink vs mesh, Network Firewall vs GWLB.
3. **Failure modes named**: TGW peering MTU, GENEVE PMTU, NACL stateless surprise, NATGW AZ outage, CGNAT on-prem collisions.
4. **Compliance + audit angle**: Reachability Analyzer as evidence, account-as-blast-radius, dedicated PCI VPCs.
5. **IPAM and CIDR planning** elevated to first-class — most network re-architectures are CIDR re-architectures.
6. **Cost reduction paths** with the actual line items: NAT processing → endpoints; per-account endpoints → centralized endpoints; per-VPC NAT → central egress; what each saves and what it costs.
7. **Anti-patterns and decision cheat sheet** — what to push back on in design review.