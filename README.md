# Atlas Breach Lab

A three-challenge Capture-the-Flag cybersecurity lab for an Ethical Hacking / EMI university course.

This project is intentionally vulnerable and is meant for authorized classroom or localhost practice only.

## Challenges

| Challenge | Difficulty | Points | Category | Concept |
|-----------|------------|--------|----------|---------|
| Challenge 1 - Ghost Bundle | Hard | 300 | Client Recon | JavaScript bundle metadata leak and reversible encoding |
| Challenge 2 - Session Masquerade | Hard | 300 | Web Access Control | Client-side role token tampering |
| Challenge 3 - Console Breakout | Hard | 300 | Web Command Injection | Unsafe diagnostic command construction |
| **Total** | | **900** | | |

Each challenge has a flag in the format `picoCTF{...}`. Runtime flags are loaded from `.env`.

## Setup

Create a local environment file:

```bash
copy .env.example .env
```

Install dependencies and start the app:

```bash
pip install -r requirements.txt
py -3 app/app.py
```

Open the lab at:

```text
http://localhost:5000
```

## Docker

You can also run the lab with Docker. First create `.env` in the project folder:

```bash
copy .env.example .env
```

Edit `.env` so it contains values for the three active challenges:

```env
CTF_M4_FLAG=picoCTF{ghost_bundle_leaks_tokens}
CTF_M5_FLAG=picoCTF{client_side_roles_are_not_auth}
CTF_M6_FLAG=picoCTF{diagnostic_console_command_breakout}
CTF_MASTER_SEED=atlas-breach-lab-seed-2026
```

Then start Docker:

```bash
docker compose up --build
```

If Docker says the env file is missing, confirm you are inside the folder that contains `docker-compose.yml`, `Dockerfile`, `.env.example`, `README.md`, and `app/`.

## Active Routes

| Route | Description |
|-------|-------------|
| `/` | Challenge briefing dashboard |
| `/mission4` | Challenge 1 - Ghost Bundle |
| `/assets/telemetry.js` | Client bundle used by Challenge 1 |
| `/mission5` | Challenge 2 - Session Masquerade |
| `/mission5/vault` | Restricted vault target for Challenge 2 |
| `/mission6` | Challenge 3 - Console Breakout |
| `/logs-demo` | Security logging demonstration |

The previous challenge entry routes redirect back to the dashboard.

## Rules

- Attack only this lab environment.
- Do not run denial-of-service attacks.
- Do not scan or interact with external systems.
- Do not use real malware or destructive payloads.
- Submit the flag, tools used, steps taken, and mitigation notes for each challenge.

## Educational Purpose

The vulnerabilities in this repository are deliberately included so students can practice identifying, exploiting, explaining, and mitigating common web security issues in a safe environment.
