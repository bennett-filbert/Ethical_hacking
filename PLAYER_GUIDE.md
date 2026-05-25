# Player Guide - Atlas Breach Lab

Start at:

```text
http://localhost:5000
```

The lab now contains three challenges only.

| Challenge | Difficulty | Points | URL |
|-----------|------------|--------|-----|
| Challenge 1 - Ghost Bundle | Hard | 300 | `/mission4` |
| Challenge 2 - Session Masquerade | Hard | 300 | `/mission5` |
| Challenge 3 - Console Breakout | Hard | 300 | `/mission6` |
| **Total** | | **900** | |

## Challenge 1 - Ghost Bundle

Inspect the frontend resources delivered to the browser. A shipped JavaScript bundle contains a retired token marker that has been transformed, not encrypted.

Recommended tools: Browser DevTools, curl, CyberChef, Python.

## Challenge 2 - Session Masquerade

The vault issues a browser role token. Decode it, inspect the claims, and test whether the backend trusts client-controlled authorization data.

Recommended tools: Browser DevTools, CyberChef, Python.

## Challenge 3 - Console Breakout

The diagnostics console accepts a host value. Test whether user input is safely separated from command syntax, then recover the simulated recovery token.

Recommended tools: Browser, Burp Suite, terminal basics.

## Rules

- Attack only the assigned VM or localhost lab.
- Do not attack external systems.
- Do not run DoS attacks.
- Do not modify or delete application files, logs, or the database during play.
- Submit each flag with a short technical write-up and mitigation suggestion.
