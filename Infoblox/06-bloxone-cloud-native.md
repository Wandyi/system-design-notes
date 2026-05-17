# 6 · BloxOne / Universal DDI — Cloud-Native Architecture

BloxOne (now branded as **Universal DDI** with **NIOS-X** as the on-prem footprint) is Infoblox's cloud-native rewrite of the DDI stack. 
The pitch: same DDI capabilities, but managed from SaaS, on Kubernetes, with microservices, and operable from anywhere. 
Most new engineering happens here, so most staff-engineer interview signal will be on this architecture.

## 6.1 The architectural shift

| | NIOS Grid | BloxOne / Universal DDI |
|---|------------|--------------------------|
| Control plane | On-prem Grid Master | SaaS on AWS/GCP/Azure |
| Data plane | NIOS appliances at sites | NIOS-X / BloxOne hosts (containers/VMs) |
| Deployment unit | Appliance image | Container image |
| Scaling | Vertical (bigger appliance) | Horizontal (more pods, K8s) |
| Updates | Coordinated cluster upgrade | Rolling deploys per service |
| DNS engine | Custom (BIND-derived) | **CoreDNS** (CNCF) |
| DHCP engine | Custom + ISC dhcpd lineage | **Kea** (ISC) |
| IPAM engine | Infoblox proprietary | Infoblox proprietary, microservices |
| Customer-visible API | NIOS WAPI (REST) | REST + gRPC + GraphQL |
| Auth | Local users / AD / RADIUS | OAuth/OIDC, SAML, federated |
| Telemetry | Reporting server | Kafka → ClickHouse → dashboards |

## 6.2 High-level architecture

```
                    +-------------------------------------+
                    |        BloxOne SaaS (cloud)         |
                    |                                     |
                    |  Portal (React)                     |
                    |  API Gateway                        |
                    |  Auth (OIDC)                        |
                    |  IPAM Service                       |
                    |  DNS Config Service                 |
                    |  DHCP Config Service                |
                    |  Threat Intel Pipeline              |
                    |  Analytics / SOC Insights           |
                    |  Notification / Event Bus (Kafka)   |
                    +-----+--------------------+----------+
                          |                    |
                          | (mTLS gRPC)        | (mTLS gRPC)
                          |                    |
              +-----------v-----+    +---------v-----------+
              |  NIOS-X host    |    |  NIOS-X host        |
              |  (customer A,   |    |  (customer B,       |
              |   on-prem)      |    |   AWS VPC)          |
              |                 |    |                     |
              |  CoreDNS        |    |  CoreDNS            |
              |  Kea DHCP       |    |  Kea DHCP           |
              |  Config-agent   |    |  Config-agent       |
              |  Telemetry-fwd  |    |  Telemetry-fwd      |
              +-----------------+    +---------------------+
                       ^
                       |
                 +-----+-----+
                 |  End      |
                 |  devices  |
                 +-----------+
```

## 6.3 Microservices on the control plane

The SaaS side runs as Kubernetes-orchestrated microservices. Concrete services you can infer (from blog posts and job listings):

- **Auth service** — OIDC, tenant-aware, SAML for SSO.
- **IPAM service** — the system-of-record for blocks, subnets, addresses, hosts. Persists to a relational DB (likely Postgres) with caching layer.
- **DNS configuration service** — manages zones, records, views, policies. Persists, then pushes config to NIOS-X hosts.
- **DHCP configuration service** — manages pools, fixed addresses, options. Same push pattern.
- **Threat intel service** — hosts blocklists, dispatches them.
- **Reporting / analytics** — aggregates telemetry from hosts. Backed by ClickHouse (high cardinality, time-series friendly).
- **Cloud sync** — discovery agents for AWS/GCP/Azure VPCs.
- **Notification** — webhooks, SIEM forwarders, Slack/email.

Communication patterns:
- **gRPC** between services internally.
- **Kafka** for async fan-out (telemetry, audit events, threat intel updates).
- **REST/GraphQL** at the API gateway for external clients.

## 6.4 The NIOS-X data plane

A NIOS-X host (a.k.a. BloxOne Host) is what runs at the customer site. It's a lightweight VM or container (or both). Inside:

- **CoreDNS** — the DNS server. Configured to forward to other CoreDNS instances, to upstream resolvers, or to serve as authoritative for managed zones.
- **Kea** — the DHCP server. Loaded with the relevant Kea hook libraries (Infoblox plugins).
- **Config agent** — connects outbound to BloxOne over mTLS gRPC, pulls config, applies it locally.
- **Telemetry agent** — buffers and forwards DNS/DHCP events, metrics, and health to the cloud.
- **Local cache / state** — leases, recent queries, last-known config.

The host is **stateful** only enough to survive a brief disconnect from the cloud. If BloxOne is offline, the host serves the last known config until reconnection.

## 6.5 CoreDNS plugin chain (deep cut)

CoreDNS is a plugin-pipeline DNS server. The config (Corefile) is a list of plugins, and *order matters* — the chain is generated at build time from `plugin.cfg`.

```
. {
    bind 0.0.0.0
    errors
    log
    cache 30
    rewrite name regex (.*)\.lan$ $1.corp.example.com
    forward . 1.1.1.1 8.8.8.8 {
        prefer_udp
        max_concurrent 1000
    }
    prometheus :9153
}
```

Each plugin sees the request, may answer it, mutate it, or pass it to the next plugin. Plugins of interest in an Infoblox context:

- `kubernetes` — service discovery from K8s API.
- `forward` — resolver upstream.
- `cache` — in-memory cache, configurable TTL bounds.
- `rewrite` — rewrite name/class/type.
- `policy` (RPZ) — apply response policy (block, redirect).
- `dnssec` — sign on the fly.
- `view` — different chains for different client groups.

A typical custom Infoblox plugin would inject policy from the cloud control plane: block a query, log it as a threat event, return NXDOMAIN or a redirect IP (the **walled garden** pattern for Threat Defense).

## 6.6 Kea hook libraries

Kea is similarly plugin-driven. Hooks are dynamically-loaded shared libraries that fire at protocol stages: `pkt4_receive`, `lease4_select`, `lease4_renew`, etc. 
Infoblox writes hooks to:

- Authenticate the lease request against IPAM policy.
- Emit telemetry events to the cloud.
- Enforce option assignment per-tenant.
- Integrate DHCP-DNS updates.

The Kea **HA hook** (built-in) does the hot-standby / load-balancing logic. Kea **lease backends** (memfile, MySQL/Postgres, Cassandra) can swap based on customerscale.

## 6.7 Multi-tenancy

The SaaS side serves many customer tenants from one platform. Tenancy isolation:

- **Logical tenants** — every API/data record carries a `tenant_id`.
- **Row-level security** in the DB (Postgres RLS or app-layer guards).
- **Per-tenant Kafka topics** or partition keys.
- **Quotas and rate limits** to prevent a noisy neighbor from impacting others.
- **Audit log** with tenant context.
- **Crypto boundaries** — tenants don't share KMS keys.

The on-prem NIOS-X host belongs to exactly one tenant.

## 6.8 Deployment model and update cadence

- **Continuous delivery** on the cloud side — services deploy multiple times per week.
- **Canary tenants** — internal first, then early-adopter customers, then general availability.
- **NIOS-X host updates** are signed and pulled on a schedule customers control (similar to a Kubernetes node upgrade).
- **Backward compatibility** between cloud and host versions is mandatory — a customer might be running a NIOS-X that's months behind the cloud version.

The protocol contract between cloud and host is therefore **versioned** — gRPC proto contracts with explicit deprecation policy.

## 6.9 Telemetry and analytics pipeline

A central reason to be in the cloud: collect telemetry at fleet scale and run analytics.

```
NIOS-X host  --mTLS-->  Telemetry Ingress  -->  Kafka  -->  Stream processor (Flink/Spark)
                                                            -->  ClickHouse (hot store)
                                                            -->  S3 / data lake (cold)
                                                            -->  Threat-intel updater
```

Volume rough numbers:
- A typical mid-size customer: **millions of DNS queries/day**.
- A large customer: **billions/day**.
- Across the fleet: easily **10s of billions/day**.
- DHCP volume is much lower (orders of magnitude).

ClickHouse is the right call here — columnar, time-series friendly, high-cardinality. Same playbook Cloudflare uses.

Stream processing detects:
- DGA-shaped domain queries.
- DNS-tunneling patterns.
- Newly registered domains being queried (Zero-Day DNS).
- Bursty queries indicating malware C2.

These events feed back as **threat intel updates** pushed to every NIOS-X host in real time.

## 6.10 Anycast for DNS at the SaaS edge

For BloxOne's recursive DNS service (Threat Defense customers pointing their resolvers at Infoblox), Infoblox runs **anycast** PoPs. BGP advertises the same recursive-resolver IP from many locations; clients route to the nearest. The same pattern as Cloudflare's `1.1.1.1` or Google's `8.8.8.8`.

## 6.11 The migration challenge (huge interview topic)

Many large customers run NIOS today. Migrating them to BloxOne is hard:

- The on-prem GM holds 10–20 years of configuration.
- Their automation, ticketing systems, and runbooks call the WAPI.
- Some features in NIOS don't have direct BloxOne equivalents (or vice versa).
- Compliance audits reference specific NIOS behaviors.
- Customers prefer to migrate at their pace.

Strategy Infoblox uses (and any staff candidate should reason about):
- **Coexistence** — let NIOS Grid and BloxOne run side by side, with limited bi-directional sync.
- **API compatibility shim** — expose a WAPI-compatible facade over BloxOne so existing automation works unchanged.
- **Phased data migration** — start with one zone or one site, validate, expand.
- **Feature parity scoreboard** — track every NIOS feature and its BloxOne equivalent.
- **Customer-driven cutover** — they decide when, with rollback for N days.

Any system-design question on Infoblox's roadmap will involve some version of this.

## 6.12 Must-internalize

- BloxOne control plane = SaaS K8s; NIOS-X = on-prem data plane container; talk over mTLS gRPC.
- Data plane built on **CoreDNS** (plugin chain) + **Kea** (hook libraries).
- Telemetry pipeline: NIOS-X → Kafka → stream processor + ClickHouse → threat-intel feedback loop.
- Multi-tenancy = tenant_id everywhere + quotas + audit.
- Anycast for the recursive-DNS PoPs.
- Migration from NIOS → BloxOne is the unsolved-product problem; coexistence + WAPI-shim + phased.

---

## Sources

- [BloxOne DDI — Infoblox product page](https://www.infoblox.com/products/bloxone-ddi/)
- [Cloud Native DDI Solution: BloxOne DDI blog](https://blogs.infoblox.com/company/a-cloud-native-ddi-solution-bloxone-ddi/)
- [BloxOne Platform — Why Infoblox](https://www.infoblox.com/company/why-infoblox/bloxone-platform/)
- [CoreDNS — DNS and Service Discovery](https://coredns.io/)
- [CoreDNS Kubernetes plugin](https://coredns.io/plugins/kubernetes/)
- [ISC Kea DHCP](https://www.isc.org/kea/)
- [Cloudflare — analyzing 1M DNS QPS](https://blog.cloudflare.com/how-cloudflare-analyzes-1m-dns-queries-per-second/)
- [ClickHouse for time series](https://clickhouse.com/docs/use-cases/time-series)