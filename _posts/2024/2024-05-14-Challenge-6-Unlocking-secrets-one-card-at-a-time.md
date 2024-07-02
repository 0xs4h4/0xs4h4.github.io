---
title: Challenge 6:Unlocking secrets, one card at a time
date: 2024-05-14 05:06:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     ## TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

## Challenge/Task

1. Unlocking secrets, one card at a time

**Proof-of-Concept (POC):**

## 1. Finding a Unique ID across the Web (Failed). ## 

This is my post on private page. ID for the post is 459.
![POC-otb](/img/cwae/uts1.png){: w="400" h="400" }{: .normal }

I try to change to other ID which is 453 to view other private post. But it is forbidden.
![POC-otb](/img/cwae/uts2.png){: w="400" h="400" }{: .normal }

## 2. Finding a Unique ID across the Web (Success) ## 

I try to crawl and discovering all the functionalities and web content inside the web and found out another UUID inside a PRO page.

![POC-otb](/img/cwae/uts3.png){: w="400" h="400" }{: .normal }

The request and response when I view my card details:
![POC-otb](/img/cwae/uts4.png){: w="400" h="400" }{: .normal }

UUID usually highly unlikely to be guessed. UUIDs are typically generated using algorithms that consider factors such as timestamp, network address, and random numbers, ensuring a very low probability of collision. To proof that the UUID can be used to query the other card details, I create a new account and put a new card. I copy the new UUID then put again in my previous request. Fortunately, it queries my second card details. This is vulnerable but somehow not exploitable since the UUID cannot be guess. 

![POC-otb](/img/cwae/uts5.png){: w="400" h="400" }{: .normal }

I notice the endpoint to view the card details is “/get_cc”. 
![POC-otb](/img/cwae/uts6.png){: w="400" h="400" }{: .normal }

Trying to change the method to GET (maybe can retrieve any data), however the method is not allowed to use in this endpoint 

![POC-otb](/img/cwae/uts7.png){: w="400" h="400" }{: .normal }

Last option is, I try to fire my burp and send the request to Intruder. I try to brute force the endpoint to find any interesting endpoint.

![POC-otb](/img/cwae/uts8.png){: w="400" h="400" }{: .normal }

I found one interesting endpoint which is “get_all_cc”.
![POC-otb](/img/cwae/uts9.png){: w="400" h="400" }{: .normal }

Upon checking, the endpoint listed all the UUIDs inside the website
![POC-otb](/img/cwae/uts10.png){: w="400" h="400" }{: .normal }

I copy one of the UUIDs and send it through “get_cc” endpoint and fortunately I can view the card details. 

![POC-otb](/img/cwae/uts11.png){: w="400" h="400" }{: .normal }
![POC-otb](/img/cwae/uts12.png){: w="400" h="400" }{: .normal }