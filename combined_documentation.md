
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

#### 3.2.2 Comparative Analysis of Logging Sources 

*   **Native Windows Logging:** Standard event logs proved insufficient for forensic reconstruction. Most RTA actions either left no trace or were buried in generic "Success" audits that lacked the process command-line arguments and file hashes necessary to confirm malicious intent.
*   **Default Sysmon Configuration:** While the default installation captured a vast amount of data, it was difficult to operationalize. It logged nearly every system event without categorization, leading to a high volume of "background noise" from standard OS processes. Crucially, default logs lacked descriptive rule names, making it impossible to quickly map an event to a specific threat technique without manual analysis.
*   **SwiftOnSecurity Configuration:** This configuration resolved the previous issues by filtering out known-benign noise and tagging events with specific rule names, allowing for immediate identification of RTA behavior.

### 3.3 Discussion: Forensic Insights and Gaps 

The evaluation of Sysmon on Windows, particularly with the SwiftOnSecurity ruleset, provided valuable forensic insights into various RTA simulations, alongside identifying crucial detection gaps.

#### 3.3.1 Forensic Analysis Highlights

Sysmon proved highly effective in capturing critical forensic artifacts:

*   **Execution & Persistence:** It successfully recorded full process lineage, including parent-child relationships and hashes for activities like Sliver C2 and Mimikatz, allowing for clear tracing of malicious execution. Persistence mechanisms, such as registry key modifications (SolarMarker) and unauthorized DLL drops (T1547.012), were also reliably flagged.
*   **Network Anomalies & C2:** The configuration identified Command & Control (C2) activity by detecting non-standard processes initiating network connections, such as `powershell.exe` performing reconnaissance, which would be unusual for a typical user.
*   **Universal Indicator: `C:\Users\Public\`:** A significant observation was the consistent flagging of malicious binaries executed from `C:\Users\Public\` or `C:\Windows\Temp\`. Given these directories are globally writable but rarely used for legitimate application execution, any `Process Creation` (Event ID 1) from these locations serves as a high-priority indicator of compromise.

#### 3.3.2 Identified Detection Gaps

Despite its strengths, certain limitations were observed:

*   **`nslookup` DNS Logging "Miss":** A notable gap occurred with `nslookup` DNS queries. While process creation was logged, the actual DNS query (Event ID 22) was often missed. This is typically due to configurations like SwiftOnSecurity prioritizing "High Fidelity" to reduce noise, often excluding `nslookup` from DNS logging. This creates a blind spot for potential DNS tunneling or data exfiltration, emphasizing the need for behavioral analysis beyond mere event triggers.
*   **Technique-Specific Blind Spots:** Certain RTAs, such as `port_monitor` and `crashdump_disabled`, did not generate actionable Sysmon logs. These techniques often involve low-level OS interactions or direct memory manipulation that do not trigger the specific kernel callbacks Sysmon is configured to monitor, indicating a need for supplemental logging or alternative detection methods for these scenarios.

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
