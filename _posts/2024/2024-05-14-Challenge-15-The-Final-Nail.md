---
title: Challenge 15:The Final Nail
date: 2024-05-14 05:00:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

### Challenge/Task

1. Insecure deserialization

**Proof-of-Concept (POC):**

Serialization is the process of converting data structures or objects into a format that can be easily stored, transmitted, or reconstructed later. Deserialization is the process of reconstructing serialized data back into its original data structures or objects.
Deserialization can be risky if the data being deserialized is not properly validated and sanitized. This can lead to deserialization vulnerabilities, where attackers can manipulate serialized data to execute arbitrary code or perform other malicious actions on the application or system.

When we browsing to our own user’s page, we can saw a download button. This feature will download all our post on the website. 

![POC-otb](/img/cwae/tfn1.png){: w="400" h="400" }{: .normal }

You may notice the URL have some strings. It is a Serialization process on that string. 
![POC-otb](/img/cwae/tfn2.png){: w="400" h="400" }{: .normal }
![POC-otb](/img/cwae/tfn3.png){: w="400" h="400" }{: .normal }

If we change the string or add new string between that, it will give us an error. You can see below it mentioned Pickle Deserialization error. So that’s mean the system using Pickle for Deserialization. Upon google, we found a tool to help us to hide our RCE payload, then serialize it and send back to the system. If the payload is good, we can get reverse shell.  

![POC-otb](/img/cwae/tfn4.png){: w="400" h="400" }{: .normal }

We just have to change the IP and Port of our machine to get the reverse shell.

![POC-otb](/img/cwae/tfn5.png){: w="400" h="400" }{: .normal }

After we got the serialize strings, we send back to the server, and we got a reverse shell!!

![POC-otb](/img/cwae/tfn6.png){: w="400" h="400" }{: .normal }
![POC-otb](/img/cwae/tfn7.png){: w="400" h="400" }{: .normal }

To prove we have land to the server, I have run the id and whoami command. The result showing us that the server belongs to the victim.

![POC-otb](/img/cwae/tfn8.png){: w="400" h="400" }{: .normal }