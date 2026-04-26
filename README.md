# HTB NewsPortal — Penetration Test Report

> **CONFIDENTIAL | HTB Lab | 2026-04-25**

| Field | Details |
|---|---|
| **Target** | 172.17.0.2 (NewsPortal – HTB) |
| **Engagement Type** | Black-Box Web Application Penetration Test |
| **Date** | 2026-04-25 |
| **Assessor** | Aravinda A Kumar |
| **Certifications** | OSCP+ \| CEH v12 Master |
| **Classification** | CONFIDENTIAL |

> This document contains sensitive security findings obtained during an authorised penetration test. Distribution is restricted to authorised personnel only.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope & Methodology](#2-scope--methodology)
3. [Technical Findings](#3-technical-findings)
   - [3.1 Reconnaissance & Enumeration](#31-reconnaissance--enumeration)
   - [3.2 Application Mapping](#32-application-mapping)
   - [F-01 SQL Injection – Search Endpoint](#f-01-sql-injection--search-endpoint-newsportalsearchphp)
   - [F-02 Unrestricted PHP File Upload → RCE](#f-02-unrestricted-php-file-upload--remote-code-execution)
   - [F-03 Sudo Privilege Escalation via /usr/bin/find](#f-03-sudo-privilege-escalation-via-usrbinfind-nopasswd)
   - [F-04 Exposed Admin Panel](#f-04-exposed-admin-panel-no-rate-limiting)
   - [F-05 Apache Default Page](#f-05-apache-default-page-exposed-on-root)
4. [Attack Chain Summary](#4-attack-chain-summary)
5. [Remediation Summary](#5-remediation-summary)
6. [Appendix – Raw Tool Output & Evidence](#6-appendix--raw-tool-output--evidence)

---

## 1. Executive Summary

An infrastructure and web application penetration test was performed against the target host **172.17.0.2 (NewsPortal)**, a Linux-based system running Apache and PHP. The assessment followed a black-box methodology beginning with reconnaissance and culminating in **full root-level compromise** of the target.

Two critical vulnerabilities were identified and exploited in a chained attack: a SQL injection vulnerability in the search functionality was leveraged to extract administrator credentials from the database, which in turn enabled authenticated file upload of a PHP web shell, yielding remote code execution. Privilege escalation to root was achieved via a misconfigured sudo entry granting unrestricted execution of `/usr/bin/find`.

### Findings Summary

| Severity | Count | Findings |
|---|---|---|
| 🔴 **CRITICAL** | 2 | SQL Injection (Search), PHP File Upload RCE |
| 🟠 **HIGH** | 1 | Sudo Privilege Escalation via `/usr/bin/find` |
| 🟡 **MEDIUM** | 1 | Exposed Admin Panel (No Rate Limiting) |
| 🔵 **INFO** | 1 | Apache Default Page Exposed on Root |

---

## 2. Scope & Methodology

### 2.1 Scope

| Asset | IP Address | Ports | Notes |
|---|---|---|---|
| NewsPortal Web App | 172.17.0.2 | 22/tcp, 80/tcp | Docker container – HTB lab environment |

### 2.2 Methodology

The assessment followed a structured black-box penetration testing approach aligned with industry frameworks (PTES, OWASP Testing Guide). The following phases were executed:

- **Reconnaissance & Enumeration** — Port scanning (Nmap), service fingerprinting, directory/file enumeration (Gobuster), application mapping.
- **Vulnerability Identification** — Manual testing for injection flaws, authentication weaknesses, and file upload controls; automated scanning with SQLMap.
- **Exploitation** — SQL injection to credential extraction; authenticated PHP reverse shell upload; netcat reverse shell.
- **Post-Exploitation & Privilege Escalation** — Sudo enumeration, GTFOBins-based root escalation via `/usr/bin/find`.
- **Reporting** — Evidence collection, risk rating (CVSS v3.1), and remediation guidance.

---

## 3. Technical Findings

### 3.1 Reconnaissance & Enumeration

Initial reconnaissance began with a full TCP port scan against the target to identify exposed services, followed by service version detection and web directory enumeration.

#### Nmap Full Port Scan

```bash
nmap -p- --open -vvv --min-rate 4000 -oN newsportal_ports.nmap 172.17.0.2
```

Two open TCP ports were identified:

| Port | Protocol | Service | Version |
|---|---|---|---|
| 22 | TCP | SSH | OpenSSH 8.9p1 Ubuntu 3ubuntu0.14 |
| 80 | TCP | HTTP | Apache httpd 2.4.52 (Ubuntu) |

*Figure 1 – Nmap full port scan output (172.17.0.2)*

<img width="429" height="438" alt="image" src="https://github.com/user-attachments/assets/df278ec8-73a8-471f-880b-c9cb1c99bf92" />


*Figure 2 – Nmap service version detection results*

<img width="675" height="305" alt="image" src="https://github.com/user-attachments/assets/4d3fbf91-7fc9-418e-b138-0af34ae2156b" />


#### Web Directory Enumeration

HTTP port 80 initially returned the Apache2 default page. Gobuster was used to enumerate hidden directories and PHP files.

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories-lowercase.txt -x php -t 50
```

Gobuster discovered the `/newsportal/` application directory (HTTP 301 redirect):

*Figure 3 – Apache default page on port 80*

<img width="458" height="480" alt="image" src="https://github.com/user-attachments/assets/41ce955e-9596-47e8-b1b7-0e7609cd786c" />


*Figure 4 – Gobuster directory enumeration results*

<img width="406" height="372" alt="image" src="https://github.com/user-attachments/assets/211fb315-1fe5-4c72-a6e1-d15dea69bced" />


---

### 3.2 Application Mapping

Navigating to `http://172.17.0.2/newsportal/` revealed a PHP-based News Portal application with user registration, login, an admin panel, and a news search function. Self-registration was enabled at `/newsportal/register.php`, allowing creation of a standard user account.

| Endpoint | URL |
|---|---|
| Login page | `/newsportal/index.php` |
| Registration | `/newsportal/register.php` |
| User home | `/newsportal/userhome.php` |
| Admin login | `/newsportal/adminlogin.php` |
| Search endpoint | `/newsportal/search.php` (POST parameter: `text`) |
| Add news (admin) | `/newsportal/add_news.php` |

*Figure 5 – NewsPortal login page*

<img width="473" height="412" alt="image" src="https://github.com/user-attachments/assets/9ed347d2-d5ea-4ad9-bd6a-2e479f1de48d" />


*Figure 6 – User registration page*

<img width="473" height="412" alt="image" src="https://github.com/user-attachments/assets/c46183bc-bebf-4081-9360-2419b4f49c56" />


*Figure 7 – Authenticated user home page (/newsportal/userhome.php)*

<img width="495" height="433" alt="image" src="https://github.com/user-attachments/assets/cc4897d6-e734-4350-b5c0-db0ad9390c1a" />


---

### F-01 SQL Injection – Search Endpoint (/newsportal/search.php)

> 🔴 **CRITICAL**

| Field | Details |
|---|---|
| **CVSS Score** | 9.8 |
| **CVSS Vector** | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H |
| **Severity** | CRITICAL |

#### Description

The POST parameter `text` in `/newsportal/search.php` is vulnerable to multiple SQL injection attack types including **boolean-based blind**, **time-based blind**, and **UNION-based injection**. The back-end DBMS was confirmed as MySQL running on Ubuntu 22.04. No input sanitisation or parameterised queries were observed.

The vulnerability was initially confirmed manually using classic boolean payloads (`' OR 1=1-- -` returning all posts vs `' OR 1=2-- -` returning none), then exploited fully via SQLMap.

#### Impact

Full database exfiltration is possible without authentication. The `newsportal` database was enumerated revealing five tables: `admin`, `user`, `category`, `news`, `subcategory`. The `admin` table yielded two sets of MD5-hashed credentials which were cracked offline, granting full administrative access to the application.

#### Evidence

*Figure 8 – Burp Suite request to /newsportal/search.php (intercepted)*

<img width="501" height="440" alt="image" src="https://github.com/user-attachments/assets/8937607e-87ce-41f2-b51b-4390df8b60ef" />

*Figure 9 – SQLMap confirming boolean-based blind, time-based blind, and UNION injection*

<img width="676" height="409" alt="image" src="https://github.com/user-attachments/assets/f62054a4-ceb3-4ca8-8c86-a78a04d17de5" />


*Figure 10 – SQLMap database enumeration: 5 databases identified*

<img width="676" height="409" alt="image" src="https://github.com/user-attachments/assets/a8aa2ce1-b588-4c9a-8dac-f76e773a8e55" />


*Figure 11 – SQLMap table enumeration for 'newsportal' database*

<img width="676" height="409" alt="image" src="https://github.com/user-attachments/assets/0689dcf8-4ea8-4015-8dc3-64cff11e54a8" />


*Figure 12 – SQLMap admin table dump: cracked credentials (password123 / editor123)*

<img width="595" height="439" alt="image" src="https://github.com/user-attachments/assets/2dc17527-ca08-4502-a9d6-6842ce46cf3d" />

#### Remediation

- Implement **parameterised queries / prepared statements** for all database interactions.
- Enforce strict input validation and use a Web Application Firewall (WAF).
- Hash passwords using a strong adaptive algorithm (**bcrypt, Argon2**) with appropriate work factors rather than unsalted MD5.

---

### F-02 Unrestricted PHP File Upload → Remote Code Execution

> 🔴 **CRITICAL**

| Field | Details |
|---|---|
| **CVSS Score** | 9.0 |
| **CVSS Vector** | AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H |
| **Severity** | CRITICAL |

#### Description

The admin panel's **Add News** feature at `/newsportal/add_news.php` accepts a thumbnail file upload with **no server-side restriction on file type**. A PHP reverse shell named `shell.php` was uploaded successfully. Upon triggering the uploaded file by visiting the associated news item's URL, the payload executed and established an inbound netcat connection to the attacker's listener on port 1234, resulting in a non-interactive shell as `www-data` (uid=33).

#### Impact

Remote code execution achieved on the underlying server. Combined with the SQL injection finding, the complete attack chain from unauthenticated network access to operating system shell required no brute-force or zero-day exploitation — only chaining of two application-layer misconfigurations.

#### Evidence

*Figure 13 – Admin 'Add News' panel with shell.php selected for upload*

<img width="623" height="440" alt="image" src="https://github.com/user-attachments/assets/9f92365e-a7db-4d46-814a-d16183f76154" />


*Figure 14 – Netcat listener receiving reverse shell as www-data (uid=33/www-data)*

<img width="679" height="367" alt="image" src="https://github.com/user-attachments/assets/295fe842-8162-4a3b-a990-bdbb90af82bd" />

*Figure 15 – TTY upgrade via script; shell stabilised with stty raw -echo;fg*

<img width="679" height="367" alt="image" src="https://github.com/user-attachments/assets/04e29fd4-82e7-49ae-9857-b5fb1a21dcbb" />


#### Remediation

- Restrict file uploads to permitted MIME types and extensions (e.g., `jpg`, `png`, `gif` only).
- Store uploaded files **outside the web root** or in a non-executable directory.
- Rename uploaded files server-side to a random, extension-stripped name.
- Enforce Content-Type validation and **magic byte checks**.
- Apply the principle of least privilege to the web application process.

---

### F-03 Sudo Privilege Escalation via /usr/bin/find (NOPASSWD)

> 🟠 **HIGH**

| Field | Details |
|---|---|
| **CVSS Score** | 7.8 |
| **CVSS Vector** | AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H |
| **Severity** | HIGH |

#### Description

Post-exploitation enumeration revealed that the `www-data` service account is permitted to execute `/usr/bin/find` as root without a password via the sudo configuration (`NOPASSWD: /usr/bin/find`). The `find` binary supports an `-exec` flag which spawns arbitrary processes. This was exploited using the GTFOBins technique to spawn a privileged bash shell:

```bash
sudo find . -exec /bin/bash -p \; -quit
```

#### Impact

**Full root-level compromise** of the target host. Both the user flag (`/home/newsadmin/user.txt`) and root flag (`/root/root.txt`) were successfully retrieved, confirming unrestricted access to all system resources, configuration files, credentials, and data.

#### Evidence

*Figure 16 – sudo -l output: www-data has NOPASSWD /usr/bin/find*

<img width="673" height="192" alt="image" src="https://github.com/user-attachments/assets/cd460fba-f701-4a88-b9a2-a12e0a8395b2" />


*Figure 17 – GTFOBins: find sudo shell spawn technique*

<img width="675" height="399" alt="image" src="https://github.com/user-attachments/assets/98626381-1cba-4691-ae61-9d8449865c18" />


*Figure 18 – Root shell confirmed; user.txt and root.txt captured*

<img width="675" height="254" alt="image" src="https://github.com/user-attachments/assets/c8117f09-41b5-46b8-a1dc-e6bf44f5ae25" />


#### Remediation

- Remove the `NOPASSWD` sudo entry for `/usr/bin/find`.
- Audit all sudo rules and apply the principle of least privilege — web application service accounts should not have any sudo rights.
- If elevated access is operationally required, restrict to the minimum necessary binary with **no ability to spawn subprocesses**.

---

### F-04 Exposed Admin Panel (No Rate Limiting)

> 🟡 **MEDIUM**

| Field | Details |
|---|---|
| **Severity** | MEDIUM |

#### Description

The admin login panel at `/newsportal/adminlogin.php` is publicly accessible from the internet with no rate limiting or account lockout policy in place.

#### Remediation

- Restrict the admin panel to **internal IPs** only via firewall rules or `.htaccess`.
- Enable **multi-factor authentication (MFA)** for admin accounts.
- Implement rate limiting and account lockout on login endpoints.

---

### F-05 Apache Default Page Exposed on Root

> 🔵 **INFO**

| Field | Details |
|---|---|
| **Severity** | INFO |

#### Description

The Apache2 default page is accessible at the web root (`http://172.17.0.2/`), disclosing the server technology stack and version information to unauthenticated users.

#### Remediation

- Remove or replace the default Apache page.
- Minimise information disclosure by suppressing version banners in Apache configuration (`ServerTokens Prod`, `ServerSignature Off`).

---

## 4. Attack Chain Summary

| Step | Action | Tool / Technique | Outcome |
|---|---|---|---|
| 1 | Port & Service Scan | `nmap -p- --min-rate 4000 -sV` | Ports 22, 80 discovered; Apache 2.4.52 |
| 2 | Directory Enumeration | Gobuster + raft-large wordlist | `/newsportal/` application found |
| 3 | Application Mapping | Browser / Manual testing | Login, register, search, admin panel identified |
| 4 | SQL Injection Discovery | Manual payload (`OR 1=1`) | Search endpoint confirmed injectable |
| 5 | Database Exfiltration | `SQLMap -r req.txt --dbs --dump` | Admin creds extracted & cracked |
| 6 | Admin Login | Browser – `adminlogin.php` | Full admin access to CMS |
| 7 | PHP Shell Upload | Admin 'Add News' – `shell.php` | Reverse shell payload uploaded |
| 8 | Reverse Shell | `nc -nlvp 1234` | Shell as `www-data` |
| 9 | Sudo Enumeration | `sudo -l` | `NOPASSWD /usr/bin/find` found |
| 10 | Root Escalation | `find . -exec /bin/bash -p \; -quit` | Root shell obtained; flags captured |

---

## 5. Remediation Summary

| Priority | Finding | Remediation Action | Effort |
|---|---|---|---|
| 🔴 CRITICAL | SQL Injection | Parameterised queries; WAF; re-hash passwords with bcrypt/Argon2 | Medium |
| 🔴 CRITICAL | PHP File Upload RCE | Whitelist extensions; store uploads outside web root; disable execution | Low |
| 🟠 HIGH | Sudo Misconfiguration | Remove NOPASSWD sudo entry; audit all sudoers; apply least privilege | Low |
| 🟡 MEDIUM | Exposed Admin Panel | Restrict admin panel to internal IPs; enable MFA; rate-limit logins | Low |
| 🔵 INFO | Apache Default Page | Remove or replace default Apache page; minimise information disclosure | Trivial |

---

## 6. Appendix – Raw Tool Output & Evidence

### A. Nmap Raw Output

```
nmap -p- --open -vvv --min-rate 4000 -oN newsportal_ports.nmap 172.17.0.2

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 02:42:AC:11:00:02 (Unknown)

nmap -sS -sV -p 22,80 -vvv -oA newsportal 172.17.0.2

22/tcp open  ssh   OpenSSH 8.9p1 Ubuntu 3ubuntu0.14 (Ubuntu Linux; protocol 2.0)
80/tcp open  http  Apache httpd 2.4.52 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### B. SQLMap Commands Executed

```bash
sqlmap -r req.txt -p text --batch --level 5 --risk 3
sqlmap -r req.txt -p text --batch --level 5 --risk 3 --dbs
sqlmap -r req.txt -p text --batch --level 5 --risk 3 -D newsportal --tables
sqlmap -r req.txt -p text --batch --level 5 --risk 3 -D newsportal -T admin --dump
```

### C. Privilege Escalation Commands

```bash
www-data@2b01be721078:/$ sudo -l
(root) NOPASSWD: /usr/bin/find

www-data@2b01be721078:/$ sudo find . -exec /bin/bash -p \; -quit

root@2b01be721078:/# id
uid=0(root) gid=0(root) groups=0(root)

root@2b01be721078:/# cat /home/newsadmin/user.txt
4c9604985bd84f659c2382bcb5509784

root@2b01be721078:/# cat /root/root.txt
f80b5016db9c6e51af5e9b726e81b086
```

### D. Credentials Recovered

| Username | Hash (MD5) | Cracked Password |
|---|---|---|
| admin@newsportal.htb | `cbfdac6008f9cab4083784cbd1874f76618d2a97` | `password123` |
| editor@newsportal.htb | `ef2ea8f0684b26279444be0cfc7ac395cc75df89` | `editor123` |

---

*Report prepared by: **Aravinda A Kumar** | OSCP+ | CEH v12 Master*
*Date: 2026-04-25 | Classification: CONFIDENTIAL*
