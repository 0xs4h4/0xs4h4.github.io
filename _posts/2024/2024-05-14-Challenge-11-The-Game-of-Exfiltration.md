---
title: Challenge 11:The Game of Exfiltration
date: 2024-05-15 05:02:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

# Challenge/Task

1. XXE INJECTION

**Proof-of-Concept (POC):**

There is a function inside the website that allow users to share their post. We can insert any short URL to share it. 

![POC-otb](/img/cwae/goe1.png){: w="400" h="400" }{: .normal }

Upon checking the endpoint, it allowed XML request in the request. If it allowed XML, then there are some security issues with XML configuration. 

![POC-otb](/img/cwae/goe2.png){: w="400" h="400" }{: .normal }

I try to prove the server accept the XML using Content Type Converter tool inside burp suite. 

![POC-otb](/img/cwae/goe3.png){: w="400" h="400" }{: .normal }

After changing to XML format, the Json format earlier for URL is changing to <url></url>. I send again the request and saw the response from servers using the XML format too. So that’s mean the server accept XML type.

![POC-otb](/img/cwae/goe4.png){: w="400" h="400" }{: .normal }

We try to perform XXE (XML External Entity) injection attack. Below is  the payload. 

```jsx
<!DOCTYPE data [<!ENTITY file SYSTEM "file:///etc/passwd">]>
```
This declaration defines the structure of the XML document and can include entity declarations.
Entity Declaration is 
```jsx
<!ENTITY file SYSTEM "file:///etc/passwd">
```
This entity declaration defines an entity named file that references the /etc/passwd file on the server's filesystem using the file:/// URI scheme.
Within the XML document, the file entity is referenced inside the <url> element. This means that the contents of the /etc/passwd file will be inserted into the XML document at the location where "&file"; is referenced.


![POC-otb](/img/cwae/goe5.png){: w="400" h="400" }{: .normal }

This payload will get the etc/passwd file from the server and display to us in the response