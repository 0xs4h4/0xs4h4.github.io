---
title: Active Directory (AD) | Understanding Golden, Silver and Diamond Ticket Attacks
date: 2024-07-08 00:00:00 +0800
categories: [Cert, CRTP]
tags: [cert, active directory]
render_with_liquid: false
description: Explore vulnerabilities in Active Directory (AD) with Golden, Silver, and Diamond Ticket attacks, revealing how they grant persistent access to attackers.
---

## **Introduction**
Active Directory (AD) is crucial for managing access in enterprises, making it a prime target for cyber attacks like Golden, Silver, and Diamond Ticket exploits. These techniques exploit AD's weaknesses in Kerberos and certificate services.

## **Golden Ticket Attack**
A Golden Ticket attack crafts a Kerberos Ticket Granting Ticket (TGT) using the KRBTGT account's password hash, granting unrestricted access across the domain. This method guarantees full control over AD, even after password changes, and is hard to detect due to its authentic-looking tickets.

## **Silver Ticket Attack**
A Silver Ticket attack forges a Kerberos Ticket Granting Service (TGS) ticket with a service account's password hash, accessing specific services discreetly. This method allows lateral movement within the network by exploiting targeted services, evading detection compared to broader attacks.

## **Diamond Ticket Attack**
A Diamond Ticket attack manipulates AD Certificate Services (AD CS) certificates, creating valid credentials to bypass Kerberos authentication. This technique provides persistent access to resources authenticated via certificates, bypassing traditional monitoring and potentially leading to widespread compromise.
