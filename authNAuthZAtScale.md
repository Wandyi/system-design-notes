# Authentication & Authorization at Scale — Staff-Level Q&A

A reference of staff-level scenarios covering identity, session management, token strategies, delegated access, multi-tenant authorization, zero-trust, and operational concerns.
Each section frames a realistic problem, the trade-offs, and the design a staff engineer is expected to defend in a system design interview or design review.

---

## Section 1 — Foundational Concepts (Calibration)

### Q1.1 Distinguish authentication, authorization, and accounting. Why do real systems blur the line and what damage does that cause?

- **AuthN** answers *who is the principal*. AuthN produces a verified identity (user, service, device).
- **AuthZ** answers *what can this principal do on this resource at this time*. AuthZ produces a decision (permit/deny) given (subject, action, resource, context).
- **Accounting / audit** answers *what did they actually do*, and is the only line of defense once the first two fail.

Systems blur them because:
- JWTs frequently embed roles/scopes, so the *identity token* becomes the *authorization decision*. Revocation then requires either short TTLs or stateful blocklists.
- "Admin" roles in monolith DBs grow into both an identity claim and a permission set. Splitting later requires a migration that touches every service.
- API gateways do "auth" — meaning a mix of token validation and coarse policy checks. Downstream services then trust the gateway and skip their own checks. A gateway bypass becomes a full compromise.

The damage: tokens can't be revoked instantly, audit logs lose the *why* of a decision (only the outcome), and least-privilege becomes impossible because permissions are baked into identity claims rather than evaluated per request.

### Q1.2 Why is "stateless authentication" a misleading phrase?

A JWT is stateless on the *verifier* side, but the system is not stateless overall:
- Revocation lists, refresh tokens, and session bindings live somewhere stateful.
- Key rotation (JWKS) requires distribution and caching state.
- "Logout everywhere," password reset, and account compromise all need a stateful counter (e.g., `tokens_issued_after`) to invalidate prior tokens.

The accurate framing is: *the verifier does not need a session lookup on the hot path*, at the cost of weaker revocation semantics and more complex key management.

---

## Section 2 — Session, Token, and Credential Strategy

### Q2.1 You inherit a system using long-lived JWTs (24h) embedding roles, scopes, and tenant ID. The CISO asks for sub-minute revocation. Walk through the redesign.

**Problem.** JWT verification is local; revocation requires either expiry or shared state. 24h TTL means a stolen token is valid for 24h.

**Options and trade-offs.**

1. **Short-lived access token + refresh token (recommended baseline).**
   - Access token TTL: 5–15 minutes; refresh token TTL: hours to days, rotated on use.
   - Refresh exchange happens at the IdP, which *can* check a revocation table cheaply because exchange is a relatively cold path.
   - Sub-minute revocation requires access TTL ≤ 60s — usually too aggressive due to JWKS/clock skew. The honest answer: you get *minute-grained* revocation, not sub-minute, without per-request server-side checks.

2. **Reference tokens (opaque) + introspection.**
   - Every request hits the IdP introspection endpoint or a cache fronted by it.
   - Revocation is instant. Cost is latency and a hard dependency on the IdP.
   - Mitigation: short-lived (sub-second) introspection cache at the gateway, keyed by token hash.

3. **JWT + revocation cache (hybrid).**
   - JWT carries identity; gateway/sidecar checks a Bloom filter or Redis set of revoked `jti`s.
   - False positives in Bloom filter trigger a definitive lookup. Memory bound is predictable.
   - Best of both: fast verification + fast revocation, at the cost of one extra in-memory check.

**What a staff engineer flags.** The actual question hidden in "sub-minute revocation" is usually "we had an incident." Ask whether the requirement is *all* tokens or *specific* tokens (compromised user, compromised service). The design differs:
- Mass revocation → bump a global epoch claim; all tokens with epoch < current become invalid.
- Targeted revocation → `jti` blocklist with TTL = remaining token lifetime.

### Q2.2 Compare cookie-based sessions vs bearer tokens for a B2C web + mobile + third-party API product.

| Concern | Cookie session (server-side) | Bearer token (JWT/opaque) |
|---|---|---|
| CSRF | Vulnerable; requires SameSite + CSRF tokens | Not applicable (no ambient auth) |
| XSS | `HttpOnly` mitigates token theft | `localStorage` tokens are stolen by any XSS; even memory storage leaks via service workers |
| Mobile clients | Awkward (cookie jars, no domain alignment) | Native fit |
| Third-party API access | Requires OAuth on top anyway | Native fit |
| Revocation | Trivially server-side | Needs TTL + revocation infra |
| Cross-origin | CORS + credentials; cumbersome | Authorization header is clean |

**Staff-level answer.** Don't pick one — pick by surface:
- Web app: cookie-based session with `HttpOnly`, `Secure`, `SameSite=Lax`, and a separate CSRF token for mutating endpoints.
- Mobile: OAuth 2.0 with PKCE, refresh tokens stored in Keychain/Keystore.
- Third-party API: OAuth 2.0 with scoped access tokens.
- All three terminate at the same IdP, which exchanges credentials for the appropriate session/token type per surface.

### Q2.3 Where do refresh tokens belong, and how do you stop refresh-token theft from being game-over?

Refresh tokens are bearer credentials with longer TTL — losing one is worse than losing an access token. Defenses, layered:

1. **Rotation on use.** Each refresh exchange returns a new refresh token and invalidates the old one. If the old one is reused, it indicates theft → kill the entire token family (descendant chain).
2. **Token binding.** Bind to a device fingerprint (DPoP, mTLS, or device-bound public key). A stolen token without the private key is unusable.
3. **Sender constraint via PKCE** for public clients (no client secret available).
4. **Sliding expiration with absolute cap.** 30-day sliding, 90-day absolute, then forced re-auth.
5. **Storage.** Mobile: Keychain/Keystore. Web SPAs: do not store refresh tokens in JS-accessible storage. Use Backend-for-Frontend (BFF) pattern — refresh token lives in `HttpOnly` cookie scoped to the BFF, which exchanges it server-side.

**The "BFF for SPA" call** is the modern staff-level recommendation. SPAs holding refresh tokens in `localStorage` is a 2018 pattern that doesn't survive XSS.

### Q2.4 Design password reset for a global product. What goes wrong if you do it naively?

Naive design: email a link with a UUID token, expire in 1 hour, set new password.

What breaks at scale:
- **Account enumeration.** Returning "user not found" vs "email sent" distinguishes accounts. Always return the same response regardless.
- **Token reuse.** Token must be single-use *and* invalidate on password change *and* invalidate prior sessions (otherwise attacker who got in via stolen session keeps access).
- **Race condition.** User clicks link twice; two parallel requests both validate the token. Use atomic "consume token" with `UPDATE ... WHERE token = ? AND used = false RETURNING ...`.
- **Email-as-identity assumption.** What if the email account is compromised? For high-value accounts, require a second factor (SMS, authenticator) before accepting the reset.
- **Timing side channels.** bcrypt the *real* password even when the user doesn't exist, so response time doesn't leak account existence.
- **Cross-region delivery.** The token store must be reachable from wherever the user clicks the link. Either replicate globally with strong consistency or pin reset flow to the home region.

---

## Section 3 — Authorization Models

### Q3.1 RBAC vs ABAC vs ReBAC — when is each correct?

- **RBAC (Role-Based).** Permissions attach to roles; users have roles. Best when permissions are coarse and stable: internal admin tools, simple SaaS with `admin/member/viewer`.
  - Breaks when: permissions depend on the resource ("can edit *this* doc"), exception grants proliferate, or roles explode combinatorially.

- **ABAC (Attribute-Based).** Decisions are functions of subject, resource, action, and environment attributes (`subject.dept == resource.dept AND time_of_day in business_hours`). Best for compliance-heavy domains: healthcare, finance, regulated data.
  - Breaks when: policies become unreadable, evaluation requires fetching attributes from many systems on the hot path, or attribute drift makes "why did this allow/deny" unanswerable.

- **ReBAC (Relationship-Based, Zanzibar-style).** Permissions derive from a graph of relationships (`user X is editor of folder Y; folder Y contains doc Z; therefore user X can edit Z`). Best for collaborative products where sharing is the core feature: Drive, Figma, Notion, GitHub.
  - Breaks when: relationships are simple and a graph is overkill, or when policies need rich attribute conditions Zanzibar can't express.

**Staff-level answer.** Real systems combine them. Zanzibar handles "who has access to which object." ABAC layers on top for context-sensitive constraints ("editor, but only during working hours from corporate IPs"). RBAC remains as a coarse pre-filter before the more expensive checks.

### Q3.2 Design the authorization layer for a Google-Drive-class product. What is Zanzibar and why did Google build it?

**The problem Zanzibar solves.** A doc has owner, editors, viewers. Folders inherit permissions. Groups can be members of other groups. Sharing creates relationships. A check ("can Alice read doc X?") may require traversing groups → folders → docs across sharded data.

Doing this in a relational DB collapses under join depth and scale. Doing it ad-hoc per service produces inconsistent policy.

**Zanzibar's design.**
- **Relation tuples**: `(object#relation@user)` — e.g., `(doc:42#viewer@user:alice)` or `(doc:42#viewer@group:eng#member)` (userset rewrites).
- **Namespaces** define how relations compose: `viewer = viewer ∪ editor ∪ owner`, `viewer = viewer + parent.viewer` (folder inheritance).
- **Check API**: "does Alice have viewer on doc:42?" — recursive evaluation following the namespace rules.
- **Zookies (consistency tokens)**: solve the "new ACL not yet replicated" problem. After writing an ACL change, the writer gets a zookie; reads passing that zookie are guaranteed to see at least that version.

**What a staff engineer designing a Drive clone says.**
1. Use a Zanzibar-style service (OpenFGA, SpiceDB, or build on Spanner/CockroachDB). Don't reinvent it in your monolith.
2. Cache check results aggressively but bound by zookie freshness — never serve a cached *allow* if the user just got revoked.
3. The check API is on the hot path of every read; budget it at single-digit ms p99. If your DB can't, you've chosen wrong.
4. **List endpoints are the hard part.** "Show all docs Alice can see" is `O(N)` checks naively. Zanzibar offers `Expand` and `Lookup` APIs, but at scale you maintain a search index reflecting the permission graph and rebuild on changes.

### Q3.3 You have a multi-tenant SaaS where one tenant's admin should manage their tenant only, but a "support" role at your company can act on behalf of any tenant. How do you model and enforce this safely?

**The two failure modes to design against:**
1. *Cross-tenant data leak* — query forgets `WHERE tenant_id = ?`, returns another tenant's rows.
2. *Privilege escalation via support* — support engineer abuses access, or their account is compromised.

**Design.**

- **Tenant isolation in the data layer.**
  - Postgres row-level security policies keyed on a session variable set from the JWT's `tenant_id`. Even a forgotten `WHERE` clause is filtered by RLS. Defense in depth.
  - Or schema-per-tenant for stronger isolation at the cost of operational complexity.

- **Two distinct identities for support actions.**
  - Support engineer authenticates as themselves (their own SSO identity).
  - To act on a tenant, they request an *impersonation grant* tied to a ticket/case ID. The grant produces a token with both `acting_as: tenant_X` and `actor: support_engineer_Y` claims.
  - Audit logs record both. The tenant sees in their own audit log "support engineer Y acted on your account at time T for ticket Z."

- **Approval workflow for sensitive actions.**
  - Read access: self-service grant, time-limited (e.g., 4 hours).
  - Write access: requires a second support engineer's approval (4-eyes). Logged. Tenant-notified.
  - Production-data export: requires the tenant's own consent via signed link.

- **Detection.**
  - Anomaly detection on impersonation: support engineer accessing 50 tenants in an hour, accessing a tenant they have no ticket for, accessing outside business hours.

**The principle.** Impersonation is not "support becomes the user" — it is "support performs an action *as themselves* with delegated authority." Audit, scope, and time-bound it.

### Q3.4 Walk through OAuth 2.0 scopes vs fine-grained permissions. When does a scope-only model break?

**Scopes** are coarse: `read:repo`, `write:repo`, `admin:org`. They live in the access token and are checked at the API boundary.

**Fine-grained permissions** are per-resource: "can this token write to *this specific repo*?" — checked by the resource server against an authorization service.

Scopes break when:
- Third-party apps need access to *some* of a user's resources, not all. A `write:repo` scope grants every repo, which is over-broad.
- Compliance requires least-privilege at resource granularity (HIPAA, PCI).
- Users expect "share this folder with this app" semantics (Drive, Dropbox).

**Modern pattern (GitHub fine-grained PATs, Google Drive picker, etc.):**
- Scopes describe the *kinds* of operations.
- Resource selection is bound at consent time and embedded in the token (or referenced by an opaque grant ID the resource server resolves).
- The resource server enforces both — scope is necessary but not sufficient.

---

## Section 4 — Federation, SSO, and Identity Providers

### Q4.1 Compare SAML, OIDC, and OAuth 2.0. Where does each belong?

- **OAuth 2.0** — *delegated authorization*. "Let this app access my resource." Doesn't define authentication. Misusing OAuth as an auth protocol (e.g., trusting the access token as proof of identity) is a classic security bug.
- **OIDC** — *authentication on top of OAuth 2.0*. Adds an `id_token` (JWT) with verified user identity claims. This is what you want for "log in with Google."
- **SAML 2.0** — *enterprise SSO*. XML-based, signed assertions, browser-redirect-driven. Heavy, but every enterprise IdP (Okta, Azure AD, Ping) speaks it. B2B SaaS must support it.

**Staff-level decision.** New consumer flow: OIDC. Enterprise B2B: support both SAML and OIDC, with SAML often required for procurement. Service-to-service: OAuth 2.0 client credentials grant or mTLS with SPIFFE.

### Q4.2 Your B2B SaaS needs to support customer-managed SSO (each tenant brings their own IdP). What changes?

- **Per-tenant IdP configuration.** Each tenant uploads SAML metadata or registers an OIDC issuer. Stored encrypted at rest. Validated on registration.
- **IdP-initiated vs SP-initiated.** Support both, but IdP-initiated has CSRF-like risks (no `RelayState` validation can let an attacker replay assertions). Validate signature, audience, `NotBefore`/`NotOnOrAfter`, and a server-side state nonce.
- **JIT provisioning.** First login from a tenant's IdP creates the user record. Map IdP attributes to internal roles via a tenant-configurable mapping.
- **SCIM** for provisioning/deprovisioning (otherwise users keep access after they leave the customer's company — a very real breach vector).
- **Domain verification.** When a tenant claims `@acme.com`, prove ownership before binding the SSO connection — otherwise anyone can register an "acme" tenant and harvest logins.
- **Fallback admin.** Always preserve a non-SSO break-glass admin account; tenant SSO outages must not lock the customer's owners out.

### Q4.3 Why is "Login with Google" insufficient for an enterprise product?

- No SAML — many enterprises mandate it.
- No SCIM — no automated deprovisioning.
- No conditional access (device posture, IP restrictions).
- No tenant isolation — a personal Gmail and an enterprise Workspace look identical to your app unless you check the `hd` claim *and* trust it (attackers can spoof `hd` if you accept tokens from any Google account without verifying the audience and issuer).
- The customer wants their IdP to be the source of truth, not Google's consumer identity.

---

## Section 5 — Service-to-Service & Zero Trust

### Q5.1 How do services authenticate each other in a zero-trust microservices environment?

The shift: don't trust the network. Every request between services authenticates the caller, regardless of VPC or namespace.

**Mechanisms, ranked by maturity:**

1. **mTLS with SPIFFE/SPIRE.** Each workload gets an X.509 SVID issued by SPIRE based on attestation (Kubernetes service account, EC2 instance role, etc.). Identities are short-lived (hours), auto-rotated. Service mesh (Istio, Linkerd) handles the TLS termination. The receiving service reads the peer's SPIFFE ID from the cert and authorizes against it.
2. **OAuth 2.0 client credentials + JWT.** Services obtain JWTs from an internal IdP. Simpler than mTLS but doesn't authenticate the network path; a stolen JWT works from anywhere.
3. **Signed requests (AWS SigV4-style).** Each request signed with a workload key. Good for stateless edges but key distribution is its own problem.

**Authorization layer on top.** SPIFFE ID alone doesn't authorize. You still need a policy engine (OPA, Cedar) consulted by the service or its sidecar to decide which calls are allowed: "service-A may call service-B's `GET /orders` but not `DELETE /orders/*`."

### Q5.2 Where does API key authentication still belong, and where does it not?

**Belongs.**
- Server-to-server integrations where the partner can't or won't run an OAuth client.
- Webhooks (HMAC-signed body using a shared key — different from bearer keys).
- Build/CI tokens with narrow, well-known permissions and aggressive rotation.

**Does not belong.**
- User-facing authentication (no MFA, no session revocation primitives).
- Anything that needs delegated access — that's OAuth's job.
- Long-lived secrets in mobile or browser code.

**Operational requirements when you do use API keys.**
- Show the key once at creation; store hashed.
- Bind to a service account, not a user (so it survives offboarding).
- Per-key scopes and IP allowlists.
- Last-used timestamp; auto-warn / auto-disable on inactivity.
- Programmatic rotation API; secrets manager integration.

### Q5.3 Your security team mandates "no long-lived credentials anywhere." How do you remove static AWS keys, DB passwords, and service account tokens from your fleet?

- **AWS keys → IRSA (IAM Roles for Service Accounts) on EKS, or instance profiles on EC2.** Workloads assume roles via STS; credentials are short-lived and never written to disk.
- **DB passwords → IAM database authentication** (RDS), or a sidecar that fetches a fresh password from Vault per connection (Vault dynamic secrets), or mTLS to the DB with a SPIFFE cert.
- **Service account tokens → workload identity federation.** GCP and AWS support exchanging Kubernetes-projected SA tokens for cloud credentials. SPIFFE IDs broker this.
- **Third-party API keys → secrets manager + rotation hooks.** When the third party doesn't support OAuth, accept the API key but rotate it on a schedule with a secrets manager (HashiCorp Vault, AWS Secrets Manager) and hot-reload in the app.
- **Build pipelines → OIDC federation.** GitHub Actions, GitLab CI, Buildkite all issue OIDC tokens that AWS/GCP/Azure can exchange for short-lived credentials. No more long-lived deploy keys in CI.

The pattern is consistent: replace static credentials with short-lived ones derived from a verifiable workload identity.

---

## Section 6 — High-Stakes Edge Cases

### Q6.1 A user reports their account was accessed from another country. What is the right response, and what should already exist?

**Response runbook (assumes infrastructure exists):**
1. Force logout of all sessions for that user. Bump their `tokens_issued_after` epoch; all current tokens become invalid on next request.
2. Force password reset and require MFA enrollment if not present.
3. Audit recent actions: data exports, API key creations, permission grants, money movements. Surface them to the user.
4. If sensitive actions occurred: compensating actions (refund, revert permission, revoke API keys created in the suspicious window).
5. Notify the user out-of-band (email + SMS) with the timeline.

**What should already exist:**
- A "logout everywhere" primitive (epoch claim or session table).
- Per-action audit logs immutable to the user themselves.
- Anomaly detection that surfaces *before* the user reports it: impossible travel, new device + sensitive action within minutes, MFA disablement followed by password change.
- Cooling-off windows: changing email or disabling MFA invalidates active sessions and triggers a 24h hold on sensitive actions (money out, data export).

### Q6.2 Your IdP is down. What still works?

This is a staff-level question because the answer reveals the architecture.

- **JWT-based access tokens with cached JWKS** — services can validate tokens for the duration of the cache TTL and the token's lifetime. Authentication continues to work for already-logged-in users.
- **Login is broken.** New sessions can't be issued.
- **Token refresh is broken** unless the IdP is highly available with multi-region failover.
- **mTLS service-to-service** continues if the cert authority is separate (SPIRE servers should be regionally redundant) and existing certs haven't expired.

**Mitigations.**
- IdP must be active-active multi-region with a globally replicated user store. RTO measured in seconds.
- JWKS cached with stale-while-revalidate semantics; tokens validated against any cached key set during outage.
- Token TTLs sized so a 30-minute IdP outage is invisible to logged-in users.
- Break-glass admin path that doesn't depend on the IdP (hardware token, separate emergency credential vault). Audited heavily.

### Q6.3 You have a feature that lets users grant other users access. How do you prevent confused-deputy and CSRF-on-grant attacks?

**Confused deputy** — service A holds privileges B doesn't have, and B tricks A into using them on B's behalf.

Concrete: a "share via link" feature. The link, when clicked, calls `POST /share/accept?token=...`. If the share target isn't validated against the *clicker's* identity, an attacker emails the link to a victim, the victim's browser auto-authenticates, and the share is accepted into the *attacker's* control.

**Defenses:**
- Share tokens are not bearer credentials — they encode "this object is sharable" plus optionally "to this email address." Acceptance binds to the *authenticated user's* identity, not the token holder.
- Email-bound tokens — only the user logged in with the matching email may redeem.
- All grant operations require an explicit user gesture *and* a CSRF token (or `Sec-Fetch-Site: same-origin` enforcement).
- Display the action to be performed and the resulting state before completion.

### Q6.4 You're rate-limiting login attempts. What goes wrong with naive per-IP limits?

- **Shared NAT** — a corporate office or mobile carrier behind one IP gets locked out collectively.
- **IPv6 sparsity** — attacker rotates through a /64; per-IP doesn't bind.
- **Distributed credential stuffing** — a botnet of 10k IPs each tries 5 logins; per-IP limits never trigger but the account is breached.

**Layered approach:**
- Per-account exponential backoff on failures (with cap, to prevent DoS-by-bad-password from locking real users out — emit a CAPTCHA at threshold instead of hard-locking).
- Per-IP and per-IP-prefix (/24 for v4, /64 for v6) limits as a secondary signal.
- Device fingerprinting and cookie-based "this browser has succeeded before" allowance.
- Anomaly detection on the *account population* — sudden spike in failures across thousands of accounts indicates credential stuffing; respond with global CAPTCHA or temporarily disabling password login in favor of email magic links.
- Bot detection (hCaptcha, reCAPTCHA Enterprise) gating high-risk attempts.
- Have-I-Been-Pwned-style breached password checks at login time, prompting re-credentialing.

---

## Section 7 — Operational & Organizational

### Q7.1 How do you roll out a new authorization model (e.g., RBAC → ReBAC) in a live system without breaking it?

This is the question that separates senior from staff. Migrations, not greenfield.

**The pattern:**
1. **Dual-write.** New writes go to both old and new authorization stores.
2. **Backfill.** Translate existing RBAC grants into ReBAC tuples. Validate row counts and spot-check.
3. **Shadow read.** On every check, query the new system *as well as* the old one. Compare. Log mismatches without failing the request. Drive the mismatch rate to zero.
4. **Cutover, dark launch.** Behind a feature flag, start using the new system's decision. Old system still runs as a comparator and fallback.
5. **Fail-closed validation.** Once mismatches are zero for a sustained period, switch the old system to read-only, then decommission.

**Staff-level wrinkles:**
- Negative permissions (deny rules) translate poorly between models — surface them early.
- Default-allow vs default-deny mismatches between systems are silent landmines. Force default-deny in the new system and explicitly grant.
- Customer-visible permissions UI must keep showing the same labels even when the underlying model changes.

### Q7.2 How do you run an authorization service that thousands of services depend on, without it becoming a single point of failure?

- **Embed the policy decision point (PDP), not just call it.** OPA-as-a-sidecar, Cedar-as-a-library. Policies and data fetched from a control plane and cached locally. Decisions are local, sub-millisecond, and survive control-plane outages.
- **Bound the data the PDP needs.** Don't ship the entire user/permission graph to every sidecar. Push relevant slices (e.g., "this service's relevant policies and the subset of users who can hit it") via partial evaluation.
- **Versioned policy bundles.** Atomic rollouts; rollback is reverting to a previous bundle. Canary by service tier.
- **Decision logs streamed asynchronously.** PDPs emit decisions to a central log for audit and anomaly detection. Failure to ship logs must not block decisions.
- **Test policies as code.** Every policy change has unit tests. Critical decisions have property-based tests ("no policy version ever grants `delete:any` to a non-admin").

### Q7.3 Your auth code is sprinkled across 40 microservices. Each implements `userHasAccess()` slightly differently. How do you converge?

The technical answer is "extract a shared library or sidecar." The staff-level answer addresses the social problem.

- **Standardize the question, not the answer.** All services must phrase auth checks the same way: `(subject, action, resource, context) → decision`. That alone eliminates half the inconsistencies.
- **Provide the easiest path.** A library that does the right thing in 5 lines, with sensible defaults. If using it is harder than rolling your own, teams will roll their own.
- **Make the wrong path visible.** Lint rules, code review checklists, and runtime telemetry that flags services not emitting standardized authz decision logs.
- **Consolidate on a schedule.** Don't try to converge all 40 in a quarter. Pick the highest-risk five (anything touching money, PII, admin), migrate, prove the value, then propagate.
- **Own the migration.** A platform team that does the work *for* product teams, not one that hands them a library and a deadline.

### Q7.4 What does "least privilege" mean operationally for a 500-engineer org, and how do you keep it from becoming "everyone has admin because requesting access takes a week"?

- **Just-in-time elevation, not standing access.** Engineers have read-only baseline. Write/admin obtained via short-duration grants (PIM, Teleport, Vault) tied to a ticket, with auto-expiry.
- **Self-service approval.** Most grants approved by a peer in seconds, not by a security team in days. Risky grants escalate.
- **Default-deny, exceptions audited.** A monthly review pulls every standing high-privilege grant and asks the owner to re-justify or revoke.
- **Make safe paths fast.** If "deploy via the safe pipeline" is slower than "ssh to prod," people will ssh to prod. Invest in tooling so the secure path is the path of least resistance.
- **Detect, don't only prevent.** No prevention catches everything. Anomaly detection on access patterns (someone reading a table they've never read; admin action outside business hours) is the safety net.

The operational principle: least privilege only sticks when the secure path is also the convenient path. Otherwise it decays into security theater.

---

## Section 8 — Quick-Fire Decision Prompts

These are the kinds of questions a staff engineer should answer in under a minute, with confidence.

- **JWT in `localStorage` for an SPA?** No. BFF pattern with `HttpOnly` cookie holding the refresh token, access token in memory.
- **Same JWT signing key across all services?** No. Asymmetric keys (RS256/EdDSA), public keys distributed via JWKS, private key only at the IdP.
- **Roll your own password hashing?** No. Argon2id with appropriate parameters, or bcrypt if Argon2 isn't available. Never SHA-anything.
- **Storing TOTP secrets in plaintext in the user table?** No. Encrypted at rest with a KMS key, ideally per-user envelope encryption.
- **Magic link login as a primary auth method?** Acceptable for low-risk consumer products; weak for high-risk (email account compromise = full takeover). Pair with device binding or step-up MFA for sensitive actions.
- **Rotating JWT signing keys without breaking active tokens?** Publish both old and new keys in JWKS during rotation. Sign new tokens with new key, accept tokens signed with either, drop old key after max-token-lifetime has elapsed.
- **Putting tenant ID in the URL path vs the JWT?** Both. URL for routing/observability, JWT for authoritative authorization. Mismatch → reject the request.
- **Allow users to disable MFA via support ticket?** Only after strong out-of-band verification. This is the #1 social-engineering vector for account takeover.
- **Letting `prod` and `staging` share an IdP?** Tenants/realms can be shared, but signing keys, sessions, and audit logs must be isolated. A staging compromise must not yield prod tokens.
- **Trusting `X-Forwarded-For` for security decisions?** Only after the load balancer has stripped client-supplied values and replaced them. Otherwise the attacker controls the IP your auth code sees.

---

## Section 9 — What Senior vs Staff Answers Look Like

A senior engineer answers *what* to build. A staff engineer answers *what to build, what tradeoffs, what to migrate from, what to monitor, and what to say no to.*

For a question like "design the auth system for our new product":
- **Senior:** OIDC with an IdP, JWT access tokens, refresh tokens, RBAC roles, MFA via TOTP.
- **Staff:** Same, plus — which IdP and why, how we'll migrate the existing user base, BFF for the SPA, mTLS for service-to-service, OPA for policy, decision logs to detect drift, dual-write during model migration, break-glass paths, what we'll *not* build (custom IdP, in-house WebAuthn, our own password reset flow), and which two engineers will own the on-call rotation for the auth service in year one.

The staff answer treats authentication and authorization as a *product* with stakeholders, lifecycle, and failure modes — not a feature to bolt on.
