---
title: Challenge 5:Altering the Unalterable
date: 2024-05-14 05:05:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

# Challenge/Task

1. Altering the Unalterable

**Proof-of-Concept (POC):**

I discover a feature/endpoint/parameter which is not supposed to be modified or changed by the user. It is an email. 

![POC-otb](/img/cwae/atu1.png){: w="400" h="400" }{: .normal }

To proof it, I send a POST method to update my profile. Referring to the screenshot below, no parameter referring to an email. That’s mean, user cant change the email 
![POC-otb](/img/cwae/atu2.png){: w="400" h="400" }{: .normal }

Looking at the source code, the id and name for the Email is “email”. I suspect that is the parameter
![POC-otb](/img/cwae/atu3.png){: w="400" h="400" }{: .normal }

We try again sending the same request. But this time, I include additional parameter “email” and assign any value to that. The response from server showing no error
![POC-otb](/img/cwae/atu4.png){: w="400" h="400" }{: .normal }

I go to my profile and find that my email address is changed. So here we can alter the unalterable parameters.

![POC-otb](/img/cwae/atu5.png){: w="400" h="400" }{: .normal }