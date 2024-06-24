---
title: Challenge 2:Case Of Client Side Code (Part 1)
date: 2024-05-14 05:01:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

## Challenge/Task

1. Stored Cross-Site Scripting (XSS)
2. DOM-Based Cross-Site Scripting (XSS)

## 1. First XSS - Stored Cross-Site Scripting (XSS)

**Proof-of-Concept (POC):**

From previous challenge, we have found a website which is https://the-abstractors.com/. After further checking, there is a signup page that everyone can register as a user.

![The Abstractors website](/img/cwae/theabstractor.png){: w="400" h="400" }{: .normal }
![The Abstractors sign-up page](/img/cwae/signup.png){: w="400" h="400" }{: .normal }

As a tester, we will play around all the functions/button/form inside the website. There is a page where we can write a post. I try to inject xss script inside all the field. In the Title field, the payload can’t be executed because the <> tag is been encoded
![The Abstractors post page](/img/cwae/xss1.png){: w="400" h="400" }{: .normal }
![POC-xss](/img/cwae/xss2.png){: w="400" h="400" }{: .normal }

However, in content, seems we can execute the payload but the alert is not pop-up. 
![POC-xss](/img/cwae/xss3.png){: w="400" h="400" }{: .normal }
![POC-xss](/img/cwae/xss4.png){: w="400" h="400" }{: .normal }

Upon checking the source code, there is a line to avoid the system from execute the alert and prompt function. 

![POC-xss](/img/cwae/xss5.png){: w="400" h="400" }{: .normal }

We try to use other function which is confirm function.
![POC-xss](/img/cwae/xss6.png){: w="400" h="400" }{: .normal }

XSS found using confirm function.

![POC-xss](/img/cwae/xss7.png){: w="400" h="400" }{: .normal }

## 2. Second XSS – DOM Based XSS

Upon checking the chat function, my message is not sent to the server. 
![POC-xss](/img/cwae/xss8.png){: w="400" h="400" }{: .normal }

Looking at the source code, it just a DOM function.
![POC-xss](/img/cwae/xss9.png){: w="400" h="400" }{: .normal }

Send the XSS payload and it popup the confirm ().
![POC-xss](/img/cwae/xss10.png){: w="400" h="400" }{: .normal }

So from the vulnerabilities, I can craft a link that contains xss payload and send it to everyone.
![POC-xss](/img/cwae/xss11.png){: w="400" h="400" }{: .normal }