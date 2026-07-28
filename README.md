# SHADOWPIPE
> **AI-Powered Supply Chain Attack Simulation & Autonomous Defense**

<p align="center">
  <img src="https://img.shields.io/badge/Red%20Team-10%20Stages-red">
  <img src="https://img.shields.io/badge/Blue%20Team-17%20Stages-green">
  <img src="https://img.shields.io/badge/Wazuh-SIEM-blue">
  <img src="https://img.shields.io/badge/Suricata-IDS-orange">
  <img src="https://img.shields.io/badge/Groq-Llama%203.3%2070B-purple">
</p>

## Overview

**SHADOWPIPE** demonstrates how a single exposed `.git` repository can escalate into a complete supply-chain compromise—and how an autonomous AI SOC detects, contains, remediates, and documents the incident.

### Lab Architecture

```text
                 VMware Host-Only Network (192.168.20.0/24)

      Kali Linux
    192.168.20.50
         │
         ├──────────────┐
         │              │
         ▼              ▼
   Web Server      GitLab CE
192.168.20.10   192.168.20.20
         │
         ▼
   Wazuh + Suricata
   192.168.20.30
         │
         ▼
     Client VM
   192.168.20.40
```

---

# Attack Chain

| Stage | Description |
|-------|-------------|
|1|Discover exposed `.git`|
|2|Dump repository|
|3|Recover deleted credentials|
|4|Authenticate to GitLab|
|5|Poison CI/CD pipeline|
|6|DNS data exfiltration|
|7|Probe deception|
|8|Command Injection → Reverse Shell|
|9|Supply Chain compromise|
|10|Database exfiltration|

---

# AI Response

```text
Alerts
   │
   ▼
Wazuh API
   │
   ▼
Llama 3.3 70B
   │
   ▼
Decision Engine
   │
   ├── Block Attacker
   ├── Revoke GitLab Tokens
   ├── Restore Pipeline
   ├── Patch Application
   ├── Deploy Canary
   └── Generate Incident Report
```

---

# Project Structure

```text
shadowpipe/
│
├── attack.py
├── ai_agent.py
├── dashboard.py
├── listener.py
├── reset_lab.sh
├── BillingPro/
├── Suricata/
├── Wazuh/
├── evidence/
└── reports/
```

---

# Demo

## 1. Start the Listener

```bash
python3 listener.py
```

Output

```text
[*] HTTP Listener Started
[*] Waiting for attack status...
```

---

## 2. Launch Attack

```bash
python3 attack.py
```

Output

```text
[1/10] .git exposed
[2/10] Repository dumped
[3/10] GitLab token recovered
[4/10] Pipeline poisoned
...
[10/10] Database exfiltration complete
```

---

## 3. AI Detection

```text
[AI] Critical Supply Chain Attack Detected
Threat Score : 98/100
MITRE : T1195.002
```

---

## 4. Autonomous Remediation

```text
✓ Firewall updated
✓ GitLab tokens revoked
✓ Malicious pipeline removed
✓ .git exposure fixed
✓ Vulnerabilities patched
✓ Canary deployed
✓ Incident report generated
```

---

# Example Vulnerability

### Vulnerable Code

```javascript
db.get(
 "SELECT * FROM users WHERE username='"+user+
 "' AND password='"+pass+"'"
);
```

### Patched Version

```javascript
db.get(
 "SELECT * FROM users WHERE username=? AND password=?",
 [user, pass]
);
```

---

# Technologies

- Python
- Node.js / Express
- GitLab CE
- SQLite
- Wazuh SIEM
- Suricata IDS
- VMware Workstation
- Groq API
- Llama 3.3 70B

---

# MITRE Coverage

- T1552.001
- T1078
- T1195.002
- T1071.004
- T1041
- T1059.004

---

# Security Improvements

- Removed exposed `.git`
- Revoked compromised PATs
- Secure SQL queries
- Removed command injection
- Restored CI/CD
- Released clean client
- Canary deception deployment
- Encrypted forensic reporting

---

# Author

**Kshitij Shrivastava**

CDAC PGCP-ITISS Capstone Project

> Educational use only. All attacks execute inside an isolated VMware lab.

