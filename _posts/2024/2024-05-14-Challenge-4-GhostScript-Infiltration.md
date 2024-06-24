---
title: Challenge 4:GhostScript Infiltration
date: 2024-05-14 05:04:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

### Challenge/Task

1. Out of The Box

**Proof-of-Concept (POC):**

There are multiple places in this system have an upload function. For example in this page, we can upload our profile image.  

![POC-otb](/img/cwae/otb1.png){: w="400" h="400" }{: .normal }
![POC-otb](/img/cwae/otb2.png){: w="400" h="400" }{: .normal }

https://dp-abstractors.s3.amazonaws.com/edc3535b19bbae04.png
Upon checking the address, it is an S3 bucket hosting all the profile images of the users. 
![POC-otb](/img/cwae/otb3.png){: w="400" h="400" }{: .normal }

Previously in another challenge we already performed a subdomain scanning. I found out there is a subdomain which refers to the S3 which is s3.the-abstractors.com. Upon digging, the cname record is referring to internal-s3-abstractors-data.s3.amazonaws.com
![POC-otb](/img/cwae/otb4.png){: w="400" h="400" }{: .normal }

I visited the domain and found out few interesting files including details.rtf. 

![POC-otb](/img/cwae/otb5.png){: w="400" h="400" }{: .normal }

I visited the domain and found out few interesting files including details.rtf.
![POC-otb](/img/cwae/otb6.png){: w="400" h="400" }{: .normal }

Using cat command, I read the file and found an interesting credential which may belongs to dev team,

![POC-otb](/img/cwae/otb7.png){: w="400" h="400" }{: .normal }

I use the credential and try to login to the system using the credential. 
![POC-xss](/img/cwae/otb8.png){: w="400" h="400" }{: .normal }