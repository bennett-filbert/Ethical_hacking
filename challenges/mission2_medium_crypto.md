# Mission 2 — Hard Crypto/Encoding

## Challenge Creator Form

| Field       | Value |
|-------------|-------|
| **Name**    | Hard Crypto/Encoding |
| **Category**| Crypto / Encoding |
| **Difficulty** | Hard |
| **Points**  | 300 |
| **Route**   | `/mission2` |

### Story / Description

Having recovered intelligence from the Atlas Core staff portal, your team intercepts a second transmission. The message has been processed through a custom encoding pipeline that one of their analysts built in-house. The payload is displayed on the page.

A field operative's note reads:

> *"They used their own scheme. It is not real encryption. But there is a key involved — and if you have been paying attention, you already have it."*

The key was embedded in the deployment artifact discovered during Mission 1. Retrieve it, reconstruct the encoding chain, reverse every step in the correct order, and recover the original message.

**This mission cannot be completed without intelligence from Mission 1.**

### Hints

1. The message was transformed more than once.
2. Intelligence from earlier in the operation may be relevant.

---

## Challenge Solver Report

### Initial Analysis

The `/mission2` page displays a Base64-looking payload. The narrative clue says a key is involved and that intelligence from Mission 1 is relevant. The Mission 1 archive file (`/static/archive/staff_audit_731.txt`) contained the line:

```
Project keyword: ATLAS
```

This is the XOR key. The challenge is to reconstruct and reverse the full encoding chain using this key.

### Tools Used

- Python 3 (`base64` module, built-in `bytes`)
- CyberChef (https://gchq.github.io/CyberChef/)
- Notes from Mission 1 archive

### Payload (displayed on the page)

```
MTM3MmQyYTMwMmEyYjBlMjcyMTM1MzYyMjJjMGMyMzIwMjIzNjIyMjMxZjMwMjYzNjM3MzIxNTEwMWUyZjJkMzEz
```

### Encoding Chain (Blue Team Design — Forward Direction)

The server applied these transformations to the flag in order:

```
Original flag: picoCTF{weak_custom_crypto_fails}

Step 1 — XOR each byte with repeating key ATLAS:
(produces raw binary bytes)

Step 2 — Hex encode:
(the XOR output expressed as a lowercase hex string)

Step 3 — Reverse the hex string:
(the entire hex string reversed character by character)

Step 4 — Base64 encode:
MTM3MmQyYTMwMmEyYjBlMjcyMTM1MzYyMjJjMGMyMzIwMjIzNjIyMjMxZjMwMjYzNjM3MzIxNTEwMWUyZjJkMzEz
```

### Decoding Chain (Red Team Performs — Reverse Direction)

```
Payload
→ Step 1: Base64 decode
→ Step 2: Reverse string
→ Step 3: Hex decode
→ Step 4: XOR bytes with key ATLAS (repeating)
→ Flag
```

### Full Python Solver

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

### CyberChef Recipe

Input: paste the payload  
Operations (add in this order):
1. **From Base64**
2. **Reverse**
3. **From Hex**
4. **XOR** — Key: `ATLAS`, Scheme: `Standard`

Output: `picoCTF{weak_custom_crypto_fails}`

### Flag Found

```
picoCTF{weak_custom_crypto_fails}
```

### Learning Points

- **XOR with a known key is not encryption.** XOR is a symmetric bitwise operation — if you know the key, decryption is identical to encryption. The strength of the scheme entirely depends on key secrecy.
- **The key was not secret.** It was embedded in a plaintext deployment artifact (`staff_audit_731.txt`) accessible via a public URL. Once an attacker has the Mission 1 artifact, they have the Mission 2 key. This is how vulnerability chaining works in real attacks — information from one weakness enables exploitation of another.
- **Encoding layers add inconvenience, not security.** Hex encoding, string reversal, and Base64 are all keyless, fully reversible transformations. Layering them does not produce confidentiality. CyberChef can reverse all of them in seconds.
- **Custom crypto is almost always weaker than standard algorithms.** Unless you are a professional cryptographer, any scheme you design will have flaws. AES-256-GCM has been publicly analyzed for decades and provides provably strong confidentiality.
- This vulnerability is classified under **OWASP A02:2021 — Cryptographic Failures** because a custom, weak scheme was used instead of proper encryption, and the key was exposed through a secondary misconfiguration.

---

## Blue Team — Vulnerability Explanation

### Root Cause

The flag was obfuscated using a custom four-step encoding pipeline: XOR with a repeating ASCII key, hex encoding, string reversal, and Base64 encoding. While this pipeline looks complex, every step is fully reversible by anyone who knows the key. The critical failure is that the key (`ATLAS`) was stored in a plaintext deployment artifact that was publicly accessible via the web server — the exact artifact that Mission 1 exploited.

This creates a cross-mission vulnerability chain: the Security Misconfiguration of Mission 1 directly enables the Cryptographic Failure of Mission 2.

### Security Impact

- Any attacker who completes Mission 1 (finding and reading the archive file) automatically obtains the XOR key for Mission 2.
- XOR with a known repeating key is trivially reversible with a five-line Python script or a four-step CyberChef recipe.
- In a real deployment, this class of failure could expose encrypted session tokens, protected API responses, or obfuscated credentials — all rendered meaningless if the key is accessible.
- The pattern of "custom obfuscation + exposed key" is common in real-world findings during penetration tests of applications built by developers without security training.

### Key Distinction: Encoding vs Encryption

| Property | XOR with exposed key | AES-256-GCM |
|----------|---------------------|-------------|
| Reversible without key | Yes (key is public) | No |
| Provides confidentiality | No | Yes |
| Provides integrity | No | Yes (authenticated encryption) |
| Key management required | Yes (and it failed here) | Yes |
| Suitable for protecting secrets | Never | Yes |

### Mitigation Strategy

| Control | Description |
|---------|-------------|
| Use real encryption | AES-256-GCM for symmetric data protection; RSA-OAEP for asymmetric |
| Protect all keys | Keys must never appear in source code, HTML, static files, or logs |
| Secrets manager | Store keys in HashiCorp Vault, AWS KMS, or Azure Key Vault — inject at runtime |
| Eliminate custom crypto | All cryptographic operations should use vetted, standard library implementations |
| Key rotation | Rotate keys on a schedule and immediately after any suspected exposure |
| Defense in depth | Even if the key leaks, the data should be protected by access controls and network-layer encryption (TLS) |

### OWASP Reference

- **A02:2021 — Cryptographic Failures** (custom XOR scheme; key exposed via deployment artifact)
- **A05:2021 — Security Misconfiguration** (key stored in publicly accessible static file — inherited from Mission 1 chain)
