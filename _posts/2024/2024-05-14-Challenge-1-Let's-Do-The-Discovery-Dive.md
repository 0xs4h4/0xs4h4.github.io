---
title: Challenge 1:Let's Do The Discovery Dive.
date: 2024-05-14 05:00:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

## Challenge/Task :

- Enumerate the domain
- Find all subdomain
- Find open ports/services for all live subdomain

### 1. Subdomain Enumeration

Tools Used: Sublist3r, Netlas

### a. Netlas
![Netlas Scanning Result](/img/cwae/netlas.png)
### b. Sublist3r
![Sublist3r Scanning Result](/img/cwae/sublist3r.png)

**Findings:**

- `health.the-abstractors.com`
- `the-abstractors.com`

### 2. Open Ports for Each Accessible Host

Tool Used: Custom Tool
![Port Scanning Result](/img/cwae/portscan.png)


- **`health.the-abstractors.com`**
    - Open Ports: [22,8080,9091]
- **`the-abstractors.com`**
    - Open Ports: [25,80,443]

### 3. Technology Used
![Technology Scanning Result](/img/cwae/wappalyzer.png)
