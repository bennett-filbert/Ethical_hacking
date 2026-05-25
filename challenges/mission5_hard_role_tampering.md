# Challenge 2 - Session Masquerade

## Challenge Creator Form

| Field | Value |
|-------|-------|
| **Name** | Session Masquerade |
| **Category** | Web / Broken Access Control |
| **Difficulty** | Hard |
| **Points** | 300 |
| **Routes** | `/mission5`, `/mission5/vault` |

### Story / Description

Atlas Core introduced a temporary browser role token for a case archive migration. The vault reads the token and decides whether the visitor is a guest or an administrator.

The token is not signed, encrypted, or validated against a server-side session. It is only Base64URL-encoded JSON.

**Your mission:** Decode the token, modify the role claim to `admin`, place the modified token back in the browser cookie, and open the vault.

### Hints

1. Browser cookies are user-controlled.
2. Base64URL is encoding, not trust.

---

## Challenge Solver Report

### Initial Analysis

Opening `/mission5` sets a cookie named `atlas_role`. The vault displays decoded token data, which reveals that the token contains JSON with a role field.

### Tools Used

- Browser DevTools Application/Storage tab
- CyberChef or Python 3

### Exploitation Steps

**Step 1 - Inspect the cookie**

Open DevTools, inspect cookies for the site, and locate `atlas_role`.

**Step 2 - Decode Base64URL**

Decode the cookie value. It resolves to:

```json
{"user":"case-viewer","role":"guest","scope":"read"}
```

**Step 3 - Modify the role**

Change `guest` to `admin`:

```json
{"user":"case-viewer","role":"admin","scope":"read"}
```

Encode the modified JSON with URL-safe Base64 and remove trailing `=` padding.

**Step 4 - Replace the cookie and open the vault**

Set the `atlas_role` cookie to the modified value, then visit:

```text
http://<target-ip>:5000/mission5/vault
```

The vault trusts the role value and returns the flag.

### Flag Found

The flag is loaded from `CTF_M5_FLAG` at runtime.

### Learning Points

- Authorization decisions must never rely on unsigned client-controlled data.
- Encoding a cookie does not protect it from modification.
- Server-side sessions or signed tokens are required when role claims affect access.

---

## Blue Team - Vulnerability Explanation

### Root Cause

The server trusts a role claim supplied entirely by the browser. The cookie has no signature, no integrity check, and no server-side lookup.

### Security Impact

Any user can alter their role to `admin` and access restricted vault content.

### Mitigation Strategy

| Control | Description |
|---------|-------------|
| Server-side authorization | Store user roles in the database or server-side session |
| Signed tokens | Use HMAC-signed tokens if claims must be client-carried |
| Validate every request | Re-check authorization on each restricted resource |
| HttpOnly cookies | Reduce script access to session cookies |

### OWASP Reference

- **A01:2021 - Broken Access Control**
- **A07:2021 - Identification and Authentication Failures**
