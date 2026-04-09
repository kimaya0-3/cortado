# Cortado documentation for Linux 

This report evaluates the efficacy of Linux auditd telemetry as a standalone solution for identifying advanced cyber threats on Ubuntu Server. Utilizing the Cortado RTA (Red Team Automation) framework developed by Elastic, we simulated a diverse range of adversary behaviors across a controlled Linux environment.

We primarily focused on comparing a default auditd installation against a hardened configuration based on the Neo23x0 ruleset. Our preliminary findings indicate that a well-tuned auditd deployment, especially with comprehensive community rules, achieves a strong detection rate across sampled MITRE ATT&CK techniques, providing high-fidelity visibility into process execution, file system modifications, and network activity.

---
## 1. Motivation

The goal of this project is to determine if robust threat detection is achievable on Linux systems without the exclusive reliance on third-party endpoint agents. While dedicated security agents provide valuable visibility, they can often introduce significant management overhead, compatibility challenges, and resource constraints in diverse Linux environments. Our focus is on leveraging native Linux auditd telemetry to identify malicious activity, aiming to provide a powerful, integrated, and potentially more lightweight alternative for security monitoring.

---

## 2. Methodology


### 2.1 RTAs Execution and Initial Assessment
All simulations were conducted within a controlled sandbox environment to ensure safety and consistency.

**Operating System:** The primary testbed was a standard Ubuntu Server 24.04 LTS Virtual Machine.

**Execution:** To ensure an effective evaluation, we sampled 20 RTAs out of the 62 Linux-based RTAS available within Cortado. This targeted sample allowed us to assess a representative set of adversary behaviors relevant to Linux systems.

---

## 3. Results 

### 3.1 Detection Performance Summary

#### 3.1.1 Statistical Summary of Findings
After analyzing the sampled RTAs, the following trends were identified regarding the effectiveness of the standalone auditd deployment:

* **Detection Rate:** **100%** of the sampled RTAs produced high-fidelity logs suitable for immediate detection. 

#### 3.2 Comparative Analysis of Logging Sources
During the evaluation, a clear hierarchy of visibility emerged between native logging, default auditd, and the hardened configuration:

**Native Linux Logging (e.g., syslog, journalctl):** Standard system logs proved insufficient for forensic reconstruction of advanced threats. Most RTA actions either left minimal trace or generated generic entries that lacked the granular detail, process lineage, and specific event types necessary to confirm malicious intent or map directly to MITRE ATT&CK techniques.

**Default auditd Configuration:** While a default auditd installation captures a significant amount of system call data, it proved challenging to operationalize for threat detection. It logs nearly every audited system event without fine-grained filtering or categorization, leading to an extremely high volume of "background noise". A basic configuration monitoring, authethication and file changes might generate 100-500 MB per day on a typical server. Approximately 80-95% of this volume is typically irrelevant to us. Crucially, default auditd logs lack descriptive rule names, making it difficult to quickly map an event to a specific threat technique without deep auditd knowledge and manual analysis. 

**Neo23x0 auditd Configuration:** For the same level of system activity, it might generate 20-100 MB per day, out of which 60-80% is highly relevant and actionable. This configuration significantly improved upon the default by introducing targeted rules designed to filter out known-benign noise and highlight suspicious behaviors. It effectively reduced the sheer volume of irrelevant logs while tagging critical events with specific rule descriptions. This allowed for more immediate identification of RTA behaviors and their associated MITRE ATT&CK techniques, making the logs far more actionable for security analysts. However, even with this hardened configuration, a degree of tuning specific to your environment is often necessary to further optimize signal-to-noise ratio, as some legitimate applications might trigger rules designed for malicious activity. 

---

## 4. Discussion

### 4.1 Efficacy of auditd with Neo23x0 Ruleset
Our evaluation demonstrated that a properly configured auditd, especially when leveraging the community-driven Neo23x0 ruleset, is effective in detecting a wide array of simulated adversary behaviors. The 100% detection rate across the sampled RTAs is a strong indicator of its potential as a robust security monitoring solution. The Neo23x0 configuration provided high-fidelity logs with crucial details such as process lineage, user IDs, file paths, and specific system calls, which are all vital for understanding the full scope and context of an attack. This represents a significant improvement over a default auditd installation, transforming raw system call data into actionable security events.

### 4.2 Strengths of auditd as a Standalone Solution
The project highlights several compelling strengths of auditd as a standalone telemetry source for Linux threat detection:

**Native Integration and Deep Visibility:** As a kernel-level component, auditd offers unparalleled, deep visibility into system calls and kernel events. This allows for the capture of low-level activities that might be missed or obscured by user-space agents, providing a foundational layer of security monitoring.

**Resource Efficiency:** While log volume can be substantial, auditd itself is generally lightweight in terms of CPU and memory footprint. This makes it an attractive option for environments where resource constraints or the overhead of third-party agents are a concern.

**Customizability:** The rule-based nature of auditd allows for extensive customization. Security teams can tailor detection logic to their specific threat models, compliance requirements, and unique environmental characteristics, providing a highly adaptable security tool.

### 4.3 Identified Gaps and Limitations
Despite its strengths, the study also revealed several operational challenges and limitations that must be considered:

**Persistent Background Noise:** While the Neo23x0 ruleset dramatically improved the signal-to-noise ratio compared to a default auditd configuration, a degree of background noise persists. Distinguishing between legitimate, highly privileged administrative actions and malicious activities can still be challenging. Further environmental-specific tuning is often necessary to optimize the ruleset and minimize false positives.

**Log Volume and Storage:** Even with optimized rules, auditd can generate a massive volume of logs. This necessitates robust infrastructure for log aggregation, long-term storage, and efficient processing to prevent data loss and ensure timely analysis.

**Potential Blind Spots:** While auditd provides deep visibility, certain highly sophisticated or memory-resident attacks might still evade detection if they do not trigger specific system calls covered by the ruleset. Continuous research and rule development are essential to address evolving adversary techniques.

### 4.4 Operational Considerations

**Leveraging AI for Enhanced Analysis:** The rich, granular data provided by auditd presents a significant opportunity for advanced analytics. Specifically, when these logs are fed into a Large Language Model (LLM), the LLM can process the extensive contextual information including process execution details, file access patterns, and system call sequences to identify subtle anomalies and correlate seemingly disparate events into coherent malicious activity. This capability can dramatically reduce the manual effort required for threat hunting and incident response, allowing security analysts to efficiently pinpoint and understand complex attack chains that might otherwise be buried in the sheer volume of data.

------

## 5. Appendix

### 5.1 Prerequisites and VM Setup

To replicate the test environment, the following were used:

*   **Virtualization:** VirtualBox
*   **Operating System:** Ubuntu Server 24.04 LTS (ISO available from [ubuntu.com/download/server](https://ubuntu.com/download/server))
*   **Minimum VM Resources:**
    *   **RAM:** 2GB
    *   **Storage:** 11GB
    *   **CPU:** 1 core

**VM Creation Steps:**

1.  **Create VM:** In VirtualBox, create a new VM named `ubuntu-server` (Linux, Ubuntu 64-bit). Allocate 2048 MB RAM and an 11GB virtual disk.
2.  **Attach ISO:** In VM **Settings → Storage**, attach the downloaded Ubuntu Server 24.04 ISO.
3.  **Install Ubuntu:** Boot the VM and follow the on-screen prompts for a standard Ubuntu Server installation.
    *   Select preferred language and keyboard layout.
    *   Choose "Use entire disk" for storage.
    *   Set up a user account and password.
    *   Select "Install OpenSSH Server" for remote access.
    *   Skip "Featured Server Snaps."
4.  **Initial Setup:** After installation and reboot, log in.
    *   Update the system: `sudo apt update && sudo apt upgrade -y`

### 5.2 `auditd` Installation and Neo23x0 Ruleset

To enable and configure `auditd` with the enhanced rules:

1.  **Install `auditd`:**
    ```bash
    sudo apt install auditd -y
    ```
2.  **Start & Enable `auditd`:**
    ```bash
    sudo systemctl start auditd
    sudo systemctl enable auditd
    sudo systemctl status auditd # Verify it's running
    ```
3.  **Install `git`:**
    ```bash
    sudo apt install git -y
    ```
4.  **Download Neo23x0 Rules:**
    ```bash
    git clone https://github.com/Neo23x0/auditd.git
    cd auditd
    sudo cp audit.rules /etc/audit/rules.d/audit.rules
    ```
5.  **Apply Rules:**
    ```bash
    sudo systemctl restart auditd
    sudo auditctl -l # Verify rules are loaded
    ```

### 5.3 Basic `auditd` Usage

Understanding how to interact with `auditd` and its logs is crucial for analysis.

*   **Viewing Live Events:**
    The `ausearch` command is used to query `auditd` logs. For live monitoring, `auditd` events are typically streamed to `/var/log/audit/audit.log`. You can watch this file in real-time:
    ```bash
    sudo tail -f /var/log/audit/audit.log
    ```
    *Note: The raw logs can be verbose. The Neo23x0 ruleset adds `type=SYSCALL` entries with `key=` fields for easier filtering.*

*   **Searching Historical Events:**
    `ausearch` is the primary tool for querying past events. It allows filtering by various criteria:
    *   **By Key (Rule Name):** To find events triggered by a specific Neo23x0 rule (e.g., "T1003_passwd_dump"):
        ```bash
        sudo ausearch -k T1003_passwd_dump
        ```
    *   **By Executable:** To find all executions of a specific command:
        ```bash
        sudo ausearch -c bash
        ```
    *   **By User ID:** To find events related to a specific user (e.g., `uid=1000`):
        ```bash
        sudo ausearch -au 1000
        ```
    *   **By Time:** To search for events within a specific time range:
        ```bash
        sudo ausearch --start today --end now
        sudo ausearch --start 09/01/2024 00:00:00 --end 09/01/2024 23:59:59
        ```
    *   **Combining Filters:** You can combine multiple filters. For example, to find all `bash` executions by user `1000` today:
        ```bash
        sudo ausearch --start today -au 1000 -c bash
        ```

*   **Interpreting `auditd` Logs:**
    Raw `auditd` logs can be complex. Each event often consists of multiple lines, with `type=` fields indicating different aspects of the event (e.g., `SYSCALL`, `CWD`, `PATH`, `PROCTITLE`). The `key=` field, especially with the Neo23x0 rules, is critical as it provides a human-readable name for the rule that triggered the log, often directly mapping to a MITRE ATT&CK technique.

    Example log snippet (simplified):
    ```
    type=SYSCALL msg=audit(1678886400.000:123): arch=c000003e syscall=59 success=yes exit=0 a0=55a0b7c2b0e0 a1=55a0b7c2b110 a2=55a0b7c2b120 a3=7ffe21b2b000 items=2 ppid=1234 pid=5678 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=1 comm="bash" exe="/usr/bin/bash" key="T1059_004_command_execution"
    type=CWD msg=audit(1678886400.000:123): cwd="/home/user"
    type=PATH msg=audit(1678886400.000:123): item=0 name="/usr/bin/id" inode=123456 dev=fd:00 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
    type=PROCTITLE msg=audit(1678886400.000:123): proctitle=6964
    ```
    In this example, `key="T1059_004_command_execution"` immediately tells an analyst that a command execution event, potentially related to MITRE ATT&CK T1059.004 (Command and Scripting Interpreter: Unix Shell), occurred.
