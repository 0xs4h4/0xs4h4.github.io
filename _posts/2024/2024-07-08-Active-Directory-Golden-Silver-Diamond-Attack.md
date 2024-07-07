---
title: Active Directory (AD) | Understanding Golden, Silver and Diamond Ticket Attack
date: 2024-07-08 00:00:00 +0800
categories: [Cert, CRTP]
tags: [Cert,Active Directory]     ## TAG names should always be lowercase
render_with_liquid: false
description: Learn about Golden, Silver, and Diamond Ticket attacks in Active Directory (AD) and how they exploit vulnerabilities.
---

## **Introduction**
Active Directory (AD) is a critical component in many enterprise environments, serving as the backbone for identity and access management. As such, it is a prime target for attackers looking to gain persistent and expansive access within a network. Among the sophisticated attack techniques leveraged against AD are Golden Ticket, Silver Ticket, and Diamond Ticket attacks. These methods exploit weaknesses in Kerberos and AD Certificate Services, granting attackers significant control and access.

## **Golden Ticket Attack**
### What is It?
A Golden Ticket attack involves forging a Kerberos Ticket Granting Ticket (TGT) using the password hash of the KRBTGT account, which is the account responsible for encrypting all Kerberos tickets in the domain. With this forged TGT, an attacker can generate service tickets for any service within the domain, effectively gaining unlimited access.

### Impact
This attack provides complete control over the AD environment, with persistent access even after password changes, and is difficult to detect and mitigate due to the legitimate nature of the Kerberos tickets.

## **Silver Ticket Attack**
### What is It?
A Silver Ticket attack entails forging a Kerberos Ticket Granting Service (TGS) ticket using the password hash of a specific service account. This type of attack targets particular services (e.g., SQL, HTTP) rather than the entire domain, allowing the attacker to authenticate to those services without involving the Domain Controller.

### Impact
Access is limited to specific services without involving the Domain Controller, enabling potential lateral movement within the network by exploiting specific services, and is harder to detect compared to Golden Ticket attacks.

## **Diamond Ticket Attack**
### What is It?
A Diamond Ticket attack is a newer concept that involves manipulating certificates in an Active Directory Certificate Services (AD CS) environment. By forging a valid certificate for a user or service, an attacker can authenticate to AD CS, bypassing traditional Kerberos authentication mechanisms.

### Impact
This attack provides persistent and stealthy access to resources authenticated via certificates, bypasses traditional Kerberos authentication monitoring, and has the potential for widespread exploitation depending on the CA’s scope.

## **Differences**
- Scope of Attack:
Golden Ticket: Targets the entire AD domain, allowing for domain-wide access.
Silver Ticket: Targets specific services, limiting access to those services only.
Diamond Ticket: Focuses on manipulating certificates for authentication, potentially affecting any resource authenticated via AD CS.

- Prerequisites:
Golden Ticket: Requires access to the KRBTGT account's password hash.
Silver Ticket: Requires access to the password hash of a specific service account.
Diamond Ticket: Requires access to an enterprise CA or sufficient privileges to issue certificates.

- Detection and Mitigation:
Golden Ticket: Difficult to detect due to legitimate-looking Kerberos tickets; mitigated by rotating KRBTGT account passwords and monitoring TGT lifetimes.
Silver Ticket: Harder to detect as it does not involve the Domain Controller; mitigated by strong service account password policies and monitoring service ticket requests.
Diamond Ticket: Bypasses traditional Kerberos monitoring; mitigated by securing AD CS environments, auditing certificate issuance, and controlling certificate template permissions.