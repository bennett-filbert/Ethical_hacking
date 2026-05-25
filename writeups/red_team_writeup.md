# Red Team Write-up — Operation Digital Turf War

**Team Name:** [Your Team Name]  
**Date:** [Date of Lab]  
**Instructor:** [Instructor Name]  
**Course:** Ethical Hacking / Information Security

---

## Mission 1 — Hard Web Recon (Hard · 300 pts)

### Target Overview

| Field | Value |
|-------|-------|
| Target URL | `http://<target-ip>:5000/mission1` |
| Category | Web Recon / Information Disclosure |
| Detected Services | HTTP (Flask), Port 5000 |

### Initial Analysis

> *Describe what you first saw when you opened the page. What did the visible page tell you? What does the challenge category suggest about where to look?*

[Write your initial observations here]

### Tools Used

- Browser (Chrome / Firefox)
- View Page Source (Ctrl+U)
- Browser DevTools (F12)
- Python 3 (`base64` module) or CyberChef

### Exploitation Steps

1. Opened `/mission1` in a browser. The rendered page showed a standard corporate login form with no visible flag or clue.
2. Pressed `Ctrl+U` to view the full page source.
3. Searched the HTML for comment blocks using `Ctrl+F` and the string `<!--`.
4. Located the following HTML comment:

```html
<!-- legacy audit bundle moved to /static/archive/staff_audit_731.txt -->
```

5. Navigated directly to `http://<target-ip>:5000/static/archive/staff_audit_731.txt` in the browser.
6. The server returned a plaintext file containing:

```
ATLAS CORE SYSTEMS — Internal Audit Log
Project keyword: ATLAS
Maintenance token (base64): cGljb0NURntkZXBsb3ltZW50X2FydGlmYWN0c19sZWFrX3NlY3JldHN9
```

7. Decoded the Base64 token using Python:

```python
import base64
print(base64.b64decode("cGljb0NURntkZXBsb3ltZW50X2FydGlmYWN0c19sZWFrX3NlY3JldHN9").decode())
```

8. Captured the flag. Recorded `Project keyword: ATLAS` as intelligence for Mission 2.

### Screenshot

> *[Insert screenshot of page source with HTML comment visible]*  
> *[Insert screenshot of browser showing the archive file content]*  
> *[Insert screenshot of Python terminal or CyberChef showing decoded flag]*

### Flag Found

```
picoCTF{deployment_artifacts_leak_secrets}
```

### Mitigation Suggestion

> *Explain in your own words how this vulnerability could have been prevented.*

[Write 2–3 sentences here. Consider: why is the file accessible? why is the comment dangerous? why is Base64 not sufficient protection?]

---

## Mission 2 — Hard Crypto/Encoding (Hard · 300 pts)

### Target Overview

| Field | Value |
|-------|-------|
| Target URL | `http://<target-ip>:5000/mission2` |
| Category | Crypto / Encoding |
| Key Source | Mission 1 archive file — `Project keyword: ATLAS` |

### Initial Analysis

> *What did you notice about the encoded string? What did the case note on the page suggest? How did your intelligence from Mission 1 become relevant?*

[Write your initial observations here]

### Tools Used

- Python 3 (`base64` module, built-in `bytes`)
- CyberChef
- Notes from Mission 1 archive (`Project keyword: ATLAS`)

### Payload (from the page)

```
MTM3MmQyYTMwMmEyYjBlMjcyMTM1MzYyMjJjMGMyMzIwMjIzNjIyMjMxZjMwMjYzNjM3MzIxNTEwMWUyZjJkMzEz
```

### Encoding Chain (Blue Team applied forward)

```
Flag → XOR with key ATLAS → Hex encode → Reverse hex string → Base64 encode → Payload
```

### Exploitation Steps

The decoding chain is the exact reverse of the encoding chain:

1. **Base64 decode** the payload.
2. **Reverse** the resulting string.
3. **Hex decode** the reversed string.
4. **XOR** each byte with the repeating key `ATLAS` (from Mission 1).

**Python solver:**

```python
import base64

payload = "MTM3MmQyYTMwMmEyYjBlMjcyMTM1MzYyMjJjMGMyMzIwMjIzNjIyMjMxZjMwMjYzNjM3MzIxNTEwMWUyZjJkMzEz"
key = "ATLAS"

step1 = base64.b64decode(payload).decode()                              # Base64 decode
step2 = step1[::-1]                                                      # Reverse string
step3 = bytes.fromhex(step2)                                             # Hex decode
flag  = bytes([step3[i] ^ ord(key[i % len(key)]) for i in range(len(step3))]).decode()
print(flag)
# picoCTF{weak_custom_crypto_fails}
```

**CyberChef recipe:**  
Input: paste the payload  
Operations: From Base64 → Reverse → From Hex → XOR (key: `ATLAS`, scheme: Standard)

### Screenshot

> *[Insert screenshot of Python terminal or CyberChef showing each decode step and the final flag]*  
> *[Insert screenshot of Mission 1 archive file showing "Project keyword: ATLAS" — evidence of cross-mission intelligence]*

### Flag Found

```
picoCTF{weak_custom_crypto_fails}
```

### Mitigation Suggestion

> *Explain why this encoding scheme is not real encryption, and how the cross-mission key exposure made it worse.*

[Write 2–3 sentences here. Consider: what makes XOR weak when the key is known? what should be used instead? how does the Mission 1 misconfiguration directly enable Mission 2?]

---

## Mission 3 — Digital Turf Breach (Hard · 300 pts)

### Target Overview

| Field | Value |
|-------|-------|
| Target IP | `<assigned VM IP>` |
| Target URL | `http://<target-ip>:5000` |
| Detected Services | Port 5000/tcp — HTTP (Flask/Werkzeug) |
| Tools Used | nmap, Browser, Burp Suite |

### Vulnerability Identification

| Vulnerability | Type | OWASP Reference |
|--------------|------|-----------------|
| SQL Injection | Injection | A03:2021 |
| IDOR | Broken Access Control | A01:2021 |

**Related CVE:** CWE-89 (SQL Injection), CWE-639 (Authorization Bypass Through User-Controlled Key)

### Phase 1 — Network Reconnaissance

```bash
nmap -sV <target-ip>
```

**Output:**
```
PORT     STATE SERVICE VERSION
5000/tcp open  http    Werkzeug/3.0 Python/3.11
```

> *[Insert screenshot of nmap output]*

### Phase 2 — SQL Injection (Authentication Bypass)

**Step 1:** Opened `http://<target-ip>:5000/login`.

**Step 2:** Read the page notice: "Account migration batch 102 is pending review. Assigned support analyst: maya." This reveals the target username.

**Step 3:** Entered the targeted SQL injection payload:

| Field    | Value |
|----------|-------|
| Username | `maya' --` |
| Password | `anything` |

**Step 4:** Observed that login succeeded. Redirected to `/profile?id=102`.

**Why it works:**

The vulnerable backend constructs the query as:
```sql
SELECT * FROM users WHERE username='maya' --' AND password='anything'
```
The `--` comments out the remainder of the query. The password check is never executed. Maya's row (id=102, employee role) is returned and a session is established.

> *[Insert screenshot of login page with the payload entered]*  
> *[Insert screenshot of Maya's employee profile at /profile?id=102 — no flag visible]*

### Phase 3 — IDOR (Broken Access Control)

**Step 1:** After login, observed URL: `/profile?id=102`. The `id` parameter controls which profile is returned.

**Step 2:** Changed URL to: `http://<target-ip>:5000/profile?id=1`

**Step 3:** Server returned the administrator profile. No authorization check was performed — any authenticated user can request any profile by ID.

**Step 4:** Notes field contained the flag.

> *[Insert screenshot of URL bar changed to /profile?id=1 and admin profile page with flag visible]*

### Flag Found

```
picoCTF{digital_turf_war_compromised}
```

### Screenshot Evidence

> *[Insert all relevant screenshots: nmap scan, login payload, Maya's profile, IDOR to admin profile, flag visible, log entries]*

### Mitigation Suggestion

> *Explain both the SQL injection fix and the IDOR fix in your own words.*

**SQL Injection:** [Write 2–3 sentences — focus on parameterized queries vs. string concatenation]

**IDOR / Broken Access Control:** [Write 2–3 sentences — focus on server-side session validation]

---

## Attack Flow Diagram

```
[Red Team Machine]
       |
       | Step 1: nmap -sV <target-ip>
       v
[Assigned Ethical VM — Port 5000]
       |
       | Step 2: Access /login
       | Step 3: Read page notice → username hint: maya
       v
[Vulnerable Login Form]
       |
       | Step 4: SQL Injection — maya' --
       v
[Authentication Bypassed — Session: Maya (id=102, employee)]
       |
       | Step 5: Observe URL /profile?id=102 — Maya's profile, no flag
       | Step 6: Change to /profile?id=1 (IDOR)
       v
[Admin Profile Returned — No Authorization Check]
       |
       | Step 7: Read flag from Notes field
       v
[FLAG CAPTURED: picoCTF{digital_turf_war_compromised}]
```

---

## Summary

| Mission | Difficulty | Points | Flag | Captured |
|---------|-----------|--------|------|----------|
| Hard Web Recon | Hard | 300 | `picoCTF{deployment_artifacts_leak_secrets}` | [ ] |
| Hard Crypto/Encoding | Hard | 300 | `picoCTF{weak_custom_crypto_fails}` | [ ] |
| Digital Turf Breach | Hard | 300 | `picoCTF{digital_turf_war_compromised}` | [ ] |
| **Total** | | **900** | | |
