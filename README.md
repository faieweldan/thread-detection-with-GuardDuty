# GuardDuty Project – Threat Detection (Juice Shop)

---

## Overview

In this project, I studied how an intentionally vulnerable web app (OWASP Juice Shop) can be deployed using an AWS CloudFormation template and how it can be exploited in a cloud environment.

Based on the NextWork walkthrough demonstration, the project shows how an attacker could exploit common web vulnerabilities to steal temporary AWS credentials from an EC2 instance and use them to access an S3 bucket.

It also demonstrates how Amazon GuardDuty detects this suspicious activity. The walkthrough further includes testing GuardDuty Malware Protection for S3 using an EICAR test file.

Due to GuardDuty and Malware Protection cost considerations, I did not execute the full attack chain in my own AWS account. This README documents my understanding of the deployment, attack flow, and detection process.

### Attack Flow Diagram

<img width="545" height="134" alt="Screenshot 2026-02-19 at 3 14 43 PM" src="https://github.com/user-attachments/assets/e3e91905-6488-425e-a49b-3e5e7196cb46" />

---

# 1️⃣ Deployment Phase (CloudFormation Setup)

## What Was Deployed

The CloudFormation stack in the walkthrough created a full environment for the web app, including:

- **EC2** instance hosting the Juice Shop web app  
- **VPC + networking resources** (subnets, security group, routing, etc.)  
- **CloudFront** distribution for a public URL  
- **S3 bucket** containing a “secret” file (simulated sensitive data)  
- **Amazon GuardDuty** enabled for monitoring and threat detection  

### OWASP Juice Shop

<img width="457" height="286" alt="Screenshot 2026-02-19 at 3 15 30 PM" src="https://github.com/user-attachments/assets/532e5a66-168e-4e4d-b6e7-fa59d1d830da" />

The OWASP Juice Shop is intentionally vulnerable and is used for security training. It allows learners to practice identifying and exploiting web vulnerabilities in a safe environment.

---

## CloudFormation Deployment Process (Walkthrough Summary)

- Created stack using uploaded template
- Enabled rollback on failure
- Allowed IAM resource creation
- Verified stack status as `CREATE_COMPLETE`
- Reviewed 27 deployed resources

### Infrastructure Breakdown

### 🌐 Web App Infrastructure
- EC2 instance
- Load balancer
- Auto Scaling Group
- Launch templates
- CloudFront distribution

### 🌍 Networking Components
- VPC
- Subnets
- Route tables
- Internet Gateway
- Security Groups
- VPC Endpoints
- Prefix Lists

### 🪣 S3 Storage
- S3 bucket storing simulated sensitive data (`important-information.txt`)

### 🛡️ GuardDuty
GuardDuty was enabled to monitor:
- CloudTrail activity
- Network traffic
- Credential usage patterns

---

# 2️⃣ Attack Phase (Walkthrough Summary)

## Step 1: SQL Injection (Login Bypass)

The demonstration used the following payload:

- Email: `' or 1=1;--`
- Password: anything

This manipulates the backend SQL query and allows authentication bypass.  
It demonstrates how unsanitized user input can compromise authentication systems.

---

## Step 2: Command Injection (Credential Exfiltration)

In the admin profile page, malicious JavaScript was injected into the username field.

The payload:

- Executed shell commands on the EC2 instance
- Accessed the **EC2 Instance Metadata Service (IMDS)**
- Retrieved temporary IAM credentials
- Saved them into a public file:

`/assets/public/credentials.json`

This shows how improper input validation can lead to credential exposure.

---

## Step 3: Accessing Stolen Credentials via CloudShell

In the walkthrough, AWS CloudShell was used to simulate an external attacker.

The attacker:

- Downloaded `credentials.json`
- Created an AWS CLI profile named `stolen`
- Used the stolen credentials to access S3
- Copied `secret-information.txt`
- Verified contents using `cat`

### Credentials Exposed in Web App

<img width="463" height="219" alt="Screenshot 2026-02-19 at 3 18 41 PM" src="https://github.com/user-attachments/assets/d426ff33-3429-4b9e-b8a0-9178bf709635" />

### CloudShell Download Commands

<img width="448" height="104" alt="Screenshot 2026-02-19 at 3 19 35 PM" src="https://github.com/user-attachments/assets/98b1225b-6fbb-4606-aa7a-c2acd0bc1fb2" />

<img width="459" height="133" alt="Screenshot 2026-02-19 at 3 20 26 PM" src="https://github.com/user-attachments/assets/a8a99836-b072-4490-a5a2-30718fbc59c2" />

This demonstrates how exposed EC2 instance credentials can be abused to access AWS resources.

---

# 3️⃣ Detection Phase (GuardDuty Analysis)

## High Severity Finding

GuardDuty generated a high severity finding similar to:

`UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.InsideAWS`

This occurred because EC2 instance credentials were used in an unusual context (CloudShell / separate AWS environment), which is not normal behavior for that IAM role.

GuardDuty provided:

- Affected resource
- IAM role used
- S3 object retrieval action
- IP and location details
- Severity level

### GuardDuty Finding Screenshots

<img width="458" height="230" alt="Screenshot 2026-02-19 at 3 20 59 PM" src="https://github.com/user-attachments/assets/7dff366a-b1ab-48ac-be47-706487aee869" />

<img width="506" height="253" alt="Screenshot 2026-02-19 at 3 21 43 PM" src="https://github.com/user-attachments/assets/890297be-c747-485d-b851-313d67d35e8c" />

This shows how GuardDuty detects anomalies in credential usage patterns.

---

# 4️⃣ Malware Protection (Walkthrough Demonstration)

The walkthrough also enabled **GuardDuty Malware Protection for S3** and uploaded:

- `EICAR-test-file.txt`

GuardDuty generated a finding:

`Object:S3/MaliciousFile`

### Malware Detection Flow

<img width="679" height="228" alt="Screenshot 2026-02-19 at 3 12 43 PM" src="https://github.com/user-attachments/assets/182ffd06-9e27-4ddb-91e1-b2f620f51bc2" />

<img width="455" height="204" alt="Screenshot 2026-02-19 at 3 13 39 PM" src="https://github.com/user-attachments/assets/126efe02-64b0-44ec-aa81-7ebe9350706e" />

This demonstrates GuardDuty’s ability to scan S3 objects for malware signatures.

---

# Key Concepts Learned

- Infrastructure as Code using **CloudFormation**
- Web vulnerabilities:
  - SQL Injection
  - Command Injection
- EC2 Instance Metadata (IMDS) risks
- IAM credential exposure impact
- Anomaly detection using GuardDuty
- S3 Malware Protection basics

---

# Reflection

This project helped me understand how multiple vulnerabilities can chain together:

1. Login bypass  
2. Command injection  
3. Credential theft from IMDS  
4. Unauthorized S3 access  

It also strengthened my understanding of how GuardDuty uses behavioral analysis and anomaly detection to identify suspicious credential usage inside AWS.

Because GuardDuty Malware Protection is not fully covered under the Free Tier, I chose to learn through the official walkthrough instead of deploying everything in my own account.
