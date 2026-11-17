# Cybersecurity Essentials — Study Notes

[![Read online](https://img.shields.io/badge/Read%20online-Blue%20Team%20Notes-2747c9?logo=bookstack&logoColor=white)](https://ammaryasser28.github.io/?t=essentials) ![Defensive Security](https://img.shields.io/badge/Defensive%20Security-informational) ![SOC](https://img.shields.io/badge/SOC-informational)

Structured notes for the **Cybersecurity Essentials** course of a SOC-track Cybersecurity Diploma. They cover the defensive side of security end to end: identity and access, Active Directory, network defense, input attacks and injection, vulnerability management, mobile security, data loss prevention, ransomware resilience, and the frameworks SOC teams use to describe attacks.

📖 **Read it online:** every note in this repo is on [Blue Team Notes](https://ammaryasser28.github.io/?t=essentials), with interactive diagrams, search across all four courses, and progress tracking.

> 🌐 **Language:** the notes are written in Arabic (Egyptian dialect), with every technical term and command kept in English. Each note also has a full English version: switch to **EN** at the top of [the site](https://ammaryasser28.github.io/?t=essentials&lang=en).

## What's Covered

- Security frameworks: MITRE ATT&CK, Cyber Kill Chain, Unified Kill Chain, NIST CSF, Pyramid of Pain
- Log analysis and digital forensics: web server, network, and authentication logs, impossible-travel and VPN detection
- Active Directory core concepts, Group Policy, and Windows authentication hashes (LM, NTLM, NTLMv2)
- Zero Trust, IAM, PAM, and access control models
- Network security foundations: secure topology design, VLANs, router and switch attacks, firewalls, proxies and load balancers, NAC and honeypots
- Input attacks and injection: OS command injection, buffer overflow, SQL injection, and cross-site scripting (XSS)
- Vulnerability management: asset inventory, CVE and CVSS scoring, the Log4Shell case study, penetration testing, and red, blue, and purple teaming
- Mobile threats, jailbreaking and rooting, and mobile device management (MDM)
- DLP operations and SIEM integration, ransomware (LockBit case study), backup and recovery
- Password hashing, salting, peppering, cracking techniques, and SSH key authentication

## Contents (52 notes)

### Active Directory Fundamentals

- [Active Directory Core Concepts](Active-Directory-Fundamentals/Active%20Directory%20Core%20Concepts.md)
- [Group Policy & Credential Storage](Active-Directory-Fundamentals/Group%20Policy%20%26%20Credential%20Storage.md)
- [Local Identity & the Scaling Problem](Active-Directory-Fundamentals/Local%20Identity%20%26%20the%20Scaling%20Problem.md)
- [Windows Authentication Hashes: LM, NTLM & NTLMv2](Active-Directory-Fundamentals/Windows%20Authentication%20Hashes%3A%20LM%2C%20NTLM%20%26%20NTLMv2.md)

### Data Loss Prevention and Exfiltration

- [Data Exfiltration and DNS Tunneling](Data-Loss-Prevention-and-Exfiltration/data-exfiltration-and-dns-tunneling.md)
- [Data Loss Fundamentals and DLP Operations](Data-Loss-Prevention-and-Exfiltration/data-loss-fundamentals-and-dlp-operations.md)
- [Interview Questions Review: Data Loss and Exfiltration](Data-Loss-Prevention-and-Exfiltration/dlp-interview-questions-review.md)
- [DLP Tools and SIEM Integration](Data-Loss-Prevention-and-Exfiltration/dlp-tools-and-siem-integration.md)

### Input Attacks and Injection

- [Blind and Advanced SQL Injection with Defences](Input-Attacks-and-Injection/blind-and-advanced-sql-injection-with-defences.md)
- [Buffer Overflow](Input-Attacks-and-Injection/buffer-overflow.md)
- [Cross-Site Scripting (XSS)](Input-Attacks-and-Injection/cross-site-scripting-xss.md)
- [Input Attacks Fundamentals](Input-Attacks-and-Injection/input-attacks-fundamentals.md)
- [OS Command Injection](Input-Attacks-and-Injection/os-command-injection.md)
- [SQL Injection Basics and In-Band Attacks](Input-Attacks-and-Injection/sql-injection-basics-and-in-band-attacks.md)

### Log Analysis and Digital Forensics

- [Core Log Types and Web Server Logs](Log-Analysis-and-Digital-Forensics/core-log-types-and-web-server-logs.md)
- [Deleted Data, Forensics, and Secure Deletion](Log-Analysis-and-Digital-Forensics/deleted-data-forensics-and-secure-deletion.md)
- [Impossible Travel and VPN Detection](Log-Analysis-and-Digital-Forensics/impossible-travel-and-vpn-detection.md)
- [Network and Authentication Logs](Log-Analysis-and-Digital-Forensics/network-and-authentication-logs.md)

### Mobile Security and Device Management

- [Jailbreaking, Rooting, and Mobile Malware](Mobile-Security-and-Device-Management/jailbreaking-rooting-and-mobile-malware.md)
- [Mobile Device Management (MDM)](Mobile-Security-and-Device-Management/mobile-device-management-mdm.md)
- [Mobile Threats and Platform Security](Mobile-Security-and-Device-Management/mobile-threats-and-platform-security.md)

### Network Security Foundations

- [Asset Awareness & Traffic Flow](Network-Security-Foundations/Asset%20Awareness%20%26%20Traffic%20Flow.md)
- [Firewalls and Network Placement](Network-Security-Foundations/firewalls-and-network-placement.md)
- [Network Access Control and Honeypots](Network-Security-Foundations/network-access-control-and-honeypots.md)
- [Proxies and Load Balancers](Network-Security-Foundations/proxies-and-load-balancers.md)
- [Router & Switch Attacks](Network-Security-Foundations/Router%20%26%20Switch%20Attacks.md)
- [Virtualization & Cloud Computing](Network-Security-Foundations/Virtualization%20%26%20Cloud%20Computing.md)
- [VLANs, Protocols, & Secure Topology Design](Network-Security-Foundations/VLANs%2C%20Protocols%2C%20%26%20Secure%20Topology%20Design.md)

### Password Security and SSH Authentication

- [Password Cracking Techniques](Password-Security-and-SSH-Authentication/Password%20Cracking%20Techniques.md)
- [Password Hashing, Salting, & Peppering](Password-Security-and-SSH-Authentication/Password%20Hashing%2C%20Salting%2C%20%26%20Peppering.md)
- [SSH Key Authentication and Privilege Escalation](Password-Security-and-SSH-Authentication/ssh-key-authentication-and-privilege-escalation.md)

### Ransomware and Backup Resilience

- [Backup Strategies and Data Recovery](Ransomware-and-Backup-Resilience/backup-strategies-and-data-recovery.md)
- [Ransomware and the LockBit Case Study](Ransomware-and-Backup-Resilience/ransomware-and-the-lockbit-case-study.md)
- [Interview Questions Review: Ransomware and Backup Resilience](Ransomware-and-Backup-Resilience/ransomware-backup-interview-questions-review.md)
- [Redundancy, High Availability, and RAID](Ransomware-and-Backup-Resilience/redundancy-high-availability-and-raid.md)

### Security Framework

- [Cyber Kill Chain](Security%20Framework/Cyber%20Kill%20Chain.md)
- [MITRE ATT&CK Framework](Security%20Framework/MITRE%20ATT%26CK%20Framework.md)
- [NIST Cybersecurity Framework (CSF)](Security%20Framework/NIST%20Cybersecurity%20Framework%20%28CSF%29.md)
- [Unified Kill Chain](Security%20Framework/Unified%20Kill%20Chain.md)

### Vulnerability Management and Security Testing

- [Log4Shell Case Study (CVE-2021-44228)](Vulnerability-Management-and-Security-Testing/log4shell-case-study.md)
- [Penetration Testing, Red Teaming, and Purple Teaming](Vulnerability-Management-and-Security-Testing/penetration-testing-red-teaming-and-purple-teaming.md)
- [Pentest Types, Phases, and Attack Surface Management](Vulnerability-Management-and-Security-Testing/pentest-types-phases-and-attack-surface-management.md)
- [Vulnerability Assessment and Risk Scoring](Vulnerability-Management-and-Security-Testing/vulnerability-assessment-and-risk-scoring.md)

### Zero Trust and Access Management

- [Access Control Models](Zero-Trust-and-Access-Management/Access%20Control%20Models.md)
- [Identity and Access Management (IAM)](Zero-Trust-and-Access-Management/Identity%20and%20Access%20Management%20%28IAM%29.md)
- [Privileged Access Management (PAM)](Zero-Trust-and-Access-Management/Privileged%20Access%20Management%20%28PAM%29.md)
- [Zero Trust Model](Zero-Trust-and-Access-Management/Zero%20Trust%20Model.md)

### General Topics

- [Pyramid of Pain](Pyramid%20of%20Pain.md)
- [Secure Network Design](Secure%20Network%20Design.md)
- [SOC Analyst Fundamentals](SOC%20Analyst%20Fundamentals.md)
- [WAF (Web Application Firewall)](WAF%20%28Web%20Application%20Firewall%29.md)
- [Web Application Security](Web%20Application%20Security.md)

---

**Author:** Bn Yasser — SOC Analyst  
[LinkedIn](https://www.linkedin.com/in/bn-yasser-424bb4215/) · [GitHub](https://github.com/ammaryasser28)

⭐ If these notes help you, consider giving the repo a star.
