# NGINX Ingress ➜ Istio Gateway API Migration (Zero-Downtime) — using Azure Application Gateway (As-Is)

This README describes a **zero-downtime** migration runbook for moving an application from **NGINX Ingress Controller** to **Istio Gateway API (gateway.networking.k8s.io)** on AKS, while keeping **Azure Application Gateway** as the primary entry (As-Is).

Architecture overview (per attached diagram):
- Layer 1: **Azure Application Gateway** (existing)
- Layer 2: **AKS Service type=LoadBalancer** (public/private IP per ingress)
- Layer 3: **Ingress data plane**
  - Existing: `nginx ingress controller` (IP A / IP B)
  - New: `istio ingress gateway` (IP C / IP D)
- Layer 4: **Application / Microservices** (unchanged; no redeploy)

---

## Goals & Constraints

✅ **1) No downtime during switch**  
✅ **2) Minimal regression testing** (focus on critical paths and compatibility checks)  
✅ **3) Rollback must be possible** (switch back to NGINX quickly)  
✅ **4) No application/microservice redeploy required**

---

## Key Idea

We run **NGINX and Istio ingress in parallel** (“dual-run”), validate Istio using **direct access to the new LB IPs (C/D)**, then perform a **single cutover** by switching Azure Application Gateway backend from **NGINX IPs (A/B)** to **Istio IPs (C/D)**.

> Azure Application Gateway does **not** support traffic weighting split (e.g., 90/10).  
> Therefore, the safe strategy is **Blue/Green switch** + **health probes** + **connection draining**, with fast rollback.

---

## Terminology

- **As-Is NGINX IPs**: `IP_A` (AKS-01), `IP_B` (AKS-02)
- **New Istio GW IPs**: `IP_C` (AKS-01), `IP_D` (AKS-02)
- **Backend Pools (AppGW)**:
  - `nginx_pool`: { IP_A, IP_B }
  - `istio_pool`: { IP_C, IP_D }  ← new
- **Gateway API resources**:
  - `GatewayClass` (platform-owned)
  - `Gateway` (platform-owned)
  - `HTTPRoute` (app-owned)

---

## Responsibilities (recommended)

### Platform Team
- Install & operate **Istio control plane** and **istio ingress gateway** data plane.
- Create/maintain `GatewayClass` and shared `Gateway`.
- Provide guardrails (admission policy / templates) for `HTTPRoute`.
- Configure Azure Application Gateway backend pools, health probes, and draining.

### App Team
- Create/maintain `HTTPRoute` in application namespace.
- Validate routing behavior via test endpoints / direct LB IP.
- No microservice redeploy required.

---

## Pre-Migration Checklist (must pass)

### A) Baseline current behavior (NGINX)
- ✅ Current public hostnames and paths list
- ✅ TLS termination point (AppGW vs ingress)
- ✅ Required headers behavior (e.g., `Host`, `X-Forwarded-*`)
- ✅ Timeouts and max body size expectations
- ✅ WebSocket/long-lived connections (if any)
- ✅ Health endpoint used by AppGW probe (`/healthz`, `/ready`, etc.)

### B) Observability readiness (minimum)
- ✅ AppGW backend health view is working
- ✅ Basic metrics/logs for istio ingress gateway are visible
- ✅ Error budget guardrails for cutover (5xx threshold, latency threshold)

---

## Phase 0 — Prepare Istio Gateway API (No prod impact)

> Objective: bring up Istio ingress gateway and Gateway API resources **without changing production traffic**.

### 0.1 Install/enable Istio (platform)
- Deploy `istiod` (control plane)
- Deploy `istio ingress gateway` (data plane) in each AKS cluster
- Expose ingress gateway via `Service type=LoadBalancer`
  - AKS-01 gets **IP_C**
  - AKS-02 gets **IP_D**

### 0.2 Create GatewayClass + Gateway (platform)
- Ensure the `GatewayClass` exists and is bound to Istio implementation.
- Create a shared `Gateway` (recommended in platform namespace) with:
  - Listener ports (80/443 as needed)
  - TLS mode consistent with current design (see TLS section below)
  - AllowedRoutes policy (who can attach)

### 0.3 Deploy HTTPRoute (app)
- Create `HTTPRoute` in `NS-App` that routes to the existing `Service`
- Use same hostnames and paths as current Ingress rules, but **do not cut over yet**
- Verify `HTTPRoute` is **Accepted** and **Attached** to the `Gateway`:
  - `kubectl describe httproute <name>`
  - Check conditions:
    - `Accepted=True`
    - `ResolvedRefs=True`
    - `Parents: Accepted=True`

**Result:** Istio path is ready but unused by prod traffic.

---

## Phase 1 — Shadow Validation (Minimal regression)

> Objective: validate Istio path using **direct access to new LB IPs (C/D)** while prod stays on NGINX.

### 1.1 Direct test to IP_C / IP_D
Perform a small set of high-value tests (minimal regression):
- ✅ Health endpoint
- ✅ Auth handshake (JWT/OIDC/MTLS if applicable)
- ✅ Top 3–5 critical API endpoints (happy path)
- ✅ One negative test (auth fail / validation fail)
- ✅ Header integrity (Host, XFF, correlation-id)
- ✅ Timeouts and payload size (only if your app is sensitive)

Example:
- `curl -H "Host: api.bank.com" https://IP_C/<path>`
- `curl -H "Host: api.bank.com" https://IP_D/<path>`

### 1.2 Compare behavior with NGINX (spot check)
- Status code equivalence (2xx/4xx/5xx)
- Redirect behavior
- Response headers (if enforced)
- Latency sanity (no unexpected spikes)

**Exit criteria (Phase 1):**
- Istio ingress returns correct responses for critical endpoints
- No systemic 5xx from Istio path
- AppGW probes would pass (simulated via same path)

---

## Phase 2 — Prepare Azure Application Gateway for Cutover (No downtime)

> Objective: add new backend pool for Istio and validate health probes **before switching**.

### 2.1 Create new backend pool: `istio_pool`
Include:
- `IP_C` (AKS-01 istio LB)
- `IP_D` (AKS-02 istio LB)

### 2.2 Configure health probe for Istio backend
- Use **same probe path** as production (e.g., `/healthz`)
- Ensure probe includes correct `Host` header if your routing requires it
- Confirm AppGW shows both backends **Healthy**

### 2.3 Enable Connection Draining (recommended)
Enable connection draining on the backend setting:
- Suggested: **30–120 seconds**
- This reduces risk for long-lived or in-flight requests during switch

**Exit criteria (Phase 2):**
- `istio_pool` backends are Healthy
- Probes are stable for at least 10–30 minutes

---

## Phase 3 — Cutover (Zero downtime switch)

> Objective: switch production traffic from NGINX to Istio with **no downtime** and **instant rollback option**.

### 3.1 Switch AppGW routing rule to `istio_pool`
- Update the AppGW rule (or backend setting reference) from:
  - `nginx_pool` ➜ `istio_pool`

**During cutover:**
- Do not delete nginx resources
- Keep both ingress paths alive

### 3.2 Monitor immediately (first 5–30 minutes)
Minimum monitoring signals:
- AppGW: backend health (should remain Healthy)
- AppGW: 5xx count/rate
- Latency: p95/p99 (compare with baseline)
- Istio ingress gateway pod CPU/memory (ensure not throttling)
- Application error rate (from APM/logs)

**Success criteria:**
- No elevated 5xx
- Latency within acceptable delta
- No routing anomalies

---

## Phase 4 — Stabilize (Keep rollback ready)

> Objective: run on Istio for a stabilization window while retaining NGINX as quick fallback.

Recommended stabilization:
- 24–72 hours (depending on criticality)

Continue to keep:
- `nginx_pool` intact
- NGINX ingress controller running
- Existing Ingress resources unchanged

---

## Rollback Plan (Fast switch back to NGINX)

> Rollback is a **single AppGW config change**. No redeploy.

### When to rollback
- Sustained 5xx increase
- Incorrect routing/headers causing functional failures
- Auth/TLS mismatch
- Performance regression beyond acceptable threshold

### How to rollback
1. Switch AppGW rule back:
   - `istio_pool` ➜ `nginx_pool`
2. Confirm `nginx_pool` backends Healthy
3. Monitor error rate returns to baseline

> Rollback typically completes as fast as AppGW applies config + connection draining.

---

## Phase 5 — Decommission NGINX (After confidence)

Only after stabilization and formal sign-off:

1. Freeze window / change record
2. Remove references to NGINX in AppGW (already unused)
3. Scale down nginx ingress controller (optional staged)
4. Delete nginx `Service type=LoadBalancer` to release IPs (A/B) if desired
5. Remove old Ingress resources (optional)

---

## HTTPRoute Deployment Guidance (No downtime, minimal risk)

When production is already on Istio, treat HTTPRoute updates as **live config**.

### Safe deployment rules
- Never remove the only matching rule first
- Prefer **two-phase change**:
  1) Add new match/filters
  2) Verify
  3) Remove old match/filters

### Recommended patterns (pick one)
- **Dedicated canary host**: `api-canary.bank.com` for validation
- **Header-based test route**: match `x-canary: true` (no prod impact)
- **Path-based test**: `/__test/*` routes to same backend

> These patterns keep regression tests minimal because you can validate route behavior without touching the main host.

---

## TLS & Header Consistency Notes (to avoid surprises)

### TLS termination must remain consistent
Pick one and keep behavior unchanged during migration:
- **Option A:** TLS terminates at AppGW; backend uses HTTP/HTTPS internal
- **Option B:** End-to-end TLS to ingress (AppGW passes HTTPS)

If your current As-Is is Option A, keep Option A for Istio cutover.

### Preserve forwarded headers
Ensure Istio gateway config preserves/sets:
- `X-Forwarded-For`
- `X-Forwarded-Proto`
- `X-Forwarded-Host`
- `Host`

Mismatch here is a common cause of “works in test but fails in prod”.

---

## Minimal Regression Test Set (Recommended)

To minimize testing effort while staying safe:

1. Health check endpoint (probe path)
2. Top 3–5 critical API calls (happy path)
3. Auth check (valid token) + 1 negative auth case
4. One large request (only if your system uses it)
5. One long-lived / streaming test (only if applicable)

---

## Acceptance Criteria (Go/No-Go)

**Go** if:
- `istio_pool` healthy in AppGW
- Critical endpoints pass on IP_C/IP_D
- No spike in 5xx during a short validation window

**No-Go / Rollback** if:
- Any systemic routing mismatch (host/path rewrite)
- Auth/TLS failure for primary flows
- Sustained 5xx or latency beyond agreed threshold

---

## Deliverables (Suggested)

- [ ] AppGW config change record (nginx_pool ⇄ istio_pool)
- [ ] GatewayClass/Gateway manifests (platform)
- [ ] HTTPRoute manifests per app (app team)
- [ ] Smoke test checklist (minimal regression)
- [ ] Rollback playbook (1-page)

---

## Appendix — What does NOT change (No redeploy)

- ✅ Kubernetes `Service` and `Pod` for microservices
- ✅ App deployment artifacts
- ✅ Database / dependencies
- ✅ Internal service discovery

Only routing layer changes:
- AppGW backend pool selection
- Ingress data plane selection
- Gateway API resources (HTTPRoute)

---

## Appendix — Quick “One-Page” Migration Summary

1) Deploy Istio + LB (get IP_C/IP_D)  
2) Deploy GatewayClass + Gateway + HTTPRoute (no prod impact)  
3) Validate by hitting IP_C/IP_D directly (minimal regression)  
4) Add `istio_pool` in AppGW + probes + draining  
5) Cutover: switch AppGW backend `nginx_pool → istio_pool` (zero downtime)  
6) If issue: rollback by switching back immediately  
7) After stability: decommission nginx

---
