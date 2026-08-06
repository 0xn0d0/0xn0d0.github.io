# Burp Suite Pro — API Security Testing Runbook (REST)

Authorized-target runbook. Keep this open in a terminal pane beside Burp.
Tick boxes as you complete items.

---

## Phase 0 — Pre-flight & Authorization

- [ ] Written authorization / rules of engagement confirmed for every target host
- [ ] Scope defined: in-scope domains, IPs, ports, API versions, auth creds
- [ ] Burp Pro license activated; `Update` check done (Project > Project options)
- [ ] Proxy listening: default `127.0.0.1:8080` (Proxy > Proxy settings)
- [ ] Burp CA cert installed & trusted in test browser/device store
- [ ] Test browser / API client pointed at proxy; all HTTP(S) routed
- [ ] Scope set: Project > Scope > Include; `Use advanced scope control` on
- [ ] Session handling rules added for cookies/tokens (Project > Sessions)
- [ ] Out-of-scope items set to drop silently (don't clutter history)
- [ ] Baseline response recorded (to compare against post-fix retests)

> Record target info here: host `____________` | auth type `____________`
> token source `____________` | API version(s) `____________`

---

## Phase 1 — Discovery & API Recon

### 1a. Passive mapping (no tampering)
- [ ] Browsed/navigated app normally; captured traffic in Proxy > HTTP history
- [ ] Filtered history to `MIME type: JSON` to isolate API traffic
- [ ] Target > Site map reviewed; expanded to see full request tree
- [ ] Noted host, path, method, params, content-type, response codes

### 1b. OpenAPI / schema import (fastest win)
- [ ] Located spec: `/openapi.json`, `/swagger.json`, `/v2/swagger.json`,
      `/redoc`, `/docs`, `/api-docs`, `/swagger-ui`
- [ ] Imported via Burp > Project > Import/Export > OpenAPI definition
- [ ] Endpoints from spec now in site map — cross-checked against live behavior

### 1c. Content discovery (Intruder)
- [ ] Target > Engagement tools > Content discovery (or Intruder with wordlist)
- [ ] Wordlists loaded (all in `~/wordlists/`, see table below)
- [ ] Extensions probed: `.json`, `.yaml`, `.xml`, `.tar`, `.bak`, `~`
- [ ] Unusual findings verified manually in Repeater
- [ ] HTTP methods tested on each discovered path: `GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD`
- [ ] OPTIONS responses reviewed for allowed methods & CORS headers

### 1d. Parameter & header enum
- [ ] Param Miner extension run on key endpoints (hidden params)
- [ ] Arjun params list fuzzed on discovered endpoints (`params_medium` first, `params_large` if time allows)
- [ ] Custom headers probed: `X-API-Key, X-Api-Version, Authorization,
      Content-Type, X-Forwarded-For, X-Original-URL`
- [ ] Versioning tested: `/v1/` vs `/v2/`, `Accept: application/vnd.foo.v2+json`

### 1f. Wordlist library (`~/wordlists/`)

How to use: **Intruder Sniper** on the path/dir segment for small lists;
**Content Discovery tool** (Target > Engagement tools > Content discovery)
for the big assetnote lists. Intruder on 958k lines will take days — don't.

| List | Lines | Job |
|------|------:|-----|
| `assetnote/httparchive_apiroutes.txt` | 290k | PRIMARY REST discovery — real /api & /v1..v2 routes from HTTP Archive. Use Content Discovery tool. |
| `assetnote/swagger-wordlist.txt` | 958k | Deep Swagger-derived paths. Last resort / very deep passes only. |
| `api-endpoints.txt` | 295 | Common REST paths — fast Intruder baseline |
| `api-endpoints-res.txt` | 10.9k | Real-world resource/method tokens |
| `api-seen-in-wild.txt` | 7.6k | API surfaces observed in live traffic |
| `api-objects.txt` | 3.1k | Resource nouns (users, orders) for path fuzzing |
| `common-api-endpoints-mazen160.txt` | 174 | Curated API roots + spec files (swagger, openapi, raml) |
| `graphql.txt` | 106 | GraphQL endpoint/IDE discovery |
| `common.txt` | 4.7k | General dirs baseline |
| `params/params_medium.txt` | 11k | Hidden-param fuzzing (start here) |
| `params/params_large.txt` | 25.9k | Deep param fuzzing |
| (optional fallback: 1n3/intruderpayloads `FuzzLists/dirbuster-*.txt`) | — | General web dirs, secondary |

Intruder tip: `httparchive_apiroutes.txt` entries already include leading
`/` — send base request `GET /§§ HTTP/1.1` with target URL stripped, or trim
slashes via a payload-processing rule (add prefix/suffix) to reuse paths
against your base path (e.g. `/api`).

### 1e. Baseline record
- [ ] One "normal" request/response per major endpoint captured as baseline
      (used for diffing after each test)

---

## Phase 2 — Authentication & Session Baseline

- [ ] Full login/register/refresh/MFA/SSO flow captured and understood
- [ ] Token format identified (JWT? opaque? API key?) via History
- [ ] If JWT: JWT Editor extension loaded; alg/claims examined
- [ ] Session handling rule created to auto-replay tokens where needed
- [ ] Logout / token revoke behavior documented
- [ ] Password reset + account recovery flows mapped

---

## Phase 3 — OWASP API Security Top 10 Test Matrix

> For each test: reproduce in Repeater (or Intruder for fuzzing), then
> confirm manually. Do NOT trust the active scanner alone.

### 3.1 Broken Object Level Authorization (BOLA / IDOR)
- [ ] Object IDs enumerated: `/users/1` ... `/users/2`, `/orders/1001`
- [ ] IDs in query, path, JSON body, and headers each tampered
- [ ] UUIDs swapped between own and other account's resources
- [ ] IDOR via list endpoints (`/users?user_id=`) and nested resources
- [ ] Recursive: `/accounts/1/orders/2/payments/3`

### 3.2 Broken Function Level Authorization (BFLA)
- [ ] Admin endpoint called with user-level token (Repeater)
- [ ] Method-level checks: `GET /admin` vs `POST /admin` vs `DELETE /admin`
- [ ] Hidden endpoints from 1c probed with low-priv token
- [ ] Route spoofing: `/users/me/role` vs `/users/1/role`
- [ ] Case/encoding bypass: `/Admin`, `/ADMIN`, `/admin/../admin`, `%2f`

### 3.3 Broken Authentication
- [ ] JWT `alg:none` test (JWT Editor)
- [ ] JWT algorithm confusion: HS256<->RS256 key confusion
- [ ] Token expiry / `exp` / `nbf` manipulation
- [ ] Token replayed after logout (revocation test)
- [ ] Session fixation: fixed session ID before/after login
- [ ] Weak credential fuzz on login (Intruder, small ethical wordlist)
- [ ] MFA/2FA bypass: replay old token, or use `remember-me` cookie
- [ ] Rate limiting on login (see 3.6)

### 3.4 Excessive Data Exposure
- [ ] Every response inspected for fields beyond what the client needs
- [ ] Narrow requests crafted (`?fields=id`) to see if server over-returns
- [ ] Error messages reviewed for leaks (stack traces, SQL, paths)
- [ ] Verbose response bodies from `/health`, `/metrics`, `/debug`

### 3.5 Lack of Resources & Rate Limiting
- [ ] Intruder burst (e.g. 100 rapid requests) against one endpoint
- [ ] Pagination abuse: `?page=999999&limit=1000`
- [ ] Concurrent session abuse / token reuse across sessions
- [ ] Response to throttling documented (429? ignored? by IP or by token?)

### 3.6 Mass Assignment
- [ ] JSON body extended with: `"role":"admin", "isAdmin":true,
      "admin":true, "permissions":["*"], "verified":true, "balance":99999`
- [ ] Query-string equivalent: `/users?role=admin`
- [ ] Compare response / resulting resource for reflected fields
- [ ] Batch endpoints (GraphQL aliasing) used to mass-edit

### 3.7 Security Misconfiguration
- [ ] Stack traces / verbose errors triggered with malformed input
- [ ] CORS tested: evil origin in `Origin:` header; check `Access-Control-Allow-Origin`
- [ ] TLS/transport: only HTTPS, no weak ciphers (Scanner TLS checks)
- [ ] Default creds attempted on `/admin`, `/api`, dashboards (within scope)
- [ ] Unnecessary methods open (e.g. `DELETE` on GET-only resource)
- [ ] Missing security headers (`HSTS`, `CSP`, `X-Content-Type-Options`) noted

### 3.8 Injection
- [ ] SQLi: `' OR 1=1--`, `1 AND SLEEP(5)` (time-based), `1' UNION SELECT`
- [ ] NoSQLi: `{"$ne":null}`, `{"$gt":""}`, `{"$where":"1"}` in JSON bodies
- [ ] Command injection: `;id`, `$(whoami)`, `` `whoami` ``
- [ ] XSS in reflected JSON values (test in browser context)
- [ ] XXE: `Content-Type: application/xml` with external entity payload
- [ ] SSTI: `{{7*7}}`, `${7*7}`, `<%= 7*7 %>`
- [ ] Path traversal: `../..`, URL-encoded and double-encoded variants
- [ ] LDAP injection where applicable
- [ ] Backslash Powered Scanner extension run for recursive injection

### 3.9 Improper Inventory Management
- [ ] Old versions probed: `/v1`, `/v2`, `/old`, `/deprecated`
- [ ] Staging/env paths: `/dev`, `/staging`, `/test`, `/qa`
- [ ] Deprecated endpoints checked for weaker auth or known vulns
- [ ] Documentation endpoints confirm which versions are live

### 3.10 Unsafe Consumption of APIs
- [ ] SSRF: URL params fed internal addresses (`http://169.254.169.254/latest/meta-data/`,
      `http://localhost:80`, `http://127.0.0.1:8080`), also via redirects
- [ ] Open redirect in `redirect_uri`, `return_url`, `next` params
- [ ] HTTP Request Smuggling: CL.TE / TE.CL checks (extension or manual)

---

## Phase 4 — Burp Pro Active Scanning

- [ ] Live scan from site map: right-click endpoint > Active scan
- [ ] Scan configuration: insertion points = all, checks = all except
      non-ethical (DoS) — keep `resource-based` within scope limits
- [ ] Scan queue monitored (Dashboard); priority: auth endpoints first
- [ ] Each finding manually verified in Repeater before accepting
- [ ] False positives triaged: rerun, adjust payload, or mark FP
- [ ] Re-scanned after any fixes (confirm remediation)

### Extensions to install (BApp Store)
- [ ] JWT Editor
- [ ] Autorize (authz bypass automation)
- [ ] AutoRepeater (mass param/method mutation)
- [ ] Param Miner
- [ ] Backslash Powered Scanner
- [ ] HTTP Request Smuggler
- [ ] OpenAPI/Swagger parser (if not using built-in import)
- [ ] Upload Scanner (if file uploads in scope)

---

## Phase 5 — Manual Verification Rules

- [ ] Every automated finding reproduced manually with exact request/response
- [ ] Impact confirmed by cross-account or cross-role test (where authorized)
- [ ] Severity re-scored with impact evidence, not just scanner confidence
- [ ] No data harmed: test data only, no destructive operations in prod
- [ ] Cleanup: test accounts/resources created during testing removed

---

## Phase 6 — Reporting

- [ ] Findings logged with: title, severity (CVSS v3.x), affected endpoint,
      request+response pair, reproduction steps, impact, remediation
- [ ] Each finding includes OWASP API Top 10 mapping (e.g. API3:2019)
- [ ] Executive summary: critical count, exposure scope, risk trend
- [ ] Technical detail annex with raw Burp captures
- [ ] Remediation guidance per finding (from OWASP ASVS where applicable)
- [ ] Retest plan + confirmation of fixed items

### Findings template (copy per finding)
```
ID:        API-0XX
Title:     <short>
Severity:  <Critical|High|Medium|Low|Info> (CVSS base score)
OWASP:     API1:2023 Broken Object Level Authorization (or other)
Endpoint:  <METHOD> <url>
Request:   <paste from Burp>
Response:  <paste relevant response>
Steps:     1) ... 2) ... 3) ...
Impact:    <what an attacker gains>
Remediation: <fix guidance, per OWASP ASVS ref>
Retest:    [ ] Pending  [ ] Fixed  [ ] Accepted risk
```

---

## Phase 7 — Cleanup & Wrap-up

- [ ] All test accounts deleted, tokens revoked
- [ ] Burp CA cert removed from test devices (or noted for the env)
- [ ] Test data mutations reverted
- [ ] Findings shared with the API owner/authorization holder
- [ ] Lessons learned / hardening backlog noted

---

## Fast-reference cheatsheet

| Goal | Burp tool |
|------|-----------|
| Replay/edit single request | Repeater |
| Fuzz params/payloads | Intruder (Sniper/Battering ram/Pitchfork/Cluster bomb) |
| Automate multi-request flows | Repeater macros / extensions |
| Hidden content | Content discovery / Param Miner |
| Authz bypass across roles | Autorize |
| Mass method mutation | AutoRepeater |
| JWT tampering | JWT Editor |
| GraphQL schema/batching | GraphQL Raider (if API is GraphQL) |
| Full automated sweep | Active scan (scope-limited) |
| Live traffic analysis | HTTP history + filters |
