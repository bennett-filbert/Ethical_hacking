# Challenge 3 - Console Breakout

## Challenge Creator Form

| Field | Value |
|-------|-------|
| **Name** | Console Breakout |
| **Category** | Web / Command Injection |
| **Difficulty** | Hard |
| **Points** | 300 |
| **Route** | `/mission6` |

### Story / Description

Atlas Core exposes a support diagnostic helper that accepts a host or IP address. The backend builds a command using user input without separating data from shell syntax.

This lab uses a simulated command context so students can learn the vulnerability without executing real operating-system commands.

**Your mission:** Test the input boundary, inject an additional command, and recover the sixth flag from the simulated recovery file.

### Hints

1. Shell metacharacters can join multiple commands.
2. The diagnostic output may reveal where recovery files live.

---

## Challenge Solver Report

### Initial Analysis

The form accepts a host value and prints command-like output. Entering shell metacharacters such as `;`, `&&`, or `|` changes the response, showing that user input is treated as command syntax.

### Tools Used

- Browser
- Burp Suite or DevTools Network tab
- Terminal command-line knowledge

### Exploitation Steps

**Step 1 - Submit a normal host**

Enter:

```text
127.0.0.1
```

The page returns simulated ping output.

**Step 2 - Test command chaining**

Enter:

```text
127.0.0.1; id
```

The page reports that a command executed in the diagnostic context and hints at `/opt/atlas/recovery/`.

**Step 3 - Read the recovery flag**

Enter:

```text
127.0.0.1; cat /opt/atlas/recovery/flag.txt
```

The diagnostic output includes the flag.

### Flag Found

The flag is loaded from `CTF_M6_FLAG` at runtime.

### Learning Points

- Never pass unsanitized user input into shell commands.
- Prefer structured APIs over shelling out.
- If a shell command is unavoidable, pass arguments as an array with shell execution disabled.
- Logging suspicious metacharacters can help defenders detect exploitation attempts.

---

## Blue Team - Vulnerability Explanation

### Root Cause

The diagnostic helper models a backend that concatenates user input into a command string. Shell metacharacters let attackers append commands.

### Security Impact

In a real system, command injection can expose files, credentials, environment variables, and internal network data. It can also become remote code execution.

### Mitigation Strategy

| Control | Description |
|---------|-------------|
| Avoid shell execution | Use native network libraries instead of shell commands |
| Argument arrays | Use subprocess APIs with `shell=False` |
| Strict validation | Accept only valid hostnames/IP addresses |
| Least privilege | Run diagnostics under a low-privilege account |
| Alerting | Log and alert on metacharacters in diagnostic inputs |

### OWASP Reference

- **A03:2021 - Injection**
- **A05:2021 - Security Misconfiguration**
