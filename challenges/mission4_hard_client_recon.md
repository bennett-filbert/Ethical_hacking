# Challenge 1 - Ghost Bundle

## Challenge Creator Form

| Field | Value |
|-------|-------|
| **Name** | Ghost Bundle |
| **Category** | Client Recon / Information Disclosure |
| **Difficulty** | Hard |
| **Points** | 300 |
| **Routes** | `/mission4`, `/assets/telemetry.js` |

### Story / Description

Atlas Core shipped a frontend telemetry bundle during a rushed incident response. The rendered page does not show a secret, but the browser still receives JavaScript assets and response metadata.

The telemetry bundle contains a retired token marker. It was lightly transformed before release, but not encrypted.

**Your mission:** Inspect the loaded client resources, find the marker, reverse the transformation, and recover the fourth flag.

### Hints

1. The visible DOM is not the full client-side attack surface.
2. Reversing a string is not encryption.

---

## Challenge Solver Report

### Initial Analysis

The page points to a telemetry build and the response header names `/assets/telemetry.js`. Because the category is client-side disclosure, the first step is to inspect Network/Sources in DevTools or fetch the JavaScript directly.

### Tools Used

- Browser DevTools
- `curl`
- Python 3 or CyberChef

### Exploitation Steps

**Step 1 - Open the client bundle**

Navigate to:

```text
http://<target-ip>:5000/assets/telemetry.js
```

The bundle contains:

```javascript
retiredTokenMarker: "<reversed_base64_value>"
```

**Step 2 - Reverse the marker**

Reverse the marker string character by character.

**Step 3 - Decode Base64**

Decode the reversed string from Base64:

```python
import base64

marker = "<value_from_telemetry_js>"
flag = base64.b64decode(marker[::-1]).decode()
print(flag)
```

### Flag Found

The flag is loaded from `CTF_M4_FLAG` at runtime.

### Learning Points

- JavaScript, source maps, and client config are fully visible to users.
- Reversible transformations are obfuscation, not encryption.
- Build pipelines should prevent debug metadata and retired tokens from reaching production.

---

## Blue Team - Vulnerability Explanation

### Root Cause

The app exposes sensitive operational metadata inside a public JavaScript asset. The value is Base64 encoded and reversed, but those steps provide no confidentiality.

### Security Impact

Any visitor can fetch the asset and recover the token without authentication.

### Mitigation Strategy

| Control | Description |
|---------|-------------|
| Remove secrets from client assets | Never ship tokens, credentials, or privileged metadata to browsers |
| Build-time secret scanning | Scan generated JS bundles before deployment |
| Server-side authorization | Keep sensitive operations and tokens on the backend |
| Short-lived scoped tokens | Use limited tokens and rotate them after exposure |

### OWASP Reference

- **A05:2021 - Security Misconfiguration**
- **A02:2021 - Cryptographic Failures**
