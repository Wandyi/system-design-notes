# Application Security at Scale — Staff-Level Q&A and Case Studies

A staff-level reference for designing, operating, and defending application security across realistic scenarios. Each section frames the problem, walks the corner cases, and addresses how the design holds up under scale and availability constraints.

The companion document `authNAuthZAtScale.md` covers identity, sessions, tokens, and authorization models in depth. This document focuses on the *rest* of application security: input handling, secrets, cryptography, supply chain, abuse, data protection, network/edge, and incident response.

---

## Section 1 — Threat Modeling as a Discipline

### Q1.1 What does threat modeling look like at staff level, beyond STRIDE checklists?

STRIDE and DREAD are starting templates, not deliverables. A staff-level threat model:

- **Starts from a data flow diagram** of the actual system, not the architecture diagram on Confluence. Real flows include the cron job, the analytics export, the support tool, the DB replica the data team queries.
- **Names trust boundaries explicitly.** Every line that crosses one is an authentication and authorization point. Most app vulnerabilities live on lines someone forgot to label.
- **Names attackers concretely.** "External attacker" is not enough. The set is at minimum: unauthenticated internet attacker, authenticated user, authenticated user from another tenant, malicious insider, compromised CI, compromised dependency, curious support engineer.
- **Names assets concretely.** Not "user data" — "the column `users.email_hash` used for password reset," "the S3 bucket holding signed export URLs valid for 7 days."
- **Outputs ranked, owned mitigations** with deadlines and verification plans. A model that ends in "we should look into MFA" failed.

**Corner cases people miss:**
- Read replicas that bypass the auth proxy.
- Backup pipelines copying production data to lower-trust environments.
- Debug/feature-flag endpoints disabled in prod by config — config drift re-enables them.
- Test fixtures with real PII committed to a private repo (still indexed by code search, still ends up in laptop git clones).

### Q1.2 How do you scale threat modeling across hundreds of services without it becoming a paperwork tax?

- **Tier services by blast radius.** Tier 1 (touches money, PII, auth) gets a deep model and security review per major change. Tier 3 (internal dashboards) gets a checklist.
- **Make the model live in the codebase.** A `THREAT_MODEL.md` next to the service, reviewed in PRs that change trust boundaries.
- **Automate the boring parts.** Lint for missing auth annotations, missing input validation on public handlers, secrets in commits. The threat model documents what *can't* be automated.
- **Trigger a refresh on architecture changes**, not on a calendar. A quarterly review of an unchanged service is theater; a model update when a new external integration lands is the actual control.

---

## Section 2 — Input Handling and Injection

### Q2.1 SQL injection in 2026 — why is it still a top-five issue and how do you eliminate it as a class?

It persists because:
- Dynamic query construction in legacy code (string concatenation in stored procedures, ORMs with raw `WHERE` builders).
- Search/filter/sort features where users supply column names — parameterization doesn't cover identifiers.
- ORM escape hatches (`raw()`, `Exec()`, query templates) used for performance.
- Reporting and admin tools that accept "SQL-ish" input.

**Eliminating it as a class, not as instances:**
- **Parameterized queries are the default**, with code-review and lint rules forbidding string concatenation into SQL.
- **Identifier safelist** for any user-controlled column or table name. Map user input to known identifiers; never interpolate.
- **Least-privileged DB users.** App user can `SELECT/INSERT/UPDATE` on its tables, not `DROP`, not cross-schema. A SQLi that succeeds is then bounded.
- **Read replicas with separate credentials** for analytics paths.
- **Detective controls.** DB query logs piped to anomaly detection. A sudden `UNION SELECT` from a service that never issues one is a paging signal.
- **Static analysis in CI** that flags raw query construction; failures block merge.

**Corner cases:**
- Second-order SQLi: input stored safely, then later concatenated into a query in a different code path. Fix at the query site, not just at ingestion.
- LIKE wildcard injection — user-supplied `%` characters subverting search semantics. Escape `%` and `_` before parameterizing.
- ORDER BY injection. Always map sort column from a safelist.
- Bulk imports — CSV cells starting with `=`, `+`, `-`, `@` cause CSV injection in downstream Excel/Sheets consumers. Prefix with a single quote on export.

### Q2.2 You're building a multi-tenant API gateway. What does input validation owe to scalability?

- **Validate at the edge, not in every service.** A schema-driven gateway (OpenAPI + envoy/kong/nginx + WAF) rejects malformed input before it consumes downstream capacity.
- **Bound everything.** Maximum body size, maximum array length, maximum string length, maximum nesting depth. Without bounds, a 10 MB JSON with 1M nested objects exhausts a parser long before it reaches business logic.
- **Reject early on protocol layer.** HTTP/2 RST flood, HPACK bomb, slowloris, oversized headers, suspicious method+path combos — these are capacity attacks dressed as input. Edge proxies must terminate them.
- **Per-tenant quotas.** A noisy tenant must not consume the validation budget of others. Token-bucket per `tenant_id`, with fairness enforced at the LB.
- **Validation must be fast.** A schema engine that takes 10ms per request collapses your p99 budget. Cache compiled schemas; benchmark them under fuzzing.

**Corner cases:**
- Unicode normalization. `café` and `café` look the same to humans, different to byte comparison. Normalize to NFC at the boundary before equality checks (especially for usernames, emails, URLs).
- Encoding ambiguity. Double-URL-encoded input bypasses naive decoders. Decode once, then validate; reject if a second decode changes the result.
- Content-Type vs Content sniffing. A client sends `Content-Type: application/json` but a different content-encoded body. Validate the parsed result, never trust the header.
- Trailing data after a valid JSON document — many parsers accept it silently. Strict parsers only.

### Q2.3 Cross-site scripting (XSS) — walk the modern landscape end-to-end.

**Three classes:**
1. **Reflected** — input echoed in the response of the same request.
2. **Stored** — input persisted, rendered later to other users.
3. **DOM-based** — input flows from `location.hash`/`document.referrer`/postMessage into a sink (`innerHTML`, `eval`, `setAttribute('href', ...)`) without server involvement.

**Layered defenses:**
- **Output encoding context-aware.** HTML body, attribute, JS, CSS, URL — each has its own escaping rules. A single `escape_html` function applied everywhere is wrong.
- **Templating engines that escape by default** (React, Vue, modern Jinja autoescape on). Treat `dangerouslySetInnerHTML` and `v-html` as code review red flags.
- **Content-Security-Policy with nonces or hashes**, no `'unsafe-inline'`, no `'unsafe-eval'`. CSP is the safety net when output encoding fails.
- **Trusted Types** (modern browsers) to enforce that DOM sinks only accept validated objects, not strings.
- **Subresource Integrity (SRI)** on third-party scripts — a CDN compromise (Polyfill.io 2024 incident) becomes a non-event.
- **Sanitize HTML inputs** with a vetted library (DOMPurify) — never with regex.

**Corner cases:**
- SVG uploads — SVG is an XML document and can carry `<script>`. Either strip on upload or serve from a sandbox domain with `Content-Security-Policy: sandbox` and `Content-Disposition: attachment`.
- Markdown rendering — `[click](javascript:alert(1))`. Reject non-http(s)/mailto schemes after parsing.
- `target="_blank"` without `rel="noopener noreferrer"` — leaks `window.opener`.
- PDF/HTML email rendering — many email clients render HTML; `mailto:` links and tracking pixels exfiltrate read state.

### Q2.4 SSRF — why is this the bug that ends careers?

Because cloud metadata services (`169.254.169.254`) hand out IAM credentials to anything that can reach them. An SSRF in a tier-1 service is one HTTP call from "fetch this URL" to "exfiltrate IAM keys" to "exfiltrate every S3 bucket."

**Eliminate-or-bound strategy:**
- **Outbound network egress allowlist.** Services that need to fetch user-supplied URLs do so through a forward proxy that resolves DNS and blocks non-public IP ranges (RFC1918, link-local, loopback, IPv6 equivalents, multicast).
- **DNS rebinding defense.** The proxy resolves DNS *itself* and connects to the resolved IP, refusing to follow `Host:` to a different IP. Naive code does `resolve → check IP → connect by hostname`, leaving a TOCTOU gap a rebinding server exploits.
- **IMDSv2 only** on AWS — requires session token, blocks the simple GET that SSRF triggers.
- **Disable URL schemes other than http(s).** No `file://`, no `gopher://`, no `dict://`. Many libraries support a long list by default.
- **Disable HTTP redirects, or revalidate the new URL** before following.
- **Run untrusted-URL fetchers in their own VPC/account** with no IAM role, no metadata access, no internal DNS.

**Corner cases:**
- IPv4 in oddball encodings: octal, hex, integer (`0x7f000001`, `2130706433`). The allowlist must canonicalize before comparing.
- IPv6 with embedded IPv4 (`::ffff:127.0.0.1`).
- Time-of-check vs time-of-use on DNS — use the *connection's* peer IP for validation, not a separate lookup.
- Webhook delivery (your service POSTing to user-supplied URLs) is the same threat with a different name. Same defenses apply.
- PDF/image/video processing libraries fetch external resources by default (XSLT in PDFs, remote SVG references). Sandbox the renderer.

### Q2.5 File upload — what's the full hardening checklist?

- **Validate type by content, not extension or `Content-Type`.** Magic bytes; for images, decode and re-encode through a vetted library to strip embedded payloads.
- **Cap size before reading the body.** Stream to disk with a hard byte limit; exceeding it kills the connection.
- **Store on a separate origin/domain.** Never serve user uploads from the app domain — a malicious HTML file becomes XSS-on-app-origin otherwise.
- **`Content-Disposition: attachment`** by default for downloads, not inline.
- **Random storage keys.** Don't store as `uploads/<filename>` — collision and enumeration risk. Use UUIDs; track original filename in metadata.
- **Antivirus / content scanning.** ClamAV for known-bad; cloud equivalents for known-malicious URLs in PDFs.
- **Image processing in a sandbox.** ImageMagick has had remote code execution via crafted images more than once. Run in a process-isolated sandbox with strict resource limits.
- **Block dangerous types** (`.exe`, `.dll`, `.bat`, `.scr`, archive bombs). For zip/tar uploads, decompress with bounded output size and bounded entry count to block zip bombs.
- **For video/audio**, transcode through a known-good pipeline, never serve original bytes — payloads in metadata or codec edge cases are real.

**Scalability angle.** Upload paths must be backpressure-aware. Direct-to-S3 with pre-signed URLs offloads the bandwidth, but the pre-signing service still validates content-type and size constraints, and post-upload processing (scan, transcode) is asynchronous via a queue.

---

## Section 3 — Cryptography Done Right

### Q3.1 You're picking primitives for a new service. What are the modern defaults, and what should you never roll yourself?

**Defaults (2026):**
- **Symmetric encryption:** AES-256-GCM or ChaCha20-Poly1305 (better on platforms without AES-NI). Always authenticated encryption — never raw AES-CBC.
- **Hashing:** SHA-256 or BLAKE3. SHA-1 and MD5 are defective for security purposes.
- **Password hashing:** Argon2id with parameters tuned for ~250ms on target hardware; bcrypt(12+) acceptable fallback. Never SHA-anything for passwords.
- **Key exchange / signatures:** Ed25519, X25519. RSA-2048 acceptable but boring; RSA-PKCS1v1.5 for signatures is fragile.
- **Public-key encryption:** Hybrid (X25519 + AEAD), e.g., libsodium `crypto_box`.
- **Randomness:** OS CSPRNG only (`/dev/urandom`, `getrandom()`, `crypto.randomBytes`). Never `Math.random()`, `rand()`, `time(NULL)` for security.
- **TLS:** 1.3 only. Disable 1.0/1.1; 1.2 only for explicit legacy compatibility.

**Never roll yourself:** the algorithms, the modes, the protocols, the libraries. Use libsodium, Tink, or the platform's audited crypto library. Constant-time comparison, side-channel resistance, and parameter validation are easy to get wrong.

### Q3.2 Encryption-at-rest, key management, and rotation — what does a production design look like?

**Envelope encryption** is the answer for almost everything:
- A KMS holds **root keys (KEKs)**. The KMS does encrypt/decrypt of small blobs only and never releases the KEK material.
- Each data record (or each chunk) gets a **data encryption key (DEK)** generated by the KMS. The DEK encrypts the data; the wrapped DEK is stored alongside.
- Decryption: ask the KMS to unwrap the DEK, decrypt locally.

**Why envelopes:**
- KMS calls are expensive — envelopes amortize them.
- Key rotation rotates the KEK; existing wrapped DEKs are re-wrapped lazily or in a background job. The data ciphertext doesn't move.
- Per-tenant or per-record DEKs give blast-radius isolation. A leaked DEK exposes one record, not the corpus.

**Rotation cadence:**
- KEK rotation: yearly or on suspected compromise. Old KEK retained for decryption.
- DEK rotation: tied to the data lifecycle (re-encrypt on update) or scheduled batch job.
- TLS certs: short-lived (90 days max, ideally automated via ACME or service mesh).

**Corner cases:**
- Backups must be re-encrypted under new KEKs *or* old KEKs preserved for restore. Either approach must be tested via a restore drill.
- Multi-region: KMS keys are regional. Cross-region DR requires either replicating keys (some KMS support multi-region keys) or maintaining per-region KEKs.
- Search over encrypted data is hard; deterministic encryption leaks equality, order-preserving leaks order. For most cases, encrypt the field, search via a separately-stored hash with a salt scoped to the tenant.
- HSM-backed KMS for high-value keys (signing roots, payment keys). FIPS-140 level matters for some compliance regimes.

### Q3.3 Secrets management — how do you stop secrets from spreading and leaking?

**Architecture:**
- **Single source of truth** (Vault, AWS Secrets Manager, GCP Secret Manager).
- **Workloads fetch at runtime** via workload identity, not at build time.
- **Short-lived dynamic secrets** (Vault DB engines generate per-pod DB credentials valid for hours) where supported.
- **Sidecars or init containers** inject secrets into env vars or tmpfs files; never bake into images.

**Detection:**
- **Pre-commit hooks** (gitleaks, trufflehog) catching obvious patterns.
- **Server-side secret scanning** on push (GitHub Advanced Security, GitLab Secret Detection).
- **Periodic full-history scan** — secrets sneak in via merge commits and squash flattening. The "delete the commit" reflex doesn't help; the secret is in the reflog and on every clone.
- **Cloud-provider scanning** (GitHub partners with AWS, GCP, etc., to invalidate leaked keys automatically).

**When a secret leaks:**
- **Rotate first, investigate second.** Even private repo leaks are compromise events — assume unauthorized read.
- **Audit trail of usage** for the leaked credential during the leak window.
- **Rotate transitively.** If the leaked credential could fetch other credentials, rotate those too.
- **Postmortem on how it landed there**, with a control to prevent the class.

**Corner cases:**
- Secrets in container layers — `RUN echo $SECRET > /etc/foo` persists in the layer even if a later `RM` removes the file.
- Secrets in build logs — CI configs that print env vars, third-party actions that dump context. Mask in CI; review third-party action permissions.
- Secrets in error messages, stack traces, and bug reports. Sanitize before logging; don't include request bodies in logs unless explicitly redacted.
- Browser history / referer headers carrying tokens in URL query strings. Tokens belong in headers, not URLs.

---

## Section 4 — Supply Chain Security

### Q4.1 SolarWinds, log4j, xz-utils — what does a credible supply chain defense look like for a typical product company?

**You can't audit every dependency.** The realistic strategy is to reduce blast radius and detect compromise quickly.

**Reduce intake:**
- **Minimize dependencies.** Each package is a trust decision. Reject packages with one maintainer, sub-1k stars, or no recent activity, unless the alternative is worse.
- **Pin and lock.** Lockfiles (`package-lock.json`, `go.sum`, `Cargo.lock`, `poetry.lock`) committed; CI verifies integrity. No floating versions in production builds.
- **Verify provenance.** Sigstore/cosign for container images; SLSA attestations from build pipelines; Go module checksums via `sumdb`.
- **Mirror through an internal proxy.** Artifactory, Nexus, or cloud equivalents. Public registries can disappear, get attacked, or yank packages.

**Reduce blast radius:**
- **Build in hermetic, sandboxed CI.** No outbound network during build (other than to known registries). A malicious package with a `postinstall` script can't exfiltrate.
- **Capability-bound runtimes** where supported (Deno, Wasm sandboxes for plugins). Untrusted code shouldn't have unconstrained file/network access.
- **SBOMs (Software Bill of Materials)** generated at build time, stored, queryable. When the next CVE drops, the question "are we affected" is a SQL query, not a fire drill.

**Detect:**
- **Continuous CVE scanning** of running images, not just at build time. New CVEs publish daily; the build was fine yesterday.
- **Anomaly detection on outbound traffic** from build agents and production. A build calling out to a Pastebin URL is a signal.

**Corner cases:**
- **Typosquatting / dependency confusion.** Internal package names matching a public registry — `pip install internal-utils` may pull from PyPI's namesake. Use scoped packages (`@yourorg/utils`) and configure registries to refuse public fallbacks for internal scopes.
- **Upgrade attacks.** A trusted package's new minor version turns malicious (xz). Pin, review changelogs for high-trust packages, delay automatic upgrades for non-security releases.
- **Transitive dependencies** are most of your dependency tree. The auditing has to be on the resolved tree.

### Q4.2 You're shipping a product where customers run your binary on-prem. What changes about supply chain?

- **Reproducible builds** — customers can verify the binary matches the published source.
- **Signed releases** with a hardware-rooted signing key, ideally with transparency logs (cosign + Rekor).
- **Vulnerability disclosure SLA** customers can hold you to.
- **No phone-home that leaks customer data.** Telemetry must be opt-in, transparent, and locally inspectable.
- **Auto-update with rollback.** A bad release pushed to all customers is a customer-property security incident. Staged rollouts, signed updates, atomic rollback.

---

## Section 5 — Abuse, Bots, and Fraud

### Q5.1 You launch a free-tier signup. Within hours, you have thousands of fake accounts mining your free quota. Walk through the response.

**The asymmetry.** Attackers automate; defense is per-account or per-IP. Every defense has a cost in legitimate-user friction.

**Layers of defense, in deployment order:**
1. **Email/phone verification.** Eliminates the laziest bots. Doesn't stop a determined attacker with disposable email or SIM farms.
2. **Disposable email blocklists** (10minutemail, Mailinator domains). Updated continuously.
3. **CAPTCHA** at signup, ideally invisible (hCaptcha, reCAPTCHA Enterprise, Cloudflare Turnstile). Tunable difficulty by risk score.
4. **Device fingerprinting.** Browsers reveal a lot — canvas, fonts, timezone, screen, audio context. Risk-score by fingerprint reuse.
5. **Behavioral signals.** Time-on-page, mouse movement, typing cadence. Bots fill instantly; humans don't.
6. **Velocity rules.** N signups per IP/ASN per hour. Per-ASN matters because a single bot can rotate IPs within a cloud range.
7. **KYC for high-value tiers.** Credit card verification (with $0 auth) for any account that touches limits worth abusing.
8. **Network reputation.** Block known-bad VPN/Tor exit nodes; risk-score data center IPs (a residential signup from a Hetzner IP is suspicious).
9. **Proof of work** for heavyweight signup (sometimes useful for queueing).

**The corner case people miss.** The best abuse signal is *post-signup behavior*, not signup-time signals. A bot that gets through behaves differently in the first 24 hours than a human. Build the model that detects this and reclaims accounts asynchronously.

**Scalability angle.** Bot detection can't be inline-blocking on the hot path beyond a few ms. Score asynchronously, gate sensitive actions (creating compute, sending email, accessing PII) on a fresh score.

### Q5.2 Your login endpoint is being credential-stuffed at 100k req/s from a botnet. What now?

**Immediate response:**
- **Bot detection vendor in front of login** (Cloudflare, Akamai, PerimeterX). They have the cross-customer botnet signal.
- **Per-account exponential backoff with CAPTCHA** rather than hard lockout (hard lockouts let the attacker DOS your users).
- **Disable password login temporarily** in favor of email magic-link, if the abuse is severe and credential stuffing is the main vector. Inconvenient but effective.
- **Force password reset for accounts whose passwords appear in known breaches** (HIBP integration on login attempt).

**Sustained posture:**
- **Have-I-Been-Pwned check at password set time** to refuse breached passwords entering your system at all.
- **Anomaly detection on the login population** — sudden spike in failures across many accounts is the signature.
- **Step-up MFA on suspicious sessions.** Even if password is correct, a new device + risky geo gets a second factor.
- **Account lockouts must not enumerate users.** "This account is temporarily locked" leaks existence; "if this account exists, it may be locked" doesn't.

**Availability angle.** Login is critical-path. The bot defense must fail open for legitimate users on its own outage — better to let some bots through for 5 minutes than to lock every customer out. This is a deliberate, documented trade-off.

### Q5.3 Payment fraud — what hooks should the application provide for the fraud team to actually do their job?

- **Velocity events** at every meaningful action (signup, address change, card add, first transaction, large transaction).
- **Linkage data** — IPs, device fingerprints, emails, names — passed to the fraud system.
- **Action interception points.** Fraud system must be able to delay, hold, or step-up any high-risk action without app-team code changes.
- **Soft and hard reverses** for transactions deemed fraud post-hoc.
- **Customer recourse path** — false positives are real; legitimate customers must have a clear path to unblock.

This is a security/product co-design problem; the fraud team is a stakeholder of the application's data model.

---

## Section 6 — Data Protection and Privacy

### Q6.1 You handle PII for users in the EU, US, and India. What does the data layer owe?

- **Data classification.** Every column tagged: `public`, `internal`, `pii`, `sensitive_pii` (SSN, biometrics), `regulated` (PCI, PHI). Tagging is the prerequisite for everything else.
- **Per-field encryption for sensitive PII.** Application-layer encryption with KMS-managed keys, separate from at-rest disk encryption. Disk encryption protects against stolen disks; per-field encryption protects against compromised DB queries.
- **Data residency.** EU PII stays in EU regions. Architect tenants with a home region; replicate within a regulatory bloc, never across.
- **Right to erasure (GDPR Art. 17).** Designs that "soft-delete" with `deleted_at` flags fail this. Implement true purge with verification, including from backups (timed expiration of backups that contain deleted data is acceptable).
- **Right to access / portability.** Export tooling that produces a complete user dump in machine-readable format, with a turnaround SLA.
- **Consent and purpose binding.** Each PII field is tagged with the legal basis for its collection and the purposes it can be used for. Analytics queries that violate the purpose are blocked.

**Corner cases:**
- Logs and metrics are data stores too. PII in logs is a breach waiting to happen. Either redact at emission, or treat the log store as PII-class.
- Backups are the deletion graveyard. Backup retention windows must be shorter than the period in which deletion completes.
- Third-party processors (Datadog, Sentry, Mixpanel) are subprocessors under GDPR. Listed in your DPA, audited, with DPAs of their own.
- ML training data — once a model is trained on a user's data, "deletion" is hard. Either avoid training on raw PII, or maintain a retraining pipeline that excludes erased users.

### Q6.2 Logging — how do you log enough for security and operations without becoming the breach?

**The principle.** Logs are persistent, replicated, and broadly accessible. Anything you put in them is effectively published inside the company.

**Practices:**
- **Structured logs only.** Fields, not formatted strings. Redaction rules apply to fields by name (`password`, `ssn`, `card_pan` always dropped or hashed).
- **Centralized redaction at ingestion**, not per service. A regex layer that catches PAN-like and SSN-like patterns even when the field name is wrong.
- **Sampling and retention by class.** Audit logs (auth, authz, admin actions): long retention, immutable, restricted access. App logs: short retention, redacted, broader access.
- **Access controls on log tooling.** Datadog/Splunk/ELK access is sensitive — audit who queries what.
- **Append-only audit logs.** WORM storage (S3 Object Lock, immutable Cloud Logging). The attacker who gets in must not be able to erase their tracks.

**Corner cases:**
- Stack traces include local variables in some languages — variables can hold secrets. Sentry has a redaction config; use it.
- Request bodies as logs — never log raw bodies of authenticated endpoints. If you must, redact.
- HTTP headers carry `Authorization`, `Cookie`. Drop on emission.
- "Debug" logs left in production after a crisis. Auto-expire log levels above INFO after a deployment.

### Q6.3 Multi-tenant data isolation — what failures should you design against?

**The four failure modes:**
1. **Forgotten `WHERE tenant_id = ?`** in a query.
2. **Cache key collision** — `cache.get('user:123')` returns another tenant's user 123.
3. **Background job leaks** — a job processes records for tenant A but writes results to a table without a tenant scope.
4. **Cross-tenant feature leaks** — search index, ML model, reporting dashboards trained or aggregated across tenants and exposing one to another.

**Defenses:**
- **DB-level tenant isolation.** Postgres RLS keyed on session variable; the app sets it from the verified token. Forgetting `WHERE` becomes inert.
- **Tenant ID in cache keys, mandatory.** Lint rule on cache library calls.
- **Tenant-scoped contexts.** Every job carries a `tenant_id` from origin to completion; cross-tenant write is a runtime error.
- **Test it.** Cross-tenant pen tests are part of CI: spin two tenants, attempt every endpoint and lookup with one tenant's token against another's IDs.
- **Per-tenant isolation for the highest tier.** Schema-per-tenant or DB-per-tenant for customers who pay for it; complete blast-radius isolation.

---

## Section 7 — Network and Edge

### Q7.1 You're designing a public-facing API. Walk through the layers between the internet and your handler.

1. **DDoS protection** at edge (Cloudflare, AWS Shield, GCP Cloud Armor). Volumetric and reflection attacks absorbed before they reach you.
2. **WAF** with managed rule sets (OWASP Core Rule Set + provider-specific). Tunable; fail-open mode for outage tolerance.
3. **TLS termination** on a separate fleet from app handlers — TLS-level attacks don't reach app capacity.
4. **Bot management** — scoring and challenge for suspicious traffic.
5. **Rate limiting** — global, per-IP, per-API-key, per-endpoint. Token-bucket with burst tolerance.
6. **Authentication** — token validation, revocation check.
7. **Authorization** — policy decision (per the companion auth doc).
8. **Schema validation** — request shape, size, types.
9. **Business logic.**

Every layer fails open or closed, deliberately. WAF fails open for availability, auth fails closed.

### Q7.2 Mutual TLS, network policies, and zero-trust internal traffic — what's the minimum viable design and what's the bar?

**Minimum viable:** Service mesh (Istio, Linkerd, Cilium) with mTLS between every pod, Kubernetes NetworkPolicies restricting which services can talk to which.

**Staff-level bar:**
- **SPIFFE-based workload identity** issued by an attested platform (SPIRE, cloud workload identity).
- **Authorization policies in code** (OPA, Cedar) consulted by sidecar or library, with policy bundles versioned and rolled out gradually.
- **Default-deny network policies.** A new service must explicitly request who it can talk to.
- **East-west traffic logged and sampled.** Anomaly detection — service A starting to call service Z is a signal.
- **Egress controls.** Outbound to internet is via an audited proxy. No service has unlimited internet.
- **Database access via a proxy** (cloud SQL proxy, RDS proxy, Vault DB engine) so DB credentials are short-lived and access is auditable.

### Q7.3 An internal service is reachable from outside the corporate network because someone created a public LB. How do you prevent the next one?

This is a configuration problem, not a code problem.

- **Policy-as-code on infrastructure.** Open Policy Agent integrated with Terraform/CDK. Plans that create a public LB without an explicit annotation fail review.
- **Cloud provider service control policies.** AWS SCPs, GCP Org Policies forbid creating public S3 buckets, public LBs, public IPs in certain accounts.
- **Continuous compliance scanning** (Prowler, Cloud Custodian, native config services). Daily report of drift; high-severity drift pages on-call.
- **Staging-vs-prod account separation.** Prod accounts have stricter SCPs; experiments in staging can't escape.
- **Default-private VPCs.** New environments come with no public ingress; making something public is a deliberate, audited action.

---

## Section 8 — Case Studies

### Case Study A — Account Takeover via Magic Link

**Scenario.** A B2C product offers passwordless login: enter email, receive a magic link, click to log in. Six months in, support gets multiple reports of accounts taken over. Logs show successful logins from unfamiliar devices, but the email address on file is unchanged.

**Investigation.**
- Email provider logs are intact; the user's inbox shows the magic link arriving and being clicked.
- The magic link is a long random token, single-use, valid for 10 minutes.
- Click-tracking on the link goes through `track.example.com/r?u=<encoded url>`.

**Root cause.** The email link is wrapped by a marketing email click-tracker. The tracker's logs include the destination URL (with the magic token). Marketing operations has a SaaS tool that lets analysts view recent clicks for debugging. An analyst's account was credential-stuffed; the attacker viewed link-click logs, extracted magic tokens, and used them within the validity window.

**Fixes.**
- Magic links must not pass through marketing click-trackers. Auth emails sent through a dedicated transactional path with no tracking wrapper.
- Magic link bound to the device that requested it (cookie set when the user typed the email; the link only works in that browser session). The token alone is insufficient.
- Magic link single-use enforced atomically; concurrent click in another browser fails.
- Step-up: any login from a new device triggers "we noticed a new device" email with a 15-minute reverse window.
- Audit: every internal tool that touches user email content is gated behind 2FA, and access is logged.

**Lessons.**
- Bearer credentials in URLs are credentials wherever the URL travels — email, browser history, server logs, clipboard, click-trackers.
- Internal tools are part of the attack surface. Their access controls must match the data they expose.
- Authentication emails are not marketing emails. They share infrastructure at your peril.

### Case Study B — Stored XSS via Profile Field, Amplified by Internal Tool

**Scenario.** A user-facing app has a "bio" field, sanitized server-side. An internal admin tool renders bios in a different rendering path (Markdown with raw HTML allowed) so support can see the content as users might. An attacker injects a payload that does nothing in the public app but executes in the support tool, exfiltrating session cookies of every support agent who views the profile.

**What broke.** Two rendering paths with two sanitizers, neither path agreed on what was safe. The attacker found the gap.

**Fixes.**
- Single sanitization pipeline applied at storage time, not at render time. The raw user input is stored, but a normalized, safe representation is also stored and used by all renderers.
- Internal tools render user content in a sandboxed iframe with a strict CSP, regardless of sanitizer assumptions.
- Support sessions use scoped tokens (read-only by default) so a stolen support cookie can't immediately do damage.
- Periodic red-team exercise on internal tools, not just public surfaces.

**Lessons.**
- Defense in depth is necessary because sanitizers fail. The CSP and the iframe sandbox were the controls that should have caught this independent of the sanitizer mismatch.
- Internal tools deserve the same review rigor as customer-facing endpoints.

### Case Study C — Cache Poisoning on a Multi-Tenant CDN

**Scenario.** A SaaS product serves tenant content from `tenant-id.app.example.com`. The CDN caches by URL. A bug in cache key generation included only the path, not the host, after a routing refactor. Within minutes, tenant A users were seeing tenant B's cached content.

**What broke.**
- Cache key didn't include `Host`.
- The bug was in a config file change reviewed by engineers who weren't aware of the cache key implications.
- No alerts fire on "tenant boundary crossing" because the metrics layer also doesn't include host in its keys.

**Fixes.**
- Cache key derivation lives in code, not config, and is unit-tested with an explicit assertion: `(tenant_a, path) != (tenant_b, path)` produces distinct keys.
- Synthetic test in CI: make a request as tenant A, then as tenant B for the same path, assert different responses. Run before any CDN config change.
- Tenant-id as an HTTP response header, asserted by the CDN to match the tenant of the requesting cookie/token. Mismatch → bypass cache, log alert.
- Compliance audit: every cache layer (CDN, internal HTTP cache, app-level memoization) reviewed for tenant-key correctness.

**Lessons.**
- Cache keys are a security control. They deserve code, tests, and review.
- Multi-tenant systems must have cross-tenant invariants explicitly tested and monitored.
- Config changes can be more dangerous than code changes because they're often less reviewed.

### Case Study D — Insider Data Exfiltration via Backup Copy

**Scenario.** A departing engineer copies a production backup to their personal cloud. Discovered a week later via cloud cost anomaly. The backup contains hashed passwords and PII for millions of users.

**What broke.**
- Production backups stored in a bucket readable by all of engineering.
- Egress from the corporate network to personal cloud accounts was unrestricted.
- Audit log of bucket reads existed but wasn't monitored.

**Fixes.**
- Backups encrypted with KMS keys that only the restore service can use. An engineer who copies the bytes has unusable ciphertext.
- Production data access is JIT, ticket-tied, time-bound. Standing access removed.
- Egress controls block copying TB-scale data to personal accounts; alerts at lower thresholds.
- Off-boarding checklist includes revoking all cloud credentials within 1 hour of termination notice.
- Bucket access logs streamed to anomaly detection: a single user reading 10x their normal volume pages security.

**Lessons.**
- Insider risk is real and is rarely malicious — it's often "this is convenient" with no malice.
- The control that worked retrospectively (cost anomaly) is fine; the control that should have worked preventively (KMS encryption + access controls) wasn't there.
- Detection is necessary but not sufficient. Make the bad action mechanically harder.

### Case Study E — Privilege Escalation Through a Forgotten Debug Endpoint

**Scenario.** A `/debug/impersonate` endpoint, gated behind a feature flag set to off in prod, was used in staging to test admin flows. A misconfigured flag service returned `unknown_flag → default_value=true` for a few minutes during a deploy. Any authenticated user during that window could impersonate any other user.

**What broke.**
- Debug endpoint existed in production code, gated only by a single flag check.
- Flag service defaulted unknown flags to `true` (a "fail-open" choice made years earlier for unrelated reasons).
- No additional check — like "request must come from internal IP" or "actor must have an admin role" — backed up the flag.

**Fixes.**
- Production builds exclude debug endpoints entirely (build flag, not runtime flag). Different binaries for staging vs prod.
- Flag service defaults unknown flags to the *secure* state per flag, configured at flag definition time.
- Defense in depth: even when enabled, impersonation requires an admin role *and* a recent fresh-login *and* a ticket reference *and* notifies the impersonated user.
- Penetration test on every endpoint enumerated from the routing table, including those not in the public API docs.

**Lessons.**
- "Disabled in prod" is not a security boundary. Make the disable mechanism layered.
- Compile-time elimination beats runtime gating for high-risk code.
- Flag systems' default behavior is a security primitive. Audit it.

---

## Section 9 — Detection, Response, and Resilience

### Q9.1 What does a useful security telemetry stack look like for an app team?

- **Authentication events:** every login attempt (success + failure), MFA challenge result, password change, MFA enroll/disable, session creation, token issuance and revocation, password reset.
- **Authorization decisions:** every deny, plus a sample of allows, with subject, action, resource, policy version.
- **Sensitive actions:** money movement, PII export, admin actions, impersonation, key creation/rotation.
- **Anomalies:** geo deltas, device deltas, velocity spikes, behavioral outliers.

These feed both real-time alerting (page on-call for high-severity patterns) and offline review (weekly anomaly digest, quarterly audit).

**Corner cases:**
- Don't log only failures. The pattern "many successes from one source" is often the actual breach signal.
- Logs must be append-only and shipped off-host quickly. An attacker with shell access who can edit logs has erased your investigation.
- Synthetic monitoring of the security pipeline itself — a fake-attacker job that triggers known signatures and verifies the alert fires. Otherwise the pipeline can be silently broken for weeks.

### Q9.2 Walk through your incident response runbook for a credential leak.

**Hour 0–1:**
- Verify the leak (is it real, is it current, what scope).
- Rotate the compromised credential.
- Identify all derived credentials and rotate them.
- Page on-call security; declare incident commander.

**Hour 1–4:**
- Audit log review for usage of the leaked credential during the leak window.
- Detect any abnormal activity (writes from unfamiliar sources, data egress).
- Communicate internally — who needs to know (legal, comms, exec, customer success).

**Hour 4–24:**
- If user/customer data was accessed without authorization: notification timeline begins. GDPR is 72 hours; some US states are tighter.
- External communications drafted and reviewed by legal and PR.
- Post-incident lockdown: enhanced monitoring, additional MFA challenges, temporary feature disables.

**Days–weeks:**
- Postmortem: blameless, technical, with concrete actions.
- Customer notifications and support response.
- Regulatory disclosure as required.

**Corner cases:**
- The leaked credential gives access to other systems via SSO trust. Map the trust graph and rotate transitively.
- The leak is in a backup or log. Even if rotated, the bytes exist; disclose conservatively.
- The credential was leaked weeks ago; you're discovering it now. The blast radius is everything that happened in the interval.

### Q9.3 How do you ensure security doesn't cause its own outages?

This is the most overlooked staff-level concern. Security controls have outages too, and a fail-closed control during its own outage is indistinguishable from an attack.

**Practices:**
- **Decide fail-open vs fail-closed per control, deliberately.** WAF: fail-open. Auth: fail-closed. Authz cache: fail-open with tight TTL. Document the decision.
- **Multi-region for security infra.** IdP, KMS, secrets manager, policy decision points all multi-region with automatic failover. Test the failover.
- **Cache aggressively, expire safely.** Authz decisions cached for seconds, JWKS for minutes, allow-stale-on-error.
- **Break-glass paths.** A documented, audited, hardware-token-protected path to bypass normal auth in a true emergency. Tested quarterly. Used precisely zero times in normal operation.
- **Game days.** Simulate IdP outage, KMS outage, certificate expiry, secrets manager outage. Measure customer impact. Fix the parts that hurt.

The principle: security infrastructure is critical-path infrastructure. It deserves the same reliability investment as your primary database.

---

## Section 10 — Quick-Fire Decision Prompts

- **Same library across services for crypto?** Yes. Roll-your-own diverges; libsodium/Tink converges.
- **Allow `eval` anywhere in the JS bundle?** No. CSP `'unsafe-eval'` off; lint rule banning `eval`, `new Function`.
- **Random ID for security purposes via `Math.random`?** No. CSPRNG only.
- **Bcrypt cost factor 10 in 2026?** Too low. Aim for ~250ms hash time on target hardware (cost 12+ for bcrypt, or Argon2id with tuned parameters).
- **Encrypt PII column-level when disk is already encrypted?** Yes. Disk encryption protects against stolen disk; column encryption protects against stolen DB query.
- **Allow `*` in CORS for an API?** Only if the API is genuinely public-read with no credentials. With credentials, never `*` — explicit allowlist.
- **Trust `User-Agent` for security decisions?** No. Client-controlled.
- **Trust `X-Forwarded-For` for IP-based security decisions?** Only after the LB has stripped client-supplied values.
- **Leave default credentials in any config?** No. CI scan that fails on known-default strings.
- **Log the request body for debugging?** No. Log the request *envelope* (method, path, status, duration) and a redacted summary. Never raw body of authenticated requests.
- **One Vault namespace for all environments?** No. Per-environment with separate root credentials.
- **Same KMS key for prod and staging?** No. Separate keys, separate IAM permissions.
- **Allow service-to-service traffic to skip TLS in private VPC?** No. mTLS everywhere, no exceptions; VPC perimeter is not a trust boundary.
- **Run security tooling synchronously on the request hot path?** No. Score asynchronously; gate sensitive actions on the score.
- **Treat a private GitHub repo as a secret store?** No. Anyone with read access reads it; many integrations have read access.

---

## Section 11 — Senior vs Staff Framing

A senior engineer ships a secure feature. A staff engineer leaves the system more secure than they found it.

For "we need to launch this new endpoint":
- **Senior:** input validation, authn check, authz check, logging, tests. Done.
- **Staff:** same, plus — what threat model does this endpoint live in, what other endpoints share its trust boundary, what telemetry detects abuse, what does the rollback look like, what changes about the team's on-call burden, what is *not* being built (in-house WAF, custom rate limiter, hand-rolled crypto), and what's the migration path for the existing similar endpoints to share the same controls.

The staff lens treats security as a property of the *system over time*, not the feature in front of you. The unit of work is "the org's security posture moved forward," not "this PR has no findings."