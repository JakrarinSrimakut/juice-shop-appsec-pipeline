# AppSec Walkthrough — ZAP + Burp + Juice Shop

**One file** with step-by-step instructions to:  
- start the environment,  
- run automated ZAP scans,  
- do manual testing with Burp Suite,  
- capture evidence (screenshots/reports), and  
- write reproducible findings.

> **Warning / Legal**: Only run these tests against systems you own or have explicit permission to test (e.g., the Juice Shop instance in this repo). Never scan third-party sites.

---

## Table of contents

1. Prerequisites
2. Repo layout & docker-compose
3. Start environment (one-shot and manual)
4. ZAP walkthrough (automated baseline + manual)
5. Burp walkthrough (intercept, repeater, manual tests)
6. Practical test payloads (SQLi, XSS, IDOR, JWT)
7. Screenshot & evidence checklist
8. Writing up findings (template)
9. Commit & push artifacts
10. Quick troubleshooting & tips
11. References

---

## 1 — Prerequisites

- Docker & Docker Compose installed and working.  
- (Optional) Burp Suite Community installed for manual testing.  
- This repo checked out and current directory is the repo root.

---

## 2 — Repo layout & docker-compose

Recommended repo root structure:

```
appsec-pipeline/
│   docker-compose.yml
│   README.md
│
├── scans/
│   ├── zap/
│   │   └── screenshots/
│   └── burpsuite/
│       └── screenshots/
│
├── docs/
│   └── appsec_walkthrough.md    # this file
│
└── config/
```

Example `docker-compose.yml` (place at repo root):

```yaml
version: "3.8"

services:
  juice-shop:
    image: bkimminich/juice-shop
    container_name: juice-shop
    ports:
      - "3000:3000"

  zap-baseline:
    image: zaproxy/zap-stable
    container_name: zap-scan
    depends_on:
      - juice-shop
    volumes:
      - ./scans/zap:/zap/wrk
    command: >
      zap-baseline.py
      -t http://juice-shop:3000
      -r zap_report.html
```

> This `zap-baseline` service will run the baseline scan and save `zap_report.html` into `./scans/zap/`.

---

## 3 — Start environment

### Option A — One-shot baseline scan (recommended for CI-like runs)
From repo root:

**bash / WSL / macOS**
```bash
docker-compose up zap-baseline
```

**PowerShell**
```powershell
docker-compose up zap-baseline
```

When finished, the container will exit and `scans/zap/zap_report.html` will exist.

### Option B — Manual (run Juice Shop then use ZAP interactively)
Bring up Juice Shop only:
```bash
docker-compose up -d juice-shop
```
Start a long-running zap container for manual use (example):
```bash
docker run --rm -d --name zap -v "${PWD}/scans/zap:/zap/wrk" zaproxy/zap-stable sleep infinity
```
Enter the container:
```bash
docker exec -it zap bash
# inside the container:
zap-baseline.py -t http://juice-shop:3000 -r /zap/wrk/zap_report.html
```

---

## 4 — ZAP walkthrough

### 4.1 Verify the report exists
On host (repo root):

**Linux**
```bash
ls -la scans/zap/zap_report.html
xdg-open scans/zap/zap_report.html
```
**macOS**
```bash
open scans/zap/zap_report.html
```
**Windows PowerShell**
```powershell
Get-ChildItem scans\zap\zap_report.html
Start-Process scans\zap\zap_report.html
```

**Screenshot suggestion:** `scans/zap/screenshots/report-open.png`

### 4.2 Read top alerts
Open the HTML report and go to **Alerts**. Prioritize **High/Medium**. For each alert record:
- Title
- Endpoint
- Evidence (request/response snippet)
- Steps to reproduce

### 4.3 Manual verification (Request Editor)
Use ZAP's manual request editor to re-send/modify requests:
1. From **Sites** or **History**, right-click request → **Open/Resend with Request Editor**.
2. Modify parameters/headers/payloads.
3. Click **Send** → inspect response.

Log manual tests to: `scans/zap/manual_tests.md` (example entries below).

### 4.4 Example ZAP manual tasks
- Check missing security headers (Content-Security-Policy, X-Frame-Options).  
- Test reflected parameters by modifying query string or POST bodies.  
- Try simple XSS payloads in parameters to confirm.  
- Capture any interesting JSON fields (IDs, debug info).

---

## 5 — Burp walkthrough (manual pentesting)

> Burp is used for manual interactive testing (Intercept, Repeater, Intruder).

### 5.1 Configure your browser
- Set browser proxy to `127.0.0.1:8080` (Burp default).
- In Burp → Proxy → Options, make sure the listener is enabled.

### 5.2 Intercept & send to Repeater
1. Turn **Intercept ON** in Burp Proxy.  
2. Perform the action in the app (login, submit feedback, add to basket).  
3. Burp will capture the request. **Right-click → Send to Repeater** (or to Intruder if fuzzing).
4. In **Repeater**, edit the body/header, click **Send**, inspect response.

### 5.3 Repeater workflow (example)
- Capture `POST /rest/user/login` → right-click → Send to Repeater.
- Replace payload with SQLi attempt:  
  ```json
  {"email":"' OR 1=1--","password":"anything"}
  ```
- Send and inspect response.

### 5.4 Use HTTP history
- Proxy → HTTP history shows every request. Use it to find endpoint names, tokens, Authorization headers, etc. Right-click → Show in Repeater.

### 5.5 Burp features to use
- **Repeater:** manual request edits & replays.  
- **Intruder:** fuzzing many payloads across parameters.  
- **Decoder:** decode/encode tokens (Base64, URL encode).  
- **Comparer:** diff responses.  

---

## 6 — Practical test payloads & exact steps

> Use these in Repeater / ZAP manual editor. Keep JSON valid.

### 6.1 SQL Injection (login attempt)
**Request body example**
```json
{"email":"' OR 1=1--","password":"anything"}
```
**Goal:** see if authentication bypass occurs or errors leak info.

### 6.2 XSS (search - DOM)
**Payload (search input)**
```
<svg/onload=alert(1)>
```
**Where:** search bar (client-side DOM reflection). Verify alert or DOM insertion.

### 6.3 XSS (stored in feedback)
**Author field payload (break out of quotes)**
```
"><iframe src=javascript:alert(1)>
```
**Where:** Customer Feedback → Author field. Inspect reviews page for execution.

### 6.4 IDOR (Insecure Direct Object Reference)
**Example**
- Request: `GET /api/Users/5`
- Modify ID to `GET /api/Users/6` and check if other user data is returned.

### 6.5 Basket tampering (quantity)
**Intercept**
- `POST /api/BasketItems` body example:
```json
{"ProductId": 2, "BasketId": 3, "quantity": 1}
```
- Change `"quantity": -100` or `"quantity": 9999`, forward, then view basket/checkout totals.

### 6.6 Weak JWT handling (forging)
**High level steps**
1. Capture token from `Authorization: Bearer <token>`.
2. Decode header & payload (jwt.io or Burp Decoder).
3. Modify payload (`"role":"admin"`).
4. Re-sign with weak secret (`secret`) using jwt.io or a script — HS256 requires signature.
5. Replace header in requests → try admin endpoints (`GET /api/Users`).

**Note:** Some servers may accept `alg: none`. If that fails, re-sign properly.

---

## 7 — Screenshot & evidence checklist

For every vulnerability or meaningful result, capture:

1. **Setup screenshot**: Browser open to the vulnerable page (`scans/.../setup.png`).  
2. **Intercepted request**: show request before modification (`req-<name>.png`).  
3. **Modified request**: show edited payload in Repeater/Request Editor (`mod-<name>.png`).  
4. **Response evidence**: server response showing payload reflected or action performed (`res-<name>.png`).  
5. **Alert view / report**: ZAP Alerts or HTML report screenshot (`alert-<name>.png`, `report-open.png`).

Store images under:
```
scans/zap/screenshots/
scans/burpsuite/screenshots/
```

**Filename convention**: `req-login.png`, `res-xss-search.png`, `alert-stored-xss.png`

---

## 8 — Writing up findings (template)

Use this template in `docs/` or `scans/`:

```markdown
### [Vulnerability Title] — [Severity]

**Description:** Short description.

**Endpoint:** URL and HTTP method.

**Steps to reproduce:**
1. Step one (exact clicks / form fields and payload).
2. Step two.

**Evidence:**
- Request screenshot: `scans/.../req-...png`
- Response screenshot: `scans/.../res-...png`
- ZAP report: `scans/zap/zap_report.html`

**Impact:** What an attacker can do.

**Mitigation:** Short recommended fix.
```

---

## 9 — Commit & push artifacts

**Important:** remove/redact sensitive values (full JWTs, API keys, local usernames) before pushing.

```bash
git add scans/zap/zap_report.html scans/zap/screenshots/* scans/burpsuite/screenshots/* docs/appsec_walkthrough.md
git commit -m "Add scan report, screenshots, and walkthrough"
git push
```

---

## 10 — Quick troubleshooting & tips

- **If the container cannot reach Juice Shop** when running ZAP in a container: use `host.docker.internal:3000` (Docker Desktop) or run both services on same Docker network and target `http://juice-shop:3000`.  
- **Base64URL vs URL encode**: JWT parts must be Base64URL-encoded (no `%` percent-encoding). Use jwt.io or a small script to encode properly.  
- **PowerShell volume quoting**: prefer `-v "${PWD}:/zap/wrk"` or use `${PWD}` unquoted.  
- **Permissions**: ensure `scans/zap` is writable by Docker (host folder permissions).  
- **Use small steps**: run baseline → inspect report → pick top 1-2 alerts to verify manually.

---

## 11 — References & further learning

- OWASP Juice Shop — https://owasp.org/www-project-juice-shop/  
- OWASP ZAP — https://www.zaproxy.org/  
- Burp Suite Community — https://portswigger.net/burp
