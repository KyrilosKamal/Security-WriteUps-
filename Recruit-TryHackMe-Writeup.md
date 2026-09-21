Recruit — TryHackMe

Target: "Recruit"

Recruit is a hiring-portal box built around a straightforward attack chain. An exposed mail log leaks information about the application's credentials, a vulnerable file-fetch endpoint allows arbitrary local file reads through the "file://" URI scheme, and an authenticated SQL injection ultimately exposes administrator credentials.

The interesting part isn't any single vulnerability. It's how each finding leads directly to the next stage.

---

Attack Chain

HTTP Enumeration
      ↓
Exposed mail.log
      ↓
HR username + config.php disclosure
      ↓
Vulnerable file-fetch endpoint
      ↓
file:// → Arbitrary Local File Read
      ↓
config.php → HR password
      ↓
HR Authentication
      ↓
SQL Injection
      ↓
recruit_db.users
      ↓
Admin credentials
      ↓
Administrative Access

---

1. Port Scan

I started with RustScan to quickly identify the exposed attack surface and then used Nmap's service/version detection against the discovered ports.

rustscan -b 500 -a 10.114.146.247 --top -- -sCV -Pn

"rustscan output" (https://i.postimg.cc/vZJvD5nP/01-rustscan-1.png)

"rustscan output continued" (https://i.postimg.cc/MpCYX1B9/02-rustscan-2.png)

Two ports came back:

Port| Service| Version
53/tcp| DNS| ISC BIND 9.16.1 (Ubuntu)
80/tcp| HTTP| Apache httpd 2.4.41 (Ubuntu)

Port 80 was the obvious primary attack surface, so I moved on to web enumeration.

---

2. Web Enumeration

I used Feroxbuster with the standard Dirbuster medium wordlist:

feroxbuster -u http://10.114.146.247 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

"feroxbuster output" (https://i.postimg.cc/rwB9z54Y/03-feroxbuster.png)

Three findings stood out:

/mail/mail.log
/api.php
/phpmyadmin/

"phpMyAdmin" was interesting but wasn't required for the eventual attack path.

The two important discoveries were the exposed mail log and the API documentation.

---

3. Information Disclosure — Exposed Mail Log

I opened:

/mail/mail.log

The file contained an internal email conversation between HR and IT regarding the portal rollout.

"mail.log contents" (https://i.postimg.cc/XYRgX95L/04-maillog.png)

The important part was the disclosure that the HR credentials were stored in:

config.php

The information obtained at this stage was:

Username: hr
Password location: config.php

This didn't give me the password directly, but it gave me a specific file to target.

The next step was therefore to determine whether the application exposed any functionality capable of reading local files.

---

4. API Enumeration

The "/api.php" endpoint documented a CV-fetching feature:

«How can I fetch a candidate CV using the API?
"/file.php?cv=<URL>"»

The important observation was that the application accepted a user-controlled URL.

Instead of assuming the vulnerability type immediately, I tested whether alternative URI schemes were accepted by the endpoint.

The "file://" scheme was accepted.

That provided an arbitrary local file read primitive.

---

5. Proof of Concept — Arbitrary Local File Read

I tested whether I could retrieve the configuration file disclosed by the mail log:

curl "http://10.114.146.247/file.php?cv=file:///var/www/html/config.php"

"config.php source via LFI" (https://i.postimg.cc/rwB9z54b/05-config-php.png)

The application returned the PHP source of "config.php", including the HR password.

This completed the first major exploitation chain:

mail.log
    ↓
HR username
    ↓
config.php disclosed as credential location
    ↓
file.php?cv=<URL>
    ↓
file:// URI
    ↓
config.php read
    ↓
HR password

Why this matters

The issue isn't simply that the application can fetch a URL.

The application accepts an attacker-controlled URI and allows the "file://" scheme, allowing local files accessible to the web-server process to be read.

This is more accurately described here as arbitrary local file read rather than treating it as a conventional SSRF vulnerability.

---

6. Initial Access — HR Account

Using the credentials recovered from "config.php", I authenticated to the application as the "hr" user.

The HR dashboard contained the candidate interface and exposed the first flag.

"HR dashboard with flag" (https://i.postimg.cc/4xD1npcq/06-hr-dashboard-flag.png)

At this point I had authenticated access, but the dashboard exposed another interesting attack surface: the candidate search functionality.

---

7. SQL Injection Discovery

The dashboard contains a candidate search field.

I started with a basic input-manipulation test:

search=123'

The application returned a SQL-related error, indicating that the parameter was being incorporated into a database query.

Rather than relying solely on the error, I captured the request in Burp Suite for controlled testing.

"Burp SQLi request" (https://i.postimg.cc/t4Kz7F6L/07-burp-sqli.png)

I saved the request as:

request.txt

---

8. SQL Injection Exploitation

I passed the captured request to sqlmap:

sqlmap -r request.txt -dbs

Sqlmap confirmed that the parameter was injectable and identified:

- Boolean-based blind SQL injection
- UNION-based SQL injection
- 4 columns
- MySQL as the backend DBMS

The discovered databases included:

information_schema
mysql
performance_schema
phpmyadmin
recruit_db
sys

The application-specific database was:

recruit_db

---

9. Database Enumeration

I enumerated the tables inside "recruit_db":

sqlmap -r request.txt -D recruit_db --tables

The relevant tables were:

+------------+
| candidates |
| users      |
+------------+

The "users" table was the obvious next target because it could contain application authentication data.

I dumped it with:

sqlmap -r request.txt -D recruit_db -T users --dump

"sqlmap dump of users table" (https://i.postimg.cc/t4Kz7FPj/08-sqlmap-dump.png)

The dump exposed administrator credentials.

At this point the SQL injection had provided a direct path from the authenticated HR account to administrative credentials.

---

10. Administrative Access

I logged out of the HR session and authenticated using the administrator credentials recovered from the database.

The second flag was then accessible from the administrative interface.

[REDACTED]

---

11. Complete Attack Chain

The entire compromise can be summarized as:

1. Port 80 discovered
        ↓
2. Web enumeration
        ↓
3. mail.log discovered
        ↓
4. HR username + config.php location disclosed
        ↓
5. API endpoint discovered
        ↓
6. file:// accepted by file.php
        ↓
7. config.php read
        ↓
8. HR password obtained
        ↓
9. HR authentication
        ↓
10. Candidate search discovered
        ↓
11. SQL injection confirmed
        ↓
12. recruit_db enumerated
        ↓
13. users table dumped
        ↓
14. Admin credentials obtained
        ↓
15. Administrative access

The important part of the attack wasn't any single command. Each stage provided the information or access required to reach the next one.

---

12. Flags

Flag| Value
User| "[REDACTED]"
Admin| "[REDACTED]"

---

13. Security Findings

Finding 1 — Publicly Accessible Internal Log

The application exposed an internal mail log through the web server.

Impact

An unauthenticated attacker could retrieve internal application information and use it to guide further exploitation.

Remediation

- Store internal logs outside the web-accessible document root.
- Restrict access to application and system logs.
- Review web-server directory exposure.
- Avoid placing sensitive operational information in publicly accessible files.

---

Finding 2 — Arbitrary Local File Read

The "file.php" endpoint accepted a user-controlled URL and allowed the "file://" URI scheme.

Impact

An attacker could read local files accessible to the web-server process.

Potential targets could include:

- Application configuration files
- Source code
- Credentials
- Environment/configuration data
- Other sensitive local files

Remediation

- Do not allow arbitrary user-controlled file paths or URLs.
- Allowlist permitted resources instead of accepting arbitrary URIs.
- Disable unsupported URI schemes such as "file://".
- Canonicalize and validate file paths before processing them.
- Run the application with least-privilege filesystem permissions.

---

Finding 3 — Plaintext Credential Storage

The HR password was stored in plaintext inside the application configuration.

Impact

Once the configuration file became readable, the HR account could be compromised immediately.

Remediation

- Never store user passwords in plaintext.
- Store passwords using a strong password-hashing algorithm.
- Protect application secrets using appropriate secret-management mechanisms.
- Rotate credentials after exposure.

---

Finding 4 — SQL Injection

The candidate search parameter was vulnerable to SQL injection.

The injection supported both boolean-based blind and UNION-based techniques.

Impact

An authenticated attacker could enumerate the database and extract application data, including administrator credentials.

Remediation

- Use parameterized queries / prepared statements.
- Never concatenate user-controlled input directly into SQL statements.
- Apply server-side input validation as a secondary control.
- Use a database account with only the privileges required by the application.

---

Finding 5 — Administrative Credential Exposure

The "users" table contained administrator authentication data that could be extracted through the SQL injection.

Impact

An attacker who reached the SQL injection could obtain administrative credentials and take over the administrative interface.

Remediation

- Store passwords using a strong password-hashing algorithm.
- Never store recoverable plaintext passwords.
- Enforce strong, unique administrative credentials.
- Implement additional authentication controls such as MFA where appropriate.

---

14. Lessons Learned

The biggest lesson from Recruit was that vulnerabilities shouldn't always be viewed in isolation.

The initial finding was just an exposed log file:

mail.log

But that file disclosed the location of a credential.

The application then provided a way to read that credential:

file.php?cv=file://...

The recovered credentials provided authenticated access.

That authenticated access exposed SQL injection:

search=123'

And the SQL injection exposed the administrator account.

The complete chain was:

Information Disclosure
        ↓
Local File Read
        ↓
Credential Disclosure
        ↓
Authentication
        ↓
SQL Injection
        ↓
Database Enumeration
        ↓
Credential Disclosure
        ↓
Administrative Access

The useful takeaway is the methodology:

Enumerate
   ↓
Identify interesting information
   ↓
Understand how the application uses it
   ↓
Validate the suspected weakness
   ↓
Exploit it
   ↓
Use the resulting access to enumerate again
   ↓
Repeat
