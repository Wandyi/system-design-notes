# 8 · DRM, Security, and Anti-Piracy

For premium VOD (studio content), DRM isn't optional. Studio contracts mandate specific DRM systems, security levels, output protection, and concurrency rules. A staff engineer needs to understand the three major DRM systems, how CMAF + CENC unifies them, and how forensic watermarking is used for traceability.

## 8.1 The three major DRM systems

| | Widevine (Google) | FairPlay (Apple) | PlayReady (Microsoft) |
|---|-------------------|-------------------|------------------------|
| Devices | Android, Chrome, Chromecast, smart TVs, Chromebooks | iOS, macOS, tvOS, Safari | Windows, Xbox, smart TVs |
| Encryption modes | cenc, cbcs | cbcs only | cenc, cbcs |
| Security levels | L1 (TEE/SoC), L2, L3 (software) | "Streaming" / "Persistence" | SL150 / SL2000 / SL3000 |
| License server | Google or self-hosted (uses google's KMS for some flows) | Self-hosted (Apple's signed key) | Microsoft / self-hosted |
| Codec quirks | All | HEVC + H.264 (no VP9/AV1 historically; AV1 now via FairPlay 2 on newer devices) | All |

To serve all devices, you need all three. To save encoding, you use **CMAF + CENC** so the same encrypted bytes work across systems.

## 8.2 Common Encryption (CENC) — one ciphertext, multiple DRMs

ISO/IEC 23001-7 defines **Common Encryption** for ISO Base Media File Format (the mp4 family):

- Encrypted **at the sample level** — only video/audio samples are encrypted; metadata stays clear so players can navigate.
- Two cipher modes:
  - **cenc** — AES-128 CTR mode. Standard.
  - **cbcs** — AES-128 CBC with constant initialization vectors and pattern encryption (encrypt 1, skip 9). Required for FairPlay.

For "encrypt once, deliver to any device":
- Pick **cbcs** mode (FairPlay's requirement; Widevine and PlayReady both support it).
- Package as CMAF segments.
- Each DRM system has a **`pssh` box** (Protection System Specific Header) in the file that the player uses to find the license server.

The result: one set of encrypted segment bytes on disk, three license endpoints (Widevine, PlayReady, FairPlay).

## 8.3 The DRM flow

1. **Player** starts playback, parses manifest, sees encrypted content.
2. **EME (Encrypted Media Extensions)** browser API requests a session.
3. Player gets a **key request message** from the CDM (Content Decryption Module).
4. Player POSTs key request to **license server** with an entitlement token.
5. License server checks entitlement (does this user have rights? are they within concurrency limits?), generates a license containing the content key wrapped with the device's public key.
6. Player passes license to CDM. CDM stores the key in secure hardware where possible.
7. CDM decrypts samples as the player decodes.

The license can carry **policy**:
- **Persistence** — can the license be cached for offline playback? For how long?
- **Output protection** — require HDCP; minimum HDCP version (2.2 for 4K).
- **Hardware vs. software DRM** — for 4K, studios demand hardware-backed (L1 / SL3000 / FairPlay HW).
- **Concurrency** — implicit via session tokens.

## 8.4 Concurrency limits

"Only 2 streams per account" is a real product rule for many SVODs. Implementation:

- Every playback session gets a token from a **session service**.
- License is bound to the session token.
- Heartbeat from player to keep session alive (e.g., every 60s).
- When concurrency exceeds limit, oldest session is revoked (or a new session is rejected).

The session store is hot — every playback start/end mutates it. Typically Redis-class.

## 8.5 Geographic restrictions and rights enforcement

Studios license content per territory. A title may be available in India but not in the UK. Enforcement:

- **IP geolocation** at the license issuance step — refuse to issue if the user's IP is outside permitted regions.
- **VPN detection** — IP reputation services flag known VPN/proxy ranges. Iffy at scale; some users will be wrongly blocked.
- **Account country** as an additional check.
- **Content blackouts** — e.g., live sports often blacked-out in the home market for the local TV broadcast.

Geographic enforcement is a frequent source of customer complaints. The interview answer: "we apply policy at license issuance, plus a manifest-level check for redundancy; we accept some VPN false positives".

## 8.6 Forensic watermarking — when DRM isn't enough

DRM stops casual piracy. Determined attackers (organized rings) can capture content via:
- Screen recording (DRM doesn't fully block this).
- HDMI capture (HDCP-broken or stripped devices).
- Decrypting via a compromised CDM (rare but happens — Widevine L3 has been broken before).

To trace leaks back to a specific user, **forensic watermarking** is used. Two techniques:

1. **Pre-baked watermark** — encode a barely-visible per-user identifier into the video. Done at edge or with two pre-encoded variants and stitching.
2. **A/B variant watermarking** — encode each segment in two slightly different variants (A and B). Per-user, deliver a unique A/B pattern. Recovering the pattern from a leak identifies the user.

A/B is dominant for premium content because the cost scales (it's *2 encodings*, *1 manifest*, *N delivery sequences*).

## 8.7 Output protection — HDCP

For 4K/HDR content, studios demand HDCP 2.2 on the HDMI output. The player tells the OS "require HDCP 2.2"; OS refuses to output if the connected display can't comply. End users sometimes see "this content requires HDCP" errors — that's the chain enforcing.

For browser playback: the EME implementation can require HDCP via a license policy flag.

## 8.8 Offline downloads

For mobile / commute / flight use cases. Implementation:

- Player downloads encrypted segments to local storage.
- License server issues a **persistent license** with a TTL (e.g., 30 days, or 48 hours after first play).
- Player stores license in secure storage; uses it offline.
- License can be **renewed** when online.

Subtleties:
- Concurrent device limits — downloads count toward concurrency.
- Disk encryption — segments shouldn't be readable outside the player.
- TTL enforcement — clock manipulation attacks; license servers use server-time-anchored TTL.

## 8.9 Token-based playback authorization

The DRM flow doesn't authorize playback in isolation. There's usually a **playback token** layered on top:

1. User starts a playback session.
2. Server (the "playback session service") verifies entitlement (subscription, region, concurrency, device fingerprint).
3. Server issues a short-lived **playback token** (JWT-class).
4. Token is attached to manifest and license requests.
5. CDN edge validates token signature; license server validates entitlement encoded in the token.
6. Token TTL is short (minutes); player refreshes during long sessions.

This gives quick revocation: invalidate the token issuer's signing key and all in-flight sessions terminate (heavy-handed but available).

## 8.10 Streaming-specific attacks and mitigations

- **Stream ripping** — automated tools download segments, save the content. Mitigation: token-based auth, IP-bound tokens, behavior detection (1000 segments downloaded in 30 seconds = bot).
- **Key extraction** — attacker reverse-engineers a player to extract content keys. Mitigation: hardware DRM, periodic key rotation, obfuscation.
- **Re-streaming / pirate IPTV** — someone subscribes legally, then re-broadcasts the stream to a paying piracy service. Mitigation: forensic watermarking, anti-piracy intelligence / DMCA.
- **Credential stuffing** — attackers try leaked passwords from other sites. Mitigation: MFA, anomaly detection, CAPTCHA on login burst.
- **Geo-spoofing via VPN** — addressed in §8.5.
- **Manifest manipulation / parameter tampering** — mitigated by signed URLs and tokens.

## 8.11 Privacy

DRM, telemetry, and recommendations all touch user data. Compliance angle:
- **GDPR** in Europe — explicit consent, right to erasure, data-portability requests.
- **DPDP Act 2023** in India — analogous; JioHotstar must comply.
- **Children's content** — COPPA in the US, age-appropriate consent flows.
- **Data residency** — some countries require data on-shore.

A staff engineer should be able to articulate how playback session data, recommendation training data, and DRM session logs are stored, retained, and deleted on request.

## 8.12 Encryption-at-rest, in-transit, and key management

- **At rest**: segments encrypted under CENC keys. Keys stored in a KMS (HSM-backed in production).
- **In transit**: TLS 1.3 between player and CDN; mTLS between CDN and origin where possible.
- **License server keys**: per-platform private keys (Widevine, FairPlay, PlayReady) stored in HSM. License server itself is high-availability — outage means no new playback starts.
- **Key rotation**: per-title content keys can be rotated, but it requires re-encoding segments. In practice, content keys are stable for the title's life; access is controlled via license-issuance policy.

## 8.13 The full security stack diagram

```
   User device
       |
       | TLS 1.3
       v
   CDN edge  ----  signed playback token (JWT, short TTL)
       |
       | mTLS
       v
   Origin / Manifest service
       |
       v
   License server  ----  per-DRM private keys (HSM)
       ^
       |
       | entitlement check
       |
   Auth + Session service  ----  user DB (encrypted at rest)
       |
       v
   Recommendation service (uses session data, NOT personally tied at this layer)
       |
       v
   Telemetry / analytics (anonymized for aggregate, identifiable per session for QoE)
```

## 8.14 Must-internalize

- Widevine + FairPlay + PlayReady are the trio; serve all three via **CMAF + CENC (cbcs)** with one set of encrypted bytes.
- License server issues per-session licenses with policy (HDCP, persistence, output protection).
- Concurrency limits enforced by a session service plus license binding.
- Geo + VPN detection at license issuance; some false positives are accepted.
- Forensic watermarking (A/B variants) for traceable leaks on premium content.
- Token-based playback authorization layered over DRM for quick revocation.
- Encryption + KMS + HSM for keys; TLS 1.3 + mTLS for transit.
- GDPR/DPDP and data residency shape architecture.

---

## Sources

- [Common Encryption (ISO/IEC 23001-7)](https://www.iso.org/standard/68042.html)
- [W3C Encrypted Media Extensions (EME)](https://www.w3.org/TR/encrypted-media/)
- [Widevine docs](https://developers.google.com/widevine)
- [Apple FairPlay Streaming docs](https://developer.apple.com/streaming/fps/)
- [Microsoft PlayReady docs](https://docs.microsoft.com/en-us/playready/)
- [DASH-IF Content Protection guidelines](https://dashif.org/guidelines/)
- [Forensic watermarking — NAGRA NexGuard](https://dtv.nagra.com/nexguard-forensic-watermarking)
