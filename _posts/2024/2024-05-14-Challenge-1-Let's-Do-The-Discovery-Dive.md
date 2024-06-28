---
title: Challenge 1:Let's Do The Discovery Dive.
date: 2024-05-14 05:00:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     ## TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

## Challenge/Task

1. Enumerate the domain : **the-abstractors.com**
2. Find all subdomain
3. Find Open Ports/Service for all live subdomain

## 1. Subdomain Enumeration

**Tools Used: Sublist3r, Netlas**

**Netlas :** Attack Surface tool . Can be used to find subdomain
![Netlas Scanning Result](/img/cwae/netlas.png){: w="400" h="400" }{: .normal }

**Sublist3r :** Open Source tool. Can find here on the [github.](https://github.com/aboul3la/Sublist3r) 
![Sublist3r Scanning Result](/img/cwae/sublist3r.png){: w="400" h="400" }{: .normal }

**Findings:**

- `health.the-abstractors.com`
- `the-abstractors.com`

## 2. Open Ports for Each Accessible Host

Tool Used: PortWhisper. My own custom tool. Can find on my [github](https://github.com/beardenx/PortWhisper)
![Port Scanning Result](/img/cwae/portscan.png){: w="400" h="400" }{: .normal }

- **`health.the-abstractors.com`**
    - Open Ports: [22,8080,9091]
- **`the-abstractors.com`**
    - Open Ports: [25,80,443]

## 3. Technology Used
![Technology Scanning Result](/img/cwae/wappalyzer.png){: w="400" h="400" }{: .normal }