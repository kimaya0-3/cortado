# OSSEC Documentation
---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [OSSEC Variants & Distributions](#ossec-variants--distributions)
3. [Installation](#installation)
   - 3a. [Local Installation (Classic OSSEC)](#3a-local-installation-classic-ossec)
   - 3b. [Agent-Server Setup (Classic OSSEC)](#3b-agent-server-setup-classic-ossec)
4. [What OSSEC Monitors on This System](#what-ossec-monitors-on-this-system)
5. [Known Gaps & Limitations](#known-gaps--limitations)
6. [Recommended Improvements](#recommended-improvements)

---

## 1. Prerequisites
- Ubuntu Server (up to date)
- `sudo` privileges
- Internet access
- Required packages:

```bash
sudo apt-get update
sudo apt-get install -y build-essential make gcc libssl-dev git wget unzip
```

---

## 2. OSSEC Variants & Distributions

### The OSSEC Family Tree

```
OSSEC (Original Open Source)
│
├── OSSEC HIDS          → Classic, community-maintained
├── Atomic OSSEC        → Commercial, enhanced by Atomicorp
├── OSSEC+              → Extended community fork
└── Wazuh               → Most popular modern fork
```

---

### 2a. OSSEC HIDS (Classic)
> **Basic setup**

| Property | Details |
|---|---|
| **Developer** | Community / OSSEC Project |
| **License** | Open Source (GPLv2) |
| **Cost** | Free |
| **Latest Version** | 3.8.0 |
| **Website** | ossec.net |

#### What it Does
- Host-based Intrusion Detection
- File Integrity Monitoring (FIM)
- Log analysis & correlation
- Rootkit detection
- Active response

#### Best For
- Small to medium environments
- Learning & lab setups
- Basic security monitoring

#### Limitations
- No modern web UI (out of the box)
- Community support only
- Less frequent updates
- No built-in threat intelligence feeds

---

### 2b. Atomic OSSEC (by Atomicorp)
> **Commercial, enhanced version of OSSEC**

| Property | Details |
|---|---|
| **Developer** | Atomicorp |
| **License** | Commercial (paid) |
| **Cost** | Paid subscription |
| **Website** | atomicorp.com |

#### What it Does
- All classic OSSEC features
- **Atomic Secured Linux (ASL)** integration
- **ModSecurity WAF** rules included
- Built-in **threat intelligence feeds**
- **Compliance reporting** (PCI-DSS, HIPAA, etc.)
- Commercial support & SLA

#### Best For
- Enterprise environments
- Organizations needing compliance (PCI, HIPAA)
- Web server protection (Apache/Nginx + ModSecurity)

#### Limitations
- Costs money
- Vendor lock-in
- Overkill for small/lab setups

---

### 2c. OSSEC+ (Plus)
> **Community extended fork**

| Property | Details |
|---|---|
| **Developer** | Community contributors |
| **License** | Open Source |
| **Cost** | Free |

#### What it Does
- All classic OSSEC features
- Additional decoders & rules
- Extended log format support

#### Best For
- Users who want more than classic OSSEC without paying for Atomic OSSEC

#### Limitations
- Smaller community than Wazuh
- Not as actively maintained

---

### 2d. Wazuh (Most Recommended Modern Fork)
> **The most popular and actively maintained OSSEC fork**

| Property | Details |
|---|---|
| **Developer** | Wazuh Inc. |
| **License** | Open Source (GPLv2) |
| **Cost** | Free (Cloud version is paid) |
| **Website** | wazuh.com |

#### What it Does
- All classic OSSEC features
- **Modern Web UI (Wazuh Dashboard)**
- **MITRE ATT&CK** framework mapping
- **Realtime FIM** via inotify
- **Process & syscall monitoring** via auditd/eBPF
- **Vulnerability detection**
- **Cloud & container security**
- **Compliance dashboards** (PCI, HIPAA, GDPR)
- **REST API** and very active development

#### Best For
- Modern enterprise environments
- Teams wanting a full SIEM solution
- Anyone wanting free but powerful detection

#### Limitations
- Heavier resource usage
- More complex to set up
- Requires more infrastructure (Elastic/OpenSearch)

---

## 3. Installation

### Choose Installation Type
| Type | Description |
|---|---|
| `local` | Monitors only the local machine |
| `agent` | Sends logs to a central OSSEC server |
| `server` | Receives logs from multiple agents |
| `hybrid` | Acts as both server and agent |

> **For this setup we initially chose: `local` for Classic OSSEC, but have since migrated to Wazuh.**

---

### 3a. Local Installation (Classic OSSEC)

#### Download OSSEC
```bash
cd /tmp
wget https://github.com/ossec/ossec-hids/archive/3.8.0.tar.gz
tar -xvzf 3.8.0.tar.gz
cd ossec-hids-3.8.0
```

#### Run the Installer
```bash
sudo ./install.sh
```

#### Installation Prompts & Answers
| Prompt | Recommended Answer |
|---|---|
| Language | `en` |
| Installation type | `local` |
| Installation directory | `/var/ossec` (default) |
| Enable integrity check | `yes` |
| Enable rootkit detection | `yes` |
| Enable active response | `yes` |
| Enable email alerts | `no` (optional) |

---

### 3b. Agent-Server Setup (Classic OSSEC) (Optional / Advanced)

> Use this if you want **multiple machines** reporting to **one central OSSEC server**.

#### On the SERVER Machine:
```bash
cd /tmp
wget https://github.com/ossec/ossec-hids/archive/3.8.0.tar.gz
tar -xvzf 3.8.0.tar.gz
cd ossec-hids-3.8.0
sudo ./install.sh
# Choose: server
```

Then add an agent:
```bash
sudo /var/ossec/bin/manage_agents
```
- Press `A` to add an agent
- Enter agent name and IP address
- Press `E` to extract the agent key - **copy this key!**

#### On the AGENT Machine:
```bash
cd /tmp
wget https://github.com/ossec/ossec-hids/archive/3.8.0.tar.gz
tar -xvzf 3.8.0.tar.gz
cd ossec-hids-3.8.0
sudo ./install.sh
# Choose: agent
# Enter the OSSEC server IP when prompted
```

Then import the key from the server:
```bash
sudo /var/ossec/bin/manage_agents
```
- Press `I` to import the key
- Paste the key extracted from the server

#### Restart Both Server and Agent:
```bash
sudo /var/ossec/bin/ossec-control restart   # Run on both machines
```

#### Verify Agent is Connected (on server):
```bash
sudo /var/ossec/bin/agent_control -l
```

#### Expected Output:
```
OSSEC HIDS agent_control. List of available agents:
   ID: 001, Name: agent-hostname, IP: 192.168.x.x, Active
```

#### Allow OSSEC Port Through Firewall:
```bash
sudo ufw allow 1514/udp
sudo ufw reload
```

---

## 4. What OSSEC HIDS Monitors on This System

> This section documents observed behaviour from live system logs on `ubuntu-server`.

### Log Files Monitored
| Log File | Purpose |
|---|---|
| `/var/log/auth.log` | Authentication events (sudo, SSH, PAM) |
| `/var/log/syslog` | General system events (AppArmor, kernel) |
| `/var/log/dpkg.log` | Package installations and updates |

### File Integrity Monitoring (FIM)
Monitors these directories on a **scheduled scan** using MD5 + SHA1 checksums:
```
/etc    /bin    /sbin    /usr/bin    /usr/sbin
```

> FIM is **scheduled** - not realtime. Fast file changes may be missed.

### Real Alerts Observed while running RTAs
| Alert | Rule | Level | Verdict |
|---|---|---|---|
| AppArmor DENIED — Firefox snap | 52002 | 3 | Normal |
| `/bin/pkcheck` checksum changed | 550 | 7 | Package update |
| sudo session closed (kim03) | 5502 | 3 | Normal |

### Detection Flow
```
Something happens on ubuntu-server
              │
              ▼
      Does it write to a monitored log file?
         │                    │
        YES                   NO
         │                    │
         ▼                    ▼
   OSSEC reads it       OSSEC is blind
         │
         ▼
   Does a rule match?
      │         │
     YES         NO
      │         │
      ▼         ▼
  Alert fires  Silently ignored
```

---

## 5. Known Gaps & Limitations HIDS

| Attack Type | Why OSSEC Misses It |
|---|---|
| Process execution from `/tmp` | No process monitoring |
| Credential dumping via `/proc` (T1003.007) | No syscall monitoring |
| Fast file create + delete | FIM scan too slow |
| Fileless / memory-only attacks | No memory visibility |
| MITRE ATT&CK techniques | No built-in ATT&CK rules |

### Capability Summary
| Capability | Status |
|---|---|
| Auth / syslog / dpkg log monitoring | Working |
| File integrity monitoring (core dirs) | Working (scheduled) |
| Rootcheck | Working |
| AppArmor alert detection | Working |
| Realtime FIM | Not configured |
| `/tmp` monitoring | Not configured |
| Process execution monitoring | Not available |
| MITRE ATT&CK rule coverage | Not available |
| Active Response | Not configured |
| Vulnerability detection | Not available |

---

## 6. Recommended Improvements

### 1. Enable Realtime FIM on `/tmp` (if using HIDS)
```xml
<!-- Add to ossec.conf syscheck section -->
<directories realtime="yes" check_all="yes">/tmp</directories>
```

### 2. Add Process Monitoring via auditd (if using HIDS)
```bash
sudo apt install auditd -y
sudo auditctl -w /tmp -p rwxa -k tmp_execution
```

```xml
<!-- Add to ossec.conf -->
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

### 3. Migrating to Wazuh
Wazuh extends OSSEC with built-in MITRE ATT&CK rules, realtime FIM,
process/syscall monitoring, vulnerability detection, and a web dashboard with zero extra rule writing required.

#### Wazuh Installation

We have decided to migrate from Classic OSSEC to Wazuh due to its enhanced features, modern web interface, and active development. Wazuh provides a more comprehensive security monitoring solution with capabilities like real-time FIM, vulnerability detection, and MITRE ATT&CK mapping, which are crucial for our security posture.

For a detailed guide on Wazuh installation, refer to the official documentation: [Wazuh Quickstart Guide](https://documentation.wazuh.com/current/quickstart.html#installing-wazuh)

#### 3a. Wazuh Installation Steps

The Wazuh installation assistant simplifies the process of deploying the Wazuh central components (Wazuh server, Wazuh indexer, and Wazuh dashboard).

1. **Download and Run the Wazuh Installation Script:**
   We used the `curl` command to download the Wazuh installation script and then executed it with `sudo`. The `-a` flag indicates an all-in-one installation, meaning it installs the server, indexer, and dashboard on a single machine.

   ```bash
   curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
   ```


#### 3b. Wazuh Access Credentials

Upon successful completion of the installation, the script displays the credentials needed to access the Wazuh Dashboard:

```
INFO: --- Summary ---
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP_ADDRESS>
    User: admin
    Password: <ADMIN_PASSWORD>
INFO: Installation finished.
```

**To access the Wazuh web interface:**

| Field | Value |
|---|---|
| **URL** | `https://<WAZUH_DASHBOARD_IP_ADDRESS>` |
| **Username** | `admin` |
| **Password** | Displayed in terminal output after successful install |

> **Important Note on Certificates:** When accessing the Wazuh Dashboard for the first time, your browser will show a warning that the certificate was not issued by a trusted authority. This is expected for a self-signed certificate. You can safely accept the certificate as an exception, or configure a trusted CA certificate if required.

---

