## Windows RTA Summary 

| RTA Name      | Status                   | Error                                  | Fix                             | Expected Failure? | Sysmon Logs Collected | Windows Event Logs |
|---------------|--------------------------|----------------------------------------|---------------------------------|-------------------|--------------------|--------------------|
|adobe_hijack      | Functional after fixing  |File already exists/ OS error |Delete the directory C:\\Programado Files (x86)\\Adobe\\Acrobat Reader DC\\Reader\\AcroCEF after running RTA| |
|appcompat_shim    |Does not work, file missing         |Error- Error while executing command in a subprocess: Command '\['sdbinst.exe', '-q', '-p', 'bin/CVE-2013-3893.sdb']' returned non-zero exit status 1. Reason- bin/CVE-2013-3893.sdb does not exist | | | 
|at_command        |Does not work             | Can't get time from host               | Formatting error? | |
|bitsadmin_download|Works ||||
|bitsadmin_execution|Works || Add source and destination url to code ||
|browser_debugging  |Works |||
|browser_exec_downloaded_file| Works |Permission error during RTA cleanup occurs because the file remains running after its execution timeout, as the timeout only stops waiting, not the process itself, thus locking the file from deletion. | ||
|brute_force_login| Does not work || Need to provide remote host ||
|c2_dns_from_iso  | Works ||||
|certutil_file_obfuscation|Works||||
|certutil_webrequest |Works||||
|child_w3wp | Works ||||
|clr_logs_creation |Works ||||
|cmd_shell_via_word|Does not work | Execution error: bin/renamed.exe fails to execute complex command strings. bin/remnamed.exe file is a resource bundled with the cortado package that's supposed to act as a command interpreter accepting /c flags (similar to cmd.exe). How ever its failing to properly parse and execute complex command-line arguments with spaces, quotes and nested commands. This is most likely a bug in the cortado package. |||
|cmstp_image_load  |Works ||||
|collection_keylog_hook_keystate| Works ||||
|collection_keylog_rawinputdevice| Works ||||
|comsvcs_dump | Works ||||
|crashdump_disabled | Works ||||
|credaccess_reg_query_privesc_token_manip | Works ||||Yes| No|
|credaccess_sam_from_vss| Functional after fixing ||install pywin32 package and change line sam\_path = f"{vss\_list\[0]}\\\\Windows\\\\System32\\\\config\\\\SAM" to sam\_path = f"{vss\_list}\\\\Windows\\\\System32\\\\config\\\\SAM" if necessary ||Yes| No|
|credential_access_known_utilities| Works |Permission error while removing file ProcessDump.exe|||Yes| No|
|credman_discovery |Works ||||Yes| No|
|cscript_suspicious_args |Works ||||U | No|
|dcom_lateral_movement_with_mmc |Works ||||Yes| No|
|ddns_lolbas | Does not work |Execution error: This error is caused by incorrect Powershell command_line syntax in the RTA implementation itself.|Change line to [powershell, "-c", "iwr https://www.noip.com -UseBasicParsing"], after this step the RTA will throw another execution error due to not being able to reach https://www.noip.com; this is because of the Siemens Firewall.||Yes| No|
|ddns_unsigned | Does not work |Error similar to ddns_lolbas |||U | No|
|delete_bootconf | Works ||||Yes| No|
|delete_catalogs | Works |Install windows backup command line tool if not already installed|Install using the following command "Install-WindowsFeature -Name Windows-Server-Backup"||Yes| No|
|delete_usnjrnl  | Works ||||Yes| Yes? Security logs|
|delete_volume_shadows | Works ||||Yes| No|
|disable_windows_fw | Works ||||Yes| No|
|double_persist | Works ||||Yes| No|
|dynwrapx_image_load | Works ||||U| No|
|eicar | Works ||||U| No|
|enum_commands | Works? |||Fails on VM because it is not part of a domain/connected to a Windows network|U| No|
|evasion_addinproc_certoc_odbc_gfxdwn|does not work| False powershell code syntax|||U| No|
|evasion_loadlib_via_callback|Works|Execution error when commands return non-zero exit codes preventing the rest of the RTA from running|try-catch blocks| The detection rules are designed to trigger based on process creation telemetry, not command success.| No| No|
|evasion_ntdll_from_unusual_path | Works ||||U| No|
|evasion_oversized_dll_load | Works ||||U? | No|
|evasion_patch_etw_amsi | Works ||||U| No|
|evasion_unhook_ldrloaddll|Works|Execution error||Detection rules are triggered|U| No|
|exec_certoc_dll| Works |Permission error|||Yes| No|
|exec_cmd_adfind | Works ||||Yes| No|
|exec_cmd_appcmd_logging | Works ||||Yes| No|
|exec_cmd_arp | Works ||||U| No|
|exec_cmd_aspnet_regiis | Works ||||Yes| No|
|exec_cmd_attrib_hidden | Works ||||Yes| No|
|exec_cmd_auditpol | Works ||||Yes| No|
|exec_cmd_clear_history | Works ||||U| No|
|exec_cmd_compiled_html | Works ||||Yes| No|
|exec_cmd_endpoint_security_masquerading | Works |Permission error while removing the file|||No| No|
|exec_cmd_fltmc_unload | Works ||||Yes| No|
|exec_cmd_fsutil_fsinfo | Works ||||U| No|
|exec_cmd_hidden_share | Works ||||Yes| No|
|exec_cmd_mklink | Works ||||Yes| No|
|exec_cmd_mpcmdrun_download | Works ||||Yes| No|
|exec_cmd_msdt | Works |Permission error while removing the file |||U| No|
|exec_cmd_mssql_xp_cmdshell | Works ||||Yes| No|
|exec_cmd_net_stop | Works ||||Yes| No|
|exec_cmd_net_use | Works ||||
|exec_cmd_netsh_advfirewall_network_discovery | Works ||||
|exec_cmd_netsh_remotedesktop | Works ||||
|exec_cmd_nltest | Works ||||
|exec_cmd_ntdsdit| Works ||||
|exec_cmd_posh_mailbox | Works ||||
|exec_cmd_psexesvc | Works |Permission error while removing the file|||
|exec_cmd_pwd_appcmd |Works |Execution error||Detection rules are triggered regardless|
|exec_cmd_rundll32 | Works ||||
|exec_cmd_rundll32_davsetcookie | Works ||||
|exec_cmd_set_casmailbox | Works ||||
|exec_cmd_set_mppreference | Works ||||
|exec_cmd_short_name | Works|Permission error while removing the file|||
|exec_cmd_susp_args | Works ||||
|exec_cmd_windows_firewall_disabled | Works ||||
|exec_cmd_wmi_cmdexe | Does not work ||||
|exec_cmd_wmi_subscription | Works ||||
|exec_cmd_wmic_antivirus_enum |Works |Execution error|||
|exec_cmd_workfolders | Works |Execution error||Detection rules are triggered regardless|
|exec_cmd_xwizard | Works |Execution error||Detection rules are triggered regardless|
|exec_conhost_indirect |Works |Permission error|||
|exec_control_panel_cpl | Works |Permission error|||
|exec_cscript_archive_args | Works ||||
|exec_cscript_suspicious_powershell | Works ||||
|exec_curl_output | Works |Execution error||Detection rules are triggered regardless|
|exec_dll_file_compressed | Works ||||
|exec_dnguard_program | Works ||||
|exec_echo_named_pipe |Works |Execution error||Detection rules are triggered regardless|
|exec_explorer_trampoline | Works ||||
|exec_forfiles | Works |Execution error||Detection rules are triggered regardless|
|exec_gfxdownloadwrapper | Works ||||
|exec_ingress_tool_posh | Works |Permission error|||
|exec_java_via_scripting |Does not work|File not found error|||
|exec_ms_dotnet_clickonce | Works ||||
|exec_msdt_diagcab | Works ||||
|exec_msiexec_dllregisterserver |Works| Execution error ||Detection rules are triggered regardless|
|exec_nodejs | Works |Permission error|||
|exec_odbcconf | Works ||||
|exec_officecmd | Works ||||
|exec_persistence_from_iso | Works ||||
|exec_renamed_msbuild | Works |Execution error||Detection rules are triggered regardless|
|exec_renamed_winword |Works |Permission error|||
|exec_scripting_persistence_locations| Works |Execution error||Detection rules are triggered regardless|
|exec_scripting_unusual_extension |Works |Execution error||Detection rules are triggered regardless|
|exec_scripting_via_html_app |Works |Execution error|||
|exec_shortcut_embedded_obj | Does not work |File not found error|||
|exec_sliver_posh | Works ||||
|exec_sqlserver_suspicious_child | Works ||||
|exec_susp_explorer | Works |Execution error||Detection rules are triggered regardless|
|exec_susp_msiexec |Works |Execution error||Detection rules are triggered regardless|
|exec_susp_parent_child | Works |Permission error|||
|exec_svchost_child_schedule | Works ||||
|exec_unusual_directory | Works ||||
|exec_unusual_path_msmpeng | Works |Permission error|||
|exec_vs_prebuildevent | Works ||||
|exec_vsls_agent | Works ||||
|exec_winword_susp_parent | Works|||| 
|execution_iso_dll_rundll32 | Works ||||
|execution_iso_dll_sideload | Works ||||
|execution_pubprn |Works ||||
|extexport_sideload | Works |Execution error||Detection rules are triggered regardless|
|file_ads_creation | Works ||||
|file_create_dpapi_key | Works ||||
|file_create_exchange_um | Works ||||
|file_create_exec_pdf_reader | Works |Permission error while trying to remove the file|||
|file_create_lsass_dump | Works ||||
|file_create_mimilsa_log | Works ||||
|file_create_ms_addins | Works ||||
|file_create_mstsc_startup| Works ||||
|file_create_outlook_vba | Works ||||
|file_create_powershell_profile | Works ||||
|file_create_scripting_startup | Works ||||
|file_create_smss_exec | Works ||||
|file_create_task_file | Works ||||
|file_create_vbs_startup | Works ||||
|file_creation_teamviewer | Works |||| 
|file_delete_spool_driver | Works ||||
|file_delete_vbk | Works |||| 
|file_double_extension | Works |Execution error||Detection rules are triggered regardless|
|file_exe_ususual_extension | Works |Execution error||Detection rules are triggered regardless|
|file_html_smuggling | Works |Execution error||Detection rules are triggered regardless|
|file_ms_template_macros | Works ||||
|file_potential_dll_hijacking | Does not work |File not found error|||
|file_script_startup_folder | Works ||||
|file_susp_browser_extension | Works ||||
|findstr_pw_search | Does not work |File not found error|||
|firewall_allowlist_modif_unsigned |Works |Execution error||Detection rules are triggered regardless|
|fltmc_unload | Works |OS Error|||
|git_creds_access | Works ||||
|globalflags | Works ||||
|hosts_file_modify | Works ||||
|html_help_file_written_exec | Works |Execution error||Detection rules are triggered regardless|
|image_load_dnguard | Works ||||
|image_load_msbuild_vaultcli | Works ||||
|image_load_phantomdll | Works ||||
|image_load_rdp_client_dll | Works ||||
|image_load_script_interpreter_wmiutils | Works |Permission error while trying to remove the file|||
|image_load_taskhost | Works ||||
|image_load_vaultcli | Works ||||
|impersonate_trusted_installer | Works ||||
|inhibit_system_recovery | Works ||||
|inhibit_system_recovery_and_rename | Works |Execution error||Detection rules are triggered regardless|
|inhibit_system_recovery_cmd |Works|Execution error||Detection rules are triggered regardless|
|inhibit_system_recovery_lolbas_child | Works |Permission error|||
|inhibit_system_recovery_office | Works |Execution error||Detection rules are triggered regardless|
|inhibit_system_recovery_renamed | Works |Permission error|||
|installutil_network | Works ||||
|ip_discovery_unsigned | Works ||||
|iqy_file_writes | Works ||||
|kerberos_netconn_file_creation | Works ||||
|lateral_command_psexec | Does not work |File not found error|||
|lateral_commands | Works |Execution error||Detection rules are triggered regardless|
|lua_image_load | Works ||||
|mimikatz_cmdline | Works ||||
|modification_of_wdigest_security_provider | Works ||||
|modify_bootconf | Works ||||
|ms_office_drop_exe | Works ||||
|ms_office_task_creation | Works ||||
|msbuild_network | Works ||||
|msbuild_unusual_args | Works |Execution error||Detection rules are triggered regardless|
|msequationeditor_file_written_exec | Works |Execution error||Detection rules are triggered regardless|
|msequationeditor_net_conn | Works ||||
|mshta_network | Works ||||
|msiexec_http_installer | Works ||||
|msiexec_remote_msi | Works |Execution error|||
|msiexec_remote_msi_install | Works ||||
|msoffice_addins_file | Works ||||
|msoffice_dcom_accessvbom | Works ||||
|msoffice_descendant_reg_mod_persistence | Works ||||
|msoffice_dll_image_load | Works ||||
|msoffice_file_dll_sideload | Works ||||
|msoffice_file_drop_exec_wmi | Works |Execution error||Detection rules are triggered regardless|
|msoffice_file_exec_script_interpreter | Works ||||
|msoffice_onenote_susp_child | Works ||||
|msoffice_potential_proc_inj | Works ||||
|msoffice_reg_mod | Works ||||
|msoffice_signed_binary_spawn | Works |OS Error||Detection rules are triggered regardless|
|msoffice_startup_persistence | Works ||||
|msoffice_untrusted_exec | Works ||||
|msoffice_wmi_imageload | Works ||||
|msxsl_image_load | Works ||||
|msxsl_network | Does not work |File not found error|||
|net_user_add | Works | Execution error ||Detection rules are triggered regardless|
|network_connection_desktopimgdownldr | Works ||||
|network_connection_download_powershell | Works ||||
|network_connection_download_script_interpreter | Works ||||
|network_connection_external_ip_lookup_non_browser | Works ||||
|network_connection_freesslcert | Works ||||
|network_connection_iexplore_rundll32 | Works ||||
|network_connection_kerberos_port | Works ||||
|network_connection_nslookup | Works ||||
|network_connection_process_unusual_args | Works ||||
|network_connection_rdp_tunneling | Works ||||
|network_connection_unusual_rundll32 | Works ||||
|obfuscated_cmd_commands | Works |Execution error||Detection rules are triggered regardless|
|obfuscated_powershell | Works |Execution error||Detection rules are triggered regardless|
|office_application_startup | Works ||||
|outlook_suspicious_child | Works |Permission error|||
|persistence_startup_unusual_process | Works |Execution error||Detection rules are triggered regardless|
|persistent_scripts | Works |Execution error||Detection rules are triggered regardless|
|ping_delayed_exec | Works ||||
|port_monitor | Works ||||
|powershell_args | Works ||||
|powershell_base64_gzip | Does not work? | Closes the power shell window|||
|powershell_delete_shadow_copy | Works ||||
|powershell_from_script | Works ||||
|powershell_unsigned_defender_exclusion | Works |Execution error||Detection rules are triggered regardless|
|powershell_vault_access | Does not work |File not found error|||
|process_double_extension | Does not work ||||
|process_extension_anomalies | Does not work |File not found error|||
|process_name_masquerade | Works |Permission error while trying to remove the file|||
|ransomnote_delete_shadows | Works |Execution error||Detection rules are triggered regardless|
|recycle_bin_process | Works ||||
|reg_creation_servicedll | Works ||||
|reg_dll_control_panel | Works ||||
|reg_mod_amsienable | Works ||||
|reg_mod_appcertdlls | Works ||||
|reg_mod_appinitdlls | Works ||||
|reg_mod_autodialdll | Works ||||
|reg_mod_base64_executable | Works ||||
|reg_mod_builtindnsclientenabled | Works ||||
|reg_mod_disable_uac | Works ||||
|reg_mod_disableantispyware | Works ||||
|reg_mod_driver_blocklist | Works ||||
|reg_mod_enableat | Works ||||
|reg_mod_enablescriptblocklogging | Works ||||
|reg_mod_ifeo | Works ||||
|reg_mod_lsa_ssp | Works ||||
|reg_mod_netwire | Works ||||
|reg_mod_networkprovider | Works ||||
|reg_mod_nullsessionpipes | Works ||||
|reg_mod_plugx| works ||||
|reg_mod_point_and_print_dll | Works ||||
|reg_mod_port_forwarding | Works |||| 
|reg_mod_print_processors | Works ||||
|reg_mod_remcos | Works ||||
|reg_mod_run_key_unusual_proc | Works ||||
|reg_mod_shadow_rdp | Works ||||
|reg_mod_shim_sb | Works ||||
|reg_mod_startup_shell_folder | Works ||||
|reg_mod_suspicious_service | Works ||||
|reg_mod_systemcertificates | Works ||||
|reg_mod_time_provider | Works |||| 
|reg_mod_unusual_startup_folder | Works ||||
|reg_mod_windir | Works ||||
|reg_run_key_asterisk | Works ||||
|reg_vss_service_disable | Works ||||
|registry_hive_export | Works ||||
|registry_persistence_create | Works ||||
|registry_rdp_enable | Works ||||
|regsvr32_scrobj | Works ||||
|regsvr32_unusual_args | Works ||||
|renamed_autoit | Works |Permission error while trying to remove the file|||
|renamed_automaton_interpreter | Works |Permission error while trying to remove the file|||
|rubeus_alike_commandline | Works ||||
|rundll32_inf | Works ||||
|rundll32_inf_callback | Works ||||
|rundll32_javascript_callback | Works ||||
|rundll32_unusual_args | Works ||||
|rundll32_unusual_dll_extension | Works ||||
|schtask_escalation | Works |Execution error||Detection rules are triggered regardless|
|schtasks_xml_masqueraded | Works |Execution error||Detection rules are triggered regardless|
|scrobj_com_hijack | Works ||||
|secure_file_deletion | Does not work |File not found error|||
|sensitive_file_access | Works ||||
|settingcontentms_files | Works ||||
|sevenzip_encrypted | Does not work |File not found error|||
|shellcode_load_ws2_32_unbacked | Works ||||
|shellcode_winexec_calc | Works |Execution error||Detection rules are triggered regardless|
|shortcut_file_suspicious_process | Works ||||
|signed_proxy_file_written_exec | Works |Permission error while trying to remove the file|||
|silentprocessexit_lsass | Works ||||
|sip_provider | Does not work |File not found error the following files are missing sigcheck64.exe / sigcheck32.exe, TrustProvider64.dll / TrustProvider32.dll|||
|smb_connection | Works ||||
|solarmaker_backdoor | Works ||||
|sticky_keys_write_execute |Works |Execution error during clean up|ignore_failures=True||
|susp_control_panel_dll_explorer | Functional after fixing|Syntax error|change renamer = _common.get_resource_path("binrcedit-x64.exe") to renamer = _common.get_resource_path("bin/rcedit-x64.exe")||
|susp_scheduled_task_creation | Works |Execution error||Detection rules are triggered regardless|
|susp_script_file_name | Works ||||
|suspicious_bits_job_notify |Works|Permission while trying to remove file|||
|suspicious_child_acrobat | Works |Execution error||Detection rules are triggered regardless|
|suspicious_child_childless_process | Works |Execution error||Detection rules are triggered regardless|
|suspicious_child_compattelrunner | Works ||||
|suspicious_child_dns | Works ||||
|suspicious_child_exchange_um | Works ||||
|suspicious_child_explorer | Works ||||
|suspicious_child_services | Works |Execution error|| Detection rules are triggered regardless|
|suspicious_child_solarwinds_businesslayerhost | Works ||||
|suspicious_child_solarwindsdiagnostics | Works ||||
|suspicious_child_svchost_sch | Works |Execution error||Detection rules are triggered regardless|
|suspicious_child_wmiprvse | Works |Execution error||Detection rules are triggered regardless|
|suspicious_child_zoom | Works |Permission error while removing file|||
|suspicious_dll_registration_regsvr32 | Works |Execution error||Detection rules are triggered regardless|
|suspicious_lineage_script |Works |Permission error while trying to remove the file|||
|suspicious_msiexec_child | Works ||||
|suspicious_office_child | Works ||||
|suspicious_office_children | Works |Execution Error ||
|suspicious_office_descendant_fp | Works ||change ["%s /c %s /c %s" % (office_path, browser_path, command)] to a list "%s /c %s /c %s" % (office_path, browser_path, command)|Execution failure is expected|
|suspicious_parent_cmd | Works ||||
|suspicious_parent_csc |Works |  Permission error while trying to remove the file |||
|suspicious_parent_msbuild_explorer| Works |Permission error while trying to remove the file|||
|suspicious_parent_msbuild_office | Works |Permission error while trying to remove the file|||
|suspicious_parent_msbuild_script | Works |Permission error while trying to remove the file|||
|suspicious_parent_sc | Works ||||
|suspicious_parent_smss | Works |Permission error while trying to remove the file|||
|suspicious_powershell_download | Does not work |Execution error: missing bad.ps1 file in the rta folder |||
|suspicious_wmic_script | Works ||||
|suspicious_wscript_parent | Works ||||
|system_restore_process | Does not work |PsExec file missing from bin|||
|trust_provider | Does not work |File not found error: sigcheck32 and sigcheck64 are missing from bin|||
|uac_cdssync| Works ||||
|uac_clipup | Works ||||
|uac_computerdefaults | Works ||||
|uac_dccw | Works ||||
|uac_diskcleanup | Works ||||
|uac_dism_dll_side_loading | Works ||||
|uac_eventviewer | Works ||||
|uac_eventvwr | Works ||||
|uac_fodhelper | Works ||||
|uac_icmluautil | Works ||||
|uac_mmc_deserialization | Works ||||
|uac_mmc_hijack |Works ||||
|uac_mmc_net_core_profiler | Works ||||
|uac_sdclt | Works ||||
|uac_sysprep | Works ||||
|uac_windir_masq | Works||||
|uac_windows_activation | Works ||||
|uac_winfw_mmc | Works ||||
|uac_wow64log | Works ||||
|uac_wsreset | Works ||||
|uncommon_persistence | Works ||||
|unsigned_startup_item_netconn | Works ||||
|unusual_kerberos_client | Works ||||
|unusual_ms_tool_network | Works ||||
|unusual_parent_child | Works ||||
|unusual_parent_chrome_extension | Works |Permission Error while trying to remove the file|||
|unusual_powershell_engine_image_load | Works||||
|unusual_rdp_client | Works ||||
|user_dir_escalation | Does not work |PsExec file missing from bin|||
|user_mode_smb_connection | Works ||||
|vaultcmd_commands | Works ||||
|webservice_lolbas |Works ||||
|webservice_unsigned | Works ||||
|werfault_masquerading |Works ||||
|werfault_persistence | Functional after fixing |Cannot repeatedly run the RTA, the error occurs because New-ItemProperty cannot create a registry property that already exists, so it fails if you try to add the same property again.|Change "New-ItemProperty" to "Set-ItemProperty"||
|wevtutil_log_clear |Works ||||
|windefend_svc_stop | Works? |Requires System privileges to run|||
|windows_script_host_file_written_exec | Works |PermissionError |||
|winrar_encrypted | Does not work |Missing Rar.exe file |||
|winrar_startup_folder | Works ||||
|wmi_incoming_logon | Works? ||Need to provide remote host||
|wmic_xsl_exec | Works ||||
|wuauclt_image_load | Works ||||



