# SHADOWPIPE

**AI-Powered Supply Chain Attack Simulation & Autonomous Defense**

> Operation Shadow Pipeline — a red team / blue team cybersecurity lab demonstrating how a single developer mistake (a credential committed to Git) cascades into a full organizational compromise, and how an LLM-driven autonomous agent detects, contains, and remediates that compromise in **~52 seconds**, against an industry-average breach response time of **197 days** (IBM Ponemon Institute).

Built as a capstone project for **CDAC's PGCP-ITISS** (Post Graduate Certificate Programme in IT Infrastructure Security & Surveillance).

---

## Table of Contents

- [What This Project Actually Does](#what-this-project-actually-does)
- [Why It Exists](#why-it-exists)
- [Lab Topology](#lab-topology)
- [The Attack: 10 Stages](#the-attack-10-stages)
- [The Defense: 17-Stage AI Agent](#the-defense-17-stage-ai-agent)
- [The Canary Deception Layer](#the-canary-deception-layer)
- [Live Dashboard Output](#live-dashboard-output)
- [Running the Demo](#running-the-demo)
- [Tools & Technology Stack](#tools--technology-stack)
- [Results & Metrics](#results--metrics)
- [Framework Coverage](#framework-coverage)
- [Known Limitations (Intentional)](#known-limitations-intentional)
- [Repository Layout](#repository-layout)
- [Safety Notes](#safety-notes)
- [Possible Extensions](#possible-extensions)

---

## What This Project Actually Does

SHADOWPIPE is not a slide-deck concept — it's a working 5-machine lab in which a scripted attacker actually compromises a fictional company ("**FakeCorp**") end-to-end, and a real AI agent (LLaMA 3.3 70B via Groq) actually detects, blocks, patches, and re-deceives that attacker, with every action executed through live API calls, SSH, iptables, and GitLab's REST API — not simulated log entries.

The core story it tells:

1. A developer commits `credentials.txt` to Git and later deletes it — but git history keeps it forever.
2. The web server exposes `.git/` over HTTP, so a remote attacker doesn't need any foothold to steal it.
3. That one exposed folder chains into: GitLab admin token theft → CI/CD pipeline poisoning → DNS exfiltration → a webserver root shell via command injection → a supply-chain-backdoored client → full database exfiltration — all fully automated, in seconds.
4. An AI SOC agent, triggered the instant the attack finishes, independently triages the incident, contains it, revokes every stolen credential, patches every vulnerability it just watched get exploited, ships a fixed client release, and plants a **honeytoken-based trap** for the attacker's next attempt — before generating an encrypted, framework-mapped incident report.

## Why It Exists

The project is built around one thesis: **the attack surface most teams ignore (leaked credentials in version control + unmonitored supply chains) is now more dangerous than the surface they spend the most money defending.** SHADOWPIPE demonstrates that thesis end-to-end rather than describing it, and pairs it with a second thesis: **autonomous, LLM-driven response can close the 197-day industry gap down to under a minute** — while being explicit about where that automation is still naive (see [Known Limitations](#known-limitations-intentional)).

---

## Lab Topology

Everything runs on **VMware Workstation Pro 17**, across **5 VMs** on a single **host-only network** (`VMnet2`, `192.168.20.0/24`) with **no route to the internet or the host machine** — attack traffic (reverse shells, DNS exfil, credential theft) never leaves the lab.

| Machine | IP | OS | Role |
|---|---|---|---|
| **Kali** | `192.168.20.50` | Kali Linux 2024 | Red Team — runs `attack.py`, hosts reverse shell (`:4444`) and credential-exfil (`:5555`) listeners |
| **WebServer** | `192.168.20.10` | Ubuntu Server 24.04 | FakeCorp's public site + **BillingPro** app (Node.js/Express + SQLite3, port 80) |
| **GitLab** | `192.168.20.20` | Ubuntu Server 24.04 | Self-hosted GitLab CE — code hosting, CI/CD, REST API |
| **Wazuh** | `192.168.20.30` | Ubuntu Server 24.04 | Blue Team — Suricata IDS, Wazuh SIEM, the AI agent, and the live dashboard |
| **Client** | `192.168.20.40` | Ubuntu Desktop | Victim workstation that downloads and runs the (initially backdoored) billing client |

**Port map**

| Port | Protocol | Purpose |
|---|---|---|
| `80` | HTTP | BillingPro web app / GitLab web + API |
| `55000` | HTTPS | Wazuh API |
| `9999` | HTTP | Dashboard status listener — exempted from the attacker's iptables block so status updates keep flowing on the second attack run |
| `4444` | TCP | Attacker's reverse shell listener |
| `5555` | HTTP | Attacker's credential-exfiltration listener |

---

## The Attack: 10 Stages

Run from Kali as a single script (`attack.py`), fully automated except for two shells the operator interacts with manually.

```
.git Scan → Repo Dump → Secrets → GitLab Auth → Pipeline
   → DNS Exfil → Decoy Probe → Web Shell → Supply Chain → Data Exfil
```

| # | Stage | What Happens |
|---|---|---|
| 1 | **`.git` exposure discovery** | `GET /.git/HEAD` returns `200 OK` — Express is serving dotfiles (`dotfiles: 'allow'`) it should never expose. |
| 2 | **Repository reconstruction** | `git-dumper` rebuilds the *entire* repo — including deleted files — purely from the exposed `.git/objects/`. |
| 3 | **Secret extraction** | `git log --all -p` greps every historical diff for `GITLAB_TOKEN`, `DB_PASSWORD`, `AWS_KEY` — all of which were "removed" but still live in git's object store. |
| 4 | **GitLab authentication** | The stolen Personal Access Token authenticates directly to GitLab's REST API (`GET /api/v4/user`) — PATs carry the exact privilege level of the user who created them. |
| 5 | **CI/CD pipeline poisoning** | `.gitlab-ci.yml` is overwritten via the API. The pipeline now beacons to the attacker on every future push — persistent C2 that survives a simple password reset. |
| 6 | **DNS exfiltration** | The stolen token is base64-encoded and tunneled out as DNS queries (`{encoded}.attacker.lab`) — a covert channel that many firewalls never inspect. |
| 7 | **Deception probe** | The attacker checks `/.canary`, `/.honeypot`, `/.decoy` — obvious file-level tripwires. (This is a deliberate decoy for the *real* trap — see [Canary Deception](#the-canary-deception-layer).) |
| 8 | **Webserver compromise** | SQL injection (`admin' OR '1'='1' --`) bypasses login; command injection in `/api/system/diagnostics` delivers a Python-launched Bash reverse shell — landing as **root**, since BillingPro runs with sudo. |
| 9 | **Supply chain attack** | `BillingPro_Client.py` is swapped for a backdoored version. The backdoor is a module-level daemon thread (`_svc_heartbeat()`) that opens a reverse shell via `/dev/tcp` **the instant the file is executed** — before any login prompt, before any user interaction. A second listener on `:5555` captures typed credentials live. |
| 10 | **Data exfiltration** | `UNION`-based SQL injection against `/api/customers` dumps the full `users` table (usernames, password hashes, roles). |

After Stage 10 completes, `attack.py` calls `trigger_agent()` — a single `GET` to the Wazuh dashboard listener (`:9999/trigger`) that hands control to the AI defender.

> **Why "deleting" a secret from Git doesn't work:** `git rm` only removes the file from the *working tree*. The original blob still exists, permanently, in `.git/objects/`. `git revert` doesn't help either — it adds a new commit, it doesn't erase the old one. Only `git filter-repo` / `BFG Repo-Cleaner` actually rewrites history.

---

## The Defense: 17-Stage AI Agent

Wazuh flags the attack (Active Response at severity level 15), launches `ai_agent.py`, and from there the agent runs autonomously — no human in the loop.

**Stages 1–3 — Triage**
1. Pull the latest Wazuh alerts via API
2. Ask **LLaMA 3.3 70B** (via Groq) to classify the incident: attack type, severity, attacker IP, confidence
3. Look up the attacker's IP against threat-intel feeds (AbuseIPDB, VirusTotal)

**Stages 4–7 — Containment**
4. `iptables DROP` the attacker's IP (with an explicit `ACCEPT` exemption for port `9999`, so the dashboard keeps receiving status updates even from a now-blocked attacker on the next run)
5. Revoke **every** GitLab Personal Access Token (list, then delete each one via API)
6. Overwrite the poisoned `.gitlab-ci.yml` with a clean pipeline definition
7. Patch the `.git` exposure over SSH (`rm -rf .git`)

**Stages 8–10 — Preservation**
8. Collect forensic evidence into a timestamped `INC-YYYYMMDD-HHMMSS/` folder
9. Verify the fix actually worked (`.git` → 404, IP genuinely blocked, pipeline clean)
10. Generate a **Fernet-encrypted** (AES-128-CBC + HMAC-SHA256) incident report mapped to NIST CSF

**Stages 11–17 — Remediation & Deception**
11. Detect, scan, and patch the three real RCE vectors in BillingPro (SQLi, diagnostic command injection, report-generation command injection)
12. Actively verify each patch (e.g., re-testing the exact payload that broke it originally)
13. Release a safe **v2.4.2** client build (no embedded backdoor, no credential exfiltration)
14–15. Roll out the patched client, confirm distribution
16. Deploy a **canary deception trap** — a decoy `.git` history planted specifically for the attacker's next attempt
17. Mark the incident `RESOLVED`

### The LLM in the loop

| Parameter | Value | Reason |
|---|---|---|
| Model | `llama-3.3-70b-versatile` (Groq) | Strong reasoning at free-tier cost, sub-second inference on Groq's LPU hardware |
| Temperature | `0.1` | Near-deterministic — minimizes hallucinated classifications |
| Max tokens | `256` | Forces concise, reliably parseable JSON |
| Input | Top 5 alerts, first 2000 characters | Fits comfortably in context, keeps latency low |

**Fail-safe design:** if Groq is unreachable, the agent falls back to a hardcoded worst-case decision (`CRITICAL / Supply Chain / attacker IP known`) and proceeds with full containment anyway. If the model *does* respond but returns malformed JSON, `setdefault('attack_detected', True)` forces the agent to treat it as a real attack. The system is deliberately **biased toward action over inaction** — a real design trade-off, not an oversight (see [Limitations](#known-limitations-intentional) for what that trade-off costs in a false-positive scenario).

---

## The Canary Deception Layer

This is the part of SHADOWPIPE that's easy to undersell in a demo but is arguably the most interesting design decision in the project.

- Stage 7's `/.canary`, `/.honeypot`, `/.decoy` probes are **not** the real trap — they're bait for the trap, deliberately obvious so a sophisticated attacker checks for them and feels reassured when they come back empty.
- The **actual** trap, deployed at Stage 16, is a `CANARY-{sha256[:24]}` token planted inside the **deleted git object history** of a freshly generated fake `.git` directory — structurally identical to how the original credential leak looked (3 commits, secret committed then "removed").
- Because the canary lives inside git objects rather than as a file, none of Stage 7's filesystem probes can ever find it.
- On the attacker's **next** run: they extract `CANARY-...` at Stage 3 believing it's a real token, attempt to authenticate with it at Stage 4, get an `HTTP 401` — and the `canary_trap()` function fires a "you have been caught" banner.

The attacker has to invest effort through 4 full stages of their own reconnaissance before discovering the trap — which is exactly the point: **the deception survives the attacker's own deception-detection routine.**

---

## Live Dashboard Output

A terminal-native dashboard (Python **Rich Live**, 2 Hz refresh) runs alongside the attack/defense, polling two JSON files:

| File | Written by | Contains |
|---|---|---|
| `/tmp/attack_status.json` | `attack.py` → HTTP POST to `:9999` | Current red team stage, label, status |
| `/home/wazuh/evidence/AGENT_STATUS.json` | `ai_agent.py` | Current blue team stage, label, elapsed breach time |

It shows, live:
- Header: overall state (`ATTACK` / `DEFENSE` / `RESOLVED`) and elapsed time
- A 10-row red team table and a 17-row blue team table with per-stage status
- The active evidence folder and the path to the encrypted report
- A running **breach cost counter** — `$0.29/second` (IBM Ponemon Institute: $4.88M average breach cost ÷ 197 days), which freezes the moment the incident resolves and displays the dollar amount "saved" by responding in ~52 seconds instead of 197 days.

---

## Running the Demo

```bash
# 1. Wazuh VM — resets the entire environment and starts the dashboard
bash /home/wazuh/reset_all.sh

# 2. Kali VM — kicks off the fully automated 10-stage attack
python3 /home/kali/attack.py

# 3. Kali VM — interact with Shell 1 (webserver root shell), then Ctrl-D to continue

# 4. Client VM — download and run the (currently backdoored) client
wget http://192.168.20.10/api/client/download -O bp.py && python3 bp.py
#    Log in with any credentials — the reverse shell already fired on execution

# 5. Kali VM — interact with Shell 2 (client, via supply chain RCE), then Ctrl-D

# 6. Attack completes → AI agent triggers automatically on the Wazuh VM
#    Watch the dashboard: IP blocked → tokens revoked → .git patched →
#    BillingPro patched → v2.4.2 released → canary deployed → RESOLVED

# 7. Run the attack again to see one of two outcomes:
python3 /home/kali/attack.py
#    - If .git was hard-patched:        Stage 1 fails — "VULNERABILITY PATCHED BY BLUE TEAM"
#    - If canary .git was restored:     Stage 4 triggers "YOU HAVE BEEN TRAPPED"
```

`reset_all.sh` (Wazuh side) and `reset_web.sh` (WebServer side, called over SSH) together handle firewall flushing, GitLab **token rotation** (so the leaked token in git history is valid again on the next run), and recreating the vulnerable 3-commit git history — making the whole demo repeatable indefinitely with no manual cleanup.

---

## Tools & Technology Stack

| Category | Tool | Purpose |
|---|---|---|
| Virtualization | VMware Workstation Pro 17 | 5-VM host-only lab |
| Web app | Node.js 20 + Express 4 + SQLite3 | BillingPro Enterprise (the vulnerable target app) |
| Code hosting | GitLab CE 17 + GitLab Runner | Git + CI/CD, exploited via its own REST API |
| Network IDS | Suricata 7 | 4 custom rules (DNS exfil, git-dumper signature, reverse shell, SQLi) |
| SIEM/XDR | Wazuh 4.9 | Alert correlation + Active Response trigger |
| AI | Groq API — LLaMA 3.3 70B | Autonomous SOC triage and decision-making |
| Attack tooling | `git-dumper` | Reconstructs full repos from exposed `.git/` |
| Encryption | Python `cryptography` (Fernet) | AES-128-CBC + HMAC-SHA256 signed incident reports |
| Threat intel | AbuseIPDB v2, VirusTotal v3 | IP reputation lookups |
| Dashboard | Python `rich` (Live) | Real-time terminal visualization |

---

## Results & Metrics

| Metric | Value |
|---|---|
| Attack stages | 10 |
| Defense stages | 17 |
| Independent RCE vectors | 4 |
| Virtual machines | 5 |
| MITRE ATT&CK techniques mapped | 8 |
| NIST CSF categories mapped | 12 |
| Industry average breach response | 197 days |
| SHADOWPIPE AI response time | ~52 seconds |
| Speed improvement | 99.9% faster |
| Breach cost basis | $0.29/sec (IBM Ponemon Institute) |

---

## Framework Coverage

**MITRE ATT&CK (8 techniques)** — T1592.002 (Gather Victim Host Info), T1083 (File & Directory Discovery), T1552.001 (Credentials in Files), T1078 (Valid Accounts), T1195.002 (Supply Chain Compromise), T1059.004 (Unix Shell), T1071.004 (DNS C2), T1041 (Exfiltration Over C2).

**NIST CSF (12 categories)** — IDENTIFY (2 violated), PROTECT (2 violated / 2 enforced), DETECT (4 enforced), RESPOND (5 enforced), RECOVER (3 enforced).

**OWASP Top 10** — A03 Injection (SQLi, command, report), A05 Security Misconfiguration, A07 Auth Failures, A08 Software Integrity Failures, A09 Logging Failures, A10 SSRF.

---

## Known Limitations (Intentional)

These are documented deliberately — SHADOWPIPE is meant to show that remediation is iterative, not a single silver bullet:

- **`pickle.load()` still ships in the "safe" v2.4.2 client**, used for config import/export. Pickle deserialization of untrusted data is a known RCE vector (CVSS 9.8 class). Left unfixed on purpose to demonstrate that a single remediation pass doesn't catch everything.
- **Attacker attribution is IP-based only.** A VPN or proxy would defeat the `iptables DROP` containment step — the agent would block the wrong address. Production systems would need payload-pattern correlation, not just source IP.
- **No human-in-the-loop gate before containment.** The agent's "fail open toward action" design means a false positive still triggers full containment (IP block, token revocation) with no analyst review step.
- **Sequential, not concurrent, attack/defense.** The agent only starts after all 10 attack stages finish — there's no real-time race condition, though a manual shell left open past Stage 10 could still be killed mid-session if `block_ip()` fires while it's active.
- **Reset is mandatory between runs.** Without `reset_all.sh`, the previous run's iptables block and rotated credentials persist, and the next attack simply fails at Stage 1 instead of demonstrating anything new.

---

## Repository Layout

Scripts are distributed across the 5 VMs based on role rather than living in one flat repo:

```
kali/                      # Red Team (192.168.20.50)
└── attack.py              # 10-stage automated attack

webserver/                 # Target (192.168.20.10)
├── server.js               # BillingPro (Node.js/Express + SQLite3)
├── public/.git/            # Intentionally exposed — root of the attack chain
└── reset_web.sh            # Restores vulnerable state, replants git history

gitlab/                    # Target (192.168.20.20)
└── fakecorp-app/           # GitLab CE project + CI/CD pipeline

wazuh/                     # Blue Team (192.168.20.30)
├── ai_agent.py              # 17-stage autonomous defense
├── dashboard.py             # Rich Live real-time terminal dashboard
├── reset_all.sh             # Full environment reset + token rotation
└── evidence/                # Per-incident forensic evidence + encrypted reports

client/                    # Victim (192.168.20.40)
└── BillingPro_Client.py    # Downloaded from the webserver; backdoored until Stage 9's patch ships
```

---

## Safety Notes

- The entire lab is isolated on a **host-only virtual network** with no route to the internet or the physical host — none of the attack traffic (reverse shells, DNS tunneling, credential theft) can leave the VM boundary.
- All credentials, tokens, and IPs used are fictional/lab-only and rotate automatically on every reset.
- This project is for educational and demonstrative purposes (offensive technique + AI-driven defensive response). Do not point any of the attack tooling at infrastructure you don't own or have explicit authorization to test.

## Possible Extensions

- Replace the hardcoded LLM fallback with a secondary model or a human analyst review queue
- Add SBOM generation and code signing to catch the supply-chain attack before distribution, not after
- Swap the IP-based containment for payload-signature-based attribution to survive VPN/proxy evasion
- Add a human-in-the-loop approval gate before irreversible containment actions (IP block, token revocation)
- Replace `pickle` with JSON throughout the client for config storage
- Move the Fernet key out of a flat file and into a proper secrets manager (e.g., HashiCorp Vault)
