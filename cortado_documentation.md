# Internal Documentation: Cortado RTAs Overview


This report evaluates the efficacy of native Windows telemetry and Sysmon as a standalone alternative to third-party endpoint detection agents for identifying advanced cyber threats. Utilizing the Cortado RTA (Red Team Automation) framework developed by Elastic, we simulated a diverse range of adversary behaviors across a controlled Windows environment.

We primarily focused on comparing a default Sysmon installation against a hardened configuration based on the SwiftOnSecurity ruleset. Our findings indicate that a well-tuned Sysmon deployment achieves an 80% detection rate across sampled MITRE ATT&CK techniques, providing high-fidelity visibility into process lineage and unauthorized file system modifications.

However, the study also identifies critical telemetry gaps. This report concludes that while Sysmon significantly reduces the need for external agents, it requires strategic configuration to address remaining blind spots in the detection of sophisticated, low-profile attacks.

---
## 1. Motivation

The goal of this project is to determine if threat detection is achievable without the use of third-party endpoint agents. While dedicated agents provide visibility, they often introduce management overhead and resource constraints. We focus on leveraging native Windows telemetry and Sysmon to identify malicious activity.

---

## 2. Methodology

### 2.1 Tool Selection: Cortado RTAs 
To simulate malicious behavior, we utilized Cortado RTAs, an open-source suite developed by Elastic. Cortado is designed for testing and analyzing defensive telemetry through two primary methods:
* **CodeRTA:** Implements behavior directly in Python code. 
* **HashRTA:** References binary samples by hash that exhibit behaviors for detection. 

Each RTA contains metadata such as ID, name, supported platforms, associated security rules, and MITRE ATT&CK techniques.

### 2.2 RTAs Execution and Initial Assessment
All simulations were conducted within a controlled sandbox environment to ensure safety and consistency.

* **Operating System:** The primary testbed was a standard Windows 2011 (Windows 11) Virtual Machine.
* **Execution:** We began by executing individual RTAs using cortado-run-rta to identify their operational status. This initial phase helped us pinpoint which RTAs executed successfully and which encountered issues such as missing files or broken code. To ensure an effective evaluation, we sampled 20 RTAs from the total library of 258 Windows-based RTAs. 

### 2.3 Detection Validation and Enhanced Telemetry
To validate that the executed RTAs were triggering detection events, we initially utilized Elastic's Endpoint Agent (free version). However, our primary objective was to assess the detection capabilities of our core infrastructure without relying on a full endpoint agent. Therefore, our main focus shifted to analyzing **Sysmon logs** using two configurations:

* **Default Sysmon Configuration:** To establish a baseline of what is detected out-of-the-box.
* **SwiftOnSecurity Sysmon Configuration:** To evaluate the enhanced visibility provided by a robust, community-driven ruleset.

---

## 3. Results 

### 3.1 Detection Performance Summary
The implementation of the **SwiftOnSecurity** configuration significantly enhanced telemetry across the attack lifecycle, particularly for "Living off the Land" (LotL) techniques that typically evade standard Windows logging. 

#### 3.1.1 Statistical Summary of Findings
After analyzing the sampled RTAs, the following trends were identified regarding the effectiveness of the standalone Sysmon deployment:

* **Detection Rate:** **80%** of the sampled RTAs produced high-fidelity logs suitable for immediate detection. 
* **Masquerading Detection:** In **100%** of cases where an attacker renamed a binary (e.g., renaming a C2 agent to `winword.exe`), Sysmon successfully identified the deception via the `OriginalFileName` field.
* **LotL Visibility:** The configuration was highly effective at flagging the abuse of legitimate Windows tools like `certutil.exe`, `nslookup.exe`, and `vssadmin.exe`.

#### 3.2 Comparative Analysis of Logging Sources
During the evaluation, a clear hierarchy of visibility emerged between native logging, default Sysmon, and the hardened SwiftOnSecurity configuration:

* **Native Windows Logging:** Standard event logs proved insufficient for forensic reconstruction. Most RTA actions either left no trace or were buried in generic "Success" audits that lacked the process command-line arguments and file hashes necessary to confirm malicious intent.
* **Default Sysmon Configuration:** While the default installation captured a vast amount of data, it proved difficult to operationalize. It logged nearly every system event without categorization, leading to a high volume of "background noise" from standard OS processes. Crucially, the default logs lacked descriptive rule names, making it impossible to quickly map an event to a specific threat technique without manual analysis.
* **SwiftOnSecurity Configuration:** This configuration resolved the previous issues by filtering out known-benign noise and tagging events with specific rule names. This allowed for immediate identification of the RTA behavior.


#### 3.1.2 Summary of Detection Capability
The following table summarizes how the SwiftOnSecurity configuration handled key MITRE ATT&CK techniques:

| MITRE Technique | RTA Simulation | Key Sysmon Field | Detection Difficulty |
| :--- | :--- | :--- | :--- |
| **T1003.002** | Mimikatz SSP | `CommandLine` | **Easy** (Keyword Match) |
| **T1547.012** | Print Processor | `TargetFilename` | **Moderate** (Path Audit) |
| **T1490** | Shadow Copy Deletion | `IntegrityLevel` | **Easy** (Privilege Audit) |
| **T1590.005** | IP Reconnaissance | `QueryName` | **Moderate** (Process Context) |

---

## 4. Discussion

The telemetry generated during the simulations provides a highly actionable framework for detection engineering. The following sections analyze the specific forensic evidence captured during the RTA executions and the inherent trade-offs in defensive configurations.

### 4.1 Forensic Analysis of Execution and Persistence
In the **Sliver C2** and **Mimikatz** simulations, Sysmon captured the full process lineage, including `ParentCommandLine` and `Hashes`. This allowed us to trace malicious activity back to the source, identifying that the commands originated from a Python-based RTA engine. This level of detail is critical for distinguishing between legitimate administrative scripts and automated attack frameworks.

![Sysmon Event ID 1 - Mimikatz Simulation](<VirtualBox_Windows 2011_11_02_2026_09_58_23.png>)
*Figure 1: Sysmon Event ID 1 capturing the execution of a Mimikatz-related command via PowerShell, initiated by the Cortado RTA framework.*

Regarding persistence, the configuration successfully flagged the modification of sensitive registry keys during the **SolarMarker** simulation. Furthermore, it provided direct mapping to **T1547.012** (Print Processor Sideloading) by generating a `FileCreate` (Event ID 11) when an unauthorized `rta.dll` was dropped into the `\spool\drivers\` directory—a location rarely modified by standard users.

### 4.2 Network Anomalies and C2 Discovery
The configuration exposed Command & Control (C2) channels by monitoring non-standard process behavior. This was evidenced by flagging automated reconnaissance, such as `powershell.exe` querying `api.ipify.org`. Because this activity originated from a command-line tool rather than a standard web browser, it serves as a high-fidelity indicator of automated discovery.

### 4.3 Technical Gap: The `nslookup` DNS Logging "Miss"
A significant observation occurred during the **network_connection_nslookup** RTA. While Sysmon successfully recorded the **Process Creation (Event ID 1)** for `nslookup.exe`, it failed to generate a **DNS Query (Event ID 22)**. 

* **The Reason:** The SwiftOnSecurity configuration is optimized for "High Fidelity" to reduce log volume. Because `nslookup` is a standard tool used frequently by IT administrators for troubleshooting, it is often explicitly excluded from DNS Query logging to prevent "noise."
* **The Risk:** This exclusion creates a blind spot. An attacker can leverage `nslookup` for **DNS Tunneling** or data exfiltration, remaining invisible to DNS-specific monitoring rules because the tool is "trusted" by the configuration. This highlights the need for behavioral monitoring (analyzing *why* the tool is running) rather than relying solely on individual event triggers.

### 4.4 The "Public" Folder as a Universal Indicator
A recurring pattern across multiple successful detections (including `browser_debugging` and `uac_computerdefaults`) was the use of `C:\Users\Public\` for staging malicious binaries. Because this directory is globally writeable but rarely used by legitimate enterprise applications for execution, any process creation (**Event ID 1**) originating from `C:\Users\Public\` or `C:\Windows\Temp\` should be treated as a high-priority alert.

### 4.5 Analysis of Detection Gaps
Certain RTAs, such as `port_monitor` and `crashdump_disabled`, did not produce actionable Sysmon logs. These techniques often involve direct memory manipulation or internal OS flag changes that do not trigger the specific kernel callbacks Sysmon monitors. To mitigate these gaps, supplemental logging such as Windows Event ID 4657 (Registry Value Modified) or ETW (Event Tracing for Windows) providers would be required to gain visibility into kernel-level configuration changes.

## 5. Appendix

## 5.1 Setting Up Your Cortado RTAS Environment

This section details the technical prerequisites and step-by-step installation process for Cortado RTAS. Official Cortado documentation uses Poetry to install Cortado; this documentation covers installation using `pip`.

### 5.1.1 Prerequisites

Before proceeding, ensure your test environment meets the following requirements:

*   **Software Dependencies:**
    *   Python `>=3.12` (ensure `pip` is updated).
*   **Access & Permissions:**
    *   Administrative rights for running RTAs.

### 5.1.2 Cortado RTAS Installation Guide

Follow these steps to install Cortado RTAs:

1.  **Install the Wheel File using `pip`:**
    Once you are in the correct directory, run the following command to install Cortado:

    ```bash
    pip install cortado-0.1.0-py3-none-any.whl
    ```

### 5.1.3 Operating RTAs & Analyzing Logs

The Cortado RTAs CLI provides commands to execute individual or all relevant RTAs in a sandboxed environment with the minimal dependencies.

### 5.1.4 Key Execution Commands

Cortado RTAS offers two primary commands for executing RTAs:

*   **`cortado-run-rta`**: Executes a *particular* RTA. This is useful for focused testing of a specific technique.
*   **`cortado-run-rtas`**: Executes *all* RTAs that match the current operating system. This is ideal for comprehensive testing across multiple techniques.



### 5.1.5 Executing a Specific RTA (`cortado-run-rta`)

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


### 5.1.6 Executing All Relevant RTAs (`cortado-run-rtas`)

To perform a broad test across all RTAs compatible with your operating system, use the `cortado-run-rtas` command.

```bash
cortado-run-rtas 
```

**Important:** RTAS techniques are designed to simulate the *actions* of ransomware, not necessarily to fully execute a malicious payload. **It is expected and often desired that RTAS techniques may not run flawlessly or complete their full intended malicious action.**

*   **The Goal:** The primary objective is to trigger your detection rules by mimicking the *initial stages* or *attempted behaviors* of ransomware.
*   **Example:** A technique might attempt to encrypt files but fail due to permissions. However, the *attempted file access patterns* should still trigger a "Suspicious File Access" or "Ransomware Behavior" alert in Elastic Security. This validates your EDR's ability to detect early-stage TTPs.
*   For a detailed summary of the execution outcomes and functional status of each RTA, please refer to the `Windows RTA Summary.md` document.
  
#### 5.1.7 Potential Cortado Set Up Issues

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

### 5.2 RTA Functional Status Summary
[Windows RTA Summary Report](./windows_rta_summary.md)



### 5.3 Sysmon Configuration Details

To get started with Sysmon, you'll first need to download the necessary files:

- **Sysmon Executable:**
    - Go to the official Microsoft Sysinternals page: [https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
    - Download the latest version of Sysmon. It usually comes as a `.zip` file.
- **SwiftOnSecurity Sysmon Configuration (for advanced setup):**
    - Go to the SwiftOnSecurity GitHub repository: [https://github.com/SwiftOnSecurity/Sysmon-config](https://github.com/SwiftOnSecurity/Sysmon-config)
    - Download the `sysmonconfig-export.xml` file. 

#### 5.3.1 Default Sysmon Configuration

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

#### 5.3.2 SwiftOnSecurity Sysmon Configuration

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

### 5.4 Sysmon Log Analysis Summary 
[Sysmon Log Analysis Report](./sysmon_log_analysis.md)

### 5.5 Elastic Installation 
[Link to installation guidelines](https://www.elastic.co/docs/reference/fleet/install-elastic-agents)
