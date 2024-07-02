---
layout: post
title: Active Directory (AD) | Enumeration Technique
date: 2024-06-20 05:00:00 +0800
categories: [Cert, CRTP]
tags: [Cert, Active Directory]     ## TAG names should always be lowercase
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques, starting with enumeration, the essential first step in any AD attack strategy.
---

## 1. Enumerating Active Directory by using PowerView

AD enumeration involves gathering detailed information about the Active Directory environment, such as user accounts, group memberships, domain policies, and network structure.

> Enumerate Domain
{: .prompt-info }
```jsx
Get-Domain
Get-Domain -Domain <DomainName>
Get-DomainSID
Get-DomainPolicy

## Show System Access Policy Configurations
Get-DomainPolicy | Select-Object -ExpandProperty SystemAccess

## Show Kerberos Policy Configurations
Get-DomainPolicy | Select-Object -ExpandProperty KerberosPolicy

## Get Domain Policy Data
(Get-DomainPolicyData).systemaccess

## Get Domain Policy for Another Domain
(Get-DomainPolicyData -domain moneycorp.local).systemaccess
```

> Enumerate Domain Controllers
{: .prompt-info }
```jsx
## Get Domain Controllers
Get-DomainController
Get-DomainController -Domain <DomainName>
```

> Enumerate Domain Users
{: .prompt-info }
```jsx
Get-DomainUser

## Save All Domain Users to a File
Get-DomainUser | Out-File -FilePath .\DomainUsers.txt

## Return Specific User
Get-DomainUser -Identity student1

## Return All Properties of a Specific User
Get-DomainUser -Identity student1 -Properties *

## Return Specific Properties of a Specific User
Get-DomainUser -Identity [username] -Properties DisplayName, MemberOf | Format-List

## Return Specific Properties for Users in the Current Domain
Get-DomainUser -Properties samaccountname, logoutCount

## List Only the samaccountname
Get-DomainUser | select -ExpandProperty samaccountname

## Search for a Particular String in a User’s Attributes
Get-DomainUser -LDAPFilter "Description=*built*" | Select name,Description

## Enumerate User Logged on a Machine
Get-NetLoggedon -ComputerName <ComputerName>

## Enumerate Session Information for a Machine
Get-NetSession -ComputerName <ComputerName>

## Enumerate Domain Machines Where Specific Users Are Logged Into
Find-DomainUserLocation -Domain <DomainName> | Select-Object UserName, SessionFromName
```

> Enumerate Domain Computers
{: .prompt-info }
```jsx
Get-DomainComputer

## Get a List of Live Computers in the Current Domain
Get-DomainComputer -Ping

## Get a Specific List Property of All Computers (dnshostname)
Get-DomainComputer | select -ExpandProperty dnshostname

## Get a List of Computers with Specific Operating System
Get-DomainComputer 'OperatingSystem -like "*Server 2022*"'

## Save All Domain Users to a File
Get-DomainComputer -Properties OperatingSystem, Name, DnsHostName | Sort-Object -Property DnsHostName | Out-File -FilePath .\DomainComputers.txt

## Enumerate Live Machines
Get-DomainComputer -Ping -Properties OperatingSystem, Name, DnsHostName | Sort-Object -Property DnsHostName
```


> Enumerate Domain Groups and Domain Group Members
{: .prompt-info }
```jsx
Get-DomainGroup

## Get Details of the Domain Admins Group
Get-DomainGroup -Identity "Domain Admins"

## Get Groups Containing Specific Words in Group Name
Get-DomainGroup *admin*

## Get Group Membership of a Specific User
Get-DomainGroup -UserName "student1"

## Save All Domain Groups to a File
Get-DomainGroup | Out-File -FilePath .\DomainGroups.txt

## Get All Groups in Another Domain
Get-DomainGroup -Domain <targetdomain>

## Return Members of a Specific Group
Get-DomainGroup -Identity '<GroupName>' | Select-Object -ExpandProperty Member

## Return Members of a Specific Group (Using Get-DomainGroupMember)
Get-DomainGroupMember -Identity '<GroupName>' | Select-Object MemberDistinguishedName

## Get Details of Enterprise Admins Group
Get-DomainGroupMember -Identity "Enterprise Admins"

## Get All Members of the Domain Admin Group
Get-DomainGroupMember -Identity "Domain Admins" -Recurse

## Enumerate Local Groups on Local or Remote Machine
Get-NetLocalGroup

## Enumerate Local Groups on Local or Remote Machine (Select GroupName)
Get-NetLocalGroup | Select-Object GroupName

## List All Local Groups on a Machine
Get-NetLocalGroup -ComputerName dcorp-dc

## Enumerate Members of a Specific Local Group
Get-NetLocalGroupMember -GroupName Administrators | Select-Object MemberName, IsGroup, IsDomain

## Get Member of Local Group on Machine
Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators

## Get Activity Logged on Users on a Computer
Get-NetLoggedon

## Get Activity Logged on Users on a Computer (With Local Admin Rights)
Get-NetLoggedon -ComputerName dcrop-adminsrv

## Get Locally Logged Users on Computer
Get-LoggedonLocal

## Get Locally Logged Users on Computer (With Remote Registry)
Get-LoggedonLocal -ComputerName dcrop-adminsrv

## Get Last Logged User on Computer
Get-LastLoggedon

## Get Last Logged User on Computer (With Admin Rights and Remote Registry)
Get-LastLoggedon -ComputerName dcrop-adminsrv

## Return All GPOs in a Domain That Modify Local Group Memberships
Get-DomainGPOLocalGroup | Select-Object GPODisplayName, GroupName
```

> Enumerate Domain Admin and Privilege Groups
{: .prompt-info }
```jsx
Get-DomainGroup *admin*

## Save All Domain Groups to a File
Get-DomainGroup | Out-File -FilePath .\DomainGroups.txt

## Get Details of the Domain Admins Group
Get-DomainGroup -Identity "Domain Admins"

## Get Details of Members of Enterprise Admins Group
Get-DomainGroupMember -Identity "Enterprise Admins"

## Get Details of Members of Account Operators Group
Get-DomainGroupMember -Identity 'Account Operators'

## Get Details of Members of Administrators Group
Get-DomainGroupMember -Identity 'Administrators'

## Get All Members of the Domain Admin Group
Get-DomainGroupMember -Identity "Domain Admins" -Recurse

## Enumerate Members of a Specific Local Group
Get-NetLocalGroupMember -GroupName Administrators | Select-Object MemberName, IsGroup, IsDomain

## Get Member of Local Group on Machine
Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators
```

> Enumerate Shares
{: .prompt-info }
```jsx
Find-DomainShare

## Enumerate Domain Shares Current User Has Access
Find-DomainShare -CheckShareAccess

## Enumerate "Interesting" Files on Accessible Shares
Find-InterestingDomainShareFile -Include *passwords*

## Find Shares on Hosts in Current Domain
Invoke-ShareFinder -Verbose

## Find Sensitive Files on Computers in the Domain
Invoke-FileFinder -Verbose

## Get All Fileservers of the Domain
Get-NetFileServer
```

> Enumerate Group Policy Objects (GPO)
{: .prompt-info }
```jsx
Get-DomainGPO

## Get GPO Applied on an OU
Get-DomainGPO -Identity "{ID}"

## Get List of GPOs in Current Domain
Get-DomainGPO -Properties DisplayName | Sort-Object -Property DisplayName

## Enumerate All GPOs to a Specific Computer
Get-DomainGPO -ComputerIdentity <ComputerName> -Properties DisplayName | Sort-Object -Property DisplayName

## Get GPOs Using Restricted Groups or Groups.xml for Interesting Users
Get-DomainGPOLocalGroup

## Get Users in a Local Group of a Machine Using GPO
Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity dcorp-student1

## Get Machines Where a User is Member of a Specific Group
Get-DomainGPOUserLocalGroupMapping -Identity (Get-DomainOU -Identity StudentMachines).distinguishedname | %{Get-DomainComputer -SearchBase $_} | select name

## Get Users in a Machine's Local Admin Group Using GPO
Get-DomainGPOComputerLocalGroupMapping -ComputerName <ComputerName>

## Enumerate GPO Applied on a Certain OU Using PowerView
Get-NetOU -Identity "OU=Test,OU=test,DC=test,DC=local" | Get-NetGPO

## Enumerate GPO Applied on the StudentMachines OU
(Get-DomainOU -Identity StudentMachines).gplink [LDAP://cn={7478F170-6A0C-490C-B355-9E4618BC785D},cn=policies,cn=system,DC=dollarcorp,DC=moneycorp,DC=local;0]

## Get GPO by Identity
Get-DomainGPO -Identity '{7478F170-6A0C-490C-B355-9E4618BC785D}'
```

> Enumerate Organizational Units (OU)
{: .prompt-info }
```jsx
Get-DomainOU

## See Just Name of the OUs
Get-DomainOU | select -ExpandProperty name

## List All Computers in the StudentsMachines OU
(Get-DomainOU -Identity StudentMachines).distinguishedname | %{Get-DomainComputer -SearchBase $_ } | select name

## Enumerate OUs Sorted by Name
Get-DomainOU -Properties Name | Sort-Object -Property Name

## List Members of Specific OU
Get-NetUser -ADSPath "OU=TestOU,OU=TestOU,DC=test,DC=local"
```

> Enumerate Access Control Lists (ACL)
{: .prompt-info }
```jsx
Get-DomainObjectAcl -SamAccountName student1 -ResolveGUIDs

## Get ACLs Associated with the Specified Prefix
Get-DomainObjectAcl -SearchBase "LDAP://CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local" -ResolveGUIDs -Verbose

## Enumerate ACLs Using ActiveDirectory Module
(Get-Acl 'AD:\CN=Administrator,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local').Access

## Search for Interesting ACEs
Find-InterestingDomainAcl -ResolveGUIDs

## Get ACLs Associated with a Specified Path
Get-PathAcl -Path "\\dcorp-dc.dollarcorp.moneycorp.local\sysvol"

## Get ACLs Associated with a Specified Account
Get-DomainObjectAcl -Identity <AccountName> -ResolveGUIDs

## Check ACLs Associated with a Specified Path (e.g., SMB Share)
Get-PathAcl -Path "\\Path\Of\A\Share"

## Enumerate ACLs for the Domain Admins Group
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs -Verbose

## Search for Interesting ACEs (Match Specific IdentityReferenceName)
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "studentx"}
```

> Enumerate Domain Trusts
{: .prompt-info }
```jsx
Get-DomainTrust

## Enumerate Trusts for Another Domain
Get-DomainTrust -Domain <DomainName>

## Enumerate All Trusts for Each Domain
Get-DomainTrustMapping

## Enumerate Trusts with Specific Attributes
Get-ForestDomain | %{Get-DomainTrust -Domain $_.Name} | ?{$_.TrustAttributes -eq "FILTER_SIDS"}
```

> Enumerate Forest Trusts
{: .prompt-info }
```jsx
Get-ForestDomain

## Enumerate Forest Trusts for Another Domain
Get-ForestDomain -Forest <ForestName>

## Map the Trust of the Forest
Get-ForestTrust

## Map the Trust of Another Forest
Get-ForestTrust -Forest <ForestName>
```

> User Hunting (Pretty Noisy)
{: .prompt-info }
```jsx
Find-LocalAdminAccess -Verbose

## Find Local Admins on All Machines of the Domain
Find-DomainLocalGroupMember -Verbose

## Find Computers Where a Domain Admin or Specified User Has a Session
Find-DomainUserLocation | Select-Object UserName, SessionFromName

## Confirm Admin Access
Test-AdminAccess

## Find Computers Where a Domain Admin or Specified User/Group Has Sessions
Find-DomainUserLocation -Verbose

## Find Computers Where Specified User/Group Has Sessions
Find-DomainUserLocation -UserGroupIdentity "RDPUsers"

## List Session on Remote Machine
Invoke-SessionHunter

## List Session on Remote Machine (FailSafe)
Invoke-SessionHunter -FailSafe

## Opsec Friendly Command for Listing Sessions on Remote Machines
Invoke-SessionHunter -NoPortScan -Targets <Location file save>
```