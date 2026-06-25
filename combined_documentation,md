
# Consolidated Report: Leveraging Native Telemetry and Wazuh for Advanced Threat Detection with Cortado RTAs

## Executive Summary

This report presents a comprehensive evaluation of native operating system telemetry (Sysmon for Windows, auditd for Linux) as a standalone alternative to third-party endpoint detection agents for identifying advanced cyber threats. Utilizing the Cortado Red Team Automation (RTA) framework developed by Elastic, we simulated a diverse range of adversary behaviors across controlled Windows and Linux environments.

Our findings indicate that well-tuned native telemetry deployments, specifically Sysmon with the SwiftOnSecurity ruleset on Windows and auditd with the Neo23x0 ruleset on Linux, achieve strong detection rates across sampled MITRE ATT&CK techniques. These configurations provide high-fidelity visibility into process lineage, file system modifications, and network activity.

However, the study also identifies critical telemetry gaps and operational challenges inherent in relying solely on native logging. To address these, we introduce **Wazuh** as a robust, open-source Extended Detection and Response (XDR) and Security Information and Event Management (SIEM) platform. Wazuh significantly enhances the utility of native telemetry by providing centralized log aggregation, advanced correlation, MITRE ATT&CK mapping, vulnerability detection, active response capabilities, and a user-friendly dashboard.

This report concludes that while strategically configured native telemetry significantly reduces the need for external agents, integrating it with a powerful platform like Wazuh is essential to overcome blind spots, operationalize detection, and effectively respond to sophisticated, low-profile attacks across diverse IT landscapes.

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

#### 3.2.2 Comparative Analysis of Logging Sources (Windows)

*   **Native Windows Logging:** Standard event logs proved insufficient for forensic reconstruction. Most RTA actions either left no trace or were buried in generic "Success" audits that lacked the process command-line arguments and file hashes necessary to confirm malicious intent.
*   **Default Sysmon Configuration:** While the default installation captured a vast amount of data, it was difficult to operationalize. It logged nearly every system event without categorization, leading to a high volume of "background noise" from standard OS processes. Crucially, default logs lacked descriptive rule names, making it impossible to quickly map an event to a specific threat technique without manual analysis.
*   **SwiftOnSecurity Configuration:** This configuration resolved the previous issues by filtering out known-benign noise and tagging events with specific rule names, allowing for immediate identification of RTA behavior.

### 3.3 Discussion: Forensic Insights and Gaps (Windows)

The telemetry generated during the simulations provided a highly actionable framework for detection engineering.

#### 3.3.1 Forensic Analysis of Execution and Persistence

*   **Process Lineage:** In Sliver C2 and Mimikatz simulations, Sysmon captured the full process lineage, including `ParentCommandLine` and `Hashes`, allowing tracing malicious activity back to the Python-based RTA engine.
*   **Persistence:** The configuration successfully flagged the modification of sensitive registry keys during the SolarMarker simulation and provided direct mapping to T1547.012 (Print Processor Sideloading) by generating a `FileCreate` (Event ID 11) when an unauthorized `rta.dll` was dropped into the `\spool\drivers\` directory.

#### 3.3.2 Network Anomalies and C2 Discovery

The configuration exposed Command & Control (C2) channels by monitoring non-standard process behavior, evidenced by flagging automated reconnaissance (e.g., `powershell.exe` querying `api.ipify.org`) originating from a command-line tool rather than a standard web browser.

#### 3.3.3 Technical Gap: The `nslookup` DNS Logging "Miss"

A significant observation occurred during the `network_connection_nslookup` RTA. While Sysmon successfully recorded the `Process Creation` (Event ID 1) for `nslookup.exe`, it failed to generate a `DNS Query` (Event ID 22).

*   **Reason:** The SwiftOnSecurity configuration is optimized for "High Fidelity" to reduce log volume. `nslookup` is often explicitly excluded from DNS Query logging to prevent "noise" due to its frequent use by IT administrators.
*   **Risk:** This exclusion creates a blind spot. An attacker can leverage `nslookup` for DNS Tunneling or data exfiltration, remaining invisible to DNS-specific monitoring rules. This highlights the need for behavioral monitoring (analyzing *why* the tool is running) rather than relying solely on individual event triggers.

#### 3.3.4 The "Public" Folder as a Universal Indicator

A recurring pattern across multiple successful detections (including `browser_debugging` and `uac_computerdefaults`) was the use of `C:\Users\Public\` for staging malicious binaries. Because this directory is globally writeable but rarely used by legitimate enterprise applications for execution, any `Process Creation` (Event ID 1) originating from `C:\Users\Public\` or `C:\Windows\Temp\` should be treated as a high-priority alert.

#### 3.3.5 Analysis of Detection Gaps (Windows)

Certain RTAs, such as `port_monitor` and `crashdump_disabled`, did not produce actionable Sysmon logs. These techniques often involve direct memory manipulation or internal OS flag changes that do not trigger the specific kernel callbacks Sysmon monitors. To mitigate these gaps, supplemental logging would be required.

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

#### 4.2.2 Comparative Analysis of Logging Sources (Linux)

*   **Native Linux Logging (e.g., syslog, journalctl):** Standard system logs proved insufficient for forensic reconstruction of advanced threats. Most RTA actions either left minimal trace or generated generic entries that lacked the granular detail, process lineage, and specific event types necessary to confirm malicious intent or map directly to MITRE ATT&CK techniques.
*   **Default auditd Configuration:** While a default auditd installation captures a significant amount of system call data, it was challenging to operationalize for threat detection. It logs nearly every audited system event without fine-grained filtering or categorization, leading to an extremely high volume of "background noise" (100-500 MB per day on a typical server, with 80-95% irrelevant). Default auditd logs also lacked descriptive rule names.
*   **Neo23x0 auditd Configuration:** This configuration significantly improved upon the default by introducing targeted rules designed to filter out known-benign noise and highlight suspicious behaviors. It effectively reduced the sheer volume of irrelevant logs (20-100 MB per day, with 60-80% highly relevant) while tagging critical events with specific rule descriptions. This allowed for more immediate identification of RTA behaviors and their associated MITRE ATT&CK techniques, making the logs far more actionable. However, environmental-specific tuning is often necessary to further optimize the signal-to-noise ratio.

### 4.3 Discussion: Efficacy, Strengths, and Limitations (Linux)

#### 4.3.1 Efficacy of auditd with Neo23x0 Ruleset

Our evaluation demonstrated that a properly configured auditd, especially when leveraging the community-driven Neo23x0 ruleset, is effective in detecting a wide array of simulated adversary behaviors. The 100% detection rate across the sampled RTAs is a strong indicator of its potential as a robust security monitoring solution. The Neo23x0 configuration provided high-fidelity logs with crucial details such as process lineage, user IDs, file paths, and specific system calls, vital for understanding the full scope and context of an attack.

#### 4.3.2 Strengths of auditd as a Standalone Solution

*   **Native Integration and Deep Visibility:** As a kernel-level component, auditd offers unparalleled, deep visibility into system calls and kernel events, capturing low-level activities that might be missed by user-space agents.
*   **Resource Efficiency:** auditd itself is generally lightweight in terms of CPU and memory footprint, making it attractive for resource-constrained environments.
*   **Customizability:** The rule-based nature of auditd allows for extensive customization, enabling security teams to tailor detection logic to specific threat models and compliance requirements.

#### 4.3.3 Identified Gaps and Limitations (Linux)

*   **Persistent Background Noise:** Even with optimized rules, a degree of background noise persists, making it challenging to distinguish between legitimate highly privileged actions and malicious activities without further environmental-specific tuning.
*   **Log Volume and Storage:** auditd can generate a massive volume of logs, necessitating robust infrastructure for log aggregation, long-term storage, and efficient processing.
*   **Potential Blind Spots:** Certain highly sophisticated or memory-resident attacks might still evade detection if they do not trigger specific system calls covered by the ruleset. Continuous research and rule development are essential.

#### 4.3.4 Operational Considerations

*   **Leveraging AI for Enhanced Analysis:** The rich, granular data provided by auditd presents a significant opportunity for advanced analytics. When these logs are fed into a Large Language Model (LLM), the LLM can process extensive contextual information to identify subtle anomalies and correlate seemingly disparate events into coherent malicious activity, dramatically reducing manual effort for threat hunting and incident response.

## 5. Overarching Solution: Wazuh for Unified XDR & SIEM

While native telemetry provides foundational visibility, operationalizing it for effective threat detection and response across diverse environments requires a robust platform. **Wazuh** emerges as an ideal solution, addressing the limitations of standalone native logging by offering unified XDR and SIEM capabilities.

### 5.1 Why Wazuh Over OSSEC

Wazuh is a free, open-source security platform that evolved from OSSEC. It provides unified XDR and SIEM capabilities, is actively maintained, enterprise-ready, and significantly more capable than its predecessor, OSSEC.

| Feature | OSSEC | Wazuh |
| :--------------------------- | :-------------------- | :-------------------------------------- |
| Active maintenance | Minimal | Active community + enterprise |
| Web dashboard | None (third-party) | Built-in (OpenSearch/Kibana) |
| REST API | None | Full REST API |
| MITRE ATT&CK mapping | None | Native mapping |
| Threat Intelligence | None | VirusTotal, MISP integration |
| Vulnerability detection | Limited | Full CVE scanning |
| Cloud monitoring | None | AWS, Azure, GCP |
| Compliance reporting | Manual | PCI-DSS, HIPAA, GDPR, NIST |
| Agent management | Basic | Centralized, scalable |
| Custom decoders/rules | Complex XML | Improved XML + testing tools |
| Docker/container support | None | Native |
| Scalability | Poor | Clustered architecture |
| File integrity monitoring | Basic | Advanced with whodata |
| Log analysis | Basic | Advanced with ML |
| Incident response | Limited | Active response + SOAR |

#### 5.1.1 Why OSSEC Cannot Fulfill Our Needs (for RTA-driven detection)

1.  **No MITRE ATT&CK Integration:** Cortado RTAs are mapped to specific MITRE ATT&CK techniques. OSSEC lacks native mapping, preventing alert correlation to techniques, identification of tactics, threat hunting, and measurement of detection gaps. Wazuh natively tags every alert with its corresponding MITRE ATT&CK technique ID, enabling direct correlation with Cortado RTA results.
2.  **Limited Rule Customization:** OSSEC's rigid rule engine struggles with the custom detection rules, chained logic, and threshold-based anomaly detection required for Cortado RTA simulations.
3.  **No Active Response Capabilities at Scale:** OSSEC's primitive active response is unsuitable for automated RTA response workflows (e.g., blocking attackers, isolating hosts, triggering SOAR playbooks).
4.  **No Vulnerability Context:** OSSEC cannot correlate alerts with CVE databases, identify host vulnerability, or prioritize alerts based on severity, which is crucial as RTAs often exploit known CVEs.
5.  **Community and Support:** OSSEC's lack of recent updates means it lacks modern OS support, updated detection rules for recent attack techniques, and community-driven RTA-specific rule sets.

### 5.2 Wazuh Installation & Configuration

This section outlines the process for setting up a single-node Wazuh environment and configuring it to ingest auditd logs from Linux systems.

#### 5.2.1 Prerequisites

*   **OS:** Ubuntu 20.04 / 22.04 / 24.04 LTS (or equivalent)
*   **Resources:** Minimum 4 CPU cores, 8GB RAM, 50GB disk
*   **Access:** Root or sudo access
*   **Connectivity:** Internet connectivity

#### 5.2.2 Wazuh Server Installation Steps

1.  **Download Installation Assistant:**
    ```bash
    curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh
    curl -sO https://packages.wazuh.com/4.12/config.yml
    ```
2.  **Configure Installation (`config.yml`):**
    Edit `config.yml` to define node names and IPs. Replace `<YOUR_SERVER_IP>` with your actual server IP or `127.0.0.1` for localhost.
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
3.  **Generate Configuration Files:**
    ```bash
    sudo bash wazuh-install.sh --generate-config-files
    ```
4.  **Install Wazuh Indexer:**
    ```bash
    sudo bash wazuh-install.sh --wazuh-indexer node-1
    ```
5.  **Initialize Indexer Cluster:**
    ```bash
    sudo bash wazuh-install.sh --start-cluster
    ```
6.  **Install Wazuh Server:**
    ```bash
    sudo bash wazuh-install.sh --wazuh-server wazuh-1
    ```
7.  **Install Wazuh Dashboard:**
    ```bash
    sudo bash wazuh-install.sh --wazuh-dashboard dashboard
    ```
8.  **Retrieve Admin Password:**
    ```bash
    sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
    ```
    *Save these credentials securely!*
9.  **Access Dashboard:** Open your browser to `https://<YOUR_SERVER_IP>` and log in with username `admin` and the retrieved password.
10. **Verify Services:**
    ```bash
    sudo systemctl status wazuh-indexer
    sudo systemctl status wazuh-manager
    sudo systemctl status wazuh-dashboard
    ```

### 5.3 Linux Agent Installation & Auditd Configuration for Wazuh

Linux Audit (auditd) provides deep kernel-level visibility. Wazuh natively integrates with auditd for enhanced detection.

#### 5.3.1 Install and Enable Auditd

1.  **Install Auditd:**
    ```bash
    sudo apt update
    sudo apt install auditd audispd-plugins -y
    ```
2.  **Enable and Start Auditd:**
    ```bash
    sudo systemctl enable auditd
    sudo systemctl start auditd
    sudo systemctl status auditd # Verify it's running
    ```

#### 5.3.2 Configure Audit Rules (Wazuh-Optimized)

1.  **Edit Audit Rules:**
    ```bash
    sudo nano /etc/audit/rules.d/audit.rules
    ```
2.  **Add Wazuh-Specific Rules:**
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
3.  **Apply Rules:**
    ```bash
    sudo augenrules --load
    sudo systemctl restart auditd
    ```
4.  **Verify Rules:**
    ```bash
    sudo auditctl -l
    ```
5.  **Configure Wazuh Agent to Read Audit Logs:**
    Edit the Wazuh agent configuration (`sudo nano /var/ossec/etc/ossec.conf`) and add the following inside the `<ossec_config>` block:
    ```xml
    <!-- Audit log ingestion -->
    <localfile>
      <log_format>audit</log_format>
      <location>/var/log/audit/audit.log</location>
    </localfile>
    ```
6.  **Enable Audit Whodata in Wazuh FIM:**
    Still in `/var/ossec/etc/ossec.conf`, add the following for file integrity monitoring with full user context:
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
7.  **Restart Wazuh Agent:**
    ```bash
    sudo systemctl restart wazuh-agent
    ```
8.  **Verify Audit Events in Wazuh Dashboard:**
    Navigate to **Security Events** in the Wazuh Dashboard and filter by `rule.groups: audit` to confirm events are flowing.

### 5.4 Auditd Noise: Raw vs Fine-Tuned (Wazuh Context)

*   **Default Auditd Configuration:** Generates 100–500 MB per day, with 80–95% irrelevant noise, making it difficult to operationalize.
*   **Neo23x0 Auditd Configuration:** Reduces volume to 20–100 MB per day, with 60–80% highly relevant. This configuration significantly improves signal-to-noise ratio and tags critical events with specific rule descriptions, making logs actionable for security analysts. Further environmental tuning is often necessary.

#### 5.4.1 Configuring Neo23x0 Auditd Rules with Wazuh

1.  **Download Neo23x0 Ruleset:**
    ```bash
    sudo curl -o /etc/audit/rules.d/audit.rules \
      https://raw.githubusercontent.com/Neo23x0/auditd/master/audit.rules
    ```
2.  **Review and Customize Rules:**
    ```bash
    sudo nano /etc/audit/rules.d/audit.rules
    ```
    *   **Key sections to review:** Exclusions by UID, `execve` rules, Network socket rules, Immutable flag (`-e 2`).
3.  **Apply Rules:**
    ```bash
    sudo augenrules --load
    sudo systemctl restart auditd
    ```
4.  **Verify Rules Loaded:**
    ```bash
    sudo auditctl -l | head -50
    ```
5.  **Confirm Wazuh Agent is Reading Audit Logs:** Ensure the `<localfile>` block for `/var/log/audit/audit.log` is present in `/var/ossec/etc/ossec.conf`.
6.  **Restart Wazuh Agent:**
    ```bash
    sudo systemctl restart wazuh-agent
    ```
7.  **Monitor Initial Event Volume:**
    ```bash
    sudo tail -f /var/log/audit/audit.log
    sudo aureport --summary -i --start today
    ```
8.  **Validate in Wazuh Dashboard:**
    Navigate to **Security Events** and filter by `rule.groups: audit`. Check for Neo23x0 rule keys (e.g., `recon`, `network_connection`, `persistence`) and correct MITRE ATT&CK mappings.

### 5.5 Additional Wazuh Features for Enhanced Security

Wazuh extends native telemetry with powerful capabilities:

1.  **VirusTotal Integration (Malware Detection):** Automatically scan suspicious files.
    ```xml
    <!-- Add to ossec.conf on the Wazuh Manager -->
    <integration>
      <name>virustotal</name>
      <api_key><YOUR_VIRUSTOTAL_API_KEY></api_key>
      <rule_id>550, 554, 594</rule_id>
      <alert_format>json</alert_format>
    </integration>
    ```
    Then, `sudo systemctl restart wazuh-manager`.
2.  **Vulnerability Detector:** Enable CVE scanning on all agents.
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
3.  **Active Response - Auto-Block Attackers:** Automatically block IPs triggering brute force rules (e.g., rule 5763 for SSH brute force).
    ```xml
    <!-- Add to ossec.conf -->
    <active-response>
      <command>firewall-drop</command>
      <location>local</location>
      <rules_id>5763</rules_id>
      <timeout>600</timeout>
    </active-response>
    ```
4.  **MITRE ATT&CK Dashboard:** Wazuh natively maps alerts to MITRE ATT&CK, allowing for heatmap visualization of detected techniques, cross-referencing with Cortado RTA results, and identification of detection gaps.
5.  **Custom Rules for Cortado RTAs:** Create specific rules to detect RTA behaviors.
    ```xml
    <!-- Create /var/ossec/etc/rules/cortado_rta.xml -->
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
    Test rules with `sudo /var/ossec/bin/wazuh-logtest` and restart manager `sudo systemctl restart wazuh-manager`.
6.  **Email Alerting:** Configure email notifications for critical alerts.
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
7.  **Log Retention Policy:** Implement a cron job to delete old indices and prevent disk exhaustion.
    ```bash
    # Add to crontab for automatic cleanup (e.g., daily at 2am)
    sudo crontab -e
    0 2 * * * curl -sk -u admin:admin -X DELETE \
      "https://localhost:9200/wazuh-alerts-4.x-$(date -d '90 days ago' +\%Y.\%m.\%d)" \
      >> /var/log/wazuh-index-cleanup.log 2>&1
    ```
8.  **Security Configuration Assessment (SCA):** Wazuh automatically audits system hardening against benchmarks like CIS, NIST, and PCI-DSS.

## 6. Conclusion

The journey from individual native telemetry configurations to a unified Wazuh deployment reveals a powerful strategy for comprehensive threat detection. While Sysmon and auditd, when meticulously configured with community-driven rulesets, offer significant visibility into malicious activities, they inherently possess blind spots and operational challenges.

Wazuh effectively bridges these gaps by providing a centralized platform for:

*   **Aggregating and enriching** diverse telemetry sources.
*   **Correlating events** across Windows and Linux.
*   **Mapping detections directly to MITRE ATT&CK**, making RTA results immediately actionable.
*   **Automating responses** and providing vulnerability context.
*   **Reducing noise** and improving the signal-to-noise ratio through advanced rule processing.

By combining the deep kernel-level visibility of native OS telemetry with the robust XDR and SIEM capabilities of Wazuh, organizations can achieve a highly effective, cost-efficient, and adaptable security monitoring solution capable of identifying and responding to advanced cyber threats across their entire IT infrastructure. This approach not only enhances detection capabilities but also streamlines security operations, enabling teams to proactively defend against evolving adversary techniques.
