# Mission 1 — Hard Web Recon

## Challenge Creator Form

| Field       | Value |
|-------------|-------|
| **Name**    | Hard Web Recon |
| **Category**| Web Recon / Information Disclosure |
| **Difficulty** | Hard |
| **Points**  | 300 |
| **Route**   | `/mission1` |

### Story / Description

The Red Team targets an internal staff portal belonging to Atlas Core Systems. The rendered page looks like a standard corporate login gateway — a form, a logo, standard branding. Nothing is visible in the browser that indicates a vulnerability.

However, the development team made two compounding mistakes before shipping to production. First, they left a comment in the HTML source pointing to an internal path. Second, a deployment artifact — an audit log from the staging environment — was never removed from the web server's static file directory. That artifact contains a sensitive token.

**Your mission:** Perform web reconnaissance beyond the rendered page. Find the artifact, retrieve the token, and decode it to capture the first flag.

### Hints

1. The rendered page is not the full story.
2. Deployment environments sometimes leave files behind.

---

## Challenge Solver Report

### Initial Analysis

The mission page renders a corporate login form. The form itself is not the attack surface — no credentials are provided and the login button is not the goal. The challenge category ("Web Recon / Information Disclosure") signals that reconnaissance of the page itself is required before any other action. The first step is to inspect what the browser received, not just what it displayed.

### Tools Used

- Google Chrome or Firefox browser
- View Page Source (Ctrl+U / Cmd+U)
- Browser DevTools (F12 — Elements and Network tabs)
- Python 3 standard library (`base64` module) or CyberChef

### Exploitation Steps

**Step 1 — Inspect the page source**

Open `/mission1`. Press `Ctrl+U` (or right-click → View Page Source). Search for HTML comment blocks using Ctrl+F and searching for `<!--`.

Locate the following comment near the top of the `<body>` tag:

```html
<!-- legacy audit bundle moved to /static/archive/staff_audit_731.txt -->
```

**Step 2 — Navigate to the deployment artifact**

Open the path revealed in the comment directly in the browser:

```
http://<target-ip>:5000/static/archive/staff_audit_731.txt
```

The server returns a plain text file. Its contents:

```
ATLAS CORE SYSTEMS — Internal Audit Log
Project keyword: ATLAS
Maintenance token (base64): cGljb0NURntkZXBsb3ltZW50X2FydGlmYWN0c19sZWFrX3NlY3JldHN9
```

**Step 3 — Decode the Base64 token**

The token is encoded in Base64. Decode it using Python:

```python
import base64
token = "cGljb0NURntkZXBsb3ltZW50X2FydGlmYWN0c19sZWFrX3NlY3JldHN9"
print(base64.b64decode(token).decode())
# picoCTF{deployment_artifacts_leak_secrets}
```

Or using CyberChef: paste the token → add operation **From Base64** → read the output.

**Step 4 — Record the project keyword**

Note the value `Project keyword: ATLAS` from the artifact file. This is intelligence that will be needed for Mission 2.

### Flag Found

```
picoCTF{deployment_artifacts_leak_secrets}
```

### Learning Points

- **View Page Source reveals everything the server sent**, including HTML comments that are invisible in the rendered view. Any information placed in a comment is delivered to every browser that loads the page.
- **Deployment artifacts are a real attack surface.** Backup files, audit logs, configuration bundles, and staging assets left in the web root are accessible to anyone who knows or discovers the path. Attackers enumerate common paths (`/backup/`, `/archive/`, `/.git/`, `/.env`) routinely.
- **Base64 is encoding, not encryption.** It provides no confidentiality — anyone who has the string can decode it in seconds with freely available tools.
- **Cross-mission intelligence.** The artifact contains `Project keyword: ATLAS`, which is the XOR key for Mission 2. This is intentional — real attacks chain information from multiple sources.
- The combined vulnerability is classified under **OWASP A05:2021 — Security Misconfiguration** (artifact left in production) and **OWASP A02:2021 — Cryptographic Failures** (sensitive token obfuscated with Base64 instead of protected with encryption).

---

## Blue Team — Vulnerability Explanation

### Root Cause

Two compounding failures were introduced during the development-to-production transition:

1. A developer left an HTML comment in `mission1.html` pointing to an internal file path (`/static/archive/staff_audit_731.txt`). This comment is invisible in the rendered browser view but fully present in the page source.
2. An internal audit log (`staff_audit_731.txt`) from the staging environment was not removed before the web server was deployed. Because it is inside the Flask `static/` directory, it is served directly to any client that requests it — no authentication required.

The file contains a Base64-encoded maintenance token. While Base64 looks like an encoded secret, it is a fully reversible, keyless transformation. Any party who sees the string can recover the original in one step.

### Security Impact

- Any visitor to the site can view the page source and follow the path to the artifact file.
- The artifact exposes an internal project keyword (`ATLAS`) that can be used to break Mission 2's encoding scheme.
- The decoded token reveals the flag directly. In a real deployment, this pattern could expose API keys, database connection strings, internal service tokens, or credentials.
- Real-world examples of this class of vulnerability include `.git` directories left on web servers (exposing full source code), `.env` files in the web root (exposing all application secrets), and backup archives (`.zip`, `.tar.gz`) accessible via predictable paths.

### Mitigation Strategy

| Control | Description |
|---------|-------------|
| Pre-deployment artifact audit | Script that scans the web root for non-whitelisted file extensions and paths before deployment |
| Exclude archive paths from web root | Move `archive/`, `backup/`, `staging/` outside the `static/` directory entirely |
| Web server access controls | Configure Nginx/Apache/Flask to return 403 for any path matching `/static/archive/` |
| Remove HTML comments before build | CI/CD pipeline step that strips comment tags from all HTML templates |
| Secrets manager | Store all tokens and credentials in a vault (HashiCorp Vault, AWS Secrets Manager) — never in static files |
| Static analysis | Tools like `truffleHog`, `gitleaks`, or `detect-secrets` scan for secrets in committed files |
| Use real encryption | If a token must be transmitted, encrypt it with AES-256-GCM and a properly managed key — Base64 provides no protection |

### OWASP Reference

- **A05:2021 — Security Misconfiguration** (deployment artifact left in production web root)
- **A02:2021 — Cryptographic Failures** (Base64 used as obfuscation for a sensitive token)
