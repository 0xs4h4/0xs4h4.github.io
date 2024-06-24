---
title: Challenge 7:The Return of Sceindamus Clavem
date: 2024-05-14 05:07:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

## Challenge/Task

1. The Return of Sceindamus Clavem

**Proof-of-Concept (POC):**

Most of modern applications have an API to communicate between backend and frontend. Same as the testing website here. I try to login into the API using the same format as login to the application. After successfully login, the servers gave me a JWT token. 

![POC-otb](/img/cwae/rsc1.png){: w="400" h="400" }{: .normal }

Further checking, the payload data inside the JWT contains username and expiration time.

![POC-otb](/img/cwae/rsc2.png){: w="400" h="400" }{: .normal }

Using the JWT token given, I try to connect to another endpoint within the API which is /account. The server response me a details of my account.

![POC-otb](/img/cwae/rsc3.png){: w="400" h="400" }{: .normal }

What if we change the username in the payload earlier to someone else? Is it the system checking the authentication of the JWT?

![POC-otb](/img/cwae/rsc4.png){: w="400" h="400" }{: .normal }

Unfortunately, yes. The token is invalid.

![POC-otb](/img/cwae/rsc5.png){: w="400" h="400" }{: .normal }

Checking other tools online, I try to find the secret key. I’m using JWT_Tool to crack the key. We found the key which is @b5tr4ct0r

![POC-otb](/img/cwae/rsc6.png){: w="400" h="400" }{: .normal }

I try again to get the new JWT token by using the secret key just now. 

![POC-otb](/img/cwae/rsc7.png){: w="400" h="400" }{: .normal }

Fortunately, the server replied to me the details of the others account that belonged to the username

![POC-otb](/img/cwae/rsc8.png){: w="400" h="400" }{: .normal }

Using the UUID inside the account details, I managed to find the card details of the user.

![POC-otb](/img/cwae/rsc9.png){: w="400" h="400" }{: .normal }