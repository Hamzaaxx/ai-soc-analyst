# AI-Augmented SOC Home Lab

> A production-inspired Security Operations Center built from scratch: Wazuh detects attacks, an LLM triages every alert like a Tier-1 analyst, n8n orchestrates the response, and TheHive manages cases — automatically.

---

## 📌 Project Summary

Most SOC labs stop at "I installed a SIEM and watched alerts." This project goes further:

1. **Simulate real attacks** (MITRE ATT&CK techniques) against monitored endpoints
2. **Detect them** with Wazuh + Sysmon + auditd
3. **Enrich every alert** with live threat intelligence (VirusTotal, AbuseIPDB)
4. **Triage with AI** — Llama 3.3 70B (via Groq) reads the alert, decides true/false positive, maps it to MITRE, and writes a plain-English analyst note
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
│  [Groq AI / Llama 3.3 70B] ← System prompt + enriched alert   │
│       ↓                                                         │
│  {verdict, mitre, summary, recommended_actions}                 │
│       ↓                                                         │
│  ┌────────────────────┬─────────────────┬──────────────────┐    │
│  │ True Positive      │ False Positive  │ Informational    │    │
│  │ → Slack alert      │ → Suppress log  │ → Dashboard only │    │
│  │ → TheHive (planned)│                 │                  │    │
│  └────────────────────┴─────────────────┴──────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🧰 Tech Stack

| Layer | Tool | Role |
|---|---|---|
| **SIEM** | Wazuh 4.14.5 | Detect, correlate, map to MITRE |
| **Telemetry (Windows)** | Sysmon + SwiftOnSecurity config | Process, network, registry, DNS |
| **Telemetry (Linux)** | auditd + custom rules | Syscalls, file access, commands |
| **Attacker** | Kali Linux | Attack simulation |
| **Attack framework** | Atomic Red Team | MITRE-mapped safe test techniques |
| **Orchestration / SOAR** | n8n (self-hosted, Docker) | Visual workflow automation |
| **Threat Intel** | VirusTotal API + AbuseIPDB API | IP/hash reputation enrichment |
| **AI Analyst** | Llama 3.3 70B via Groq API | LLM-powered triage & summarization |
| **Case Management** | TheHive (planned) | Incident tracking |
| **Alerting** | Slack webhook | Real-time analyst notification |
| **Infrastructure** | VirtualBox + Ubuntu VMs | Isolated home lab |

---

## 🖥️ Lab Infrastructure

| VM | OS | RAM | Role | Network |
|---|---|---|---|---|
| **Wazuh-Serverr** | Ubuntu (64-bit) | 4 GB | SIEM (manager + indexer + dashboard) | NAT + Host-Only |
| **ubuntu** | Ubuntu (64-bit) | 2.6 GB | Endpoint + n8n + Docker host | NAT + Host-Only |
| **kali-linux-2025.4** | Kali Linux | 2 GB | Attacker (use only when simulating attacks) | NAT + Host-Only |
| **Windows 11** | Windows 11 Home | host | Primary monitored endpoint | Host (192.168.56.1) |

**Host-Only network:** `192.168.56.0/24`
- Wazuh server: `192.168.56.103`
- Ubuntu server: `192.168.56.104`
- Windows host: `192.168.56.1`

---

## 📋 Prerequisites

- [x] VirtualBox installed on Windows 11 host
- [x] Wazuh all-in-one server installed and running (`192.168.56.103`)
- [x] Windows 11 Wazuh agent — Active (agent 001, WIN-NSOIB3358K2)
- [x] Sysmon installed + Wazuh ingesting `Microsoft-Windows-Sysmon/Operational`
- [x] Ubuntu server VM with Wazuh agent + auditd (`192.168.56.104`)
- [x] n8n running in Docker on Ubuntu VM
- [x] VirusTotal free API key
- [x] AbuseIPDB free API key
- [x] Groq API key (Llama 3.3 70B)
- [x] Slack workspace + incoming webhook UR
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
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.104

# Port scan (reconnaissance)
nmap -sS -sV -O 192.168.56.104

# Web directory brute force
gobuster dir -u http://192.168.56.104 -w /usr/share/wordlists/dirb/common.txt
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
      - WEBHOOK_URL=http://192.168.56.104:5678/
      - N8N_SECURE_COOKIE=false
    volumes:
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
```

```bash
docker compose up -d
```

**Access:** `http://192.168.56.104:5678` from your Windows browser.

---

### Phase 4 — Wazuh → n8n Pipeline

**Goal:** every Wazuh alert above level 7 automatically fires an n8n workflow execution.

#### n8n side: create a Webhook node
1. New workflow → add **Webhook** node
2. Method: POST, path: `wazuh-alert`
3. Copy the webhook URL (e.g. `http://192.168.56.104:5678/webhook/wazuh-alert`)
4. Activate the workflow

#### Wazuh side: integration script
On the **Wazuh server VM**, create `/var/ossec/integrations/custom-n8n`:
```bash
sudo tee /var/ossec/integrations/custom-n8n << 'SCRIPT'
#!/usr/bin/env python3
import sys, json, urllib.request, urllib.error

N8N_WEBHOOK = 'http://192.168.56.104:5678/webhook/wazuh-alert'
try:
    with open(sys.argv[1]) as f:
        alert = json.load(f)
    payload = json.dumps(alert).encode('utf-8')
    req = urllib.request.Request(
        N8N_WEBHOOK,
        data=payload,
        headers={'Content-Type': 'application/json'},
        method='POST'
    )
    urllib.request.urlopen(req, timeout=10)
except Exception as e:
    with open('/var/ossec/logs/n8n-integration.log', 'a') as log:
        log.write(f'ERROR: {e}\n')
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

**1. Extract IOCs (node name: "Code in JavaScript"):**
```javascript
const alert = $input.first().json;
const srcIp = alert?.data?.srcip || alert?.agent?.ip || null;
const fileHash = alert?.data?.win?.eventdata?.hashes?.split(',')
  .find(h => h.startsWith('SHA256='))?.replace('SHA256=','') || null;
const ruleLevel = alert?.rule?.level;
const ruleDesc = alert?.rule?.description;
const agentName = alert?.agent?.name;
const ruleId = alert?.rule?.id;

return [{ json: { alert, srcIp, fileHash, ruleLevel, ruleDesc, agentName, ruleId } }];
```

**2. VirusTotal IP check (node name: "HTTP Request"):**
- Method: GET
- URL: `https://www.virustotal.com/api/v3/ip_addresses/{{ $json.srcIp || '127.0.0.1' }}`
- Header: `x-apikey: YOUR_VT_KEY`

> Note: the `|| '127.0.0.1'` fallback handles alerts with no source IP (e.g. local Windows commands like `net user`). VirusTotal returns a benign result for localhost.

**3. AbuseIPDB check (node name: "HTTP Request1"):**
- Method: GET
- URL: `https://api.abuseipdb.com/api/v2/check`
- Query params: `ipAddress: {{ $('Code in JavaScript').first().json.srcIp || '127.0.0.1' }}`, `maxAgeInDays: 90`
- Headers: `Key: YOUR_ABUSEIPDB_KEY`, `Accept: application/json`

**4. Merge enrichment (node name: "Code in JavaScript1"):**
```javascript
const base = $('Code in JavaScript').first().json;
const vtData = $('HTTP Request').first()?.json || {};
const abuseData = $('HTTP Request1').first()?.json?.data || {};

return [{
  json: {
    alert: base.alert,
    srcIp: base.srcIp,
    agentName: base.agentName,
    enrichment: {
      ip: base.srcIp,
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

### Phase 6 — AI Analyst (Llama 3.3 70B via Groq) ⭐

**Goal:** every enriched alert gets analyzed by an LLM that produces a structured analyst verdict.

**Why this is the differentiator:** SOC analysts spend most of their day reading noisy alerts, looking up IOCs, and writing "this is a false positive because..." notes. This phase automates exactly that.

#### n8n workflow — Groq API call:
- Method: POST
- URL: `https://api.groq.com/openai/v1/chat/completions`
- Headers:
  - `Authorization: Bearer YOUR_GROQ_KEY`
  - `Content-Type: application/json`

**Request body (built via Code node):**
```javascript
const data = $input.first().json;
const prompt = `You are a Tier-1 SOC analyst. Analyze this alert and respond ONLY with valid JSON.
Format: {"is_true_positive": true, "confidence": "high", "severity": "high", "mitre_technique": "T#### - Name", "mitre_tactic": "Tactic", "summary": "explanation", "recommended_actions": ["action"], "false_positive_reason": null}
Alert: ${JSON.stringify(data.alert)}
Enrichment: ${JSON.stringify(data.enrichment)}`;

return [{ json: { ...data, groq_payload: {
  model: "llama-3.3-70b-versatile",
  temperature: 0.1,
  max_tokens: 1024,
  messages: [
    { role: "system", content: "You are a Tier-1 SOC analyst. Respond ONLY with valid JSON." },
    { role: "user", content: prompt }
  ]
}}}];
```

**Parse response (Code node):**
```javascript
const data = $input.first().json;
const raw = data.choices[0].message.content;
const verdict = JSON.parse(raw);

return [{
  json: {
    alert: $('Code in JavaScript').first().json.alert,
    enrichment: $('Code in JavaScript').first().json.enrichment,
    srcIp: $('Code in JavaScript').first().json.srcIp,
    agentName: $('Code in JavaScript').first().json.agentName,
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
`wazuh`, `siem`, `soc`, `cybersecurity`, `threat-detection`, `llm`, `groq`, `n8n`, `mitre-attack`, `homelab`, `soar`, `incident-response`, `blue-team`, `detection-engineering`

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
| Groq (Llama 3.3 70B) | console.groq.com | Free tier, generous limits |
| VirusTotal | virustotal.com/gui/join-us | 4 req/min free |
| AbuseIPDB | abuseipdb.com/register | 1,000 req/day free |
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
| LLM integration | Groq API (Llama 3.3 70B), prompt engineering |
| Incident response | TheHive case management |
| Python / scripting | Wazuh integration script |
| Linux administration | Ubuntu VM, Docker, systemd |
| Detection engineering | Custom Wazuh rules, alert tuning |
