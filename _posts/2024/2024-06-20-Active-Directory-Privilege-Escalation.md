---
title: Active Directory (AD) | Privilege Escalation 101
date: 2024-06-25 7:40:00 +0800
categories: [Cert, CRTP]
tags: [Cert, Active Directory]     ## TAG names should always be lowercase
render_with_liquid: false
render_with_liquid: false
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques
---

## Scenarios Leading to Privilege Escalation
 
 - Hunting for Local Admin Access: Gain local admin access on other machines.
 - Hunting for High Privilege Domain Accounts: Target accounts like Domain Administrator.
 - Example Scenario : 
    1. I have local admin access on a machine
    2. A Domain Admin has a session on that machine
    3. I steal his token and impersonate him
    4. Profit!

## Methods of Privilege Escalation
 
 - Missing Patches
 - Automated Deployment and AutoLogon password in clear text
 - AlwaysInstallElevated (Any user can run MSI on SYSTEM)
 - Misconfigured Services
 - DLL Hijacking and more
 - NTLM Relaying a.k.a Wont’t Fix
 - NTLM Relaying example - https://github.com/antonioCoco/RemotePotato0

## Essential Tools for Complete Coverage
 
 - https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1
 - https://github.com/enjoiz/Privesc
 - https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASexe/README.md - Very NOISY