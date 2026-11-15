# Recruit — TryHackMe Writeup

`Web` `Information Disclosure` `Server-Side File Read` `SQL Injection` `Easy`

**Platform:** TryHackMe  
**Room:** Recruit  
**Target:** `<TARGET_IP>` (replace with the room's current target IP)

> This walkthrough documents an authorized TryHackMe lab. Replace expired target details and verify commands against the current room instance before reuse. Credentials and flags are intentionally redacted.

## Overview

The room demonstrates how multiple web weaknesses can be chained:

1. Enumerate exposed web resources.
2. Read an exposed mail log that discloses an HR username and points to an application configuration file.
3. Abuse a URL-fetching endpoint to read a local server file.
4. Use the recovered HR credentials to access the application.
5. Identify and exploit SQL injection in a dashboard search function.
6. Access the administrative account using credentials exposed by the database.

The important lesson is the relationship between information disclosure, server-side file retrieval, and SQL injection. Each finding provides context for the next step.

## 1. Port scanning

```bash
rustscan -b 500 -a <TARGET_IP> --top -- -sCV -Pn
```

The original lab run identified:

| Port | Service | Version |
|---|---|---|
| 53/tcp | DNS | ISC BIND 9.16.1 (Ubuntu) |
| 80/tcp | HTTP | Apache httpd 2.4.41 (Ubuntu) |

These are observations from the recorded room instance, not a guarantee that a new instance will expose identical versions or ports.

## 2. Web content discovery

```bash
feroxbuster -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The scan identified several relevant paths:

- `/mail/mail.log`
- `/api.php`
- `/phpmyadmin/`

The exposed mail log contained an internal message identifying the HR username and indicating that the corresponding password was stored in `config.php`.

**Finding:** Public access to internal mail logs can disclose operational details and help an attacker target other weaknesses.

## 3. Analyze the API documentation

The observed `/api.php` documentation described a CV-fetching endpoint:

```text
/file.php?cv=<URL>
```

This suggests a server-side URL-fetching feature. The behavior should be tested carefully to determine whether the application restricts schemes, hosts, paths, and redirects.

The original lab test supplied a local file URL. The result demonstrated that the endpoint could retrieve a local server-side file in this challenge environment.

This is more precisely described as **arbitrary local file read through an unsafe server-side URL-fetching feature**. Calling it conventional path-based LFI may be imprecise because the input is a URL and the server-side fetch implementation is not shown.

## 4. Read the application configuration

In the authorized room instance, the following request returned the PHP configuration source:

```bash
curl "http://<TARGET_IP>/file.php?cv=file:///var/www/html/config.php"
```

The response exposed the HR password in the lab. The value is redacted from this public writeup.

The request demonstrates the impact of allowing an untrusted URL-fetch parameter to access local file resources. The exact backend function (for example, `file_get_contents`) should not be asserted without source-code evidence.

## 5. HR application access

Using the username disclosed in the mail log and the password recovered from the configuration file, the HR account could access the dashboard.

**Flag 1:** `[REDACTED]`

The dashboard also contained a search function. This became the next attack surface.

## 6. SQL injection discovery

A quote character in the search parameter caused a database error in the observed lab response. This was an initial indication that user input might be affecting a SQL query.

A database error alone is not sufficient to prove SQL injection. The finding was subsequently tested with sqlmap using a request captured in Burp Suite.

```bash
sqlmap -r request.txt --dbs
```

The recorded run reported boolean-based blind and UNION-based SQL injection against a MySQL backend, with a four-column query context.

The number of columns should be treated as a tool-validated result for this specific request. Do not infer it solely from the number of visible table fields.

### Enumerate the database

```bash
sqlmap -r request.txt -D recruit_db --tables
```

The recorded output listed:

```text
candidates
users
```

### Extract the users table

```bash
sqlmap -r request.txt -D recruit_db -T users --dump
```

The lab database dump exposed administrative credentials. Password material is redacted.

## 7. Administrative access

The recovered administrative credentials were used to access the application's admin area.

**Flag 2:** `[REDACTED]`

## Attack-chain summary

```text
Web enumeration
    |
    +--> Public mail log
    |       |
    |       +--> HR username and config-file disclosure hint
    |
    +--> API documentation
            |
            +--> URL-fetch endpoint
                    |
                    +--> Local config file read
                            |
                            +--> HR credentials
                                    |
                                    +--> HR dashboard
                                            |
                                            +--> SQL injection
                                                    |
                                                    +--> Database credentials
                                                            |
                                                            +--> Admin access
```

## Findings and remediation

| Finding | Impact | Recommended remediation |
|---|---|---|
| Publicly accessible mail log | Internal information disclosure | Store logs outside the web root; deny public access; remove sensitive data from logs |
| Unsafe server-side URL fetch | Local file disclosure and potentially broader server-side request risks | Allowlist required destinations and schemes; block local/private addresses and redirects; enforce strict path and protocol validation |
| Plaintext application credential in configuration | Credential disclosure if the file is exposed | Use a secrets manager or protected environment configuration; rotate exposed credentials; apply least privilege |
| SQL injection in search parameter | Unauthorized database access and possible credential disclosure | Use parameterized queries; avoid dynamic SQL concatenation; restrict database account privileges; avoid exposing database errors |
| Weak or exposed administrative credentials | Unauthorized administrative access | Use unique strong credentials, secure password hashing, MFA where available, and credential rotation |

Severity depends on exploitability, exposure, and business impact in the real deployment. The table describes the lab findings and does not represent a formal CVSS assessment.

## Lessons learned

- Directory and content discovery can reveal sensitive files and useful application documentation.
- Internal logs can disclose usernames, architecture, and operational details.
- A server-side URL-fetching feature must be treated as a security-sensitive input.
- Validate suspected SQL injection with reproducible request/response evidence.
- Chaining vulnerabilities can have a much greater impact than assessing each weakness in isolation.

## Evidence to retain

For a reproducible portfolio report, include sanitized copies of:

- Port-scan output.
- Feroxbuster findings.
- Relevant mail-log excerpt.
- API documentation response.
- File-fetch request and redacted response.
- Burp request used for SQL injection testing.
- Sqlmap output showing the confirmed injection technique and database.
- Screenshots of the two lab flags with flag values redacted.

Remove session cookies, passwords, tokens, and any personal information before publishing.
