# 🛡️ Sentinel — AI-Augmented SOC Home Lab

> A production-inspired Security Operations Center built from scratch: Wazuh detects attacks, an LLM triages every alert like a Tier-1 analyst, n8n orchestrates the response, and TheHive manages cases — automatically.

---

## 📌 Project Summary

Most SOC labs stop at "I installed a SIEM and watched alerts." This project goes further:

1. **Simulate real attacks** (MITRE ATT&CK techniques) against monitored endpoints
2. **Detect them** with Wazuh + Sysmon + auditd
3. **Enrich every alert** with live threat intelligence (VirusTotal, AbuseIPDB)
4. **Triage with AI** — Claude LLM reads the alert, decides true/false positive, maps it to MITRE, and writes a plain-English analyst note
5. **Respond automatically** — block malicious IPs, open TheHive cases, fire Slack alerts — all without touching a keyboard

**The result:** a measurable reduction in analyst triage time. Alert fatigue is the #1 SOC pain point. This lab quantifies the fix.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ATTACKER                                 │
│              Kali Linux — MITRE ATT&CK simulations             │
│         (nmap, hydra, Atomic Red Team, Metasploit)             │
└──────────────┬───────────────────────────┬──────────────────────┘
               │ attacks                   │ attacks
               ▼                           ▼
┌──────────────────────┐     ┌──────────────────────────────────┐
│   Windows 11 Host    │     │        Ubuntu Server VM          │
│  ┌────────────────┐  │     │  ┌────────────────────────────┐  │
│  │ Wazuh Agent    │  │     │  │ Wazuh Agent                │  │
│  │ Sysmon         │  │     │  │ auditd + audit rules       │  │
│  │ (deep telemetry│  │     │  │ (syscall monitoring)       │  │
│  └────────────────┘  │     │  └────────────────────────────┘  │
└──────────┬───────────┘     └────────────────┬─────────────────┘
           │ events (TCP 1514)                │ events (TCP 1514)
           └──────────────┬───────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Wazuh Server VM                              │
│   ┌─────────────┐  ┌────────────────┐  ┌────────────────────┐  │
│   │   Manager   │  │    Indexer     │  │    Dashboard       │  │
│   │ (rules/     │  │  (OpenSearch)  │  │   (Kibana-based)   │  │
│   │  decoders)  │  │                │  │                    │  │
│   └──────┬──────┘  └────────────────┘  └────────────────────┘  │
│          │ alert webhooks (HTTP)                                │
└──────────┼──────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    n8n — SOAR Orchestration                     │
│                                                                 │
│  [Webhook] → [Parse Alert] → [Extract IOCs]                    │
│       ↓                                                         │
│  [VirusTotal] + [AbuseIPDB]  ← Enrichment                      │
│       ↓                                                         │
│  [Claude AI] ← System prompt + enriched alert                  │
│       ↓                                                         │
│  {verdict, mitre, summary, recommended_actions}                 │
│       ↓                                                         │
│  ┌────────────┬─────────────────┬──────────────────┐           │
│  │ True Pos.  │ False Positive  │ Informational    │           │
│  │ → TheHive  │ → Suppress log  │ → Dashboard only │           │
│  │ → Slack    │                 │                  │           │
│  │ → Block IP │                 │                  │           │
│  └────────────┴─────────────────┴──────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🧰 Tech Stack

| Layer | Tool | Role |
|---|---|---|
| **SIEM** | Wazuh 4.7 | Detect, correlate, map to MITRE |
| **Telemetry (Windows)** | Sysmon + SwiftOnSecurity config | Process, network, registry, DNS |
| **Telemetry (Linux)** | auditd + custom rules | Syscalls, file access, commands |
| **Attacker** | Kali Linux | Attack simulation |
| **Attack framework** | Atomic Red Team | MITRE-mapped safe test techniques |
| **Orchestration / SOAR** | n8n (self-hosted, Docker) | Visual workflow automation |
| **Threat Intel** | VirusTotal API + AbuseIPDB API | IP/hash reputation enrichment |
| **AI Analyst** | Claude (Anthropic API) | LLM-powered triage & summarization |
| **Case Management** | TheHive | Incident tracking |
| **Alerting** | Slack webhook | Real-time analyst notification |
| **Infrastructure** | VirtualBox + Ubuntu VMs | Isolated home lab |

---

## 🖥️ Lab Infrastructure

| VM | OS | RAM | Role | Network |
|---|---|---|---|---|
| **Wazuh-Server** | Ubuntu 22.04 | 4 GB | SIEM (manager + indexer + dashboard) | NAT + Host-Only |
| **Ubuntu-Server** | Ubuntu 22.04 | 3 GB | Endpoint + n8n + TheHive host | NAT + Host-Only |
| **Kali** | Kali Linux | 2 GB | Attacker (use only when simulating attacks) | Host-Only only |
| **Windows 11** | Windows 11 Home | host | Primary monitored endpoint | Host-Only adapter |

**Host-Only network:** `192.168.56.0/24`
- Wazuh server: `192.168.56.103`
- Ubuntu server: TBD after setup
- Kali: TBD

---

## 📋 Prerequisites

- [x] VirtualBox installed on Windows 11 host
- [x] Wazuh all-in-one server installed and running
- [x] Windows 11 Wazuh agent — Active (agent 001)
- [x] Sysmon installed + Wazuh ingesting `Microsoft-Windows-Sysmon/Operational`
- [ ] Ubuntu server VM with Wazuh agent + auditd
- [ ] n8n running in Docker
- [ ] VirusTotal free API key
- [ ] AbuseIPDB free API key
- [ ] Anthropic API key (for Claude)
- [ ] Slack workspace + incoming webhook URL
- [ ] TheHive installed
- [ ] GitHub repo initialized

---

## 🗺️ Build Phases

---

### ✅ Phase 0 — Foundation

**Goal:** stable lab infrastructure, snapshots taken.

1. Allocate VM resources:
   - Wazuh VM: 4 GB RAM, 4 vCPU, 25 GB disk
   - Ubuntu server VM: 3 GB RAM, 2 vCPU, 30 GB disk
   - Kali: 2 GB RAM, 2 vCPU
2. Set both Ubuntu VMs to **NAT + Host-Only Adapter** networking
3. Take a VirtualBox snapshot of each VM before starting each phase
4. Confirm host ↔ VM connectivity: `ping 192.168.56.103` from Windows

---

### ✅ Phase 1 — Rich Telemetry

**Goal:** see deep endpoint activity, not just basic Windows event logs.

#### Part A — Sysmon on Windows

Sysmon turns Windows event logs from "a program ran" into "this process, with this hash, launched by this parent, made a connection to this IP." Essential for detection.

**Install:**
```powershell
# Download Sysmon + SwiftOnSecurity config (Admin PowerShell)
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile "$env:USERPROFILE\Downloads\Sysmon.zip"
Expand-Archive "$env:USERPROFILE\Downloads\Sysmon.zip" -DestinationPath "$env:USERPROFILE\Downloads\Sysmon" -Force
Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile "$env:USERPROFILE\Downloads\Sysmon\sysmonconfig.xml"

cd "$env:USERPROFILE\Downloads\Sysmon"
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

**Wire into Wazuh agent (`ossec.conf`):**
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**Key Sysmon Event IDs:**
| ID | What it logs |
|---|---|
| 1 | Process creation (with hash + parent) |
| 3 | Network connection |
| 7 | DLL image load |
| 11 | File creation |
| 12/13 | Registry set/delete |
| 22 | DNS query |

**Verify:** `Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5`

#### Part B — Ubuntu Wazuh Agent + auditd

**Install agent:**
```bash
# Add repo
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update && sudo apt install wazuh-agent -y
sudo sed -i 's/MANAGER_IP/192.168.56.103/' /var/ossec/etc/ossec.conf
sudo systemctl enable --now wazuh-agent
```

**Install auditd + custom rules:**
```bash
sudo apt install auditd audispd-plugins -y

sudo tee /etc/audit/rules.d/soc-lab.rules << 'EOF'
# Privilege escalation
-w /etc/sudoers -p wa -k sudoers_change
-w /etc/sudoers.d/ -p wa -k sudoers_change

# User/group changes
-w /etc/passwd -p wa -k user_change
-w /etc/shadow -p wa -k shadow_change
-w /etc/group  -p wa -k group_change

# Suspicious downloads / shells
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/wget -k download
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/curl -k download
-a always,exit -F arch=b64 -S execve -F path=/bin/nc -k netcat

# SSH key tampering
-w /root/.ssh -p wa -k ssh_keys
-w /home -p wa -k ssh_keys
EOF

sudo augenrules --load && sudo systemctl restart auditd
```

**Verify:** both `WIN-NSOIB3358K2` and `ubuntu-server` show **Active** in the Wazuh dashboard → Agents.

---

### Phase 2 — Attack Simulation

**Goal:** generate real detections that prove the pipeline works before building the AI layer.

#### Atomic Red Team (Windows)
Atomic Red Team is a library of safe, single-command tests mapped to MITRE ATT&CK techniques. Each test = one real attacker technique run safely in your lab.

**Install:**
```powershell
# Admin PowerShell
Install-Module -Name invoke-atomicredteam,powershell-yaml -Force
Import-Module invoke-atomicredteam
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics -Force
```

**High-signal test attacks (run one at a time, watch Wazuh after each):**
```powershell
# T1059.001 — PowerShell execution (attacker uses encoded PS to hide commands)
Invoke-AtomicTest T1059.001 -TestNumbers 1

# T1003.001 — LSASS credential dump attempt (attacker steals passwords)
Invoke-AtomicTest T1003.001 -TestNumbers 1

# T1136.001 — Create a local account (persistence)
Invoke-AtomicTest T1136.001 -TestNumbers 1

# T1053.005 — Scheduled task creation (persistence)
Invoke-AtomicTest T1053.005 -TestNumbers 1

# T1112 — Registry modification (persistence/defense evasion)
Invoke-AtomicTest T1112 -TestNumbers 1
```

#### Kali attacks (from Kali VM → Windows/Ubuntu)
```bash
# SSH brute force against Ubuntu
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.x

# Port scan (reconnaissance)
nmap -sS -sV -O 192.168.56.x

# Web directory brute force
gobuster dir -u http://192.168.56.x -w /usr/share/wordlists/dirb/common.txt
```

**For each detection in Wazuh, record:**
- Rule ID + level
- MITRE technique + tactic
- Raw alert fields
- True positive / false positive judgment

---

### Phase 3 — n8n Setup (SOAR Brain)

**Goal:** n8n running in Docker on the Ubuntu VM, reachable from your Windows browser.

#### Install Docker
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
```

#### n8n docker-compose
```bash
mkdir ~/n8n && cd ~/n8n
```

Create `docker-compose.yml`:
```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - N8N_ENCRYPTION_KEY=changeme-use-a-long-random-string
      - WEBHOOK_URL=http://192.168.56.x:5678/
    volumes:
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
```

```bash
docker compose up -d
```

**Access:** `http://192.168.56.x:5678` from your Windows browser.

---

### Phase 4 — Wazuh → n8n Pipeline

**Goal:** every Wazuh alert above level 7 automatically fires an n8n workflow execution.

#### n8n side: create a Webhook node
1. New workflow → add **Webhook** node
2. Method: POST, path: `wazuh-alert`
3. Copy the webhook URL (e.g. `http://192.168.56.x:5678/webhook/wazuh-alert`)
4. Activate the workflow

#### Wazuh side: integration script
On the **Wazuh server VM**, create `/var/ossec/integrations/custom-n8n`:
```bash
sudo tee /var/ossec/integrations/custom-n8n << 'SCRIPT'
#!/usr/bin/env python3
import json, sys, urllib.request

alert_file = sys.argv[1]
with open(alert_file) as f:
    alert = json.load(f)

payload = json.dumps(alert).encode()
req = urllib.request.Request(
    'http://192.168.56.x:5678/webhook/wazuh-alert',
    data=payload,
    headers={'Content-Type': 'application/json'},
    method='POST'
)
urllib.request.urlopen(req, timeout=10)
SCRIPT

sudo chmod 750 /var/ossec/integrations/custom-n8n
sudo chown root:wazuh /var/ossec/integrations/custom-n8n
```

Add to `/var/ossec/etc/ossec.conf` (inside `<ossec_config>`):
```xml
<integration>
  <name>custom-n8n</name>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

```bash
sudo systemctl restart wazuh-manager
```

**Verify:** trigger a test alert on Windows (`net user hacker /add`) and watch n8n **Executions** — you should see a live execution appear.

---

### Phase 5 — Enrichment (in n8n)

**Goal:** attach reputation data to every IOC before handing the alert to the AI.

#### n8n workflow nodes (add after Webhook):

**1. Extract IOCs (Code node):**
```javascript
const alert = $input.first().json;
const srcIp = alert?.data?.srcip || alert?.agent?.ip || null;
const fileHash = alert?.data?.win?.eventdata?.hashes?.split(',')
  .find(h => h.startsWith('SHA256='))?.replace('SHA256=','') || null;

return [{ json: { alert, srcIp, fileHash } }];
```

**2. VirusTotal IP check (HTTP Request node):**
- Method: GET
- URL: `https://www.virustotal.com/api/v3/ip_addresses/{{ $json.srcIp }}`
- Header: `x-apikey: YOUR_VT_KEY`
- Only execute if `srcIp` is not null

**3. AbuseIPDB check (HTTP Request node):**
- Method: GET
- URL: `https://api.abuseipdb.com/api/v2/check?ipAddress={{ $json.srcIp }}&maxAgeInDays=90`
- Header: `Key: YOUR_ABUSEIPDB_KEY`

**4. Merge enrichment (Code node):**
```javascript
const alert = $('Extract IOCs').first().json.alert;
const vtData = $('VirusTotal').first()?.json || {};
const abuseData = $('AbuseIPDB').first()?.json?.data || {};

return [{
  json: {
    alert,
    enrichment: {
      ip: alert.data?.srcip,
      vt_malicious: vtData?.data?.attributes?.last_analysis_stats?.malicious || 0,
      vt_total: vtData?.data?.attributes?.last_analysis_stats?.total || 0,
      abuse_confidence: abuseData?.abuseConfidenceScore || 0,
      abuse_country: abuseData?.countryCode || 'unknown',
      abuse_reports: abuseData?.totalReports || 0
    }
  }
}];
```

---

### Phase 6 — AI Analyst (Claude) ⭐

**Goal:** every enriched alert gets analyzed by an LLM that produces a structured analyst verdict.

**Why this is the differentiator:** SOC analysts spend most of their day reading noisy alerts, looking up IOCs, and writing "this is a false positive because..." notes. This phase automates exactly that.

#### n8n HTTP Request node — Claude API call:
- Method: POST
- URL: `https://api.anthropic.com/v1/messages`
- Headers:
  - `x-api-key: YOUR_ANTHROPIC_KEY`
  - `anthropic-version: 2023-06-01`
  - `content-type: application/json`

**Request body:**
```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "system": "You are a Tier-1 SOC analyst. Analyze the provided security alert and enrichment data. Respond ONLY with a valid JSON object in this exact format:\n{\n  \"is_true_positive\": true or false,\n  \"confidence\": \"high\" | \"medium\" | \"low\",\n  \"severity\": \"critical\" | \"high\" | \"medium\" | \"low\" | \"informational\",\n  \"mitre_technique\": \"T#### — Name\" or null,\n  \"mitre_tactic\": \"Tactic name\" or null,\n  \"summary\": \"2-3 sentence plain-English explanation of what happened and why it matters\",\n  \"recommended_actions\": [\"action 1\", \"action 2\"],\n  \"false_positive_reason\": \"explanation if false positive, else null\"\n}\nBe concise. Do not add any text outside the JSON.",
  "messages": [
    {
      "role": "user",
      "content": "Alert:\n{{ JSON.stringify($json.alert, null, 2) }}\n\nEnrichment:\n{{ JSON.stringify($json.enrichment, null, 2) }}"
    }
  ]
}
```

**Parse response (Code node):**
```javascript
const response = $input.first().json;
const text = response.content[0].text;
const verdict = JSON.parse(text);

return [{
  json: {
    ...$('Merge Enrichment').first().json,
    verdict
  }
}];
```

#### Sample AI verdict output:
```json
{
  "is_true_positive": true,
  "confidence": "high",
  "severity": "high",
  "mitre_technique": "T1110.001 — Password Guessing",
  "mitre_tactic": "Credential Access",
  "summary": "Multiple failed SSH authentication attempts detected from 185.220.101.47 over 30 seconds, consistent with an automated brute-force attack. AbuseIPDB reports 847 prior abuse reports for this IP with 100% confidence score. VirusTotal flags it as malicious by 12/87 vendors.",
  "recommended_actions": [
    "Block 185.220.101.47 at the firewall immediately",
    "Review SSH access logs for any successful logins from this IP",
    "Consider implementing fail2ban or similar rate-limiting"
  ],
  "false_positive_reason": null
}
```

---

### Phase 7 — Response & Case Management

**Goal:** the AI verdict automatically drives the right response — no analyst needed for the first cut.

#### n8n Switch node (branch on verdict):

**Branch 1: True Positive → High severity**
- Create **TheHive** case via API
- Post **Slack** alert with the AI summary
- Trigger **Wazuh Active Response** to block the source IP

**TheHive case creation (HTTP Request):**
```json
{
  "title": "{{ $json.verdict.mitre_technique }} — {{ $json.alert.agent.name }}",
  "description": "{{ $json.verdict.summary }}\n\nRecommended actions:\n{{ $json.verdict.recommended_actions.join('\n') }}",
  "severity": 3,
  "tags": ["{{ $json.verdict.mitre_tactic }}", "wazuh", "ai-triaged"],
  "tlp": 2
}
```

**Slack notification (HTTP Request to webhook):**
```json
{
  "text": "🚨 *Security Alert — {{ $json.verdict.severity.toUpperCase() }}*\n*{{ $json.verdict.mitre_technique }}*\n{{ $json.verdict.summary }}\n\n*Actions:*\n• {{ $json.verdict.recommended_actions.join('\n• ') }}"
}
```

**Wazuh Active Response — block IP:**
```bash
# On Wazuh server, configure active-response in ossec.conf:
# <active-response>
#   <command>firewall-drop</command>
#   <location>local</location>
#   <level>10</level>
# </active-response>
```

**Branch 2: False Positive**
- Log to a Google Sheet / local file for tuning metrics
- No alert fired (this is the alert-fatigue reduction)

**Branch 3: Informational**
- Enrich the Wazuh alert in the dashboard only
- No action

---

### Phase 8 — Metrics & Dashboard

**Goal:** quantify the value of the AI layer. This is what you put in your portfolio.

#### Metrics to track (log each alert verdict to a CSV/Sheet):
- Total alerts per day
- AI-classified as true positive / false positive / informational
- Confidence level distribution
- Time from alert → verdict (should be ~3–5 seconds)
- Cases auto-created in TheHive
- IPs auto-blocked

#### Sample before/after framing:
| Metric | Without AI | With AI |
|---|---|---|
| Alerts reviewed manually | 47/day | 8/day (high-conf TP only) |
| Avg triage time per alert | ~8 min | ~30 sec (AI pre-read) |
| False positive rate flagged | mixed in noise | isolated + logged |
| Case creation | manual | automatic |

---

### Phase 9 — Portfolio Polish

**Goal:** GitHub repo that shows up when a recruiter searches for "wazuh siem" or "ai soc."

#### Repo structure:
```
ai-soc-analyst/
├── README.md                    ← this file
├── architecture/
│   └── diagram.svg              ← architecture diagram
├── wazuh/
│   ├── ossec-windows.conf       ← agent config snippet
│   ├── ossec-ubuntu.conf        ← agent config snippet
│   └── custom-n8n               ← integration script
├── sysmon/
│   └── sysmonconfig.xml         ← SwiftOnSecurity config
├── auditd/
│   └── soc-lab.rules            ← custom auditd rules
├── n8n/
│   └── wazuh-ai-triage.json     ← exported n8n workflow
├── thehive/
│   └── setup-notes.md
├── atomic-red-team/
│   └── test-playbook.md         ← which tests, what to expect
├── ai-analyst/
│   └── system-prompt.txt        ← the Claude prompt (core IP)
├── metrics/
│   └── sample-results.csv       ← example alert verdicts
└── demo/
    └── demo-script.md           ← what to show in the video
```

#### GitHub repo topics to add:
`wazuh`, `siem`, `soc`, `cybersecurity`, `threat-detection`, `llm`, `claude-ai`, `n8n`, `mitre-attack`, `homelab`, `soar`, `incident-response`, `blue-team`, `detection-engineering`

#### Demo video script (3 minutes):
1. **0:00–0:30** — show the architecture diagram, explain the concept in 3 sentences
2. **0:30–1:00** — run `hydra` SSH brute-force from Kali, show alert appearing in Wazuh
3. **1:00–1:45** — switch to n8n, watch the workflow execute node-by-node (VirusTotal → AbuseIPDB → Claude)
4. **1:45–2:15** — show the Claude verdict JSON: true positive, T1110, summary, recommended actions
5. **2:15–2:45** — show TheHive case auto-created + Slack message received
6. **2:45–3:00** — show the metrics: "out of 47 alerts, 8 required human review"

---

## 🔑 API Keys Needed (all have free tiers)

| Service | Where to get | Free limit |
|---|---|---|
| Anthropic (Claude) | console.anthropic.com | Pay-per-token, very cheap |
| VirusTotal | virustotal.com/gui/join-us | 4 req/min |
| AbuseIPDB | abuseipdb.com/register | 1,000 req/day |
| Slack webhook | api.slack.com/apps | Free |

---

## 📈 Skills Demonstrated

This project maps to real SOC/security engineering job requirements:

| Skill | Where demonstrated |
|---|---|
| SIEM deployment & tuning | Wazuh server setup, custom rules |
| Endpoint detection | Sysmon config, auditd rules |
| MITRE ATT&CK | Every test mapped to techniques |
| Threat intelligence | VirusTotal + AbuseIPDB integration |
| SOAR / automation | n8n workflows |
| LLM integration | Claude API, prompt engineering |
| Incident response | TheHive case management |
| Python / scripting | Wazuh integration script |
| Linux administration | Ubuntu VM, Docker, systemd |
| Detection engineering | Custom Wazuh rules, alert tuning |

---

## 🚧 Current Status

- [x] Phase 0 — Infrastructure
- [x] Phase 1A — Sysmon on Windows
- [ ] Phase 1B — Ubuntu agent + auditd
- [ ] Phase 2 — Attack simulation
- [ ] Phase 3 — n8n setup
- [ ] Phase 4 — Wazuh → n8n pipeline
- [ ] Phase 5 — Enrichment
- [ ] Phase 6 — AI analyst (Claude)
- [ ] Phase 7 — Response & cases
- [ ] Phase 8 — Metrics
- [ ] Phase 9 — Portfolio polish

---

## 👤 Author

**Hamza** — Cybersecurity student, SOC internship candidate at Ribatis (Tier 3 Datacenter)

---

*Built as a hands-on preparation for a real SOC internship. Every component reflects tools and workflows used in production environments.*
