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
      | nmap / browser / Python / CyberChef / Burp Suite
      v
Assigned Ethical VM (or Docker on laptop)
      |
      | Flask Web App — Port 5000
      v
SQLite Database
      |
      +── Vulnerable routes: /login, /profile
      +── Secure routes: /secure-login, /secure-profile
      +── Logs: /logs-demo → access.log
      +── Static artifacts: /static/archive/staff_audit_731.txt
```

**Deployment options:**
- Local: `python app/app.py`
- Docker: `docker compose up --build`
- Shared on demo day: ngrok exposes port 5000 (browser/Burp only — not nmap)

**Speaking note:**  
Explain that Docker gives any machine a consistent, one-command environment. The app runs on port 5000 so nmap can discover it on the assigned VM. ngrok is for HTTP sharing only — nmap goes against the actual VM IP, not the ngrok domain.

---

## Slide 5 — Mission 1: Hard Web Recon

**Fields:**
- Difficulty: Hard — 300 pts
- Category: Web Recon / Information Disclosure
- Concepts: HTML comment inspection, deployment artifact exposure, Base64 decoding
- Technique: View Page Source → follow hidden path → decode Base64 token
- OWASP: A05:2021 — Security Misconfiguration, A02:2021 — Cryptographic Failures

**Attack flow:**
```
1. View Page Source (Ctrl+U)
      ↓ finds HTML comment with path
2. Navigate to /static/archive/staff_audit_731.txt
      ↓ browser downloads plaintext audit log
3. Read file → find Base64 maintenance token + "Project keyword: ATLAS"
      ↓
4. Decode token with Python or CyberChef
      ↓
5. FLAG: picoCTF{deployment_artifacts_leak_secrets}
6. Note: "ATLAS" → save for Mission 2
```

**Why this is Hard:**  
The flag is not directly in the HTML source — there are two steps. The player must first find the comment, then navigate to an unlisted file, then decode the token. They also need to recognize that the archive file contains intelligence for the next mission. This requires careful reading, not just a search-and-find.

**Screenshot to show:** Page source with the HTML comment visible. Browser showing the archive file content. Python terminal or CyberChef decoding the token to the flag.

---

## Slide 6 — Mission 2: Hard Crypto/Encoding

**Fields:**
- Difficulty: Hard — 300 pts
- Category: Crypto / Encoding
- Concepts: XOR cipher, repeating key, cross-mission intelligence dependency
- Technique: Base64 decode → reverse → hex decode → XOR with ATLAS key
- OWASP: A02:2021 — Cryptographic Failures

**Cross-mission dependency:**  
The key `ATLAS` is found in the Mission 1 archive file (`Project keyword: ATLAS`). Without completing Mission 1 carefully, this mission cannot be solved.

**Encoding chain (Blue Team design — forward):**
```
Flag
→ XOR with repeating key ATLAS
→ Hex encode
→ Reverse hex string
→ Base64 encode
→ Payload displayed on page
```

**Decoding chain (Red Team — reverse):**
```
Payload
→ Base64 decode
→ Reverse string
→ Hex decode
→ XOR bytes with key ATLAS
→ FLAG: picoCTF{weak_custom_crypto_fails}
```

**Why this is Hard:**  
Four encoding steps must be identified and reversed in the correct order. The key must be sourced from Mission 1 intelligence — it is not given on the page. Players who did not carefully document Mission 1's findings must go back. This models real attacker behavior: intelligence gathered earlier directly enables later exploitation.

**Screenshot to show:** CyberChef or Python terminal showing the four decode steps and the final flag. Notes from Mission 1 archive showing `Project keyword: ATLAS`.

---

## Slide 7 — Mission 3: Digital Turf Breach

**Fields:**
- Difficulty: Hard — 300 pts
- Category: Web + Network + Database + Blue Team Defense
- Concepts: Network scanning, SQL Injection, IDOR / Broken Access Control
- Tools: nmap, browser, Burp Suite
- OWASP: A01, A03, A07, A09

**Attack flow summary:**
1. nmap discovers Flask on port 5000
2. Login page notice reveals username `maya`
3. SQL injection `maya' --` bypasses authentication
4. IDOR: change `/profile?id=102` to `/profile?id=1` to access admin
5. Flag captured from admin's notes field

**Why this is Hard:**  
This mission chains three techniques: network reconnaissance, SQL injection, and IDOR. The student must first find the service, then break authentication using a targeted (not generic) payload, then recognize and exploit a second vulnerability in the profile endpoint. Each step teaches a distinct real-world attack.

**Screenshots to show:** nmap output, login page with `maya' --` payload entered, Maya's profile (no flag), URL changed to `?id=1`, admin profile with flag, log entries at `/logs-demo`.

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
      ↓ discovers port 5000 / Flask
2. Open /login in browser
      ↓ read notice: username = maya
3. Enter SQL injection payload: maya' --
      ↓ authentication bypassed as Maya (id=102)
4. Observe /profile?id=102 — employee profile, no flag
      ↓
5. Change URL to /profile?id=1 — IDOR: admin profile returned
      ↓
6. Read flag from admin Notes field
      ↓
7. Document all steps + screenshots
      ↓
8. Submit flag + write-up + mitigation suggestion
```

**Speaking note:**  
Walk through each step during the demo. Show the nmap scan result. Show the login form with the targeted payload. Show Maya's profile to demonstrate why IDOR is still needed. Show the URL change and the admin profile. If time allows, show `/logs-demo` so the class can see how the attack was recorded.

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
- [ ] **Mission 1** — View Page Source with the HTML comment pointing to the archive path highlighted
- [ ] **Mission 1** — browser tab showing the archive file at `/static/archive/staff_audit_731.txt` with the Base64 token and ATLAS keyword visible
- [ ] **Mission 1** — Python terminal or CyberChef showing the Base64 decoded flag
- [ ] **Mission 2** — the encoded payload displayed on the challenge page
- [ ] **Mission 2** — Python terminal or CyberChef showing all four decode steps (Base64 → reverse → hex → XOR) and the final flag
- [ ] **Mission 2** — Mission 1 archive note showing `Project keyword: ATLAS` (evidence of cross-mission intelligence)
- [ ] **Mission 3** — `nmap -sV <vm-ip>` output showing port 5000 open (VM/LAN deployment only)
- [ ] **Mission 3** — login page with the SQL injection payload `maya' --` entered in the username field
- [ ] **Mission 3** — Maya's employee profile at `/profile?id=102` (no flag — shows why IDOR is needed)
- [ ] **Mission 3** — URL bar changed to `/profile?id=1` and admin profile page with flag visible
- [ ] **Mission 3** — secure login page with the same SQLi payload rejected and generic error shown
- [ ] **Mission 3** — secure profile returning the 403 blocked page when `?id=1` is attempted
- [ ] **Logs demo** — `/logs-demo` showing INFO and WARNING entries from the full attack session

**Speaking note:**  
Screenshots are evidence. Each one should show a specific step clearly. For Mission 1, the archive file screenshot is especially important — it should show both the Base64 token and the `Project keyword: ATLAS` line, demonstrating that cross-mission intelligence was retrieved. Use Burp Suite's HTTP history to show the raw POST request with the Mission 3 payload if available.

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

- Three Hard real vulnerabilities demonstrated on a real backend system
- Not static HTML — Flask + SQLite + Docker Compose
- Cross-mission intelligence chain: Mission 1 artifact provides the key for Mission 2
- Both the attack path and the defensive fix are shown side by side
- Logging captures what a real attacker's activity looks like
- The project covers six OWASP Top 10 categories
- All 3 missions are Hard (300 pts each) — 900 pts total
- Designed to satisfy every rubric requirement while teaching real skills

**Closing line:**  
Operation Digital Turf War demonstrates that real attacks do not happen in a single step — they chain information across vulnerabilities. Understanding how one misconfiguration enables another is the most important lesson in this lab, and it is the mindset of every effective security engineer.

---

## Source Code Sharing Policy

> **Important:** Do not share the GitHub repository or any source files with Red Team players before the lab session ends.
>
> Give players only:
> - The running URL (`http://<vm-ip>:5000` or the ngrok URL)
> - `PLAYER_GUIDE.md`
>
> The source code contains flag locations, vulnerability implementations, credential data, and the archive artifact path. Sharing it before play ends is equivalent to handing out the answer key.
>
> Share the repository only after all flags have been submitted and the lab session is closed.

---

## Demo Order (Suggested)

1. Open home page — show mission cards
2. Mission 1 — Ctrl+U on the mission page → find HTML comment → navigate to archive file → decode Base64 token live
3. Mission 2 — show payload, note ATLAS key from Mission 1, run Python solver live (all 4 steps)
4. Mission 3 — run nmap, enter `maya' --` SQLi payload, show Maya's profile (no flag), change URL to IDOR, show admin profile + flag
5. Secure login — show the same payload failing
6. Secure profile — show 403 blocked page
7. Logs demo — show WARNING entries from the attack session

Total demo time: approximately 10–15 minutes depending on live speed.
