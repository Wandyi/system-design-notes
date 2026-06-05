# 14 · TLS, kTLS, and QUIC

The crypto layer that touches every modern packet. Kernel has long been *almost-aware* of TLS via kTLS (in-kernel TLS) — and the future is more of this. For QUIC, TLS is integrated into the transport; kernel-side QUIC offload is on the horizon.

## 14.1 TLS handshake — refresher

TLS 1.3 (RFC 8446), the modern version. 1-RTT (vs 1.2's 2-RTT) handshake; 0-RTT with PSK.

```
Client                                Server
ClientHello → key_share, ciphers
                                      ServerHello + key_share
                                      Extensions {EncryptedExtensions}
                                      Certificate
                                      CertificateVerify
                                      Finished
Finished, application data →
                                ← application data
```

Key derivation: ECDHE (or PSK), HKDF-Expand-Label to derive traffic keys.

Symmetric cipher: AEAD (AES-GCM, ChaCha20-Poly1305) — Authenticated Encryption with Associated Data.

## 14.2 The cost of TLS

Per-record overhead:
- 5-byte TLS record header.
- 16-byte AEAD tag.
- IV (mostly implicit in 1.3).

Computational cost:
- Asymmetric (handshake): ~1ms with ECDHE P-256 on modern CPU. Cost only on connection setup.
- Symmetric (record): AES-GCM with AES-NI: ~5GB/s per core. ChaCha20-Poly1305: ~2GB/s without HW.

For a high-throughput server: handshake CPU is amortized over connection lifetime; AEAD CPU is the long-term cost.

## 14.3 kTLS — TLS in the kernel

`KTLS` (4.13+): once the handshake completes in userspace (with OpenSSL or similar), the **symmetric keys are pushed to the kernel** via `setsockopt(SOL_TLS, TLS_TX, ...)` and `setsockopt(SOL_TLS, TLS_RX, ...)`.

After that:
- `write()`/`send()` on the TLS socket → kernel encrypts → ships out as TCP.
- `read()`/`recv()` decrypts → returns plaintext.
- `sendfile()` works with TLS! Kernel reads from file, encrypts, ships.

This is the **headline feature**: TLS no longer breaks `sendfile()`.

```
   userspace (OpenSSL)
            │
            │ handshake (control)
            ▼
   kernel TCP socket  ◄── push keys via SOL_TLS
            │
            ▼
   kernel encrypts/decrypts on write/read
```

### kTLS TX (4.13+)
First in mainline. Encrypt on send.

### kTLS RX (4.17+)
Decrypt on receive. Trickier (must verify auth before delivering plaintext); some restrictions on record boundaries.

### kTLS HW offload (5.0+)
NIC does the AEAD encryption. Mellanox ConnectX-6 Dx, Marvell, Intel E810 + tlsoffload.

`ethtool -k eth0 | grep tls`:
- `tls-hw-record-creation`: NIC builds records.
- `tls-hw-tx-offload`: HW AEAD encrypt.
- `tls-hw-rx-offload`: HW AEAD decrypt.

Netflix's famous 800Gbps-from-one-4U-box (2021 LPC talk) uses NIC kTLS offload.

## 14.4 OpenSSL + kTLS integration

OpenSSL 3.x has `SSL_CTX_set_options(ctx, SSL_OP_ENABLE_KTLS)` (or auto-detect). After the handshake, OpenSSL calls `setsockopt` to push keys. Subsequent SSL_read/SSL_write degrade to plain read/write on the kTLS-enabled socket.

```c
SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());
SSL_CTX_set_options(ctx, SSL_OP_ENABLE_KTLS);
SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, fd);
SSL_accept(ssl);  // handshake + push keys to kernel

// from here: plain write() works as encrypted send
write(fd, "GET /\r\n\r\n", 9);
```

The NGINX-on-FreeBSD-with-kTLS Netflix benchmark: 800Gbps with TLS on a single box, ~50% of CPU.

## 14.5 TLS 1.3 0-RTT

Pre-shared key (resumption) enables sending application data with the ClientHello (no RTT before first byte).

Risk: replay. 0-RTT data is replayable; an attacker can replay the first request. Server must either:
- Limit 0-RTT to idempotent requests (HTTP GET).
- Reject 0-RTT for state-changing operations.

`tcp_fastopen` is the TCP-level analog. Together, TFO + 0-RTT = data + connection setup in one packet.

## 14.6 Session tickets and resumption

TLS 1.2: session ID (server-side) or session ticket (client-stored). Resumption skips full handshake.
TLS 1.3: session tickets only; one-time-use to prevent replay.

Server-side: keep tickets short-lived; rotate the encryption key periodically.

For load-balanced TLS termination: tickets must be shareable across LB nodes. Either share key material (centralized rotation) or use sticky LB.

## 14.7 SNI and ESNI/ECH

**SNI** (Server Name Indication): hostname sent in plaintext in ClientHello. Used by LBs to route to the right backend.

Problem: privacy — hostname visible to passive observers (ISPs, censors).

**ECH** (Encrypted Client Hello, draft RFC): SNI encrypted with a server-published HPKE key. The "outer" ClientHello uses a generic public name; the inner has the real one.

Cloudflare, Mozilla, Apple ship ECH. Adoption: client-side rolling out; server-side mostly Cloudflare and friends.

## 14.8 OCSP stapling — perf optimization

Pre-stapling: client requests OCSP from CA to check certificate revocation → adds RTT.

OCSP stapling: server includes the signed OCSP response in TLS handshake. Client doesn't need to ask CA.

`ssl_stapling on;` in NGINX. CDNs always staple.

## 14.9 Mutual TLS (mTLS)

Both ends authenticate via certs. Service mesh sidecar pattern (Istio, Linkerd, Cilium service mesh).

Cost: extra CPU per handshake (verify client cert chain). Modern hardware shrugs.

Identity-per-pod: certs issued by SPIFFE/SPIRE-style identity systems, short-lived (~1h), auto-rotated.

## 14.10 QUIC — TLS integrated

QUIC (RFC 9000) builds TLS 1.3 directly into the transport. No separate "TCP+TLS" layering.

- Crypto handshake is the first thing in a QUIC connection.
- Each packet is authenticated (header + payload).
- Connection IDs are encrypted; even the 5-tuple isn't observable.

This is more secure than TCP+TLS at the cost of more complexity in the transport.

In-kernel QUIC support is experimental (lwn.net/Articles/958571/, 2023). Expected analog to kTLS: an `AF_QUIC` socket that does in-kernel framing + crypto.

Adoption (2026): most Google traffic, ~50% Facebook, growing on Apple. HTTP/3 is the visible layer.

## 14.11 Performance numbers

| Cipher / mode | Throughput per core |
|--------------|---------------------|
| AES-128-GCM (AES-NI) | 5-8 GB/s |
| AES-256-GCM (AES-NI) | 4-6 GB/s |
| ChaCha20-Poly1305 (no HW) | 2-3 GB/s |
| ChaCha20-Poly1305 (AVX-512) | 4-5 GB/s |
| Userspace TLS over TCP | bottlenecked by userspace ↔ kernel copy |
| kTLS sendfile | bound by AES-GCM throughput (no copy) |
| NIC TLS offload | bound by NIC TLS engine; saves ~30% CPU |

Handshakes:
- ECDHE P-256: ~1ms / core / handshake.
- A 100K-rps server with new connections needs ~100 cores just for handshakes.
- Solutions: session resumption (avoid full handshake), TLS session ticket rotation, OCSP stapling.

## 14.12 Common interview probes

- **"What does kTLS gain you?"** sendfile() with TLS, hardware offload, fewer userspace round-trips.
- **"How does TLS 1.3 0-RTT work, and what's the risk?"** Replay attacks; idempotent-only data.
- **"How does mTLS differ from TLS?"** Server requires client cert; both verify each other.
- **"How is QUIC different from TCP+TLS?"** Integrated crypto, encrypted headers, multi-stream, IP migration.
- **"How would you scale TLS to 1Tbps?"** kTLS + sendfile + NIC offload + session resumption + multi-core (one TLS context per core if possible). Netflix's 800Gbps box is the reference.

## 14.13 Corner cases

- **kTLS RX boundary mismatch.** kTLS expects record-aligned reads; partial records buffered. Don't mix BIO_read with kTLS read.
- **kTLS + sendmsg gather.** Sometimes inefficient compared to writev directly; depends on kernel version.
- **OpenSSL bypass for kTLS.** Apps that use SSL_read directly may not benefit; need plain read() on the underlying fd.
- **NIC kTLS offload races.** If a NIC driver bugs out, falls back to software kTLS; quiet perf degradation.
- **Session tickets and load balancers.** If LB doesn't share tickets between backends, resumption fails → full handshake; visible as latency spike.
- **HRR (HelloRetryRequest).** TLS 1.3 retries the handshake if the server's preferred key group differs from client's offered. Adds an RTT. Tune cipher preferences to avoid.
- **Forward secrecy concern.** Long-lived ticket keys break PFS; rotate keys hourly/daily.
- **Linux kTLS supports only TLS 1.2 and TLS 1.3 with select ciphers** (AES-GCM, ChaCha20-Poly1305). Legacy ciphers stay in userspace.

## 14.14 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Encrypt at line rate | userspace OpenSSL | kTLS | NIC TLS offload |
| Faster handshake | OpenSSL default | session resumption | TFO + 0-RTT |
| Privacy of SNI | TLS 1.2 with SNI | ECH | DoH (resolver-side hide) |
| Identity per pod | static certs | SPIRE/SPIFFE mTLS | service mesh sidecar mTLS |
| Encrypt UDP traffic | DTLS | QUIC | WireGuard (UDP+ChaCha20) |
| Avoid kernel TLS | userspace + raw TCP | userspace + io_uring | DPDK + libssl |

## 14.15 WireGuard — different beast

In-mainline since 5.6. Simpler than IPsec; uses ChaCha20-Poly1305; UDP transport; no negotiation (static config).

Performance: ~5Gbps per core in software, more with AVX-512.

Use case: VPN — site-to-site, road warrior, k8s pod-overlay (Calico uses WireGuard for inter-node encryption).

Not generally a TLS competitor; complementary.

## 14.16 Future — `AF_QUIC` and beyond

Watch LWN for in-kernel QUIC. Likely:
- `socket(AF_QUIC, SOCK_STREAM, ...)` returning a QUIC stream fd.
- `sendfile()` on a QUIC stream.
- NIC QUIC offload (some NICs already have rudimentary support).

This would extend kTLS's win to HTTP/3.

## Must-internalize

- TLS 1.3 = 1-RTT (0-RTT with resumption).
- kTLS pushes symmetric keys to kernel after handshake; `sendfile()` works with TLS.
- TX in 4.13+, RX in 4.17+, HW offload in 5.0+.
- Netflix's 800Gbps box is the canonical kTLS + sendfile reference.
- 0-RTT data is replayable; only for idempotent (GET).
- Session tickets enable resumption; rotate keys to maintain forward secrecy.
- mTLS = mutual TLS, the service mesh pattern (Istio, Cilium mesh).
- QUIC integrates TLS into transport; in-kernel QUIC is experimental.
- WireGuard is a complementary in-kernel VPN; different design from IPsec.