# Presentation Notes — Operation Digital Turf War

Use this document to prepare your slides and talking points. The structure below maps to the 14-slide presentation format.

---

## Slide 1 — Title

**Title:** Operation Digital Turf War  
**Subtitle:** Mini CTF Ethical Hacking Project  
**Include:** Team member names, course name, instructor name, date

**Speaking note:**  
Introduce the project name and the fictional scenario: two cyber-intelligence units competing for control of a restricted digital network. Explain that this is a 3-stage CTF lab where one group attacks and the other defends and designs. All three stages are Hard-level challenges, each worth 300 points, for a total of 900 points.

---

## Slide 2 — Project Overview

**Bullet points:**
- Three Hard-level Capture-the-Flag challenges — all 300 points each, 900 total
- Red Team vs Blue Team model
- Hosted in a controlled VM or Docker environment
- Covers: Web Recon, Crypto/Encoding with cross-mission intelligence, Network reconnaissance, SQL Injection, IDOR, and Blue Team defense

**Speaking note:**  
Explain the Red Team / Blue Team split. Red Team attacks and captures flags. Blue Team designs the challenges, explains the vulnerabilities, and demonstrates the secure fix. Both roles are required for a complete score. All three missions are Hard difficulty — each requires multi-step reasoning, not just a single technique. This is why the total is 900 points.

---

## Slide 3 — Assignment Requirement Mapping

**Table:**

| Requirement | Our Implementation |
|-------------|-------------------|
| Hard Challenge (300 pts) | Mission 1 — Hard Web Recon |
| Hard Challenge (300 pts) | Mission 2 — Hard Crypto/Encoding |
| Hard Challenge (300 pts) | Mission 3 — Digital Turf Breach |
| Web Vulnerability | HTML info disclosure, exposed artifact, SQL Injection, IDOR |
| Network Vulnerability | nmap scan, exposed Flask service on port 5000 |
| Encoding / Crypto Challenge | XOR + hex + reverse + Base64 chain with cross-mission key |
| Blue Team Defense | Secure comparison routes, logs, rate limiting |
| **Total Points** | **900** |

**Speaking note:**  
Show that every rubric requirement is satisfied. Point out that the project is not three static pages — it is a real Flask application with a database, intentional vulnerabilities, and secure alternatives. The upgrade to three Hard challenges makes the lab more realistic: real-world vulnerabilities require chaining multiple techniques and gathering intelligence across steps.

---

## Slide 4 — System Architecture

**Diagram:**

```
Red Team Machine
      |
      | nmap / browser / Python / CyberChef / Burp Suite / curl
      v
Assigned Ethical VM (or Docker on laptop)
      |
      | Flask Web App — Port 5000
      v
SQLite Database + Runtime Secrets (.env)
      |
      +── Discovery:  /robots.txt, /archive/ (deployment artifacts)
      +── Mission 1:  /mission1 (header/cookie), /assets/style.css (CSS comment)
      +── Mission 2:  /mission2 (3 transmissions — one real)
      +── Mission 3:  /login (SQLi), /profile (IDOR), /case (broken access control)
      +── Blue Team:  /secure-login, /secure-profile, /logs-demo
      +── Flags:      read from .env at runtime — not in committed source
```

**Deployment options:**
- Local: `copy .env.example .env` → edit values → `python app/app.py`
- Docker: `copy .env.example .env` → edit values → `docker compose up --build`
- Shared on demo day: ngrok exposes port 5000 (browser/Burp only — not nmap)

**Speaking note:**  
Explain that Docker gives any machine a consistent, one-command environment. The app runs on port 5000 so nmap can discover it on the assigned VM. ngrok is for HTTP sharing only — nmap goes against the actual VM IP, not the ngrok domain.

---

## Slide 5 — Mission 1: Hard Web Recon

**Fields:**
- Difficulty: Hard — 300 pts
- Category: Web Recon / Information Disclosure
- Concepts: robots.txt recon, deployment artifact exposure, metadata leakage across 4 channels, Base64 fragment reconstruction
- Technique: robots.txt → /archive/ → Fragment A + ATLAS keyword → CSS comment → response header → cookie → concatenate fragments → flag
- OWASP: A05:2021 — Security Misconfiguration, A02:2021 — Cryptographic Failures

**Attack flow:**
```
1. Check /robots.txt
      ↓ Disallow: /archive/
2. Access /archive/ — see directory listing of 4 files
      ↓ (decoys: staff_audit_old.txt, backup_notes.txt, checksum_manifest.txt)
3. Read /archive/staff_audit_731.txt
      ↓ Fragment A (base64) + "Project keyword: ATLAS" + "Audit Reference: BATCH-731"
4. Inspect /assets/style.css in DevTools
      ↓ /* deployment-fragment-b: <base64> */ → Fragment B
5. Inspect /mission1 response headers (Network tab / curl -I)
      ↓ X-Atlas-Audit: fragment-c=<base64> → Fragment C
6. Inspect /mission1 cookies (Application tab / curl -c)
      ↓ audit_hint=<base64> → Fragment D
7. base64_decode(A) + base64_decode(B) + base64_decode(C) + base64_decode(D)
      ↓
8. FLAG: picoCTF{deployment_artifacts_leak_secrets}
9. Save: ATLAS, BATCH-731 → needed for Mission 2 key derivation
```

**Why this is Hard:**  
The flag is split across 4 different channels (file, CSS, header, cookie). Players who only check the HTML source find nothing. Players who check only the archive find 1 fragment — insufficient. Decoy files add misdirection. Full discovery requires systematic enumeration of all response channels and understanding that robots.txt is a hint, not access control.

**Screenshot to show:** robots.txt showing Disallow. Archive listing. Archive file with Fragment A. DevTools Sources showing CSS comment. DevTools Network showing X-Atlas-Audit header. DevTools Application showing audit_hint cookie. Python/CyberChef showing reconstruction to flag.

---

## Slide 6 — Mission 2: Hard Crypto/Encoding

**Fields:**
- Difficulty: Hard — 300 pts
- Category: Crypto / Encoding
- Concepts: derived key from cross-mission metadata, custom encoding chain with byte transposition, decoy transmissions
- Technique: identify authentic transmission → derive key from Mission 1 intel → reverse 5-step chain
- OWASP: A02:2021 — Cryptographic Failures, A05:2021 — Security Misconfiguration

**Cross-mission dependency:**  
The key is derived from two pieces of intelligence found in Mission 1:
- `Project keyword: ATLAS` from `/archive/staff_audit_731.txt`
- `Audit Reference: BATCH-731` from the same file
- Key derivation: `SHA256("ATLAS:BATCH-731")[:8]`

**Three transmissions (only 731-B is the flag):**
```
731-A: simple base64 → decodes to ops text containing Mission 3 case code a7f3b2c1
731-B: complex chain → decodes to picoCTF{weak_custom_crypto_fails}  ← REAL
731-C: simple base64 → decodes to plausible ops text, no flag
```

**Encoding chain (Blue Team design — forward):**
```
Flag bytes
→ XOR each byte with repeating 8-byte SHA256-derived key
→ Separate even-indexed and odd-indexed bytes; concatenate even + odd
→ Hex encode
→ Reverse hex string
→ Base64 encode
→ Transmission 731-B displayed on page
```

**Decoding chain (Red Team — reverse):**
```
Base64 decode
→ Reverse string
→ Hex decode to bytes
→ Undo even/odd transposition (split halves, interleave)
→ XOR with SHA256("ATLAS:BATCH-731")[:8]
→ FLAG: picoCTF{weak_custom_crypto_fails}
```

**Why this is Hard:**  
5 encoding steps. Key requires cross-mission intel AND knowing the format "ATLAS:BATCH-731" + SHA256. Three transmissions shown — players must identify which is real (the harder-encoded one). 731-A looks solved easily but gives operational intel for Mission 3, rewarding thorough players.

**Screenshot to show:** /mission2 showing 3 transmissions. Decoded 731-A text containing case code. Python solver decoding 731-B step by step to the flag. Mission 1 archive notes showing ATLAS and BATCH-731.

---

## Slide 7 — Mission 3: Digital Turf Breach

**Fields:**
- Difficulty: Hard — 300 pts
- Category: Web + Network + Database + Blue Team Defense
- Concepts: Network scanning, SQL Injection, IDOR, Broken Object-Level Access Control on secondary resource
- Tools: nmap, browser, Burp Suite, DevTools
- OWASP: A01, A03, A07, A09

**Attack flow summary:**
1. nmap discovers Flask on port 5000
2. Login page notice reveals username `maya`
3. SQL injection `maya' --` bypasses authentication → lands on `/profile?id=102`
4. IDOR: change to `/profile?id=1` → admin profile shows "case AC-731, verification code required"
5. Decode Transmission 731-A from Mission 2 → find verification token `a7f3b2c1`
6. Access `/case?id=AC-731&code=a7f3b2c1` → flag revealed from case record

**Why this is Hard:**  
Four techniques chained: network recon, SQLi, IDOR, and broken access control on a secondary resource. The final flag is NOT in the admin profile — players must find the case endpoint, recognize that the route only checks a code (not authorization), and use a code discovered from Mission 2 output. All three missions are interconnected.

**Screenshots to show:** nmap output, login with `maya' --`, Maya's profile (no flag), admin profile with case reference, case denied page (no code), case record with flag, log entries at `/logs-demo`.

---

## Slide 8 — Vulnerable vs Secure Comparison

**Table:**

| Feature | Vulnerable Version | Secure Version |
|---------|-------------------|----------------|
| SQL Query | String concatenation | Parameterized query |
| Password Storage | Plain text | SHA-256 hash |
| Access Control | Trusts `?id=` parameter | Checks `session["user_id"]` |
| Error Messages | Exposes raw SQL query | Generic message only |
| Rate Limiting | None | 5 attempts / 60 seconds |
| Logging | Minimal | All suspicious events logged |
| SQLi attack | Succeeds | Blocked and logged |
| IDOR attack | Succeeds | Returns 403 Forbidden |

**Speaking note:**  
The secure version is not a different application — it is the same scenario with correct implementation. Walk through each row. Emphasize that the fixes are not complex: parameterized queries are one line different from the vulnerable version. The biggest risk is not knowing the correct pattern.

---

## Slide 9 — Red Team Attack Flow (Mission 3)

**Sequential steps:**

```
1. nmap -sV <target-ip>
      ↓ discovers port 5000 / Flask / Werkzeug
2. Open /login in browser
      ↓ read maintenance notice: username = maya, batch 102
3. Enter SQL injection payload: maya' --
      ↓ password check commented out → auth bypassed as Maya (id=102)
4. Observe /profile?id=102 — Maya's employee profile, no flag
      ↓ notice ?id= parameter in URL
5. Change URL to /profile?id=1 — IDOR: admin profile returned
      ↓ WARNING logged: Profile IDOR
      ↓ Notes field: "Recovery note archived under case AC-731. Verification code required."
6. Cross-reference Mission 2: decode Transmission 731-A (simple base64)
      ↓ plaintext contains: "Case AC-731 verification token: a7f3b2c1"
7. Access /case?id=AC-731 → denied (no code)
      ↓
8. Access /case?id=AC-731&code=a7f3b2c1
      ↓ WARNING logged: Case record accessed
      ↓ Recovery Note field reveals flag
9. FLAG: picoCTF{digital_turf_war_compromised}
      ↓
10. Document all steps + screenshots → submit flag + write-up + mitigation
```

**Speaking note:**  
Walk through each step during the demo. Emphasize the chain: SQLi bypasses auth → IDOR accesses admin data → admin data references the case → case code found in Mission 2 output → broken access control on /case reveals the flag. Show `/logs-demo` at the end so the class sees the full WARNING trail from SQLi through IDOR to case access.

---

## Slide 10 — Blue Team Defense Flow

**Sequential steps:**

```
1. Designed the challenge environment (Flask + SQLite)
      ↓
2. Identified intentional vulnerabilities (all 3 missions)
      ↓
3. Implemented cross-mission intelligence chain (ATLAS key)
      ↓
4. Implemented logging for all suspicious events
      ↓
5. Built secure comparison routes
      ↓
6. Demonstrated that attack fails on secure version
      ↓
7. Documented root cause and mitigation for each mission
      ↓
8. Prepared hardening plan
```

**Speaking note:**  
The Blue Team's job is not just to design puzzles — it is to understand why each vulnerability exists, how it is exploited, and exactly what code change eliminates it. Show the secure login blocking the SQL injection payload. Show the secure profile returning 403. Show the log entries. This is the defense mindset.

---

## Slide 11 — Course Chapter Connections

**List:**

| Topic | Where It Appears |
|-------|-----------------|
| OWASP Injection (SQLi) | Mission 3 — vulnerable login |
| OWASP Broken Access Control (IDOR) | Mission 3 — vulnerable profile |
| OWASP Sensitive Data Exposure | Mission 1 — deployment artifact in web root |
| OWASP Cryptographic Failures | Missions 1 and 2 — Base64 obfuscation; XOR with exposed key |
| OWASP Security Misconfiguration | Mission 1 — artifact left in production; HTML comment disclosure |
| OWASP Logging & Monitoring Failures | Logs Demo |
| Reconnaissance / Network Scanning | Mission 3 — nmap |
| Cross-mission intelligence chaining | Mission 2 key sourced from Mission 1 artifact |
| Ethical hacking methodology | All missions — authorization, scope, reporting |
| Penetration testing methodology | Pre-engagement, recon, exploitation, reporting |

**Speaking note:**  
Every technical element of this project maps directly to what was taught in class. This is not a standalone exercise — it applies the exact concepts from the OWASP, web security, and ethical hacking chapters to a real working system. The cross-mission key dependency is a deliberate design choice that models how real-world penetration testers chain findings.

---

## Slide 12 — Screenshots and Demo

**Checklist of screenshots to include:**

- [ ] **Home page** — mission dashboard showing all three mission cards at `http://<target>:5000`
- [ ] **Mission 1** — mission page rendered in the browser (flag is NOT visible)
- [ ] **Mission 1** — `/robots.txt` showing `Disallow: /archive/`
- [ ] **Mission 1** — `/archive/` directory listing showing the four files
- [ ] **Mission 1** — `/archive/staff_audit_731.txt` with Fragment A, `Project keyword: ATLAS`, and `Audit Reference: BATCH-731` visible
- [ ] **Mission 1** — DevTools Sources tab showing `/assets/style.css` with the `deployment-fragment-b:` comment
- [ ] **Mission 1** — DevTools Network tab showing `/mission1` response with `X-Atlas-Audit` header
- [ ] **Mission 1** — DevTools Application tab showing `audit_hint` cookie value
- [ ] **Mission 1** — Python or CyberChef showing fragment concatenation and the final flag
- [ ] **Mission 2** — `/mission2` showing all three transmissions (731-A, 731-B, 731-C)
- [ ] **Mission 2** — decoded Transmission 731-A plaintext (containing the Mission 3 case code)
- [ ] **Mission 2** — Python terminal showing the 5-step decode chain for 731-B and the final flag
- [ ] **Mission 3** — `nmap -sV <vm-ip>` output showing port 5000 open (VM/LAN deployment only)
- [ ] **Mission 3** — login page with the SQL injection payload `maya' --` entered in the username field
- [ ] **Mission 3** — Maya's employee profile at `/profile?id=102` (no flag — shows why IDOR is needed)
- [ ] **Mission 3** — Admin profile at `/profile?id=1` with case AC-731 reference in Notes (no flag yet)
- [ ] **Mission 3** — `/case?id=AC-731` access denied page (no code provided)
- [ ] **Mission 3** — `/case?id=AC-731&code=a7f3b2c1` showing the case record with the flag
- [ ] **Mission 3** — secure login page with the same SQLi payload rejected and generic error shown
- [ ] **Mission 3** — secure profile returning the 403 blocked page when `?id=1` is attempted
- [ ] **Logs demo** — `/logs-demo` showing INFO and WARNING entries from the full attack session (SQLi, IDOR, case access)

**Speaking note:**  
Screenshots are evidence of every step. For Mission 1, show all 4 channels — robots.txt, archive, CSS, header, cookie — to demonstrate that the flag was not visible in any single place. For Mission 2, show the decoded 731-A to prove the cross-mission intel flow. For Mission 3, show the case denied → case accepted sequence to demonstrate both the broken access control vulnerability and the cross-mission dependency on Mission 2 output.

---

## Slide 13 — Ethical Considerations

**Bullet points:**

- All attacks performed only against the assigned lab VM or localhost
- No denial-of-service attacks, no network flooding
- No scanning of external systems or the ngrok domain
- No real malware, no destructive payloads
- Tools used only within authorized scope: nmap, Burp Suite, CyberChef, Python, browser
- ngrok used only for HTTP sharing — not as a scan target
- All activity is educational simulation in a controlled environment
- Flags captured by exploiting challenges — not looked up or shared early

**Speaking note:**  
Ethical hacking requires written authorization and a defined scope. Our scope is this lab VM only. Every technique demonstrated here is legal within that scope and illegal outside of it. The rate limiter, the logs, and the secure versions all exist to demonstrate what real defensive systems do — this is the other side of ethical hacking.

---

## Slide 14 — Conclusion

**Key takeaways:**

- Three Hard real vulnerabilities chained across three missions
- Not static HTML — Flask + SQLite + Docker Compose + runtime secrets
- Flags live only in `.env` (gitignored) — not in the public repository; players must interact with the running app
- Full cross-mission chain: Mission 1 → key for Mission 2 → case code for Mission 3
- Four attack channels in Mission 1 (file, CSS, header, cookie) — no single source reveals the flag
- Both the attack path and the defensive fix are shown side by side
- Logging captures the complete attacker trail: SQLi → IDOR → case access
- Covers six OWASP Top 10 categories
- All 3 missions are Hard (300 pts each) — 900 pts total
- AI-resistant by design: no flag is recoverable from static source code alone

**Closing line:**  
Operation Digital Turf War demonstrates that real attacks do not happen in a single step — they chain information across vulnerabilities and require live interaction with a running system. Understanding how one misconfiguration enables another is the most important lesson in this lab, and it is the mindset of every effective security engineer.

---

## Source Code Sharing Policy

> **Important:** Do not share the GitHub repository or any source files with Red Team players before the lab session ends.
>
> Give players only:
> - The running URL (`http://<vm-ip>:5000` or the ngrok URL)
> - `PLAYER_GUIDE.md`
>
> The flags are not directly recoverable from the committed public repository alone; players must interact with the running app and collect runtime artifacts. Even so, the source reveals vulnerability logic, route structure, and the encoding scheme — sharing it before play ends gives away the full solution chain.
>
> Share the repository only after all flags have been submitted and the lab session is closed.

---

## Demo Order (Suggested)

1. Open home page — show three mission cards (all Hard, 300 pts each)
2. Mission 1:
   - Open /robots.txt → show Disallow: /archive/
   - Browse /archive/ → show directory listing with decoy files
   - Open /archive/staff_audit_731.txt → Fragment A + ATLAS + BATCH-731
   - DevTools Sources → /assets/style.css → show CSS comment with Fragment B
   - DevTools Network → /mission1 → show X-Atlas-Audit header (Fragment C)
   - DevTools Application → show audit_hint cookie (Fragment D)
   - Python: decode and concatenate A+B+C+D → flag
3. Mission 2:
   - Open /mission2 → show 3 transmissions
   - Decode 731-A (base64) live → show "Case AC-731 verification token: a7f3b2c1"
   - Python solver: derive key from ATLAS:BATCH-731 → reverse chain → 731-B flag
4. Mission 3:
   - Run nmap, show port 5000
   - Enter `maya' --` → log in as Maya → /profile?id=102 (no flag)
   - Change to /profile?id=1 → admin profile → case AC-731 reference
   - /case?id=AC-731 → access denied
   - /case?id=AC-731&code=a7f3b2c1 → flag revealed
5. Secure login — show same SQLi payload failing
6. Secure profile — show 403 blocked page
7. Logs demo — show full WARNING trail: SQLi → IDOR → case access

Total demo time: approximately 15–20 minutes depending on live speed.
