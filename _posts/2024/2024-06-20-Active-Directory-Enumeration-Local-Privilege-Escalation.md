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
 - I have local admin access on a machine - A Domain Admin has a session on that machine - I steal his token and impersonate him - Profit!

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

# 2. Local Privilege Escalation by using PowerUp

Check for any priviliege escalation path
{: .prompt-info }
```bash
Invoke-AllChecks
```