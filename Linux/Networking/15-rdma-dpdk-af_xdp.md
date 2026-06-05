# 15 · RDMA, DPDK, and AF_XDP — Kernel Bypass

The "we need more than the kernel can give" tier. Three different mechanisms, three different niches. Staff candidates should be able to pick the right one and defend the choice.

## 15.1 The kernel-bypass spectrum

```
                          kernel involvement
   highest ◄─────────────────────────────────────────► lowest
   sockets   AF_PACKET   AF_XDP    XDP   DPDK    RDMA
              + mmap   (semi-bypass)  (libs)  (no kernel)  (no CPU)
```

- **sockets / TCP** — full stack; max safety; ~1Mpps per CPU.
- **AF_PACKET + mmap** — userspace raw packet via copy; ~3-5Mpps.
- **AF_XDP** — userspace zero-copy via XDP redirect; ~20-30Mpps.
- **DPDK** — userspace owns the NIC; ~50Mpps per CPU.
- **RDMA** — NIC moves memory; bypasses CPU entirely for data.

Decision factors: pps target, latency target, complexity tolerance, operational maturity.

## 15.2 AF_XDP — the half-way house

Already touched in `07` and `11`. The recap:

A userspace ring is allocated; pre-registered with the kernel and mapped to userspace. An XDP program (`XDP_REDIRECT`) puts packets into this ring; userspace consumes via memory-mapped reads.

```
NIC ──► driver RX ──► XDP program ──► XDP_REDIRECT ──► AF_XDP ring (userspace mmap)
```

Zero-copy mode (default on modern NICs): NIC DMAs straight into userspace pages (frame pool registered with kernel).

Pros vs DPDK:
- Kernel manages buffers; memory accounting works.
- One NIC can serve both kernel sockets and AF_XDP simultaneously.
- Less invasive: drop into existing systems without DPDK's all-in adoption.
- Easier ops (kernel still owns lifecycle).

Cons vs DPDK:
- ~20-30Mpps vs DPDK's 50Mpps (kernel overhead in driver path).
- Less mature ecosystem.

Used by: Cilium for some hot paths, Suricata at line rate, custom HFT shops.

## 15.3 DPDK — userspace owns the NIC

Intel-led project; LF Networking hosts. Architecture:

```
   userspace app
        │
        ▼
   DPDK Poll Mode Driver (PMD)
        │ memory-mapped NIC registers, hugepages buffers
        ▼
   NIC ── DMA to hugepages, no IRQs
```

- **Hugepages** (2MB or 1GB) reduce TLB pressure.
- **PMD** (poll-mode driver) polls the NIC RX ring; no IRQs.
- **NUMA-aware buffer allocation** keeps everything local.
- **Vector instructions** (SSE/AVX) for batch processing.

Frameworks built on DPDK:
- **FD.io VPP** — full L2/L3/L4 dataplane (firewall, routing, NAT) at line rate.
- **OVS-DPDK** — Open vSwitch with DPDK.
- **Pktgen** — packet generator.
- **mTCP** — userspace TCP stack on DPDK.
- **F-Stack** — Tencent's userspace TCP.

Performance:
- 50-100Mpps per CPU on a modern Xeon.
- Sub-µs L3 forwarding latency.
- Saturates 100Gbps with ~4 CPUs.

Cost:
- CPU permanently pinned (busy poll).
- Power consumption stays high even when idle.
- Hard to share NIC with kernel sockets.
- Operational complexity: hugepages, IOMMU configs, driver binding (DPDK-bound NICs aren't usable by kernel).
- Cannot do TCP termination without porting a TCP stack into userspace.

When to use DPDK: high-pps middleboxes (NFV, telco), Tor exit nodes, custom L4 LBs (some VPP-based), high-rate measurement.

When NOT: regular servers (use kernel + eBPF); apps that need TCP (unless you're committed to mTCP/F-Stack).

## 15.4 VPP (Vector Packet Processing)

LF Networking flagship. Open-source dataplane on top of DPDK (or netmap, AF_XDP). VPP "vectorizes": packets travel as a batch through a graph of nodes (parse-ethernet → ip4-lookup → ip4-rewrite → output). Each node is hot, branch-predictor-friendly.

Used by: NFV deployments, some Tier-1 ISPs, some cloud providers.

Useful interview reference: "We use VPP for our 5G UPF (User Plane Function)" — Magma project, NFV vendors.

## 15.5 RDMA — Remote Direct Memory Access

Different category: not just kernel bypass; **CPU bypass** for data movement.

The NIC reads/writes peer memory directly. CPU posts a "verb" (operation), the NIC handles transfer.

Three flavors:
- **InfiniBand (IB)** — Mellanox's native fabric; lossless; specialized switches.
- **RoCE (RDMA over Converged Ethernet)** — IB protocol over Ethernet; needs DCB / PFC (lossless ethernet) on switches.
- **iWARP** — RDMA over TCP; works on any IP network; slower (kernel + TCP not fully bypassed in all impls).

Verbs:
- **SEND/RECV** — like sockets, but bypassing kernel.
- **READ** — read peer memory by address+key (RKEY).
- **WRITE** — write peer memory by address+key.
- **ATOMICS** — compare-and-swap, fetch-and-add on remote memory.

Use cases:
- **HPC / supercomputers** — MPI runs on RDMA.
- **Distributed storage** — Ceph RBD-RDMA, NVMe-oF.
- **In-memory databases** — Apache Spark, RDMA-Memcached.
- **GPU training fabrics** — NVIDIA NCCL uses RDMA + GPUDirect.
- **AI clusters** — Meta's Grand Teton, Microsoft's Azure HPC, Google's TPU pods.

### RoCE congestion control

The big problem: Ethernet drops; IB doesn't. RoCE assumes lossless ethernet. Achieved via:
- **PFC** (Priority Flow Control, IEEE 802.1Qbb) — per-class pause frame.
- **ECN** + DCQCN (Datacenter QCN, Mellanox proprietary CC algo) for fine-grained backoff.

PFC failure modes:
- **PFC storm** / pause propagation deadlock.
- **Head-of-line blocking** on shared priority queues.
- **Microbursts** that PFC can't keep up with.

Datacenter networking PhD topic; staff candidate must know this exists and roughly how it works.

### RDMA performance
- Per-NIC: ~200ns RTT (small message), saturate 200Gbps with single core.
- vs TCP: ~10× lower latency, ~3× more throughput per CPU.
- vs DPDK: similar throughput, but CPU bypass is the unique win.

### Linux RDMA stack
- `rdma-core` userspace; kernel modules per HCA.
- `libibverbs` is the standard API.
- `ucma` (RDMA Connection Manager) for connection setup.
- `nvme-rdma` is NVMe-over-fabric using RDMA.

## 15.6 NVMe-over-Fabric (NVMe-oF)

Storage protocol that uses RDMA (or TCP) for remote disk access. SCSI-equivalent over the network.

NVMe-oF/RDMA: ~µs latency to remote disk; competitive with local NVMe.
NVMe-oF/TCP: ~10× latency but works on any network.

Linux kernel: both supported (5.x+).

Use case: shared-nothing storage that looks like local; AWS's NVMe-over-EBS, Microsoft's Azure NVMe.

## 15.7 GPUDirect, GPUDirect RDMA

NVIDIA's: GPU buffer can be the source/dest of an RDMA op. Bypass the CPU and even system memory.

Used in: AI training clusters; collective ops between GPUs across hosts.

## 15.8 Decision rubric — when to bypass the kernel

| Need | Path |
|------|------|
| <10 Mpps, full TCP | kernel + tuning |
| <30 Mpps, custom L2/L3 | XDP + eBPF |
| <30 Mpps, custom + userspace logic | AF_XDP |
| >30 Mpps, dataplane, willing to ditch kernel net | DPDK / VPP |
| Ultra-low-latency, CPU-light | RDMA |
| Disaggregated storage | NVMe-oF (RDMA or TCP) |
| GPU collective | GPUDirect RDMA |
| Edge proxy with TLS | kTLS + kernel sockets |
| 100Gbps file serving | kTLS sendfile (kernel) |

## 15.9 Common interview probes

- **"DPDK vs AF_XDP, when do I pick which?"** AF_XDP if you want kernel co-existence and simpler ops; DPDK for max performance and standalone middleboxes.
- **"RoCE vs iWARP?"** RoCE if you control the switching fabric (DCB/PFC); iWARP for cross-DC over standard IP.
- **"What breaks at 100Mpps?"** sk_buff allocation; per-packet locks; cache line bouncing; NIC IRQ rate. DPDK avoids all of these.
- **"How does RDMA differ from kernel-bypass?"** RDMA bypasses CPU for data; the NIC reads/writes peer memory. Kernel bypass (DPDK) only bypasses the kernel; CPU still does the work.
- **"PFC pitfalls?"** Pause propagation, deadlock, head-of-line. Many DCs have moved to DCQCN or ECN-based.

## 15.10 Corner cases

- **DPDK + IOMMU.** Either passthrough (vfio) or use `--iova-mode=pa` for hugepage physaddr; mistakes here cause silent corruption.
- **AF_XDP frame leak.** If userspace doesn't return frames to fill ring, NIC can't RX → drops.
- **RoCE on lossy fabric.** Without PFC + careful tuning, RoCE pukes.
- **NUMA mismatch.** DPDK app on socket 1, NIC on socket 0 → 30% perf loss.
- **Power management.** PMD polling defeats C-states; box stays hot. Disable C-states or use `IDLE_PMD` mode.
- **VPP/DPDK ops surface.** Driver binding (`dpdk-devbind.py`) is fragile across kernel updates.
- **GPUDirect RDMA + PCIe Switch.** Need a PCIe topology that supports peer-to-peer; some boards don't.

## 15.11 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Line-rate L4 LB | XDP (Katran) | DPDK + VPP | hardware ASIC |
| HPC collective | MPI over RDMA | MPI over TCP | NCCL GPUDirect |
| 100Gbps file server | kTLS sendfile | userspace + DPDK | RDMA + NVMe-oF |
| Cross-DC L3 forwarder | kernel + ECMP | DPDK + VPP | hardware router |
| Service mesh sidecar | Envoy (kernel TCP) | Cilium (eBPF) | DPDK-based mesh |

## 15.12 Production stories

- **Microsoft Azure** — uses RDMA (RoCEv2) across their fabric for storage and HPC; massive PFC operational war stories.
- **Meta AI fabric** — RoCE across millions of GPUs; custom CC ("Falcon") layered on.
- **Google** — has own RDMA-like protocol on TPU pods (ICI).
- **Cloudflare** — XDP and AF_XDP heavily; not big on DPDK (kernel + eBPF is enough).
- **Cisco / Juniper / Arista** — VPP for some NFV products.

## 15.13 The 60-second pitch

> "Kernel bypass exists because the per-packet cost of sk_buff allocation, lock contention, and IRQ overhead caps the kernel at ~5-10 Mpps per CPU. Three escalating tiers exist: AF_XDP (kernel-managed bypass via XDP redirect, ~20-30 Mpps), DPDK (userspace owns the NIC, no kernel involvement, ~50 Mpps), and RDMA (the NIC reads/writes peer memory directly, bypassing not just the kernel but the CPU). AF_XDP is the lightest-touch — drop it in alongside kernel sockets. DPDK requires you to commit a CPU and the NIC. RDMA requires lossless ethernet (RoCE) or a tolerant switch fabric (iWARP), and is the only choice for sub-µs latency or for HPC/AI fabrics. The selection is by per-packet cost ceiling and ops appetite — most general-purpose servers are happy at kernel + eBPF; only when you need pure middlebox throughput or HPC latency do you escape."

## Must-internalize

- AF_XDP: kernel-managed userspace zero-copy via XDP redirect. 20-30 Mpps.
- DPDK: PMD owns NIC; hugepages; 50 Mpps; can't share NIC with kernel.
- RDMA: NIC reads/writes peer memory; CPU bypass; sub-µs latency.
- RoCE = RDMA over Ethernet, needs PFC/DCQCN; iWARP = RDMA over TCP, more lenient.
- VPP = vectorized dataplane on DPDK; LF Networking flagship.
- NVMe-oF (RDMA or TCP) = disaggregated storage protocol.
- Decision: TCP for sockets, eBPF for kernel-side custom, AF_XDP/DPDK/RDMA for the rest.