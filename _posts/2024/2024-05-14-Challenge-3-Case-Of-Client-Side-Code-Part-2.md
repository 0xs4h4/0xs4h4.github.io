---
title: Challenge 3:Case Of Client Side Code (Part 2)
date: 2024-05-14 05:03:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

### Challenge/Task

1. Blind Cross-Site Scripting (XSS)

### 1. Third XSS - Blind Cross-Site Scripting (XSS)

**Proof-of-Concept (POC):**

Blind XSS is an attack that we cannot see the payload executed. Referring to the page below, there is a feedback field that we need to send when we want to remove the credit card. 

![POC-xss](/img/cwae/xss12.png){: w="400" h="400" }{: .normal }

We want to prove the XSS using **XSS hunter**. We want to know if the payload is executed. 
![POC-xss](/img/cwae/xss13.png){: w="400" h="400" }{: .normal }

We send the payload and click okey. By default, our payload will be sent to the server.
![POC-xss](/img/cwae/xss14.png){: w="400" h="400" }{: .normal }

To see if our payload is successful or not, go to xss payload fires. The XSS is executed and appear in the console. 

![POC-xss](/img/cwae/xss15.png){: w="400" h="400" }{: .normal }
![POC-xss](/img/cwae/xss16.png){: w="400" h="400" }{: .normal }