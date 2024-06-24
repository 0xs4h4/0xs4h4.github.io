---
title: Challenge 8:Inquisitio et Expugnatio
date: 2024-05-14 05:08:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

### Challenge/Task

1. Inquisitio et Expugnatio

**Proof-of-Concept (POC):**

I try to find if there is any possible SQL injection in the system. Saw this post method that contains few parameters.

![POC-otb](/img/cwae/iee1.png){: w="400" h="400" }{: .normal }

I copy all the request to the file and named it as “sql.req”. I fire up the SQLmap and let the tools running to test the parameters. Unfortunately, all the parameters are secure.

![POC-otb](/img/cwae/iee2.png){: w="400" h="400" }{: .normal }

Then I try to visit any possible endpoints that are exist in the system, I noticed that it redirects me to the /home page. Upon checking the request below, there is a cookie “redir=eyJsb2NhdGlvbiI6ICIvaG9tZSJ9”. I suspect that redir meaning redirection which may refer to the redirection to the /home page. The value is encoded with Base64

![POC-otb](/img/cwae/iee3.png){: w="400" h="400" }{: .normal }

I try to decode the Base64. Yup it is showing me the /home page where it used to redirect any user to the home page. 

![POC-otb](/img/cwae/iee4.png){: w="400" h="400" }{: .normal }

I try to insert (‘1’) at the /home value to see what the server response to the single quote is. Fortunately, it returns 500 Server error. Usually, single quote is used to test SQL syntax error. Seems that I need to test the SQL injection payload here.

![POC-otb](/img/cwae/iee5.png){: w="400" h="400" }{: .normal }
![POC-otb](/img/cwae/iee6.png){: w="400" h="400" }{: .normal }


I try to use 1 or 1=1 payload and encoded it to base64 again and put in the redir param value.

![POC-otb](/img/cwae/iee7.png){: w="400" h="400" }{: .normal }

The server response me with 500 showing a server error. But surprisingly, it shows me all the DB inside the system because the payload we are using just now is a true statement. 

![POC-otb](/img/cwae/iee8.png){: w="400" h="400" }{: .normal }
![POC-otb](/img/cwae/iee9.png){: w="400" h="400" }{: .normal }