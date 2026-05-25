# Blue Team Defense — Operation Digital Turf War

**Team Name:** [Your Team Name]  
**Date:** [Date of Lab]  
**Instructor:** [Instructor Name]  
**Course:** Ethical Hacking / Information Security

---

## System Design Overview

### Architecture

```
Red Team Machine
      |
      | nmap / browser / Burp Suite / CyberChef / Python
      v
Assigned Ethical VM — Flask App (Port 5000)
      |
      +──── /mission1 ──── Hard Web Recon (HTML comment → artifact → Base64 token)
      |
      +──── /mission2 ──── Hard Crypto/Encoding (XOR+hex+reverse+B64 with ATLAS key)
      |
      +──── /login ──────── VULNERABLE: SQLi login → IDOR profile
      |
      +──── /secure-login ── SECURE: parameterized, hashed, rate-limited
      |
      +──── /logs-demo ───── Blue Team logging demonstration
      v
SQLite Database (users, profiles tables)
      |
      +── Vulnerable version: plain passwords, concatenated SQL
      +── Secure version: SHA-256 hashes, parameterized queries, session auth
```

### Technology Stack

| Component | Technology |
|-----------|------------|
| Backend | Python Flask 3.0 |
| Database | SQLite |
| Frontend | HTML5, CSS3, vanilla JS |
| Deployment | Docker Compose / Ubuntu VM |
| Logging | Python logging module → access.log |

---

## Mission 1 — Blue Team Design

### Intentional Vulnerability

**Type:** Exposed Deployment Artifact + Sensitive Token in Base64  
**Difficulty:** Hard  
**OWASP:** A05:2021 — Security Misconfiguration, A02:2021 — Cryptographic Failures

**Design decision:** The mission page contains an HTML comment pointing to an internal static file path. That file — a staging-environment audit log — was not removed before production deployment. It contains a Base64-encoded maintenance token. Players must view the page source to find the comment, follow the path to the artifact, read the file, and decode the token.

The Base64 encoding simulates a real developer mistake: treating encoding as a form of protection for a sensitive value. Because Base64 is keyless and universally reversible, it provides no security. The token decodes directly to the flag.

**Cross-mission design:** The archive file also contains `Project keyword: ATLAS` — the XOR key required for Mission 2. Players who read the artifact carefully will have the intelligence needed for the next mission. This models real attacker behavior: information gathered from one vulnerability informs the exploitation of another.

### Expected Exploitation Path

1. Player opens `/mission1`.
2. Player presses Ctrl+U or right-clicks → View Page Source.
3. Player finds the HTML comment: `<!-- legacy audit bundle moved to /static/archive/staff_audit_731.txt -->`.
4. Player navigates to `http://<target>:5000/static/archive/staff_audit_731.txt`.
5. Player reads the file and finds the Base64 token.
6. Player decodes the token using Python or CyberChef.
7. Flag captured. Player notes the `Project keyword: ATLAS` for Mission 2.

### Detection Strategy

In the vulnerable version, there is no way to detect this server-side — because the player reads static files via normal HTTP GET requests, which are indistinguishable from legitimate access. This is the key Blue Team lesson: once an artifact is in the public web root, it cannot be "hidden" — only removed.

**Blue Team lesson:** Security cannot rely on obscurity. A file in a static directory is public by definition, regardless of whether it appears in navigation menus. Attackers enumerate paths systematically; hidden URLs are not safe.

### Hardening Plan

| Action | Description |
|--------|-------------|
| Remove deployment artifacts | Delete staging files, audit logs, and debug bundles from the web root before going live |
| Restrict static directory | Configure the web server to serve only whitelisted file extensions from `static/` |
| Secrets manager | Store all tokens and credentials in a vault — never in files served by the web server |
| Remove HTML comments | Strip all HTML comments as part of the CI/CD build pipeline |
| Static analysis | Use tools like `truffleHog` or `detect-secrets` to catch secrets in files before deployment |
| Use real encryption | If a token must be stored or transmitted, encrypt it with AES-256-GCM — never rely on Base64 |

---

## Mission 2 — Blue Team Design

### Intentional Vulnerability

**Type:** Weak Custom Crypto — XOR with Exposed Key  
**Difficulty:** Hard  
**OWASP:** A02:2021 — Cryptographic Failures

**Design decision:** The flag is encoded using a custom four-step pipeline: XOR with a repeating ASCII key (`ATLAS`), hex encoding, string reversal, and Base64 encoding. While this chain appears complex, it has two critical weaknesses:

1. Every step is a reversible transformation. Given the key, any person can reconstruct the original message in seconds using Python or CyberChef.
2. The key (`ATLAS`) is not secret — it is embedded in the deployment artifact discovered in Mission 1. The Security Misconfiguration of Mission 1 directly enables the Cryptographic Failure of Mission 2.

**Cross-mission design:** This is intentional. Real attacks chain information across vulnerabilities. Players who carefully documented Mission 1's artifact output will immediately have the key for Mission 2. Players who skipped the artifact or did not note the `Project keyword` must go back and re-read it.

### Encoding Chain Applied

```
picoCTF{weak_custom_crypto_fails}

Step 1 — XOR each byte with repeating key ATLAS:
(binary — each byte XORed with the corresponding character of ATLAS, cycling)

Step 2 — Hex encode:
(XOR output expressed as a lowercase hex string)

Step 3 — Reverse:
(the entire hex string reversed character by character)

Step 4 — Base64 encode:
MTM3MmQyYTMwMmEyYjBlMjcyMTM1MzYyMjJjMGMyMzIwMjIzNjIyMjMxZjMwMjYzNjM3MzIxNTEwMWUyZjJkMzEz
```

The payload is generated server-side in `app.py` at startup — always mathematically correct.

### Expected Exploitation Path

1. Player reads the encoded payload on `/mission2`.
2. Player recognizes the outer layer as Base64 (alphanumeric, possibly padded).
3. Player recalls (or returns to) the Mission 1 archive file and notes `Project keyword: ATLAS`.
4. Player Base64-decodes the payload → obtains a reversed hex string.
5. Player reverses the string → obtains a hex string.
6. Player hex-decodes → obtains raw XOR bytes.
7. Player XORs bytes with repeating key `ATLAS` → recovers the flag.

### Detection Strategy

Custom encoding schemes are not encryption and cannot be detected as "attacks" — because there is no attack traffic. The server generates the payload at startup and displays it. Any party who views the page receives the payload. This is the key lesson: encoding without a properly managed secret key provides no confidentiality.

### Hardening Plan

| Action | Description |
|--------|-------------|
| Use real encryption | AES-256-GCM for all symmetric data protection requirements |
| Key management | Store encryption keys in a dedicated secrets manager (HashiCorp Vault, AWS KMS); never in static files or source code |
| Eliminate custom crypto | All cryptographic operations must use vetted, standard library implementations |
| Protect keys from cross-mission exposure | The key was leaked via a deployment artifact — fixing Mission 1's misconfiguration also eliminates the key exposure for Mission 2 |
| Avoid encoding as secrecy | Base64, hex, string reversal, and XOR with a known key all provide zero confidentiality |
| Encrypt in transit | Use TLS 1.2+ for all data transmission so that payload interception is also mitigated |

---

## Mission 3 — Blue Team Design

### Intentional Vulnerabilities

**Vulnerability 1: SQL Injection**  
**Type:** OWASP A03:2021 — Injection  
**Location:** `/login` POST handler  

**Vulnerable code:**
```python
query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
db.execute(query)
```

The user input is concatenated directly into the SQL string. The single quote character breaks out of the string literal and allows the attacker to inject arbitrary SQL conditions.

---

**Vulnerability 2: IDOR / Broken Access Control**  
**Type:** OWASP A01:2021 — Broken Access Control  
**Location:** `/profile` GET handler  

**Vulnerable code:**
```python
profile_id = request.args.get("id")
user = db.execute("SELECT * FROM profiles WHERE id = ?", (profile_id,)).fetchone()
return render_template("profile.html", profile=user)
```

The backend returns any profile whose `id` matches the URL parameter, without verifying that the requesting user is authorized to view that profile.

---

**Additional intentional weaknesses:**
- Passwords stored in plain text in the `password` column (visible in `init_db.py`)
- Error messages disclose the raw SQL query on failure (information disclosure)
- No rate limiting on the vulnerable login
- Login page notice reveals employee username `maya` — realistic information disclosure that gives the attacker a target username

### Expected Exploitation Path

1. `nmap -sV <target-ip>` → discovers Flask on port 5000
2. `/login` → read page notice for username hint → SQL injection payload (`maya' --`) → session as Maya (id=102, employee)
3. `/profile?id=102` → normal employee profile visible, no flag
4. Change URL to `/profile?id=1` → IDOR: backend returns admin profile without authorization check
5. Admin profile note field contains the flag

> Alternative payload `' OR '1'='1' --` logs in directly as admin (skips step 3–4).  
> The preferred intended path uses `maya' --` so both SQLi and IDOR are exercised.

### Detection and Logging Evidence

The logging system (`app/logs/access.log`) captures:

```
[INFO]    Login attempt  username=maya' --  ip=192.168.56.12
[WARNING] SQL injection payload detected  ip=192.168.56.12  payload_user="maya' --"
[INFO]    Login success  username=maya  ip=192.168.56.12
[WARNING] Profile IDOR  session_user=102  requested_profile=1  ip=192.168.56.12
```

View live logs at `/logs-demo`.

### IDS / Monitoring Strategy

| Event | Log Level | Action Recommended |
|-------|-----------|-------------------|
| SQL injection signature in input | WARNING | Alert security team, block IP after N events |
| Login failure | WARNING | Count failures; lock account after 5 attempts |
| IDOR access (profile mismatch) | WARNING | Alert; log for audit trail |
| Rate limit exceeded | WARNING | Block IP temporarily |
| DB error (malformed query) | ERROR | Immediate alert; could indicate active attack |

### Hardening Plan

**SQL Injection:**
```python
# Use parameterized queries — user input is a parameter, never SQL syntax
user = db.execute(
    "SELECT * FROM users WHERE username = ?", (username,)
).fetchone()
if user and user["password_hash"] == sha256(password):
    # authenticate
```

**Password Storage:**
```python
# Store SHA-256 hash (minimum); use bcrypt/argon2 in production
import hashlib
password_hash = hashlib.sha256(password.encode()).hexdigest()
```

> **Important note on SHA-256:** This lab uses SHA-256 for educational simplicity only.
> SHA-256 without a random salt is vulnerable to GPU-accelerated dictionary attacks and
> rainbow table lookups. In any real system, use **bcrypt**, **Argon2**, or **PBKDF2**,
> which include per-password salts and configurable cost factors that resist brute-force attacks.

**IDOR Fix:**
```python
# Always verify authorization server-side before returning data
requested_id = request.args.get("id")
if requested_id != str(session["user_id"]) and session["role"] != "admin":
    return "Access denied", 403
```

**Rate Limiting:**
```python
# Track failed attempts per IP; block after threshold
if is_rate_limited(ip):
    return "Too many attempts. Please wait.", 429
```

### Patch Strategy

| Component | Current State (Vulnerable) | Patched State (Secure) |
|-----------|---------------------------|------------------------|
| SQL Query | String concatenation | Parameterized query |
| Password Storage | Plain text | SHA-256 hash |
| Access Control | Trusts URL parameter | Session authorization check |
| Error Messages | Exposes raw SQL | Generic message |
| Rate Limiting | None | 5 attempts / 60 seconds |
| Logging | None | Full event logging |

### Long-term Defense Plan

1. **Input validation:** Validate and sanitize all user input at the boundary.
2. **Least privilege:** Database accounts should only have SELECT permission on needed tables.
3. **Regular security testing:** Run automated SAST/DAST tools on every deployment.
4. **Dependency monitoring:** Track Flask and SQLite CVEs; update promptly.
5. **SIEM integration:** Forward logs to a centralized SIEM for correlation and alerting.
6. **Access control reviews:** Regularly audit who can access what data.
7. **Penetration testing:** Commission annual pen tests on production systems.

---

## Vulnerable vs Secure Comparison

| Feature | Vulnerable (`/login`, `/profile`) | Secure (`/secure-login`, `/secure-profile`) |
|---------|-----------------------------------|--------------------------------------------|
| SQL query | String concatenation | Parameterized |
| Password storage | Plain text | SHA-256 hash |
| Access control | Trusts `?id=` parameter | Checks `session["user_id"]` |
| Error message | Exposes raw SQL | Generic "Invalid credentials" |
| Rate limiting | None | 5 attempts / 60 seconds per IP |
| Logging | Minimal | All suspicious events logged |
| SQLi attack | **Succeeds** | **Blocked** |
| IDOR attack | **Succeeds** | **Returns 403 Forbidden** |

---

## Scoring

| Mission | Difficulty | Points | Primary Vulnerability |
|---------|-----------|--------|----------------------|
| Mission 1 — Hard Web Recon | Hard | 300 | Exposed deployment artifact + Base64 token |
| Mission 2 — Hard Crypto/Encoding | Hard | 300 | Weak custom XOR crypto with exposed key |
| Mission 3 — Hard Web + Network | Hard | 300 | SQL Injection + IDOR |
| **Total** | | **900** | |

---

## OWASP Coverage

| OWASP Category | Challenge | How It Applies |
|----------------|-----------|----------------|
| A01 — Broken Access Control | Mission 3 | IDOR on profile endpoint |
| A02 — Cryptographic Failures | Missions 1 and 2 | Base64 as obfuscation (M1); custom XOR with exposed key (M2) |
| A03 — Injection | Mission 3 | SQL injection in login |
| A05 — Security Misconfiguration | Missions 1 and 2 | Deployment artifact in web root (M1); key exposed via artifact (M2) |
| A07 — Auth Failures | Mission 3 | Weak/plain passwords, no session management |
| A09 — Logging & Monitoring Failures | All | Logs demo; insufficient logging in vulnerable version |
