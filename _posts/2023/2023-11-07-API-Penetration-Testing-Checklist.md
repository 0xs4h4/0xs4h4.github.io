---
title: API Pentest Checklist
date: 2023-11-07 05:00:00 +0800
categories: [Pentest Checklist, API]
tags: [Checklist]        ## TAG names should always be lowercase
description: A comprehensive guideline designed to steer an API penetration tester through a series of essential steps, procedures, and considerations necessary for conducting a robust and effective penetration test.
---

## **1. Information Gathering**

**API Documentation Review:**

- [ ]  Examine the API documentation for potential security weaknesses.
- [ ]  Identify the endpoints, HTTP methods, parameters, and authentication mechanisms used.

**Endpoint Discovery:**

- [ ]  Identify all API endpoints.
- [ ]  Determine the functionality and data each endpoint handles.

---

## **2. Security Misconfiguration and Deployment Management Testing**

**SSL/TLS Configuration:**

- [ ]  Ensure that secure SSL/TLS configurations are implemented.

**HTTP Security Headers:**

- [ ]  Verify the existence and configuration of HTTP security headers.

---

## **3. Authentication Testing**

**Token Generation:**

- [ ]  Assess the token generation mechanism for vulnerabilities.
- [ ]  Ensure tokens are generated with sufficient entropy.

**Token Expiration:**

- [ ]  Check the token expiration mechanism.
- [ ]  Verify tokens are invalidated after logout.

---

## **4. Authorization Testing**

**Role-Based Access Control:**

- [ ]  Test endpoints against various user roles to verify proper access control.

**Endpoint Protection:**

- [ ]  Ensure that all endpoints are protected and not publicly accessible unless required.

---

## **5. Session Management Testing**

**Session Token Security:**

- [ ]  Ensure session tokens are securely generated and transmitted.

**Token Storage:**

- [ ]  Verify secure storage of tokens on the client side.

---

## **6. Data Validation Testing**

**Input Validation:**

- [ ]  Test API endpoints for proper input validation.
- [ ]  Check for potential SQL Injection, XSS, and Command Injection vulnerabilities.

---

## **7. Error Handling and Logging**

**Error Message Review:**

- [ ]  Ensure error messages do not reveal sensitive information or system details.

**Logging Practices:**

- [ ]  Verify that logging practices do not expose sensitive information.

---

## **8. Business Logic Testing**

**Rate Limiting:**

- [ ]  Confirm the implementation of rate limiting on endpoints to prevent abuse.

**Data Exposure:**

- [ ]  Assess endpoints to ensure they do not expose sensitive or unnecessary information.

---

## **9. API Specific Testing**

**REST API:**

- [ ]  Conduct tests specific to RESTful APIs such as HTTP method testing and endpoint security.

**GraphQL:**

- [ ]  Conduct tests specific to GraphQL such as excessive data exposure and authorization checks.

---

## **10. Miscellaneous**

**Denial of Service (DoS) Vulnerabilities:**

- [ ]  Check the API for potential DoS vulnerabilities.

**CORS Configuration:**

- [ ]  Review Cross-Origin Resource Sharing (CORS) configurations for security concerns.

---

This checklist embodies a comprehensive and meticulously curated guide to facilitate a thorough API penetration test. It aids in ensuring that every critical aspect of API security is scrutinized, vulnerabilities are unearthed, and preventive actions are recommended to bolster the API's security posture.