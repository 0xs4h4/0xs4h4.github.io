---
title: Challenge 2:Case Of Client Side Code
date: 2024-05-14 05:00:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

## Challenge/Task :

- Stored Cross-Site Scripting (XSS)
- DOM-Based Cross-Site Scripting (XSS)

### 1. Subdomain Enumeration

Tools Used: Sublist3r, Netlas

### a. Netlas
![Netlas Scanning Result](/img/cwae/xss1.png)
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
