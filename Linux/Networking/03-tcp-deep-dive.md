# 03 · TCP Deep Dive

TCP is the heaviest single topic in any kernel-networking interview. RFC 793 (1981) defined it; RFCs 1122, 5681, 2018, 7413, 8985, 8684, 9293 (the rewrite) own the modern story. The Linux implementation lives in `net/ipv4/tcp_*.c`.

## 3.1 The state machine (memorize)

```
              passive open: LISTEN
                  │  recv SYN
                  ▼
              SYN-RECEIVED
                  │  recv ACK
                  ▼
              ESTABLISHED ◄──┐ active open path: CLOSED→SYN-SENT→ESTABLISHED
                  │  send FIN
                  ▼
              FIN-WAIT-1
                  │  recv ACK
                  ▼
              FIN-WAIT-2
                  │  recv FIN
                  ▼
              TIME-WAIT (2 × MSL)
                  │  timer
                  ▼
              CLOSED

   simultaneous close path: FIN-WAIT-1 → CLOSING → TIME-WAIT
   passive close path:      CLOSE-WAIT → LAST-ACK → CLOSED
```

11 states. `ss -tan` shows them: `ESTAB`, `TIME-WAIT`, `CLOSE-WAIT`, `FIN-WAIT-1`, etc.

Interview probes:
- **Why TIME-WAIT?** To absorb duplicate packets from the closed connection so a new connection (same 4-tuple) doesn't accept them. Linux: 60s (`tcp_fin_timeout` — misleadingly named; sets TIME-WAIT, not FIN-WAIT).
- **Why CLOSE-WAIT pileup?** App got `recv()=0` (FIN) but didn't `close()` its fd. The socket sits in CLOSE-WAIT forever. **The most-asked debugging question.**
- **Why ESTAB without progress?** Often a half-open connection: peer crashed without RST, our side has no traffic to detect (keepalive disabled). `tcp_keepalive_time` default = 7200s.

## 3.2 Connection establishment — the three-way handshake

```
  Client                    Server (LISTEN)
    │ SYN seq=x ─────────────────►│
    │                              │ SYN cookies or syn_queue enqueue
    │◄──── SYN-ACK seq=y ack=x+1 ─│ SYN-RECV
    │ ACK ack=y+1 ────────────────►│
    │                              │ ESTABLISHED; deqd to accept_queue
   ESTABLISHED              accept() returns
```

Linux has two queues per listen socket:
- **SYN queue** (`tcp_max_syn_backlog`): half-open connections awaiting ACK.
- **Accept queue** (the `backlog` arg of `listen()` capped by `somaxconn`): fully-established connections awaiting `accept()`.

If accept queue is full → kernel drops the final ACK; client re-sends; server-side gets stuck. `ss -lnt` shows current backlog (`Recv-Q` for listen sockets).

### SYN cookies
With `tcp_syncookies=1`, when SYN queue overflows, the server constructs the SYN-ACK with a cryptographic cookie in `seq_y` derived from the 4-tuple + secret + time. No state stored. On final ACK, server reconstructs the connection from the cookie. Cost: forgoes SACK/timestamps options (mitigated in 4.x+ which preserve some via cookie extensions).

### TCP Fast Open (TFO, RFC 7413)
Lets the client send data with the SYN if it has a cookie from a prior connection.

- First connection: server returns a cookie via TCP option in SYN-ACK.
- Subsequent connections: client sends `SYN + cookie + data`; server accepts data immediately, opens connection.
- Saves 1 RTT for repeat clients.
- Sysctls: `tcp_fastopen=3` (both client + server), `tcp_fastopen_blackhole_timeout_sec`.
- Use case: HTTPS to known origins. Largely supplanted by QUIC 0-RTT but still relevant.

## 3.3 Data transfer — sliding window + cumulative ACK

```
seq:   1 2 3 4 5 6 7 8 9 ...
       └────┘
        snd_una        next byte un-ACKed
            └──────────┘
            snd_nxt    next byte to send
                       └────────────┘
                       snd_wnd      receiver's advertised window
```

`snd_una` (oldest unacked) and `snd_nxt` (next send seq) bound the in-flight bytes. `snd_wnd` is what the receiver said it can buffer.

Each ACK advances `snd_una`; "cumulative ACK" means an ACK for seq=N implies all seqs < N also received.

### SACK (RFC 2018)

If packets 1,2,4,5 arrived but 3 didn't, cumulative ACK is stuck at 3. SACK adds a TCP option: "I have [4..6)." Sender knows to resend only 3.

`tcp_sack=1` (default). DSACK (RFC 2883) reports duplicate ranges — helpful diagnostic.

Sender keeps a **scoreboard**: per-byte status (acked / SACKed / lost). `tcp_recovery` controls how losses are inferred.

### Window scaling (RFC 7323)

TCP option header limits window to 65535 bytes. Window-scale option says "multiply by 2^N." Both ends advertise; min is used.

Required for any BDP > 64KB. Sysctl: `tcp_window_scaling=1` (default).

### Timestamps (RFC 7323)

Each segment carries `(tsval, tsecr)` 32-bit timestamps. Uses:
- RTT measurement on retransmitted packets (without ambiguity).
- PAWS (Protection Against Wrapped Sequences) — old packets distinguished by old timestamp.
- TIME-WAIT recycling (`tcp_tw_recycle`, **removed in 4.10** because of NAT breakage).

Sysctl: `tcp_timestamps=1`. There's a 2019 attack (CVE-2019-11479) that abuses small advertised windows + low MSS to make the server burn CPU; mitigation is `tcp_min_snd_mss=536`.

## 3.4 Retransmission — RTO vs FRR vs RACK

Three loss-detection methods, evolution:

### RTO (Retransmission TimeOut)
The fallback. After SRTT + 4×RTTVAR with min `tcp_rto_min` (default 200ms), retransmit. RFC 6298.

Backoff: doubles on each retransmit. Min on Linux is 200ms (was 1s in some old textbooks).

### Fast Retransmit (RFC 5681)
Three duplicate ACKs for the same byte → retransmit immediately (don't wait for RTO).

### RACK (RFC 8985, "Recent ACKnowledgments")
Time-based, the modern Linux default since 4.4. If a segment was sent more than `reo_wnd` ago and a later segment was ACKed, assume lost.

Why RACK won: TCP suffers from "lost final segment" — fast retransmit needs 3 dup-ACKs which never come if the loss is the tail. RACK detects it from later activity.

Sysctl: `tcp_recovery=4` (RACK; bitmask of features).

### TLP (Tail Loss Probe, RFC 8985)
Sends one extra segment as a probe after the tail is sent, with a short timer (~2 × SRTT). If the probe is ACKed, no loss; if not, RACK triggers.

This is *the* reason modern Linux TCP feels much faster on lossy wireless paths than 2.6-era TCP.

## 3.5 Congestion control

Send rate = min(cwnd, rwnd). `cwnd` is grown by an algorithm of your choice:

### Reno (the textbook)
- Slow start: cwnd doubles per RTT.
- Loss → halve cwnd, then linear growth (additive increase, multiplicative decrease, AIMD).
- Almost no production stack uses Reno anymore.

### Cubic (default since 2.6.18)
Window grows as a cubic function of time since last loss:
```
W(t) = C(t - K)^3 + W_max
```
- C is aggressiveness; K is the time to grow back to W_max.
- Plateau near W_max ("max probing"), then concavely accelerates.
- Loss-driven (like Reno) but probes more aggressively on fast networks.

### BBR (Bottleneck Bandwidth and RTT)
Model-based. Maintains estimates of BtlBw (max bandwidth seen) and RTprop (min RTT seen). Targets `pacing_rate = BtlBw × pacing_gain`.

States: STARTUP, DRAIN, PROBE_BW, PROBE_RTT. Periodically probes for new bandwidth (1.25× send) and min RTT (cwnd drop to 4 segs for 200ms).

Why BBR is famous: it doesn't react to loss (treats loss as noise on the path). Wins on lossy paths (mobile, transcontinental). Co-existence problems with Cubic in shared buffers (BBR may grab more than its fair share).

Versions:
- v1 (2016) — unfair to Cubic, ramped too aggressively.
- v2 (2019) — added ECN response, calmer.
- v3 (2023) — better in-network coexistence.

Set via `tcp_congestion_control` sysctl. Per-route via `ip route ... congctl bbr`.

### Selection rubric

| Scenario | Algorithm |
|----------|-----------|
| Web traffic, mixed paths, default | Cubic |
| Bulk transfer over high-RTT (intercontinental, sat) | BBR |
| Datacenter fabric, ECN-enabled, microbursts | DCTCP |
| Long-running streaming (Netflix, YouTube) | BBR |
| Compatibility-first | Cubic (Reno's progeny is everywhere) |

### DCTCP
For datacenter fabrics. Uses ECN (Explicit Congestion Notification) instead of loss; reduces cwnd proportionally to the fraction of marked packets. Needs switch support (RED with ECN marking). Standard in modern DC.

## 3.6 Flow control — the receive window

Separate from cwnd. The receive socket says "I have B bytes of buffer free; advertise this as the window."

Linux auto-tunes:
- `tcp_rmem` (min, default, max) — controls per-socket buffer growth.
- `tcp_moderate_rcvbuf=1` — kernel grows window as throughput climbs.
- `tcp_adv_win_scale` — fraction of `sk_rcvbuf` advertised vs reserved for app processing.

Common mistake: app sets `SO_RCVBUF` explicitly. That **disables auto-tuning**. The kernel will not grow it past the user's value. Use the env-variable tuned defaults, don't set `SO_RCVBUF` unless you have a precise reason.

## 3.7 Nagle, delayed ACK, and the unholy alliance

**Nagle (RFC 896)**: don't send a small segment if there's unACKed data. Buffer until either (a) MSS-sized segment fills or (b) ACK arrives.

**Delayed ACK (RFC 1122)**: don't ACK immediately; wait up to 40ms or until 2 segments arrive, then ACK.

Together → **Nagle + Delayed ACK = 40ms stalls on small-message protocols** (RPC, SSH). Classic failure mode.

Fixes:
- `TCP_NODELAY` — disables Nagle. Most RPC libraries set this.
- `TCP_QUICKACK` — disables delayed ACK briefly (re-enabled by kernel sometimes).
- `TCP_CORK` — opposite of NODELAY: hold all data, send on uncork or full MSS.

Selection rubric:
- Streaming bulk → leave defaults.
- RPC / interactive → `TCP_NODELAY`.
- Batched / file-aligned → `TCP_CORK` then uncork after `sendfile()`.

## 3.8 Timestamps, RTO, and the RTT estimator

RTT measurement: `srtt = α×srtt + (1-α)×rtt_sample`. RTTVAR: `rttvar = β×rttvar + (1-β)|srtt - rtt|`. RTO = `srtt + 4×rttvar`, lower-bound `tcp_rto_min` (default 200ms).

Tip: low `tcp_rto_min` (down to 5ms) helps very low-RTT links (DC) but is risky on the public internet (spurious retransmits).

## 3.9 MPTCP (Multipath TCP, RFC 8684)

Allows one TCP connection to spread across multiple subflows (different paths / interfaces). In Linux mainline since 5.6.

Use cases:
- Mobile: keep a connection alive when switching WiFi ↔ LTE.
- Servers with multiple uplinks: aggregate bandwidth.
- Apple Siri / Maps: uses MPTCP since iOS 7.

Key gotchas:
- Middlebox compatibility: many NATs / DPI break MPTCP. Falls back to plain TCP.
- Scheduler: which subflow gets the next segment? Default = lowest-RTT.
- Receive buffer: needs *aggregate* BDP across subflows.

Sysctl: `net.mptcp.enabled=1`. `ip mptcp endpoint add` configures.

## 3.10 The Linux TCP code, by function

| Function | File | Role |
|----------|------|------|
| `tcp_sendmsg` | `net/ipv4/tcp.c` | userspace write entry; builds skb, calls write_xmit |
| `tcp_write_xmit` | `net/ipv4/tcp_output.c` | decides whether to send (cwnd, Nagle, pacing) |
| `tcp_transmit_skb` | `net/ipv4/tcp_output.c` | builds TCP header, calls IP |
| `tcp_v4_rcv` | `net/ipv4/tcp_ipv4.c` | RX entry; socket lookup; calls rcv_established or new conn |
| `tcp_rcv_established` | `net/ipv4/tcp_input.c` | fast path for ESTABLISHED |
| `tcp_ack` | `net/ipv4/tcp_input.c` | processes incoming ACK |
| `tcp_fastretrans_alert` | `net/ipv4/tcp_input.c` | retransmit logic |
| `tcp_retransmit_timer` | `net/ipv4/tcp_timer.c` | RTO handler |

These are the names interviewers expect you to know.

## 3.11 Corner cases (the high-leverage list)

| Symptom | Likely cause | Tool |
|---------|-------------|------|
| Many `TIME-WAIT` on outbound | Short connections; tune `tw_reuse`, prefer keepalive | `ss -tan state time-wait` |
| Many `CLOSE-WAIT` | App not closing FDs | `ss -tn state close-wait`, `lsof` |
| Slow connect after rolling restart | Backlog full; tune `somaxconn` | `ss -lnt` |
| Stalled large transfer | Window not opening; tune `tcp_rmem.max` to BDP | `ss -tin` (window size) |
| Spurious retransmits | Reordering or low `tcp_rto_min` | `tcpdump`, `nstat -az TcpExt*` |
| TCP "stuck" at MSS=536 | PMTUD blackhole; `tcp_mtu_probing=1` | `tracepath` |
| HOL blocking single connection | TCP intrinsic; consider HTTP/2 → HTTP/3 (QUIC) | n/a |
| Per-CPU softirq pegged | Single-flow GRO; enable RPS/RFS | `mpstat -P ALL 1`, `/proc/interrupts` |
| Window stuck at 0 | Receive buffer full (app slow); `tcp_wmem` shows backpressure | `ss -tin` |

## 3.12 Alternative approaches (the "3 ways" reflex)

| Need | TCP path | Alternative 1 | Alternative 2 |
|------|----------|--------------|--------------|
| Reliable streaming | TCP | QUIC (UDP+app reliability) | SCTP (multistream native) |
| Connection migration | n/a (rebind) | QUIC connection ID | MPTCP |
| Sub-RTT first byte | TFO | QUIC 0-RTT | (none) |
| Multi-path | MPTCP | QUIC migration | App-layer (BGP anycast routes to nearest) |
| HOL avoidance | TCP can't | QUIC streams | HTTP/2 + many TCP connections (parallel HOL but per-stream) |
| Encryption in path | TLS over TCP | QUIC (integrated) | kTLS (offloaded TLS over TCP) |

## Must-internalize

- 11-state machine; CLOSE-WAIT pileup is the #1 prod question.
- TIME-WAIT is by design (60s on Linux); use SO_REUSEADDR / connection reuse.
- Cumulative ACK + SACK + RACK = modern loss recovery.
- Cubic = default; BBR = bulk + high-RTT; DCTCP = datacenter.
- Nagle + delayed ACK = 40ms stalls; `TCP_NODELAY` for RPC.
- BDP > tcp_rmem.max → TCP cannot fill the link.
- `tcp_sendmsg` → `tcp_write_xmit` → `tcp_transmit_skb`: the TX function chain to memorize.