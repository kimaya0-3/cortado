# Internal Documentation: Cortado RTAS Overview

This guide provides comprehensive technical details, setup instructions, and operational procedures for working with Cortado RTAS.

---
## 1. Motivation

The goal of this project is to determine if threat detection is achievable without the use of third-party endpoint agents. While dedicated agents provide visibility, they often introduce management overhead and resource constraints. We focus on leveraging native Windows telemetry and Sysmon to identify malicious activity.

## 2. Introduction to Cortado RTAs

### 2.1 What is Cortado RTAs?

Cortado RTAS is an open-source suite developed by Elastic, designed for testing and analyzing ransomware techniques. 

Cortado supports two main types of RTAs:
* **CodeRTA:** Implements behavior directly in Python code 
* **HashRTA:** References binary samples by hash that exhibit behaviors for detection. They cannot be directly executed throught the CLI (generates an error). It is used primarily for documenting associations between malware samples and detection rules.

Each RTA contains metadata such as ID, name, supported platforms, associated security rules, and MITRE ATT&CK techniques.

### 2.2 Scope of this Document

This documentation covers:

*   Installation and configuration of Cortado RTAS.
*   Execution and analysis of RTAs.
*   Integration with Sysmon.

---

## 3. Setting Up Your Cortado RTAS Environment

This section details the technical prerequisites and step-by-step installation process for Cortado RTAS. Official Cortado documentation uses Poetry to install Cortado; this documentation covers installation using `pip`.

### 3.1 Prerequisites

Before proceeding, ensure your test environment meets the following requirements:

*   **Software Dependencies:**
    *   Python `>=3.12` (ensure `pip` is updated).
*   **Access & Permissions:**
    *   Administrative rights for running RTAs.

### 3.2 Cortado RTAS Installation Guide

Follow these steps to install Cortado RTAs:

1.  **Install the Wheel File using `pip`:**
    Once you are in the correct directory, run the following command to install Cortado:

    ```bash
    pip install cortado-0.1.0-py3-none-any.whl
    ```

## 4. Operating RTAs & Analyzing Logs

The Cortado RTAs CLI provides commands to execute individual or all relevant RTAs in a sandboxed environment with the minimal dependencies.

### 4.1 Key Execution Commands

Cortado RTAS offers two primary commands for executing RTAs:

*   **`cortado-run-rta`**: Executes a *particular* RTA. This is useful for focused testing of a specific technique.
*   **`cortado-run-rtas`**: Executes *all* RTAs that match the current operating system. This is ideal for comprehensive testing across multiple techniques.



### 4.2 Executing a Specific RTA (`cortado-run-rta`)

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


### 4.3 Executing All Relevant RTAs (`cortado-run-rtas`)

To perform a broad test across all RTAs compatible with your operating system, use the `cortado-run-rtas` command.

```bash
cortado-run-rtas 
```

**Important:** RTAS techniques are designed to simulate the *actions* of ransomware, not necessarily to fully execute a malicious payload. **It is expected and often desired that RTAS techniques may not run flawlessly or complete their full intended malicious action.**

*   **The Goal:** The primary objective is to trigger your detection rules by mimicking the *initial stages* or *attempted behaviors* of ransomware.
*   **Example:** A technique might attempt to encrypt files but fail due to permissions. However, the *attempted file access patterns* should still trigger a "Suspicious File Access" or "Ransomware Behavior" alert in Elastic Security. This validates your EDR's ability to detect early-stage TTPs.
*   For a detailed summary of the execution outcomes and functional status of each RTA, please refer to the `Windows RTA Summary.md` document.
  

## 5. Methodology


### 5.1 RTAs Execution and Initial Assessment

We began by executing individual RTAs using `cortado-run-rta` to identify their operational status. This initial phase helped us pinpoint which RTAs executed successfully and which encountered issues such as missing files or broken code. This step was crucial for understanding the baseline functionality of each technique. 

For a detailed breakdown of which RTAs were functional or encountered issues during testing, please refer to the [Windows RTA Summary](./windows_rta_summary.md) linked in the Appendix.

To ensure an effective evaluation, we sampled 20 RTAs from the total library of 258 Windows-based RTAs. 

### 5.2 Detection Validation and Focus on Sysmon

To validate that the executed RTAs were indeed triggering detection events, we initially utilized Elastic's Endpoint Agent (free version). This confirmed that malicious behaviors were observable by an endpoint security solution.

However, our primary objective was to assess the detection capabilities of our core infrastructure without relying on a full endpoint agent. Therefore, our main focus shifted to analyzing **Sysmon logs**. We conducted this analysis using two configurations:

*   **Default Sysmon Configuration:** To establish a baseline of what is detected out-of-the-box.
*   **SwiftOnSecurity Sysmon Configuration:** To evaluate the enhanced visibility provided by a robust, community-driven configuration.


**Technical Scope Note:**
During our assessment, we identified that certain RTAs are not expected to be caught by standard Sysmon configurations. For example:
crashdump_disabled: Modifies registry values that require specific Event ID 12/13 monitoring.
collection_keylog_hook_keystate: Uses low-level Windows API calls (like GetAsyncKeyState()) that operate below the layer Sysmon typically monitors.

In addition to Sysmon, we also reviewed **system, security and application logs** to identify any detections or alerts generated by the operating system's built-in security features during RTA execution.

---

## 6. Results 
Based on the 20 sampled RTAs, 4 RTAs could not be detected by Sysmon.


### 6.1 Default Configuration Results 


### 6.2 SwiftOnSecurity Configuration Results 


---

## 7. Appendix

### 7.1 Sysmon Configuration Details

To get started with Sysmon, you'll first need to download the necessary files:

- **Sysmon Executable:**
    - Go to the official Microsoft Sysinternals page: [https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
    - Download the latest version of Sysmon. It usually comes as a `.zip` file.
- **SwiftOnSecurity Sysmon Configuration (for advanced setup):**
    - Go to the SwiftOnSecurity GitHub repository: [https://github.com/SwiftOnSecurity/Sysmon-config](https://github.com/SwiftOnSecurity/Sysmon-config)
    - Download the `sysmonconfig-export.xml` file. 

#### 7.1.1 Default Sysmon Configuration

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

#### 7.1.2 SwiftOnSecurity Sysmon Configuration

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

#### 7.2 Potential Cortado Set Up Issues

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

### 7.3 RTA Functional Status Summary
[Windows RTA Summary Report](./windows_rta_summary.md)

### 7.4 Sysmon Log Analysis Summary 
[Sysmon Log Analysis Report](./sysmon_log_analysis.md)