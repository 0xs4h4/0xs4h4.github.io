---
title: Challenge 10:ShadowFetch
date: 2024-05-15 05:02:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

### Challenge/Task

1. ShadowFetch

**Proof-of-Concept (POC):**

There is a function that allows you to upload images through inserting any URLs 

![POC-otb](/img/cwae/sf1.png){: w="400" h="400" }{: .normal }
![POC-otb](/img/cwae/sf2.png){: w="400" h="400" }{: .normal }

Once you send the URLs to the server, the server will try to download the images and upload them to the S3 bucket. It will renamed the file with encoded filename with png extension.

![POC-otb](/img/cwae/sf3.png){: w="400" h="400" }{: .normal }

I try to validate the link that the server provided, it eventually download back the image from the link that I put earlier. 

![POC-otb](/img/cwae/sf4.png){: w="400" h="400" }{: .normal }

What if I put malicious link that will download the malicious file from my server? Yes, it is also downloaded the file.

![POC-otb](/img/cwae/sf5.png){: w="400" h="400" }{: .normal }

Since it allows any URL to input here, we will try to use file:///. The file:/// URI scheme is used to reference files on the local system. At first attempt I try to locate sshd_config. The server then downloads the sshd_config. It gave me the S3 bucket URL.

![POC-otb](/img/cwae/sf6.png){: w="400" h="400" }{: .normal }

I downloaded the image with PNG extension format. 

![POC-otb](/img/cwae/sf7.png){: w="400" h="400" }{: .normal }

Using cat command, I can read the file contains all the sshd_config.

![POC-otb](/img/cwae/sf8.png){: w="400" h="400" }{: .normal }

Using the same method, I want to view the users listed inside the server using /etc/passwd payload.

![POC-otb](/img/cwae/sf9.png){: w="400" h="400" }{: .normal }

Now, I can know all the users listed inside the server. 

![POC-otb](/img/cwae/sf10.png){: w="400" h="400" }{: .normal }