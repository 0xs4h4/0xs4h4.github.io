---
title: Active Directory (AD) | Local Privilege Escalation (Part 1)
author: beardenx
date: 2024-07-01 00:00:00 +0800
categories: [Cert, CRTP]
tags: [cert,active directory]     ## TAG names should always be lowercase
render_with_liquid: false
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques
---

## **Step 1: Check for Privilege Escalation Paths**

**Tools Used:** PowerView

This step ensures you can escalate your privileges on the current machine to gain local admin rights. Start by checking for any privilege escalation paths. If any services can be abused, add your domain user to the local admin group.

### Check for any priviliege escalation path
```bash
Invoke-AllChecks
```
![Result](/img/crtp/result1.png){: width="972" height="589" }
_Invoke-AllChecks_


### Abusing Services
Add your current domain user to the local Administrators group.

```bash
Invoke-ServiceAbuse
Invoke-ServiceAbuse -Name 'AbyssWebServer' -UserName 'dcorp\studentx' -Verbose
```

![Result](/img/crtp/result2.png){: width="972" height="589" }
_Now, our user is a local admin !_


## **Step 2: Identify Machines with Local Admin Access**

**Tools Used:** Find-PSRemotingLocalAdminAccess.ps1

After gaining local admin rights on one machine, identifying other machines where you have local admin access helps in expanding your control over the network. This can give you access to more resources, data, or higher-privilege accounts. So, you have to identify any computers/machines in the domain where our user has local administrative access.

### Identifying Computers with Local Admin Access

```bash
Find-PSRemotingLocalAdminAccess
```
![Result](/img/crtp/result3.png){: width="972" height="589" }
_studentx has administrative access on dcorp-adminsrv and on the student machine_

So, studentx has administrative access on **dcorp-adminsrv** and on the **student machine.** We can connect to the machine with local administrative access using winrs or powershell remoting.
### Using winrs
```bash
winrs -r:dcorp-adminsrv cmd
set username
set computername
```
![Result](/img/crtp/result4.png){: width="972" height="589" }

### PowerShell Remoting
```bash
Enter-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.local
```
![Result](/img/crtp/result5.png){: width="972" height="589" }

## **Step 3: Identify Machines with Domain Admin Sessions**

**Tools Used:** PowerView

After gaining local admin access on multiple machines, the next objective is often to escalate to **domain admin** privileges. Identifying machines where domain admin accounts are actively logged in provides an opportunity to steal their tokens and elevate privileges within the domain.

### Finding Domain User Locations
To identify machines where domain admin sessions are active, we can query the Domain Controller for information on ComputerNames and membership of the domain admin group with active sessions.
```bash
Find-DomainUserLocation
```
![Result](/img/crtp/result6.png){: width="972" height="589" }
_There is a domain admin session on dcorp-mgmt server!_

This information allows us to potentially exploit these sessions using techniques like winrs or PowerShell Remoting.