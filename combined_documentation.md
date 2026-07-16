
# Leveraging Native Telemetry and Wazuh for Advanced Threat Detection with Cortado RTAs

## Summary

This report presents a comprehensive evaluation of native operating system telemetry (Sysmon for Windows, auditd for Linux) as a standalone alternative to third-party endpoint detection agents for identifying advanced cyber threats. Utilizing the Cortado Red Team Automation (RTA) framework developed by Elastic, we simulated a diverse range of adversary behaviors across controlled Windows and Linux environments.

Our findings indicate that well-tuned native telemetry deployments, specifically Sysmon with the SwiftOnSecurity ruleset on Windows and auditd with the Neo23x0 ruleset on Linux, achieve strong detection rates across sampled MITRE ATT&CK techniques. These configurations provide high-fidelity visibility into process lineage, file system modifications, and network activity. We also introduce Wazuh as a robust, open-source Extended Detection and Response (XDR) and Security Information and Event Management (SIEM) platform.

## 1. Motivation

The primary goal of this project was to determine if robust threat detection is achievable without the exclusive reliance on third-party endpoint agents. While dedicated agents provide valuable visibility, they often introduce management overhead, resource constraints, and compatibility challenges. Our focus was on leveraging native Windows telemetry (Sysmon) and Linux telemetry (auditd) to identify malicious activity, aiming to provide powerful, integrated, and potentially more lightweight alternatives for security monitoring. The subsequent integration with Wazuh aimed to address the operationalization and advanced analytical needs identified during the native telemetry evaluations.

## 2. Methodology

### 2.1 Tool Selection: Cortado RTAs

To simulate malicious behavior, we utilized Cortado RTAs, an open-source suite developed by Elastic. Cortado is designed for testing and analyzing defensive telemetry through two primary methods:

*   **CodeRTA:** Implements behavior directly in Python code.
*   **HashRTA:** References binary samples by hash that exhibit behaviors for detection.

Each RTA contains metadata such as ID, name, supported platforms, associated security rules, and MITRE ATT&CK techniques.

### 2.2 RTAs Execution and Initial Assessment

All simulations were conducted within controlled sandbox environments to ensure safety and consistency.

*   **Execution:** We began by executing individual RTAs using `cortado-run-rta` to identify their operational status. This initial phase helped us pinpoint which RTAs executed successfully and which encountered issues.

### 2.3 Detection Validation and Enhanced Telemetry

To validate that the executed RTAs were triggering detection events, we initially utilized Elastic's Endpoint Agent (free version). However, our primary objective was to assess the detection capabilities of our core infrastructure without relying on a full endpoint agent. Therefore, our main focus shifted to analyzing logs using two configurations for each OS:

*   **Default Configuration:** To establish a baseline of what is detected out-of-the-box.
*   **Hardened Configuration:** To evaluate the enhanced visibility provided by robust, community-driven rulesets.

## 3. Windows Environment: Sysmon Evaluation

### 3.1 Testbed and Configuration

*   **Operating System:** Standard Windows 2011 (Windows 11) Virtual Machine.
*   **Sampled RTAs:** 50 RTAs from the total library of 258 Windows-based RTAs were sampled.
*   **Telemetry Sources:**
    *   Default Sysmon Installation
    *   Hardened Sysmon Configuration (based on the SwiftOnSecurity ruleset)

### 3.2 Results: Detection Performance Summary

A well-tuned Sysmon deployment achieved an 80% detection rate across sampled MITRE ATT&CK techniques, providing high-fidelity visibility into process lineage and unauthorized file system modifications.

#### 3.2.1 Statistical Summary of Findings

*   **Detection Rate:** 80% of the sampled RTAs produced high-fidelity logs suitable for immediate detection.
*   **Masquerading Detection:** In 100% of cases where an attacker renamed a binary (e.g., renaming a C2 agent to `winword.exe`), Sysmon successfully identified the deception via the `OriginalFileName` field.
*   **LotL Visibility:** The configuration was highly effective at flagging the abuse of legitimate Windows tools like `certutil.exe`, `nslookup.exe`, and `vssadmin.exe`.

#### 3.2.2 Discussion

The telemetry generated during the simulations provides a highly actionable framework for detection engineering. The following sections analyze the specific forensic evidence captured during the RTA executions and the inherent trade-offs in defensive configurations.

##### 3.2.2.1 Forensic Analysis of Execution and Persistence
In the **Sliver C2** and **Mimikatz** simulations, Sysmon captured the full process lineage, including `ParentCommandLine` and `Hashes`. This allowed us to trace malicious activity back to the source, identifying that the commands originated from a Python-based RTA engine. This level of detail is critical for distinguishing between legitimate administrative scripts and automated attack frameworks.

![Sysmon Event ID 1 - Mimikatz Simulation](<VirtualBox_Windows 2011_11_02_2026_09_58_23.png>)
*Figure 1: Sysmon Event ID 1 capturing the execution of a Mimikatz-related command via PowerShell, initiated by the Cortado RTA framework.*

Regarding persistence, the configuration successfully flagged the modification of sensitive registry keys during the **SolarMarker** simulation. Furthermore, it provided direct mapping to **T1547.012** (Print Processor Sideloading) by generating a `FileCreate` (Event ID 11) when an unauthorized `rta.dll` was dropped into the `\spool\drivers\` directory- a location rarely modified by standard users.

##### 3.2.2.2 Network Anomalies and C2 Discovery
The configuration exposed Command & Control (C2) channels by monitoring non-standard process behavior. This was evidenced by flagging automated reconnaissance, such as `powershell.exe` querying `api.ipify.org`. Because this activity originated from a command-line tool rather than a standard web browser, it serves as a high-fidelity indicator of automated discovery.

##### 3.2.2.3 Technical Gap: The `nslookup` DNS Logging "Miss"
A significant observation occurred during the **network_connection_nslookup** RTA. While Sysmon successfully recorded the **Process Creation (Event ID 1)** for `nslookup.exe`, it failed to generate a **DNS Query (Event ID 22)**.

* **The Reason:** The SwiftOnSecurity configuration is optimized for "High Fidelity" to reduce log volume. Because `nslookup` is a standard tool used frequently by IT administrators for troubleshooting, it is often explicitly excluded from DNS Query logging to prevent "noise."
* **The Risk:** This exclusion creates a blind spot. An attacker can leverage `nslookup` for **DNS Tunneling** or data exfiltration, remaining invisible to DNS-specific monitoring rules because the tool is "trusted" by the configuration. This highlights the need for behavioral monitoring (analyzing *why* the tool is running) rather than relying solely on individual event triggers.

##### 3.2.2.4 The "Public" Folder as a Universal Indicator
A recurring pattern across multiple successful detections (including `browser_debugging` and `uac_computerdefaults`) was the use of `C:\Users\Public\` for staging malicious binaries. Because this directory is globally writeable but rarely used by legitimate enterprise applications for execution, any process creation (**Event ID 1**) originating from `C:\Users\Public\` or `C:\Windows\Temp\` should be treated as a high-priority alert.

##### 3.2.2.5 Analysis of Detection Gaps
Certain RTAs, such as `port_monitor` and `crashdump_disabled`, did not produce actionable Sysmon logs. These techniques often involve direct memory manipulation or internal OS flag changes that do not trigger the specific kernel callbacks Sysmon monitors. To mitigate these gaps, supplemental logging would be required.

### 3.3 Comparative Analysis of Logging Sources 

*   **Native Windows Logging:** Standard event logs proved insufficient for forensic reconstruction. Most RTA actions either left no trace or were buried in generic "Success" audits that lacked the process command-line arguments and file hashes necessary to confirm malicious intent.
*   **Default Sysmon Configuration:** While the default installation captured a vast amount of data, it was difficult to operationalize. It logged nearly every system event without categorization, leading to a high volume of "background noise" from standard OS processes (200-300 MB per day on the system, with 75-85% irrelevant). Crucially, default logs lacked descriptive rule names, making it impossible to quickly map an event to a specific threat technique without manual analysis.
*   **SwiftOnSecurity Configuration:** This configuration resolved the previous issues by filtering out known-benign noise and tagging events with specific rule names, allowing for immediate identification of RTA behavior.

## 4. Linux Environment: auditd Evaluation

### 4.1 Testbed and Configuration

*   **Operating System:** Standard Ubuntu Server 24.04 LTS Virtual Machine.
*   **Sampled RTAs:** 20 RTAs out of the 62 Linux-based RTAs available within Cortado were sampled.
*   **Telemetry Sources:**
    *   Default auditd Installation
    *   Hardened auditd Configuration (based on the Neo23x0 ruleset)

### 4.2 Results: Detection Performance Summary

Our preliminary findings indicate that a well-tuned auditd deployment, especially with comprehensive community rules, achieves a strong detection rate across sampled MITRE ATT&CK techniques, providing high-fidelity visibility into process execution, file system modifications, and network activity.

#### 4.2.1 Statistical Summary of Findings

*   **Detection Rate:** 100% of the sampled RTAs produced high-fidelity logs suitable for immediate detection.

#### 4.2.2 Comparative Analysis of Logging Sources 

*   **Native Linux Logging (e.g., syslog, journalctl):** Standard system logs proved insufficient for forensic reconstruction of advanced threats. Most RTA actions either left minimal trace or generated generic entries that lacked the granular detail, process lineage, and specific event types necessary to confirm malicious intent or map directly to MITRE ATT&CK techniques.
*   **Default auditd Configuration:** While a default auditd installation captures a significant amount of system call data, it was challenging to operationalize for threat detection. It logs nearly every audited system event without fine-grained filtering or categorization, leading to an extremely high volume of "background noise" (100-500 MB per day on a typical server, with 80-95% irrelevant). Default auditd logs also lacked descriptive rule names.
*   **Neo23x0 auditd Configuration:** This configuration significantly improved upon the default by introducing targeted rules designed to filter out known-benign noise and highlight suspicious behaviors. It effectively reduced the sheer volume of irrelevant logs (20-100 MB per day, with 60-80% highly relevant) while tagging critical events with specific rule descriptions. This allowed for more immediate identification of RTA behaviors and their associated MITRE ATT&CK techniques, making the logs far more actionable. However, environmental-specific tuning is often necessary to further optimize the signal-to-noise ratio.

### 4.3 Discussion: Efficacy, Strengths, and Limitations 

#### 4.3.1 Efficacy of auditd with Neo23x0 Ruleset

Our evaluation demonstrated that a properly configured auditd, especially when leveraging the community-driven Neo23x0 ruleset, is effective in detecting a wide array of simulated adversary behaviors. The 100% detection rate across the sampled RTAs is a strong indicator of its potential as a robust security monitoring solution. The Neo23x0 configuration provided high-fidelity logs with crucial details such as process lineage, user IDs, file paths, and specific system calls, vital for understanding the full scope and context of an attack.

#### 4.3.2 Strengths of auditd as a Standalone Solution

*   **Native Integration and Deep Visibility:** As a kernel-level component, auditd offers unparalleled, deep visibility into system calls and kernel events, capturing low-level activities that might be missed by user-space agents.
*   **Resource Efficiency:** auditd itself is generally lightweight in terms of CPU and memory footprint, making it attractive for resource-constrained environments.
*   **Customizability:** The rule-based nature of auditd allows for extensive customization, enabling security teams to tailor detection logic to specific threat models and compliance requirements.

#### 4.3.3 Identified Gaps and Limitations 

*   **Persistent Background Noise:** Even with optimized rules, a degree of background noise persists, making it challenging to distinguish between legitimate highly privileged actions and malicious activities without further environmental-specific tuning.
*   **Log Volume and Storage:** auditd can generate a massive volume of logs, necessitating robust infrastructure for log aggregation, long-term storage, and efficient processing.
*   **Potential Blind Spots:** Certain highly sophisticated or memory-resident attacks might still evade detection if they do not trigger specific system calls covered by the ruleset. Continuous research and rule development are essential.

#### 4.3.4 Operational Considerations

*   **Leveraging AI for Enhanced Analysis:** The rich, granular data provided by auditd presents a significant opportunity for advanced analytics. When these logs are fed into a Large Language Model (LLM), the LLM can process extensive contextual information to identify subtle anomalies and correlate seemingly disparate events into coherent malicious activity, dramatically reducing manual effort for threat hunting and incident response.

## 5.  Wazuh for Unified XDR & SIEM
While native telemetry provides foundational visibility, operationalizing it for effective threat detection and response across diverse environments requires a robust platform. Wazuh is a free, open-source security platform that evolved from OSSEC. It is actively maintained, enterprise-ready, and significantly more capable than its predecessor.

### 5.1 Wazuh Features for Enhanced Security
Building upon our findings from the dedicated `auditd` evaluation, we integrated the fine-tuned `auditd` logs (configured with the Neo23x0 ruleset) into Wazuh. This combination significantly improved our detection capabilities. The Neo23x0 ruleset, as previously noted, dramatically reduced log volume and effectively filtered out benign system activity, making the remaining logs far more relevant and actionable.

When these optimized `auditd` logs were ingested by Wazuh, the platform was able to parse the descriptive rule names provided by the Neo23x0 configuration. This process enriched alerts and often directly mapped them to specific MITRE ATT&CK techniques. This integration streamlined the analysis process, enabling quicker identification and understanding of simulated RTA behaviors within the Wazuh dashboard.

Beyond log ingestion and correlation, Wazuh further extends the utility of this native telemetry by offering:
1.  **VirusTotal Integration (Malware Detection):** Automatically scan suspicious files.
2.  **Vulnerability Detector:** Enable CVE scanning on all agents.
3.  **Active Response - Auto-Block Attackers:** Automatically block IPs triggering brute force rules (e.g., rule 5763 for SSH brute force).
4.  **MITRE ATT&CK Dashboard:** Wazuh natively maps alerts to MITRE ATT&CK, allowing for heatmap visualization of detected techniques, cross-referencing with Cortado RTA results, and identification of detection gaps.
5.  **Custom Rules for Cortado RTAs:** Create specific rules to detect RTA behaviors.
6.  **Email Alerting:** Configure email notifications for critical alerts.
7.  **Log Retention Policy:** Implement a cron job to delete old indices and prevent disk exhaustion.
8.  **Security Configuration Assessment (SCA):** Wazuh automatically audits system hardening against benchmarks like CIS, NIST, and PCI-DSS.


## 6. Conclusion

The evaluation of native telemetry solutions, Sysmon for Windows and auditd for Linux, reveals their significant potential for advanced threat detection when meticulously configured with community-driven rulesets like SwiftOnSecurity and Neo23x0, respectively. Both demonstrated high-fidelity visibility into various adversary behaviors simulated by Cortado RTAs, offering a foundation for robust security monitoring without immediate reliance on third-party endpoint agents.

However, our findings also highlighted inherent limitations and operational challenges with standalone native logging. These include critical telemetry blind spots for certain sophisticated techniques, the persistent issue of high log volume and noise, and the difficulty in operationalizing raw logs for efficient threat hunting and incident response. In essence, while native OS telemetry provides invaluable deep-level visibility, augmenting it with a comprehensive platform such as Wazuh allows organizations to overcome the inherent operational hurdles, transform raw logs into actionable intelligence, and build a more mature, adaptable, and efficient threat detection and response capability against evolving cyber threats. This combined approach leverages the strengths of both native system insights and advanced security analytics platforms.

---

## 7. Appendix

## 7.1 Setting Up Your Cortado RTAS Environment

This section details the technical prerequisites and step-by-step installation process for Cortado RTAS. Official Cortado documentation uses Poetry to install Cortado; this documentation covers installation using `pip`.

### 7.1.1 Prerequisites

Before proceeding, ensure your test environment meets the following requirements:

*   **Software Dependencies:**
    *   Python `>=3.12` (ensure `pip` is updated).
*   **Access & Permissions:**
    *   Administrative rights for running RTAs.

### 7.1.2 Cortado RTAS Installation Guide

Follow these steps to install Cortado RTAs:

1.  **Install the Wheel File using `pip`:**
    Once you are in the correct directory, run the following command to install Cortado:

    ```bash
    pip install cortado-0.1.0-py3-none-any.whl
    ```

### 7.1.3 Operating RTAs & Analyzing Logs

The Cortado RTAs CLI provides commands to execute individual or all relevant RTAs in a sandboxed environment with the minimal dependencies.

### 7.1.4 Key Execution Commands

Cortado RTAS offers two primary commands for executing RTAs:

*   **`cortado-run-rta`**: Executes a *particular* RTA. This is useful for focused testing of a specific technique.
*   **`cortado-run-rtas`**: Executes *all* RTAs that match the current operating system. This is ideal for comprehensive testing across multiple techniques.



### 7.1.5 Executing a Specific RTA (`cortado-run-rta`)

To execute a single RTA, you'll typically need to specify the RTA's identifier.

1.  **List Available RTAs:**
    Before running, you might want to see which RTAs are available. 

    ```bash
    print-rtas
    ```

2.  **Execute the RTA:**
    Once you know the RTA you want to run, use the `cortado-run-rta` command followed by the RTA's identifier.

    ```bash
    cortado-run-rta rtaName
    ```


### 7.1.6 Executing All Relevant RTAs (`cortado-run-rtas`)

To perform a broad test across all RTAs compatible with your operating system, use the `cortado-run-rtas` command.

```bash
cortado-run-rtas 
```

**Important:** RTAS techniques are designed to simulate the *actions* of ransomware, not necessarily to fully execute a malicious payload. **It is expected and often desired that RTAS techniques may not run flawlessly or complete their full intended malicious action.**

*   **The Goal:** The primary objective is to trigger your detection rules by mimicking the *initial stages* or *attempted behaviors* of ransomware.
*   **Example:** A technique might attempt to encrypt files but fail due to permissions. However, the *attempted file access patterns* should still trigger a "Suspicious File Access" or "Ransomware Behavior" alert in Elastic Security. This validates your EDR's ability to detect early-stage TTPs.
*   For a detailed summary of the execution outcomes and functional status of each RTA, please refer to the `Windows RTA Summary.md` document.
  
#### 7.1.7 Potential Cortado Set Up Issues

1.  The Cortado wheel file does not contain some dependencies, so you will receive a similar error when you run `cortado --help`:

    ```bash
    PS C:\Users\vboxuser\Downloads> cortado --help
    Traceback (most recent call last):
    File "<frozen runpy>", line 198, in _run_module_as_main
    File "<frozen runpy>", line 88, in _run_code
    File "C:\Users\vboxuser\AppData\Local\Programs\Python\Python313\Scripts\cortado.exe\__main__.py", line 2, in <module>
    from cortado.cli import run_cli
    File "C:\Users\vboxuser\AppData\Local\Programs\Python\Python313\Lib\site-packages\cortado\cli.py", line 9, in <module>
    import structlog
    ModuleNotFoundError: No module named 'structlog'
    ```

These scripts are designed to run without any external dependencies beyond what's handled by the Cortado installation itself so the RTAs runs fine despite this message.

**Installing the Additional Dependencies**

1.  Manually install dependencies listed in `pyproject.toml` from the `[tool.poetry.dependencies]` section using `pip`:

    ```bash
    pip install -r requirements.txt
    ```
*It is important to verfiy that all dependencies are available and not conflicting.* 

### 7.2 RTA Functional Status Summary
[Windows RTA Summary Report](./windows_rta_summary.md)



### 7.3 Sysmon Configuration Details

To get started with Sysmon, you'll first need to download the necessary files:

- **Sysmon Executable:**
    - Go to the official Microsoft Sysinternals page: [https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
    - Download the latest version of Sysmon. It usually comes as a `.zip` file.
- **SwiftOnSecurity Sysmon Configuration (for advanced setup):**
    - Go to the SwiftOnSecurity GitHub repository: [https://github.com/SwiftOnSecurity/Sysmon-config](https://github.com/SwiftOnSecurity/Sysmon-config)
    - Download the `sysmonconfig-export.xml` file. 

#### 7.3.1 Default Sysmon Configuration

This method installs Sysmon with its most basic, default settings. 

1. **Extract Sysmon:**
    - Unzip the downloaded `Sysmon.zip` file to a convenient location, for example, `C:\Sysmon`.

2. **Navigate to the Sysmon Directory:**

    ```bash
    cd C:\Sysmon
    ```

3. **Install Sysmon with Default Configuration:**
    - For 64-bit systems (most common):

        ```bash
        Sysmon64.exe -i
        ```

    - You'll be prompted to agree to the EULA (End-User License Agreement). Type `yes` and press Enter.

#### 7.3.2 SwiftOnSecurity Sysmon Configuration

1. **Prepare Sysmon and Configuration File:**
    - Ensure you have extracted `Sysmon.zip` (e.g., to `C:\Sysmon`).
    - Place the downloaded `sysmonconfig-export.xml` file into the same directory as `Sysmon.exe` (e.g., `C:\Sysmon\sysmonconfig-export.xml`).

2. **Open an Elevated Command Prompt or PowerShell:**
    - Search for "cmd" or "PowerShell" in the Start Menu.
    - Right-click on it and select "Run as administrator."

3. **Navigate to the Sysmon Directory:**

    ```bash
    cd C:\Sysmon
    ```

4. **Install or Update Sysmon with SwiftOnSecurity Configuration:**

    - **First-time installation:**
        - For 64-bit systems:

            ```bash
            Sysmon64.exe -i sysmonconfig-export.xml
            ```


    - **Updating an existing installation:** If Sysmon is already running, you can update its configuration without reinstalling.
        - For 64-bit systems:

            ```bash
            Sysmon64.exe -c sysmonconfig-export.xml
            ```

### 7.4 Sysmon Log Analysis Summary 
[Sysmon Log Analysis Report](./sysmon_log_analysis.md)

### 7.5 Elastic Installation 
[Link to installation guidelines](https://www.elastic.co/docs/reference/fleet/install-elastic-agents)

## 7.6 OSSEC Installation

### Choose Installation Type
| Type | Description |
|---|---|
| `local` | Monitors only the local machine |
| `agent` | Sends logs to a central OSSEC server |
| `server` | Receives logs from multiple agents |
| `hybrid` | Acts as both server and agent |

---

### 7.6a Local Installation (Classic OSSEC)

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

### 7.6b Agent-Server Setup (Classic OSSEC) (Optional / Advanced)

Use this if you want **multiple machines** reporting to **one central OSSEC server**.

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

## 7.7 Wazuh Installation

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

## 7.8 Audit Installation & Configuration

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
# Covers: RTA detection, MITRE ATT&CK techniques,
#         file integrity, privilege escalation, persistence
# ============================================================

# Delete all existing rules
-D

# Set buffer size (increase if losing events)
-b 8192

# Failure mode: 1=log, 2=panic
-f 1

# -----------------------------------------------
# IDENTITY & AUTHENTICATION
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
# SSH & REMOTE ACCESS (T1021)
# -----------------------------------------------
-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /root/.ssh/ -p wa -k ssh_keys
-w /home/ -p wa -k home_ssh

# -----------------------------------------------
# SUSPICIOUS TOOLS & RECON (T1082, T1057)
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
