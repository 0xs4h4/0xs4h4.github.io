---
title: Active Directory (AD) Attack Technique
date: 2024-06-20 05:00:00 +0800
categories: [Cert, CRTP]
tags: [Cert]     # TAG names should always be lowercase
description: Designed to provide a comprehensive guide to Active Directory (AD) attack techniques, starting with enumeration, the essential first step in any AD attack strategy.
---

# 0. Reading List

> **Reading List**
> 
> - [AV Evasion](https://csbygb.gitbook.io/pentips/windows/avevasion)
> - https://hackmag.com/security/fck-amsi/
> - https://amsi.fail/
> - https://blog.f-secure.com/hunting-for-amsi-bypasses/
> - https://www.mdsec.co.uk/2018/06/exploring-powershell-amsi-and-logging-evasion/
> - https://medium.com/@vanitasnk/tales-to-redteam-ops-crtp-review-3da3ba427e76

---

# 1. Introduction

> **Purpose and Scope:**
> 
> - **Purpose**: This playbook is designed to provide a comprehensive guide to Active Directory (AD) attack techniques, starting with enumeration, the essential first step in any AD attack strategy.
> - **Scope**: Covering a range of methods and tools, this playbook will help you systematically gather information, identify vulnerabilities, and exploit them to achieve your objectives within an AD environment.

> **Importance in Red Team Operations:**
> 
> - Understanding the AD structure helps in planning and executing attacks effectively.
> - Identifies potential entry points and privileges escalation paths.
> - Provides insights into the security posture and defensive mechanisms of the target environment.
> - Understanding AD attack techniques is crucial for Red Team professionals to effectively simulate real-world attacks and assess the security posture of an organization.

---

# 2. Enumerating Active Directory by using PowerView

AD enumeration involves gathering detailed information about the Active Directory environment, such as user accounts, group memberships, domain policies, and network structure.

- Get Current Domain
    
    
    | Command | Objective |
    | --- | --- |
    | Get-Domain | Get Current Domain |
- Enumerate Other Domains4
    
    
    | Command | Objective |
    | --- | --- |
    | Get-Domain -Domain <DomainName> | Get Object of another domain |
- Get Domain SID
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainSID | Get domain SID for the currect domain |
- Get Domain Policy
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainPolicy | Get domain policy for current domain  |
    | Get-DomainPolicy | Select-Object -ExpandProperty SystemAccess | Will show us the policy configurations of the Domain about system access |
    | Get-DomainPolicy | Select-Object -ExpandProperty KerberosPolicy | Will show us the policy configurations of the Domain about kerberos |
    | (Get-DomainPolicyData).systemaccess | Get domain policy for current domain  |
    | (Get-DomainPolicyData -domain moneycorp.local).systemaccess | Get domain policy for another domain  |
- Get Domain Controllers
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainController | Get domain controller for current domain |
    | Get-DomainController -Domain <DomainName> | Get domain controller for another domain |
- Enumerate Domain Users
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainUser | Get a list of users in the current domain |
    | Get-DomainUser | Out-File -FilePath .\DomainUsers.txt | Save all Domain Users to a file |
    | Get-DomainUser -Identity student1 | Will return specific user |
    | Get-DomainUser -Identity student1 -Properties | Will return all properties of a specific user |
    | Get-DomainUser -Identity [username] -Properties DisplayName, MemberOf | Format-List | Will return specific properties of a specific user |
    | Get-DomainUser -Properties samaaccount, logoutCount | Will return specific properties for users in the current domain |
    | Get-DomainUser | select -ExpandProperty samaccountname | List only the samaccountname |
    | Get-DomainUser -LDAPFilter “Description=*built*” | Select name,Description | Search for a particular string in a user’s attributes |
    | Get-NetLoggedon -ComputerName <ComputerName> | Enumerate user logged on a machine |
    | Get-NetSession -ComputerName <ComputerName> | Enumerate Session Information for a machine |
    | Find-DomainUserLocation -Domain <DomainName> | Select-Object UserName, SessionFromName | Enumerate domain machines of the current/specified domain where specific users are logged into |
- Enumerate Domain Computers
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainComputer | Get a list of computers in the current domain |
    | Get-DomainComputer  -Ping | Get a list of live computers in the current domain |
    | Get-DomainComputer | select -ExpandProperty dnshostname | Get a specific list property of all the computers : list only the dnshostname |
    | Get-DomainComputer ‘OperatingSystem -like “*Server 2022*”’ | Get a list of computers in the current domain |
    | Get-DomainComputer -Properties OperatingSystem, Name, DnsHostName | Sort-Object -Property DnsHostName | Save all Domain Users to a file |
    | Get-DomainComputer -Ping -Properties OperatingSystem, Name, DnsHostName | Sort-Object -Property DnsHostName | Enumerate Live machines |
- Enumerate Domain Groups and Domain Group Members
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainGroup | Get all the group in the current domain |
    | Get-DomainGroup -Identity "Domain Admins” | Get the details of the Domain Admins Group |
    | Get-DomainGroup *admin* | Get all group containing the words “admin” in group name |
    | Get-DomainGroup -UserName “student1” | Get the group membership of a specific user |
    | Get-DomainGroup | select name | Get all the group in the current domain |
    | Get-DomainGroup | Out-File -FilePath .\DomainGroup.txt | Save all Domain Groups to a file |
    | Get-DomainGroup -Domain <targetdomain> | Get all the group in the other domain |
    | Get-DomainGroup -Identity '<GroupName>' | Select-Object -ExpandProperty Member | Return members of Specific Group (eg. Domain Admins & Enterprise Admins) |
    | Get-DomainGroupMember -Identity '<GroupName>' | Select-Object MemberDistinguishedName | Return members of Specific Group (eg. Domain Admins & Enterprise Admins) |
    | Get-DomainGroupMember -Identity "Enterprise Admins" | Get the details members of the Enterprise Admins group |
    | Get-DomainGroupMember -Identity “Domain Admins” -Recurse | Get all the members of the Domain Admin Group |
    | Get-NetLocalGroup | Enumerate the local groups on the local (or remote) machine. |
    | Get-NetLocalGroup | Select-Object GroupName | Enumerate the local groups on the local (or remote) machine. Requires local admin rights on the remote machine
 |
    | Get-NetLocalGroup -ComputerName dcorp-dc | List all the local group on a machine (needs administrator privs on non dc-machines) |
    | Get-NetLocalGroupMember -GroupName Administrators | Select-Object MemberName, IsGroup, IsDomain | Enumerates members of a specific local group on the local (or remote) machine. Also requires local admin rights on the remote machine |
    | Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators | Get member of local group “administrator” on machine (needs administrator privs on non dc-machines) |
    | Get-NetLoggedon | Get activity logged on users on a computer  |
    | Get-NetLoggedon -ComputerName dcrop-adminsrv | Get activity logged on users on a computer (need local admin rights on the target) |
    | Get-LoggedonLocal | Get locally logged users on computer |
    | Get-LoggedonLocal -ComputerName dcrop-adminsrv | Get locally logged users on computer (need remote registry on the target - started by default on server OS) |
    | Get-LastLoggedon | Get the last logged user on computer  |
    | Get-LastLoggedon -ComputerName dcrop-adminsrv | Get the last logged user on computer (needs administrator right and remote registry on the target) |
    | Get-DomainGPOLocalGroup | Select-Object GPODisplayName, GroupName | Return all GPOs in a domain that modify local group memberships through Restricted Groups or Group Policy Preferences |
- Enumerate Domain Admin and Privilege Group
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainGroup *admin* | Get all group containing the words “admin” in group name |
    | Get-DomainGroup | Out-File -FilePath .\DomainGroup.txt | Save all Domain Groups to a file |
    | Get-DomainGroup -Identity "Domain Admins” | Get the details of the Domain Admins Group |
    | Get-DomainGroupMember -Identity "Enterprise Admins" | Get the details members of the Enterprise Admins group |
    | Get-DomainGroupMember -Identity 'Account Operators' | Get the details members of the Account Operators group |
    | Get-DomainGroupMember -Identity 'Administrators' | Get the details members of the Administrators group |
    | Get-DomainGroupMember -Identity “Domain Admins” -Recurse | Get all the members of the Domain Admin Group |
    | Get-NetLocalGroupMember -GroupName Administrators | Select-Object MemberName, IsGroup, IsDomain | Enumerates members of a specific local group on the local (or remote) machine. Also requires local admin rights on the remote machine |
    | Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators | Get member of local group “administrator” on machine (needs administrator privs on non dc-machines) |
- Enumerate Shares
    
    
    | Command | Objective |
    | --- | --- |
    | Find-DomainShare | Enumerate Domain Shares |
    | Find-DomainShare -CheckShareAccess | Enumerate Domain Shares the current user has access |
    | Find-InterestingDomainShareFile -Include *passwords* | Enumerate "Interesting" Files on accessible shares |
    | Invoke-ShareFinder -Verbose | Find shares on hosts in current domain |
    | Invoke-FileFinder -Verbose | Find sensitive files on computers in the domain |
    | Get-NetFileServer | Get all fileservers of the domain |
- Enumerate Group Policies Object (GPO)
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainGPO | Get list of GPO in current domain |
    | Get-DomainGPO -Identity “{ID} | Get GPO applied on an OU. Read GPOname from gplink attribute from Get-NetOU |
    | Get-Domain -ComputerIdentity dcorp-student1 | Get list of GPO in current domain |
    | Get-DomainGPO -Properties DisplayName | Sort-Object -Property DisplayName | Get list of GPO in current domain |
    | Get-DomainGPO -ComputerIdentity <ComputerName> -Properties DisplayName | Sort-Object -Property DisplayName
 | Enumerate all GPOs to a specific computer
 |
    | Get-DomainGPOLocalGroup | Get GPO(s) which use Restricted Groups or groups.xml for interesting users |
    | Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity dcorp-student1 | Get users which are in a local group of a machine using GPO |
    | Get-DomainGPOUserLocalGroupMapping -Identity s(Get-DomainOU -Identity StudentMachines).distinguishedname |
    %{Get-DomainComputer -SearchBase $_} | select nametudent1 -Verbose | Get machine where the given user is member of a specific group |
    | Get-DomainGPOComputerLocalGroupMapping -ComputerName <ComputerName>
 | Get users that are part of a Machine's local Admin group
 |
    | Get-NetOU -Identity "OU=Test,OU=test,DC=test,DC=local" | Get-NetGPO | Enumerate GPO applied on the certain OU using powerview |
    | (Get-DomainOU -Identity StudentMachines).gplink [LDAP://cn={7478F170-6A0C-490C-B355-9E4618BC785D},cn=policies,cn=system,DC=dollarcorp,DC=moneycorp,DC=local;0] | To enumerate GPO applied on the StudentMachines OU |
    | Get-DomainGPO -Identity '{7478F170-6A0C-490C-B355-
    9E4618BC785D}' |  |
- Enumerate Organizational Unit (OU)
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainOU | Get OUs in a domain |
    | Get-DomainOU | select -ExpandProperty name | See just name of the OUs |
    | (Get-DomainOU -Identity StudentMachines).distinguishedname |
    %{Get-DomainComputer -SearchBase $_ } | select name | List all the computers in the StudentsMachines OU: |
    | Get-DomainOU -Properties Name | Sort-Object -Property Name |  |
    |  Get-NetUser -ADSPath "OU=TestOU,OU=TestOU,DC=test,DC=local” | List Members of specific OU |
- Enumerate Access Control List (ACL)
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainObjectAcl -SamAccountName student1 -ResolveGUIs | Get the ACLs associated with the specified object |
    | Get-DomainObjectAcl -SearchBase “LDAP://CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local” -ResolveGUIDs -Verbos | Get the ACLs associated with the specified prefix to be used for search |
    | (Get-Acl ‘AD:\CN=Administrator,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local’).Access | We can also enumerate ACLs using ActiveDirectory module but without resolving GUIDs |
    | Find-InterestingDomainAcl -ResolveGUIDs |  |
    | Get-PathAcl -Path “\\dcorp-dc.dollarcorp.moneycorp.local\sysvol” | Get the ACLs associated with the specified path |
    | Get-DomaiObjectAcl -Identity <AccountName> -ResolveGUIDs
 | Returns the ACLs associated with the specified account |
    | Find-InterestingDomainAcl -ResolveGUIDs | Search for interesting ACEs
 |
    | Get-PathAcl -Path "\\Path\Of\A\Share"
 | Check the ACLs associated with a specified path (e.g smb share)
 |
    | Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs -Verbose | Enumerate ACLs for the Domain Admins Group |
    | Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "studentx"}  |  |
- Enumerate Domain Trust
    
    
    | Command | Objective |
    | --- | --- |
    | Get-DomainTrust
 | Enumerate all trusts for the current domain  |
    | Get-DomainTrust -Domain <DomainName>
 | Enumerate all trusts for the other domain  |
    | Get-DomainTrustMapping
 | Enumerate all trusts for the current domain and then enumerates all trusts for each domain it finds |
    | Get-ForestDomain | %{Get-DomainTrust -Domain $.Name} | ?{$.TrustAttributes -eq "FILTER_SIDS"} |  |
- Enumerate Forest Trust
    
    
    | Command | Objective |
    | --- | --- |
    | Get-ForestDomain
 | Enumerate all trusts for the current domain  |
    | Get-ForestDomain -Forest <ForestName>
 | Enumerate all trusts for the other domain  |
    | Get-ForestTrust
 | Map the Trust of the Forest
 |
    | Get-ForestTrust -Forest <ForestName>
 | Map the Trust of the Forest |
- User Hunting (Pretty Noisy)
    
    
    | Command | Objective |
    | --- | --- |
    | Find-LocalAdminAccess -Verbose
 | Finds all machines on the current domain where the current user has local admin access
 |
    | Find-DomainLocalGroupMember -Verbose
 | Find local admins on all machines of the domain
 |
    | Find-DomainUserLocation | Select-Object UserName, SessionFromName
 | Find computers were a Domain Admin OR a spesified user has a session
 |
    | Test-AdminAccess
 | Confirming admin access
 |
    | Find-DomainUserLocation -Verbose | Find Computers where a domain admin(or specified user/group) has sessions: |
    | Find-DomainUserLocation -UserGroupIdentity “RDPUsers” | Find Computers where a domain admin(or specified user/group) has sessions: |
    | Invoke-SessionHunter | List Session on Remote Machine - Using   |
    | Invoke-SessionHunter -FailSafe | List Session on Remote Machine - Using   |
    | Invoke-SessionHunter -NoPortScan -Targets <Location file save> | An Opsec Friendly command |

---

# 3. Local Privilege Escalation

> **In Active Directory Environment, there are multiple scenarios which lead to privilege escalation. We had a look at the following:**
> 
> - Hunting for Local Admin access on other machines
> - Hunting for high privilege domain accounts (like a Domain Administrator)
> - I have local admin access on a machine -> A Domain Admin has a session on that machine -> I steal his token and impersonate him -> Profit!

> **There are various ways of locally escalting privilege on Windows:**
> 
> - Missing Patches
> - Automated Deployment and AutoLogon password in clear text
> - AlwaysInstallElevated (Any user can run MSI on SYSTEM)
> - Misconfigured Services
> - DLL Hijacking and more
> - NTLM Relaying a.k.a Wont’t Fix
> - NTLM Relaying example - https://github.com/antonioCoco/RemotePotato0

> **We can use below tools for complete coverage:**
> 
> - https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1
> - https://github.com/enjoiz/Privesc
> - https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASexe/README.md - Very NOISY

# 4. Local Privilege Escalation by using PowerUp

- Check for any priviliege escalation path
    
    
    | Command | Objective |
    | --- | --- |
    | Invoke-AllChecks | Check for any priviliege escalation path |