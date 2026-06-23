# Wazuh Complete Installation & Configuration Guide

## Table of Contents
1. [Why Wazuh Over OSSEC](#why-wazuh-over-ossec)
2. [Why OSSEC Cannot Fulfill Our Needs](#why-ossec-cannot-fulfill-our-needs)
3. [Wazuh Installation](#wazuh-installation)
4. [Audit Installation & Configuration](#audit-installation--configuration)
5. [Auditd Noise - Raw vs Fine-Tuned](#auditd-noise--raw-vs-fine-tuned)
6. [Additional Features](#additional-features)

---

## Why Wazuh Over OSSEC

### What is OSSEC?
OSSEC (Open Source HIDS Security) is a free, open-source host-based intrusion detection system. It was the foundation that Wazuh was originally forked from. CUrrently OSSEC has significant limitations in modern threat detection environments.

### What is Wazuh?
Wazuh is a free, open-source security platform that evolved from OSSEC. It provides unified XDR (Extended Detection and Response) and SIEM (Security Information and Event Management) capabilities. It is actively maintained, enterprise-ready, and significantly more capable than its predecessor.

### Feature Comparison

| Feature | OSSEC | Wazuh |
|---|---|---|
| Active maintenance | Minimal | Active community + enterprise |
| Web dashboard | None (third-party only) | Built-in (OpenSearch/Kibana) |
| REST API | None | Full REST API |
| MITRE ATT&CK mapping | None | Native mapping |
| Threat Intelligence | None | VirusTotal, MISP integration |
| Vulnerability detection | Limited | Full CVE scanning |
| Cloud monitoring | None | AWS, Azure, GCP |
| Compliance reporting | Manual | PCI-DSS, HIPAA, GDPR, NIST |
| Agent management | Basic | Centralized, scalable |
| Custom decoders/rules |Complex XML | Improved XML + testing tools |
| Docker/container support | None | Native |
| Scalability | Poor | Clustered architecture |
| File integrity monitoring | Basic | Advanced with whodata |
| Log analysis | Basic | Advanced with ML |
| Incident response | Limited | Active response + SOAR |

---

## Why OSSEC Cannot Fulfill Our Needs

### 1. No MITRE ATT&CK Integration
Cortado RTAs (Red Team Assessments) are mapped to specific MITRE ATT&CK techniques and tactics. OSSEC has **zero native MITRE ATT&CK mapping**, meaning:

- Alerts cannot be correlated to specific attack techniques
- There is no way to identify which ATT&CK tactic is being used
- Threat hunting across ATT&CK frameworks is impossible
- Detection gaps cannot be measured against the ATT&CK matrix

Wazuh natively tags every alert with its corresponding MITRE ATT&CK technique ID (e.g., T1059, T1078), enabling direct correlation with Cortado RTA results.


### 2. Limited Rule Customization
OSSEC's rule engine is rigid and difficult to extend. Cortado RTAs simulate specific techniques that require:

- Custom detection rules per technique
- Chained rule logic (parent-child process detection)
- Threshold-based anomaly detection

OSSEC's rule system cannot reliably support these without breaking existing functionality.

### 3. No Active Response Capabilities at Scale
When an RTA simulates an attack, you need to:

- Automatically block the simulated attacker
- Isolate the affected host
- Trigger a SOAR playbook

OSSEC's active response is primitive and not suitable for automated RTA response workflows.

### 4. No Vulnerability Context
RTAs often exploit known CVEs. OSSEC cannot:

- Correlate alerts with CVE databases
- Identify if the attacked host is actually vulnerable
- Prioritize alerts based on vulnerability severity

### 5. Community and Support
OSSEC's last major release was years ago. It lacks:

- Modern OS support (Ubuntu 22.04, 24.04, etc.)
- Updated detection rules for recent attack techniques
- Community-driven RTA-specific rule sets

---

## Wazuh Installation

### Prerequisites

- Ubuntu 20.04 / 22.04 / 24.04 LTS (or equivalent)
- Minimum 4 CPU cores, 8GB RAM, 50GB disk
- Root or sudo access
- Internet connectivity

> **Note:** This guide covers the all-in-one single-node installation. For multi-node clusters, refer to the official Wazuh documentation at https://documentation.wazuh.com

---

### Step 1 - Download the Wazuh Installation Assistant

```bash
curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.12/config.yml
```

### Step 2 - Configure the Installation

Edit config.yml to define your node names and IPs:

```bash
nano config.yml
```

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "<YOUR_SERVER_IP>"

  server:
    - name: wazuh-1
      ip: "<YOUR_SERVER_IP>"

  dashboard:
    - name: dashboard
      ip: "<YOUR_SERVER_IP>"
```

> Replace `<YOUR_SERVER_IP>` with your actual server IP or `127.0.0.1` for localhost.

### Step 3 - Generate Configuration Files

```bash
sudo bash wazuh-install.sh --generate-config-files
```

### Step 4 - Install Wazuh Indexer

```bash
sudo bash wazuh-install.sh --wazuh-indexer node-1
```

### Step 5 - Initialize the Indexer Cluster

```bash
sudo bash wazuh-install.sh --start-cluster
```

### Step 6 - Install Wazuh Server

```bash
sudo bash wazuh-install.sh --wazuh-server wazuh-1
```

### Step 7 - Install Wazuh Dashboard

```bash
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

### Step 8 - Retrieve Admin Password

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

> Save these credentials securely!

### Step 9 - Access the Dashboard

Open your browser and navigate to:

```
https://<YOUR_SERVER_IP>
```

Login with:
- **Username:** `admin`
- **Password:** (from Step 8)

### Step 10 - Verify Services Are Running

```bash
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```

---

## Auditd Installation &amp; Configuration

Linux Audit (auditd) provides deep kernel-level visibility into system calls, file access, process execution, and user activity. Wazuh natively integrates with auditd for enhanced detection.

### Step 1 - Install Auditd

```bash
sudo apt update
sudo apt install auditd audispd-plugins -y
```

### Step 2 - Enable and Start Auditd

```bash
sudo systemctl enable auditd
sudo systemctl start auditd
sudo systemctl status auditd
```

### Step 3 - Configure Audit Rules

```bash
sudo nano /etc/audit/rules.d/audit.rules
```

Add the following rules:

```bash
# ============================================================
# Wazuh Audit Rules
# Covers: RTA detection, MITRE ATT&amp;CK techniques,
#         file integrity, privilege escalation, persistence
# ============================================================

# Delete all existing rules
-D

# Set buffer size (increase if losing events)
-b 8192

# Failure mode: 1=log, 2=panic
-f 1

# -----------------------------------------------
# IDENTITY &amp; AUTHENTICATION
# -----------------------------------------------
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers

# -----------------------------------------------
# PRIVILEGE ESCALATION (T1548)
# -----------------------------------------------
-a always,exit -F arch=b64 -S setuid -S setgid -F a0=0 -F exe=/usr/bin/su -k privilege_escalation
-a always,exit -F arch=b64 -S setresuid -k privilege_escalation
-a always,exit -F arch=b64 -S setresgid -k privilege_escalation
-w /usr/bin/sudo -p x -k privilege_escalation
-w /usr/bin/su -p x -k privilege_escalation
-w /bin/su -p x -k privilege_escalation

# -----------------------------------------------
# PROCESS EXECUTION (T1059)
# -----------------------------------------------
-a always,exit -F arch=b64 -S execve -k process_execution
-a always,exit -F arch=b32 -S execve -k process_execution

# -----------------------------------------------
# NETWORK ACTIVITY
# -----------------------------------------------
-a always,exit -F arch=b64 -S socket -S connect -S accept -k network_activity
-a always,exit -F arch=b32 -S socket -S connect -S accept -k network_activity

# -----------------------------------------------
# PERSISTENCE (T1053, T1543)
# -----------------------------------------------
-w /etc/cron.d/ -p wa -k persistence
-w /etc/cron.daily/ -p wa -k persistence
-w /etc/cron.hourly/ -p wa -k persistence
-w /etc/cron.monthly/ -p wa -k persistence
-w /etc/cron.weekly/ -p wa -k persistence
-w /etc/crontab -p wa -k persistence
-w /var/spool/cron/ -p wa -k persistence
-w /etc/systemd/system/ -p wa -k persistence
-w /usr/lib/systemd/system/ -p wa -k persistence
-w /etc/init.d/ -p wa -k persistence
-w /etc/rc.local -p wa -k persistence

# -----------------------------------------------
# FILE INTEGRITY — CRITICAL BINARIES (T1574)
# -----------------------------------------------
-w /usr/bin/ -p wa -k binary_modification
-w /usr/sbin/ -p wa -k binary_modification
-w /bin/ -p wa -k binary_modification
-w /sbin/ -p wa -k binary_modification

# -----------------------------------------------
# KERNEL MODULES (T1547.006)
# -----------------------------------------------
-w /sbin/insmod -p x -k kernel_modules
-w /sbin/rmmod -p x -k kernel_modules
-w /sbin/modprobe -p x -k kernel_modules
-a always,exit -F arch=b64 -S init_module -S delete_module -k kernel_modules

# -----------------------------------------------
# SSH &amp; REMOTE ACCESS (T1021)
# -----------------------------------------------
-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /root/.ssh/ -p wa -k ssh_keys
-w /home/ -p wa -k home_ssh

# -----------------------------------------------
# SUSPICIOUS TOOLS &amp; RECON (T1082, T1057)
# -----------------------------------------------
-w /usr/bin/wget -p x -k download_tools
-w /usr/bin/curl -p x -k download_tools
-w /usr/bin/nc -p x -k netcat
-w /usr/bin/ncat -p x -k netcat
-w /usr/bin/netcat -p x -k netcat
-w /usr/bin/nmap -p x -k port_scan
-w /usr/bin/tcpdump -p x -k packet_capture
-w /usr/bin/wireshark -p x -k packet_capture

# -----------------------------------------------
# DEFENSE EVASION (T1562)
# -----------------------------------------------
-w /usr/bin/shred -p x -k defense_evasion
-w /usr/bin/unlink -p x -k defense_evasion
-w /usr/bin/dd -p x -k defense_evasion
-w /usr/bin/truncate -p x -k defense_evasion

# -----------------------------------------------
# MAKE RULES IMMUTABLE (comment out during testing)
# -----------------------------------------------
# -e 2
```

### Step 4 - Apply the Rules

```bash
sudo augenrules --load
sudo systemctl restart auditd
```

### Step 5 - Verify Rules Loaded

```bash
sudo auditctl -l
```

### Step 6 - Configure Wazuh to Read Audit Logs

Edit the Wazuh agent configuration:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add inside the `<ossec_config>` block:

```xml
<!-- Audit log ingestion -->
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

### Step 7 - Enable Audit Whodata in Wazuh FIM

For file integrity monitoring with full user context:

```xml
<!-- File Integrity Monitoring with whodata -->
<syscheck>
  <disabled>no</disabled>
  <frequency>300</frequency>

  <!-- Monitor critical directories with whodata -->
  <directories check_all="yes" whodata="yes">/etc</directories>
  <directories check_all="yes" whodata="yes">/usr/bin</directories>
  <directories check_all="yes" whodata="yes">/usr/sbin</directories>
  <directories check_all="yes" whodata="yes">/bin</directories>
  <directories check_all="yes" whodata="yes">/sbin</directories>
  <directories check_all="yes" whodata="yes">/root</directories>

  <!-- Ignore noisy paths -->
  <ignore>/etc/mtab</ignore>
  <ignore>/etc/hosts.deny</ignore>
  <ignore>/etc/mail/statistics</ignore>
  <ignore>/etc/random-seed</ignore>
  <ignore>/etc/adjtime</ignore>
  <ignore>/etc/resolv.conf</ignore>
</syscheck>
```

### Step 8 - Restart Wazuh Agent

```bash
sudo systemctl restart wazuh-agent
```

### Step 9 - Verify Audit Events in Wazuh Dashboard

1. Open Wazuh Dashboard
2. Navigate to **Security Events**
3. Filter by `rule.groups: audit`
4. Verify events are flowing in

---

## Auditd Noise - Raw vs Fine-Tuned

### Default Auditd Configuration

While a default auditd installation captures a significant amount of system call data, it proved challenging to operationalize for threat detection. It logs nearly every audited system event without fine-grained filtering or categorization, leading to an extremely high volume of background noise. A basic configuration monitoring authentication and file changes might generate **100–500 MB per day** on a typical server, of which approximately **80–95% is typically irrelevant**. Crucially, default auditd logs lack descriptive rule names, making it difficult to quickly map an event to a specific threat technique without deep auditd knowledge and manual analysis.

### Neo23x0 Auditd Configuration

For the same level of system activity, the Neo23x0 ruleset might generate **20–100 MB per day**, of which **60–80% is highly relevant and actionable**. This configuration significantly improved upon the default by introducing targeted rules designed to filter out known-benign noise and highlight suspicious behaviors. It effectively reduced the sheer volume of irrelevant logs while tagging critical events with specific rule descriptions. This allowed for more immediate identification of RTA behaviors and their associated MITRE ATT&amp;CK techniques, making the logs far more actionable for security analysts. However, even with this hardened configuration, a degree of tuning specific to your environment is often necessary to further optimize the signal-to-noise ratio, as some legitimate applications may trigger rules designed to detect malicious activity.

#### Configuring Neo23x0 Auditd Rules with Wazuh

**Step 1 - Download the Neo23x0 Ruleset**

```bash
sudo curl -o /etc/audit/rules.d/audit.rules \
  https://raw.githubusercontent.com/Neo23x0/auditd/master/audit.rules
```


**Step 2 - Review and Optionally Customise the Rules**

```bash
sudo nano /etc/audit/rules.d/audit.rules
```

Key sections to review and tune for your environment:

| Section | What to Check |
|---|---|
| Exclusions by UID | Ensure trusted service UIDs are excluded |
| `execve` rules | Confirm scope matches your workload |
| Network socket rules | Tune if generating excessive noise |
| Immutable flag `-e 2` | Comment out during initial testing |

**Step 3 - Apply the Rules**

```bash
sudo augenrules --load
sudo systemctl restart auditd
```

**Step 4 - Verify Rules Loaded Successfully**

```bash
sudo auditctl -l | head -50
```

**Step 5 - Confirm Wazuh Agent is Reading Audit Logs**

Ensure the following block exists in `/var/ossec/etc/ossec.conf` on the agent:

```xml
<!-- Audit log ingestion -->
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

**Step 6 - Restart the Wazuh Agent**

```bash
sudo systemctl restart wazuh-agent
```

**Step 7 - Monitor Initial Event Volume**

After applying Neo23x0 rules, monitor the incoming event rate for the first 30 minutes to catch any unexpected noise spikes:

```bash
# Watch live audit events
sudo tail -f /var/log/audit/audit.log

# Count events per minute
sudo aureport --summary -i --start today
```

**Step 8 - Validate in Wazuh Dashboard**

1. Navigate to **Security Events**
2. Filter by `rule.groups: audit`
3. Check that rule keys from Neo23x0 (e.g., `recon`, `network_connection`, `persistence`) are appearing as alert tags
4. Verify MITRE ATT&amp;CK technique mappings are populating correctly


## Additional Features

### 1. VirusTotal Integration (Malware Detection)

Automatically scan suspicious files against VirusTotal:

```xml
<!-- Add to ossec.conf on the Wazuh Manager -->
<integration>
  <name>virustotal</name>
  <api_key><YOUR_VIRUSTOTAL_API_KEY></api_key>
  <rule_id>550, 554, 594</rule_id>
  <alert_format>json</alert_format>
</integration>
```

```bash
sudo systemctl restart wazuh-manager
```

---

### 2. Vulnerability Detector

Enable CVE scanning on all agents:

```xml
<!-- Add to ossec.conf on Wazuh Manager -->
<vulnerability-detector>
  <enabled>yes</enabled>
  <interval>12h</interval>
  <min_full_scan_interval>6h</min_full_scan_interval>
  <run_on_start>yes</run_on_start>

  <provider name="canonical">
    <enabled>yes</enabled>
    <os>focal</os>
    <os>jammy</os>
    <os>noble</os>
    <update_interval>1h</update_interval>
  </provider>

  <provider name="nvd">
    <enabled>yes</enabled>
    <update_from_year>2010</update_from_year>
    <update_interval>1h</update_interval>
  </provider>
</vulnerability-detector>
```

---

### 3. Active Response - Auto-Block Attackers

Automatically block IPs that trigger brute force rules:

```xml
<!-- Add to ossec.conf -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>600</timeout>
</active-response>
```

> Rule 5763 = SSH brute force detected. The attacker IP is automatically blocked for 600 seconds.

---

### 4. MITRE ATT&CK Dashboard

Wazuh natively maps alerts to MITRE ATT&CK:

1. Navigate to **Wazuh Dashboard → MITRE ATT&CK**
2. View heatmap of detected techniques
3. Cross-reference with Cortado RTA results
4. Identify detection gaps per tactic

---

### 5. Custom Rules for Cortado RTAs

Create custom detection rules:

```bash
sudo nano /var/ossec/etc/rules/cortado_rta.xml
```

```xml
<group name="cortado,rta,attack">

  <!-- Detect credential dumping via /proc -->
  <rule id="100001" level="12">
    <if_group>audit</if_group>
    <match>/proc/*/mem</match>
    <description>Possible credential dumping via /proc filesystem (T1003)</description>
    <mitre>
      <id>T1003</id>
    </mitre>
    <group>credential_access,</group>
  </rule>

  <!-- Detect suspicious base64 encoded commands -->
  <rule id="100002" level="10">
    <if_group>syslog</if_group>
    <regex>base64 -d|echo.*|.*bash</regex>
    <description>Possible encoded command execution (T1027)</description>
    <mitre>
      <id>T1027</id>
    </mitre>
    <group>defense_evasion,</group>
  </rule>

  <!-- Detect reverse shell patterns -->
  <rule id="100003" level="14">
    <if_group>audit</if_group>
    <match>bash -i|/dev/tcp|/dev/udp</match>
    <description>Possible reverse shell detected (T1059.004)</description>
    <mitre>
      <id>T1059.004</id>
    </mitre>
    <group>execution,</group>
  </rule>

</group>
```

```bash
# Test rules before applying
sudo /var/ossec/bin/wazuh-logtest

# Restart manager to apply
sudo systemctl restart wazuh-manager
```

---

### 6. Email Alerting

```xml
<!-- Add to ossec.conf on Wazuh Manager -->
<global>
  <email_notification>yes</email_notification>
  <smtp_server>smtp.yourdomain.com</smtp_server>
  <email_from>wazuh@yourdomain.com</email_from>
  <email_to>security-team@yourdomain.com</email_to>
  <email_maxperhour>12</email_maxperhour>
  <email_alert_level>10</email_alert_level>
</global>
```

---

### 7. Log Retention Policy

Prevent disk exhaustion by setting index retention:

```bash
# Delete indices older than 90 days (run as cron job)
curl -k -u admin:admin -X DELETE \
  "https://localhost:9200/wazuh-alerts-4.x-$(date -d '90 days ago' +%Y.%m.%d)"
```

```bash
# Add to crontab for automatic cleanup
sudo crontab -e
```

```
# Run daily at 2am — delete 90-day old Wazuh indices
0 2 * * * curl -sk -u admin:admin -X DELETE \
  "https://localhost:9200/wazuh-alerts-4.x-$(date -d '90 days ago' +\%Y.\%m.\%d)" \
  >> /var/log/wazuh-index-cleanup.log 2>&1
```

---

### 8. Security Configuration Assessment (SCA)

Wazuh automatically audits system hardening against CIS benchmarks:

1. Navigate to **Wazuh Dashboard → Agents → Select Agent**
2. Click **Security Configuration Assessment**
3. View pass/fail checks against:
   - CIS Ubuntu Linux Benchmark
   - NIST 800-53
   - PCI-DSS requirements

To customize SCA policies:

```bash
ls /var/ossec/ruleset/sca/
# cis_ubuntu20-04.yml
# cis_ubuntu22-04.yml
# pci_dss.yml
# nist800_53.yml
```

---

## Quick Reference - Key File Locations

| File | Purpose |
|---|---|
| `/var/ossec/etc/ossec.conf` | Main Wazuh agent/manager config |
| `/var/ossec/etc/rules/` | Custom detection rules |
| `/var/ossec/etc/decoders/` | Custom log decoders |
| `/var/ossec/logs/alerts/alerts.json` | Raw alert output |
| `/var/ossec/logs/ossec.log` | Wazuh agent logs |
| `/etc/audit/rules.d/audit.rules` | Auditd rules |
| `/var/log/audit/audit.log` | Raw audit events |
| `/etc/filebeat/filebeat.yml` | Filebeat shipper config |

---

## Quick Reference - Key Commands

```bash
# Check Wazuh manager status
sudo systemctl status wazuh-manager

# Test custom rules
sudo /var/ossec/bin/wazuh-logtest

# Restart all Wazuh services
sudo systemctl restart wazuh-manager wazuh-indexer wazuh-dashboard

# Check connected agents
sudo /var/ossec/bin/agent_control -l

# Check cluster health
curl -k -u admin:admin https://localhost:9200/_cluster/health?pretty

# View live alerts
sudo tail -f /var/ossec/logs/alerts/alerts.json | python3 -m json.tool
```

---
