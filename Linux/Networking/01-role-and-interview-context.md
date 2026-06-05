# 01 · Role and Interview Context

## What the Linux Foundation actually employs SWEs to do

The Linux Foundation is **not the Linux kernel project's employer**. The kernel is built by ~5000 developers paid by ~500 companies (top contributors: Intel, Red Hat, Linaro, AMD, Google, IBM/Suse, Meta, Huawei, Arm). LF the org provides legal, IP, infrastructure, events, and project-management support.

Where LF *does* employ SWEs:

| Org unit | What it does | Networking depth needed |
|----------|-------------|------------------------|
| **LFX** | CI/CD, security, mentorship platforms for OSS | Light (web infra) |
| **CNCF** | Hosts Kubernetes, Envoy, eBPF Foundation, Cilium project | **Deep** (k8s netnow + CNI + service mesh) |
| **LF Networking** | FD.io VPP, ONAP, Tungsten Fabric, Magma, Anuket | **Very deep** (VPP/DPDK, SDN, telco) |
| **LF Edge** | Akraino, Edge orchestration | Medium (edge networking) |
| **LF AI & Data** | ML platforms | Light |
| **Hyperledger** | Blockchain platforms | Light |
| **OpenJS / OpenSSF** | Web / security foundations | Application security |

If the role title says "Linux Networking Engineer," confirm which sub-foundation. **FD.io VPP** = DPDK-based userspace dataplane, very different bar from CNCF (eBPF/Cilium). **Magma** = LTE/5G core, telco protocols. **ONAP** = orchestration, more Java/Python than dataplane.

If you can't disambiguate during recruiter screen, prep both. The kernel-side topics in this pack cover the common floor.

## The interview loop (typical Staff SWE; LF and adjacent companies)

1. **Recruiter screen** (30 min) — role fit, comp, sponsor (for LF: which sub-project).
2. **Hiring-manager technical screen** (45–60 min) — résumé deep dive, one shallow technical probe (e.g., "what happens when I `curl 1.1.1.1`").
3. **Coding round** (1–2 × 60 min) — usually one DSA-ish problem in C or Go, with a networking flavor: implement an LRU cache for connection state; parse a binary packet; implement a producer-consumer queue with `epoll`. Bring a kernel mindset (no GC pauses, alignment, error code handling).
4. **System design round** (60 min) — design an L4 load balancer; design a CDN edge; design a multi-region anycast service. Failure modes are the grading criterion.
5. **Networking / kernel deep dive** (60 min) — the technical anchor. Pick one: TCP internals; eBPF/XDP; netfilter/conntrack; netns/CNI; QoS/qdisc. Expect 3-layer-deep probing.
6. **Behavioral / bar-raiser** (45 min) — STAR stories. For LF specifically: "describe a time you contributed upstream"; "how do you balance vendor agendas in an OSS project"; "how do you handle a maintainer who blocks your patch."
7. **Sometimes a take-home** — a small write-up: read this kernel patch and explain what it does, or design a small XDP program with rationale.

## What "Staff" means (kernel-networking flavor)

A staff SWE in a kernel/networking role is expected to:

- **Own a subsystem.** Example: "I own the conntrack scaling work in our cilium fork"; "I own the L4 LB control plane"; "I own the kTLS integration in our edge proxy."
- **Be the architecture review for the team.** When a senior engineer proposes a feature, you should be the one to ask "what happens when conntrack overflows" or "have you considered XDP_REDIRECT instead of `tc-bpf egress`."
- **Communicate to two audiences.** Speak fluent network engineering to the BGP/NOC folks; speak fluent kernel internals to the kernel hackers. Bridge them.
- **Have produced upstream impact.** Patches to kernel, Cilium, Envoy, BPF tools. Not required for every role, but a strong signal.
- **Make build-vs-buy / build-vs-OSS decisions.** "Should we adopt Cilium 1.x or stick with IPVS?" is a staff-call question. They want to see *your decision framework*, not just an answer.

## Calibration: what they grade on

| Signal | Junior answer | Staff answer |
|--------|--------------|--------------|
| **"How would you debug high latency?"** | "Check the application logs." | "First ack/test: is it per-connection or per-host? `ss -tin` shows TCP retransmits; if those are high, check NIC drops via `ethtool -S`. If retransmits are low but tail latency is high, it's likely a queueing issue — check `tc -s qdisc show`, see if a single CPU is hot via `mpstat -P ALL 1`, and look at the IRQ distribution in `/proc/interrupts`. If queueing looks fine, the app is the next suspect — `bpftrace tcp:tcp_send_loss_probe`." |
| **"What's TCP slow start?"** | "TCP grows its window slowly when starting." | "Initial cwnd is 10 segments since RFC 6928 (Linux default `initcwnd`=10 since 2.6.39 backport from Google's 2010 paper). Slow start doubles cwnd per RTT until ssthresh. The interaction with hybrid slow start (`HYSTART_DELAY`) means cwnd freezes earlier on long-RTT paths to avoid overshooting. For high-BDP paths (think 100ms × 100Gbps) the slow-start exit is the dominant latency cost; that's why BBR and tuning `tcp_slow_start_after_idle=0` matter." |
| **"How does eBPF differ from iptables?"** | "It's a newer way to filter packets." | "iptables walks a linear chain per packet, recompiles to a flat list, lives in netfilter hook points. nftables compiles to a bytecode interpreted by a small VM. eBPF compiles to native code via a verifier and runs at the eBPF hook of your choice: XDP (pre-skb, line-rate), tc (post-skb-allocation, easier), socket (per-connection), cgroup (per-process). The win isn't features — it's *being able to write a custom dataplane* without forking the kernel." |
| **"Why does conntrack matter?"** | "It tracks connections." | "Conntrack is the in-kernel stateful firewall + NAT engine. Two things matter at staff level: (1) it has a *table* with a fixed max (`nf_conntrack_max`), and overflow causes hash collisions then drops — see `dmesg` for 'nf_conntrack: table full'. (2) Entry expiry is by timer (`nf_conntrack_tcp_timeout_established` default 5d), so long-lived idle connections eat slots. The cloud-scale workaround is conntrack-bypass via eBPF (Cilium does this) or a totally different forwarding plane (Katran uses XDP with its own flow table)." |

The shift from junior to staff is: **specifics, layers, scaling limits, and the named alternative.**

## Behavioral patterns specific to kernel/LF interviews

LF and kernel-shop interviews lean on OSS-culture questions. STAR stories should include:

- **A patch you upstreamed.** Even small ones count. Bonus if it was reviewed publicly.
- **A reviewer pushback you handled.** Maintainers reject patches harder than internal review. Show calm + iteration.
- **A conflict with a vendor.** OSS communities have competing employers; how you navigate that matters.
- **A scaling story.** Not "we used Redis" — a story where you hit a kernel limit (conntrack overflow, ephemeral port exhaustion, TIME_WAIT pile-up, NAPI budget) and fixed it.
- **An incident.** What did the on-call do, what telemetry was missing, what did you change.

If you have no kernel patches, that's OK — show OSS engagement (filing bugs with reproducers, contributing docs, helping in mailing lists/Slack). Empty OSS history hurts here.

## Stack you should be ready to discuss

Pick 1–2 to be *the experts on* and have a story for each:

- A **Kubernetes CNI plugin** (Calico, Cilium, Flannel, AWS VPC CNI, kube-router). Know one deeply.
- A **service mesh** (Istio/Envoy or Linkerd). Know how it injects, where the latency goes.
- A **load balancer** (Envoy, HAProxy, NGINX, IPVS, Maglev, Katran). One you'd deploy at 1M qps.
- An **observability stack** (Pixie, Cilium Hubble, Pyroscope, OpenTelemetry over eBPF). At least familiar.
- **eBPF tools** (bcc, bpftrace, libbpf). Be ready to write a 10-line bpftrace.

## Anti-patterns in your answers (avoid these)

- **"I'd just add more nodes."** Staff is supposed to find the bottleneck, not paper over it.
- **Vague references.** "Modern kernels handle this better" — name the version, the commit, the sysctl.
- **Refusing to commit.** "It depends" without a follow-up *decision framework* is a junior tic. Decision frameworks are the staff move.
- **No failure mode.** Every design should have at least one named failure mode. "What happens when conntrack overflows / when BGP flaps / when a POP loses fiber / when the leader dies."
- **Ignoring ops.** Staff is part of the on-call. "How would you roll this out without downtime; how would you roll back; what's the SLI; what's the alarm."

## Three preparation principles

1. **Trace one packet end-to-end every day.** RX path. TX path. With conntrack. Through netns. Through XDP. Through tc. Through TLS. You should be able to do it blind by interview day.
2. **Build a lab.** `ip netns add ns1 ns2`; `veth`-pair them; `bridge` them; add a `tc-bpf` program; `tcpdump` each side. Doing it once teaches you more than reading 5 papers.
3. **Read one RFC per week.** Start with 793 (TCP), 1122 (host requirements), 5681 (congestion control), 8985 (RACK), 7413 (TFO), 8684 (MPTCP), 9000 (QUIC), 2018 (SACK). The phrasing in interviews mirrors RFC language.

## Must-internalize

- LF the org ≠ Linux kernel project. Know which sub-foundation you're applying to.
- The bar is *staff*: layered diagnosis, named alternatives, failure-mode storytelling, tool fluency.
- Build a netns lab; trace a packet end-to-end; read the actual code (`net/ipv4/tcp_input.c` is famous for a reason).
- Have one *deep* OSS story (a patch, a debug session, an incident) ready in your back pocket.