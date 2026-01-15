## Sysmon log analysis 

| RTA Name | Useful | Default Sysmon Configration logs | SwiftOnSecurity Configuration Logs | Comments |
|---------------|--------------------------|----------------------------------------|---------------------------------|-------------------|
|adobe_hijack | No | | File created; Process Terminated : from python  |  | 
|bitsadmin_download | Yes | | File created; Network Connection Detected; Dns query| |
|bitsadmin_execution | Yes | | File created; Process created; Process terminated | Process created - winword launching bitsadmin to download files|
|browser_debugging | Yes | | File created; Process create; Process Terminated | Process created - chrome is being lauched from C:\Users\Public\ instead of Program files, file version, description, company etc are all - , legitmate executables from major software vendors would have these fields populated. |
|browser_exec_downloaded_file | Yes  |  | File created; Process create; File created | Process created - product, company etc from legitimate source should not be - |
|c2_dns_from_iso| Yes | | File created; Process Create; Dns query; Process terminated | Process created - legitimate ping.exe should reside in C:\Windows\System32, executing it from D drive is unusual| 
|certutil_file_obfuscation | Yes | |  Process create; File created; Process Terminated | use of certutil.exe for encoding and decoding arbitrary files is a known technique for Defense Evasion & Ingree Tool Transfer. Legitimate use of certutil.exe is rare in typical user environments. Encoding and the decoding executables and creation of decoded.exe by certutil.exe |
|certutil_webrequest | Yes | | Process create;  File created; Process terminated; Dns query; Network connection detected  |
|child_w3wp | Yes | | Process terminated; Process Create; File created |  the presence of w3wp.exe in C:\Users\Public\ instead of its legitimate location (C:\Windows\System32\inetsrv\) + the lack of file version information |
|clr_logs_creation | Yes | | Process create; File created; Process terminated | | 
|cmd_shell_via_word| Yes | | File created; Process Create; Process Create; Process terminated; Registry value set | winword.exe should be located in C:\Program Files\Microsoft Office \ not in C:\Windows\System32\, original file name rta.exe was renamed to winword.exe -> masquerading | 
|cmstp_image_load | Yes | | File created; Process Create; Process terminated | |
|collection_keylog_hook_keystate| No | | Process terminated |
|collection_keylog_rawinputdevice| No | |Process terminated |
|comsvcs_dump | Yes | | Process create; File created; Process terminated | CommandLine: "C:\WINDOWS\system32\rundll32.exe" this specific command line is a well-known technique for credential dumping (MITRE ATT&CK T1003.001 - OS Credential Dumping: LSASS Memory). Attackers use this to extract credentials (like NTLM hashes or plaintext passwords) from memory, often targeting the Local Security Authority Subsystem Service (lsass.exe). |
|crashdump_disabled | No | | Process terminated  | |
|credaccess_reg_query_privesc_token_manip | Yes || Process create; Process terminated | | 




