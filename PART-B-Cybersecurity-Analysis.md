# Part B — Cybersecurity Analysis

## System scope

CostScope is a plain HTML, CSS, and JavaScript dashboard. In its current form it has no accounts, project authorization, backend, or database. Cost rows remain in page memory unless a user sends them to Gemini or downloads a report. Excel parsing and fonts are loaded from public CDNs. This reduces the server-side attack surface, but it does not make an online deployment safe for confidential data: browser code and CDN resources run in the page, and there is no authentication or server-side rate limiting.

## Relevant threats, scenarios, and mitigations

| Threat | Realistic attack scenario | Prevention or mitigation |
|---|---|---|
| **1. API-key theft and quota abuse** | A user enters a Gemini key. A malicious browser extension, compromised page script, shared-device user, or screen-sharing session reads the key while it is active. An attacker reuses it to consume quota or incur charges. | Do not use a personal or unrestricted provider key in a public frontend. For production, move Gemini calls to an authenticated backend, keep the key in a server-side secret store, restrict the key to the required API and quotas, rotate exposed keys, and monitor usage. The current app keeps the key in page memory only, but that is not a security boundary. |
| **2. Sensitive cost data disclosure** | A user submits real project budgets through Gemini, or a compromised CDN script reads the in-memory line items and sends them elsewhere. Cost data can reveal business plans, suppliers, staffing, or project status. | Minimize and classify data before sending it to any external service; use synthetic data for demos. Disclose the provider transfer to users. Production should use a reviewed server-side integration, access controls, contractual/privacy review, and retention settings appropriate to the data. Self-host and pin reviewed dependencies where practical. |
| **3. Cross-site scripting (XSS) through imported values or reports** | An attacker supplies a CSV/XLSX containing HTML or script-like text in a category or description. If the application later inserts that value as HTML, code could execute in the app page and read visible data or an active key. A downloaded report opened in a browser could also execute unescaped values. | Treat imported cells and model output as untrusted. Escape values in HTML contexts and prefer `textContent`/DOM text nodes. Keep a restrictive Content Security Policy (CSP), avoid inline scripts, validate file size and structure, and test both the dashboard and generated report with payloads such as `<img src=x onerror=alert(1)>`. The current implementation escapes category/description in table and report HTML, and puts Gemini output in `textContent`; retain these safeguards. |
| **4. Malicious or malformed upload / parser denial of service** | A user uploads a very large CSV or a crafted workbook that consumes browser memory/CPU or exploits a flaw in the Excel parsing library. A misleading extension can also pass a simple filename check. | CostScope limits imported rows to 5,000 and each cost to $1,000,000,000,000, but it does not yet enforce a byte-size or robust file-signature limit. Add those controls, inspect workbook structure, parse defensively, and keep SheetJS patched and pinned or bundle a reviewed version. For production, consider parsing in an isolated worker or server with resource limits. Do not open unexpected files from untrusted senders. |
| **5. Third-party dependency/CDN compromise** | A CDN account, package, DNS path, or delivery endpoint is compromised and serves malicious JavaScript. Since the SheetJS script executes in the CostScope page, it can read the page DOM, line items, and active Gemini key. | Prefer a reviewed, pinned local dependency; verify releases and integrity hashes. If using a CDN, use Subresource Integrity (SRI) with `crossorigin`, pin a specific version, use CSP allowlists, and have a dependency update/monitoring process. Avoid loading scripts from arbitrary sources. |
| **6. Unauthorized project access (deployment/future backend risk)** | A future version adds shared projects or an API, but relies on hidden buttons or client-supplied project IDs for access control. A user changes an ID in a request and reads another team's cost rows (broken object-level authorization/IDOR). | The current frontend has no shared projects or server endpoint, so it cannot enforce or demonstrate user/project permissions. Before adding them, authenticate users and enforce authorization on every server request and object lookup; deny by default, test cross-account access, and never treat client-side UI checks as authorization. |
| **7. Excessive Gemini requests / service abuse** | A user or automated script repeatedly triggers Gemini analysis, consuming quota and potentially creating costs. Disabling the button only while a request is in flight does not provide a per-user or global rate limit. | In production route calls through an authenticated backend with per-user/IP rate limits, concurrency caps, daily quotas, request-size limits, abuse monitoring, and a kill switch. The current frontend has no reliable rate limiter; avoid exposing a live key in public deployments. |

## Three threats with greatest potential harm

1. **API-key theft and quota abuse:** a reusable credential can create direct financial loss, exhaust service quota, and expose the provider account. The current browser-only key approach cannot protect a key from someone who controls or inspects the active browser session.
2. **Sensitive cost data disclosure:** budgets and actual spend may reveal commercially sensitive plans. The user explicitly sends rows to a third-party AI service when requesting analysis, while a compromised page dependency could read them without that request.
3. **XSS or compromised third-party script:** script execution in this page could read all current in-memory cost rows and an entered key, alter reports, or send data elsewhere. The public CDN dependency increases the importance of controlling script delivery. Current escaping reduces the imported-text XSS risk, but does not defend against a malicious script dependency.

These are ranked for the current browser-only project. If shared accounts or server-side project storage are introduced, unauthorized project access could become one of the highest-impact threats and must be addressed before release.

## Pre-deployment security checklist

- [ ] Do not ship a Gemini secret in HTML, JavaScript, source maps, or browser storage; proxy AI requests through an authenticated backend.
- [ ] Apply authentication and server-side authorization to every project and record lookup; test that one account cannot read or alter another account's objects.
- [ ] Define what data may be sent to Gemini; show a clear notice and obtain user action before transfer; use synthetic data during demonstrations.
- [ ] Keep API secrets in a secret manager, restrict provider permissions and quotas, rotate keys, and set usage alerts.
- [ ] Rate-limit AI requests on the server by account and IP; cap concurrency, payload size, and daily usage.
- [ ] Pin and review dependencies; self-host where practical; use SRI for CDN scripts and a restrictive CSP (`script-src`, `connect-src`, `object-src 'none'`, `base-uri 'self'`, and `frame-ancestors`).
- [ ] Enforce HTTPS, secure response headers, and appropriate cross-origin rules on the hosting service.
- [ ] Validate upload type, size, row count, headers, numeric bounds, and workbook sheet count. CostScope currently caps row count and cost values, but needs a file byte-size/signature check before public deployment. Treat extensions and browser `accept` filters as convenience, not security validation.
- [ ] Escape untrusted values in the table and exported report; keep AI output in text nodes; test for reflected/stored XSS and spreadsheet formula injection in any CSV export.
- [ ] Handle malformed files and provider errors without exposing stack traces, keys, or sensitive payloads. Avoid logging secrets or full cost records.
- [ ] Decide data retention, backup, deletion, privacy notice, and incident/key-rotation procedures before storing or sharing user data.
- [ ] Run the safe tests below, then add automated regression tests for every fix and have a second reviewer inspect the deployment configuration.

## Safe security test demonstrations

Use a local copy and synthetic records only. These tests do not require attacking a real service or using a real API key. The outcomes below describe the current implementation and its known limits.

### Test 1 — Reject an unsupported upload

**Purpose:** Check that an unsupported file is not imported as cost data.

**Steps:** Create a text file named `safe-test.txt` containing `not a spreadsheet`. In CostScope, try to select it in the file picker; if the picker allows it, drop it onto the import area.

**Expected/observed behavior:** The file picker filters for CSV/Excel files, and the importer rejects the `.txt` extension with a clear message. No rows should be added. The extension check is only a basic guard; it does not prove that a `.csv` or `.xlsx` file is safe or genuine.

**Result:** Pass for unsupported-extension rejection. Production follow-up: add file-size and content/signature validation.

### Test 2 — XSS payload in imported text and generated report

**Purpose:** Verify that imported text is displayed as text rather than executed as HTML.

**Safe test CSV:**

```csv
Category,Description,Optimistic,Most likely,Pessimistic,Actual
"<img src=x onerror=alert(1)>","<script>alert(1)</script>",10,20,30,25
```

**Steps:** Save this as `safe-xss-test.csv`, import it, and observe the dashboard. Download the HTML report and open it locally in a browser.

**Expected/observed behavior:** The payload text should be visible literally and no alert should appear. The current table and report escape imported values before insertion, so the payload is rendered inert. Remove the test row afterward.

**Result:** Pass by source review of the HTML escaping path; repeat in the target browser before deployment. This test does not cover malicious third-party scripts, which execute independently of imported cells.

### Test 3 — Excessive AI request controls (mocked, no live API calls)

**Purpose:** Check whether repeated analysis can be rate-limited without consuming a real Gemini quota.

**Steps:** Use synthetic sample data and do not enter a real key. In a local browser test harness, replace/intercept `fetch` for the Gemini endpoint with a mock that returns a delayed `429 Too Many Requests` response. Enter a dummy key, press **Analyze with Gemini**, and attempt repeated clicks while the mocked request is pending; then retry after it completes.

**Expected/observed behavior:** The button is disabled during the in-flight call, reducing simultaneous requests. It becomes enabled again afterward, and the frontend has no per-user cooldown or request quota; successive requests can still be attempted. No live request is sent when `fetch` is mocked.

**Result:** Partial protection only; rate-limiting fails after each request completes. Production follow-up: enforce quotas and rate limits on an authenticated backend. A client-only limit can always be bypassed.

## Deployment conclusion

The project is suitable as a local demonstration with synthetic data. It is **not ready for public deployment with real user data or a real Gemini API key** until the key is moved server-side, request abuse is controlled, dependencies and hosting headers are secured, and identity/access controls are added if users will share projects. The current implementation has no backend, so account-based authorization cannot be implemented or meaningfully tested within this frontend alone.
