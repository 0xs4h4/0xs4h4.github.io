---
title: Web Application Pentest Checklist
date: 2024-06-20 05:00:00 +0800
categories: [Pentest Checklist]
tags: [Checklist]     # TAG names should always be lowercase
description: This checklist is meticulously curated to guide a web application penetration tester through a series of steps, tasks, and checks necessary for performing a comprehensive and effective penetration test.
---


This checklist is meticulously curated to guide a web application penetration tester through a series of steps, tasks, and checks necessary for performing a comprehensive and effective penetration test. Each section is strategically categorized, focusing on various vital aspects of web application security, ranging from information gathering to meticulous testing of authentication, session management, data validation, and business logic, among others. Utilize this checklist as a roadmap to navigate through the penetration testing process, ensuring each critical component of the web application's security posture is assessed, vulnerabilities are identified, and potential threats are mitigated.

### **1. Information Gathering**

**WHOIS Lookup**

- [ ]  Perform WHOIS lookups to gather domain information.
- [ ]  Gather Network/Host Information using tools like **whois, dig, Sublist3r**.

**DNS Interrogation**

- [ ]  Identify DNS records (A, AAAA, MX, NS, SOA, TXT).

**Identify Open Ports and Services**

- [ ]  Utilize tools like Nmap for port scanning.
- [ ]  Identify Application Entry Points.

**User/Directory Enumeration**

- [ ]  Enumerate User Roles and Application Functionalities.
- [ ]  Directory and File Enumeration using **DirBuster, gobuster, wfuzz**.

**Subdomain Discovery**

- [ ]  Use tools like Sublist3r, Amass, or Subfinder.
- [ ]  Verify the existence of discovered subdomains.

**Identify Web Technologies**

- [ ]  Identify Technologies and Frameworks Used with **Wappalyzer, BuiltWith**

**HTTP Methods**

- [ ]  Identify allowed HTTP methods (GET, POST, PUT, DELETE)

**Content Discovery**

- [ ]  Utilize tools like Burp Suite's Content Discovery function to uncover hidden directories, files, and parameters.

---

### **2. Security Misconfiguration and Deployment Management Testing**

**HTTP Security Headers**

- [ ]  Examine HTTP Headers for Security Configurations on [**SecurityHeaders.io**](https://securityheaders.io/).

**Error Handling**

- [ ]  Check error pages for sensitive information leakage.
- [ ]  Customize error pages.

**Directory Listing**

- [ ]  Test for Directory Listing and Path Traversal Issues.
- [ ]  Review File Permissions and Access Controls.

**SSL/TLS**

- [ ]  Check for Secure SSL/TLS Configurations using **sslscan, SSLyze, [TestSSL.sh](https://testssl.sh/)**.

---

### **3. Identity Management Testing**

- [ ]  Verify Role-Based Access Controls.
- [ ]  Test for Insecure Direct Object References (IDOR).
- [ ]  Verify User and Session Enumeration.

**Password Recovery**

- [ ]  Evaluate the security of password recovery mechanisms, ensuring they don’t expose sensitive information or allow for account enumeration.

---

### **4. Authentication Testing**

**Default Credentials**

- [ ]  Check for default usernames and passwords.

**Weak Password Policies**

- [ ]  Check the effectiveness of password complexity requirements.
- [ ]  Test password resets and recovery mechanisms.

**Account Lockout Mechanisms**

- [ ]  Test the account lockout policy.
- [ ]  Analyze account lockout duration and thresholds.
- [ ]  Test for Bypassing Authentication Mechanisms.
- [ ]  Check for Insecure Password Storage.
- [ ]  Test for Exposure of Internal Implementation Details.

**Multi-Factor Authentication (MFA)**

- [ ]  Test the resilience of MFA implementations against phishing and bypass attacks.
- [ ]  Evaluate the consistency of MFA prompts in different parts of the application.

---

### 5**. Authorization Testing**

- [ ]  Test for Horizontal and Vertical Privilege Escalation.
- [ ]  Verify Proper Function Level Access Control.

**Object-Level Authorization**

- [ ]  Ensure that access controls are enforced on the server-side and that client-side controls are not solely relied upon.

---

### **6. Session Management Testing**

**Session Timeout**

- [ ]  Test for Session Timeout and Expiration Handling.
- [ ]  Test for Session Fixation.
- [ ]  Verify session termination after logout.

**Session Token Security**

- [ ]  Assess security of session tokens (length, complexity).
- [ ]  Test session token randomness

**Cookie Security**

- [ ]  Ensure cookies are secure and HttpOnly.
- [ ]  Validate that sensitive data is not stored in cookies.

**Cross-Site Request Forgery (CSRF)**

- [ ]  Confirm that anti-CSRF tokens are in place and effectively protecting against cross-site request forgery attacks.

---

### **7. Data Validation Testing**

**Cross-Site Scripting (XSS)**

```jsx
<x onmousedown=alert(1)>click this!

<h1 draggable="true" ondrag="alert(1337)">test</h1>

<input draggable="true" ondrag="alert(1)">test</input>

<marquee onload=confirm(1337)>@tulisanPAPE
```

- [ ]  Test for reflected XSS
- [ ]  Test for stored XSS
- [ ]  Test for DOM-based XSS

**SQL Injection**

- [ ]  Test for SQL Injection (SQLi)
- [ ]  Check error handling for sensitive information leakage.

**File Upload Vulnerabilities**

- [ ]  Check file upload functionalities for malicious files.
- [ ]  Validate file type and content.

**Others Injection**

- [ ]  Examine for Cross-Site Request Forgery (CSRF).
- [ ]  Test for XML External Entity (XXE) Attacks.
- [ ]  Check for Unvalidated Redirects and Forwards.
- [ ]  Check for Open Redirection -

```
<head>
<meta http-equiv="refresh" content="0; url=https://beardenx.github.io/index.html" />
</head>
```

**API Parameter Tampering**

- [ ]  Test for the ability to manipulate parameters in API requests to bypass security controls or access sensitive data.

---

### **8. Error Handling and Logging**

- [ ]  Test for Verbose Error Handling.
- [ ]  Verify Secure Logging Practices.
- [ ]  Examine Log Management and Monitoring.

---

### **9. Business Logic Testing**

**Role-Based Access Control**

- [ ]  Test access controls and role definitions.
- [ ]  Validate direct object references.

**Data Validation**

- [ ]  Test for proper input validation on all forms and fields.
- [ ]  Test for excessive data exposure.

**Function Level Access Controls**

- [ ]  Ensure that functions have proper access controls.
- [ ]  Examine Workflow Bypass and Circumvention.

**Rate Limiting and Throttling**

- [ ]  Verify that rate limiting is implemented and effectively protecting against automated attacks or brute-force attempts.

---

### **10. API and Web Service Testing**

**REST API**

- [ ]  Test for authentication on endpoints.
- [ ]  Validate input on all API endpoints.
- [ ]  Verify Proper HTTP Method Implementation.
- [ ]  Test for Misconfigurations and Default Credentials.

**OAuth & OpenID Connect**

- [ ]  Test the implementation of OAuth and OpenID Connect for weaknesses and misconfigurations that could allow unauthorized access.

**GraphQL**

- [ ]  Test for excessive data exposure.
- [ ]  Test for authentication and authorization flaws.

---

### **11. Miscellaneous**

- [ ]  Check for Potential Denial of Service (DoS) Vulnerabilities.
- [ ]  Test for Insecure File Uploads.
- [ ]  Examine for Potential Subdomain Takeovers.
- [ ]  Test WebSocket implementations for proper authentication, authorization, and data validation.

---

This compiled checklist includes all necessary tests and ensures a thorough web application penetration test. It allows you to track each stage of the testing process meticulously and ensures that no aspect is overlooked.