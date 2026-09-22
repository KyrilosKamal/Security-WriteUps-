# Recruit (TryHackMe)

`Web` `LFI` `SQL Injection` `Information Disclosure` `Easy`

**Target:** `Recruit`

Recruit bills itself as a recruitment portal, and functionally it is one — candidates, statuses, an HR view, an admin view. But almost every corner of it leaks something it shouldn't. A log file tells you where the password lives. A "fetch a CV" feature turns into arbitrary file read. And the search box just... trusts you. I didn't need anything clever here, just patience to follow one leak into the next.

## Port scan

Rustscan first, because waiting on a full nmap sweep before I even know what's open feels like a waste of a coffee break:

```
rustscan -b 500 -a 10.114.146.247 --top -- -sCV -Pn
```

![rustscan output](https://i.postimg.cc/vZJvD5nP/01-rustscan-1.png)
![rustscan output continued](https://i.postimg.cc/MpCYX1B9/02-rustscan-2.png)

Two ports, nothing surprising:

| Port | Service | Version |
|------|---------|---------|
| 53/tcp | domain | ISC BIND 9.16.1 (Ubuntu) |
| 80/tcp | http | Apache httpd 2.4.41 (Ubuntu) |

DNS wasn't going to give me anything useful on a box like this, so port 80 was the entire attack surface.

## Digging through the web root

Feroxbuster with the standard dirbuster medium wordlist:

```
feroxbuster -u http://10.114.146.247 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![feroxbuster output](https://i.postimg.cc/rwB9z54Y/03-feroxbuster.png)

Buried in the usual pile of assets and locale folders, three things actually mattered:

- `/mail/mail.log` — a mail log. Just sitting there. Publicly readable.
- `/api.php` — documentation for some internal API
- `/phpmyadmin/` — a DB admin panel I never ended up needing, but nice to know it's there

## The log file gives away the game

`/mail/mail.log` turned out to be a real internal email thread between HR and IT about the portal's rollout.

![mail.log contents](https://i.postimg.cc/XYRgX95L/04-maillog.png)

One line does all the work here:

> HR login credentials (username: `hr`) are currently stored in the application configuration file (`config.php`) for ease of access during the initial rollout phase.

"For ease of access during the initial rollout phase" is doing a lot of heavy lifting in that sentence. Now I had a username and I knew exactly which file to go read. I just needed a way to read it.

## api.php points at the bug

`/api.php` documents a CV-fetching feature:

> **How can I fetch a candidate CV using the API?**
> `/file.php?cv=<URL>`

A parameter that takes an arbitrary URL and fetches it is basically an invitation. My guess for the backend is something like `file_get_contents($_GET['cv'])` or `fopen()` fed the raw request straight — PHP's stream wrappers (`file://`, `http://`, `php://`, and friends) are all enabled by default unless someone explicitly locks them down, and nothing here checks the scheme or restricts it to `http(s)://`. So instead of a CV URL, I can hand it a local path with `file://` and let the server read its own disk back to me.

## Reading config.php

```
curl "http://10.114.146.247/file.php?cv=file:///var/www/html/config.php"
```

![config.php source via LFI](https://i.postimg.cc/rwB9z54b/05-config-php.png)

Straight to the raw PHP source, HR password and all (redacted above). Combined with the username from the mail log, that's a working login.

## Flag 1 — logging in as HR

Logged in, landed on `dashboard.php`. Candidate table, and sitting right in a banner above it, the flag:

![HR dashboard with flag](https://i.postimg.cc/4xD1npcq/06-hr-dashboard-flag.png)

Nice — but the more interesting thing on this page wasn't the flag. It was the search box underneath it.

## The search box is the real way in

Threw a stray `'` at it (`search=123'`) and got a SQL error back. That's the tell: a broken quote in user input reaching a database error means the query is being built by concatenating the raw parameter straight into the SQL string — something like `SELECT * FROM candidates WHERE name LIKE '%$search%'` — instead of using a parameterized query. The unclosed quote breaks the syntax, and the fact that MySQL's error makes it back into the response confirms both the injection point and that error output isn't being suppressed here.

Grabbed the request in Burp and saved it to `request.txt`:

![Burp SQLi request](https://i.postimg.cc/t4Kz7F6L/07-burp-sqli.png)

From there I let sqlmap do the actual work:

```
sqlmap -r request.txt -dbs
```

It confirmed boolean-based blind and UNION-based injection, 4 columns, MySQL on the back end. sqlmap gets to that column count with an `ORDER BY` probe — bump `ORDER BY N` until the query errors, and whatever N you land on is how many columns the original `SELECT` returns. Four lines up neatly with the four fields the candidates table actually shows (ID, Name, Position, Status), which is also why a `UNION SELECT` payload comes back clean instead of erroring on a mismatch. Database list:

```
information_schema, mysql, performance_schema, phpmyadmin, recruit_db, sys
```

`recruit_db` was obviously the one worth looking at:

```
sqlmap -r request.txt -D recruit_db --tables
```

```
+------------+
| candidates |
| users      |
+------------+
```

`users` is never subtle about what it wants you to do next.

```
sqlmap -r request.txt -D recruit_db -T users --dump
```

![sqlmap dump of users table](https://i.postimg.cc/t4Kz7FPj/08-sqlmap-dump.png)

Admin credentials, straight out of the database, password redacted above.

## Flag 2 — admin login

Logged out of the HR session, logged back in as admin. Second flag showed up right away:

```
[REDACTED]
```

## How it all chains together

1. Recon finds HTTP on port 80
2. Directory brute-force finds `mail.log` and `api.php`
3. `mail.log` leaks the HR username and points at `config.php` for the password
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

| Severity | Issue |
|----------|-------|
| **Medium** | `mail.log` publicly reachable — leaks internal comms and a credential-storage hint |
| **High** | `file.php` trusts user-supplied URLs; `file://` wrapper allows arbitrary local file read (LFI) |
| **Medium** | HR password stored in plaintext in `config.php` |
| **Critical** | `search` parameter on `dashboard.php` is SQL-injectable (boolean-blind + UNION-based) — full DB read |
| **High** | Admin password is weak/guessable once the DB is dumped |

None of these are individually exotic. What makes the box work is that they chain: the log tells you where to look, the LFI gets you there, and the SQLi finishes the job. Fix any one link and the rest of the chain stops mattering.
