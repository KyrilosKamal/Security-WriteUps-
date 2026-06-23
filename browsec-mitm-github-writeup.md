# How a VPN Extension Hijacked My GitHub Session (and How I Caught It)

_A real-world browser extension MITM attack, caught live through DevTools._

---

## Background

So there I was, just checking my GitHub dashboard like any other day. You know the drill coffee in hand, scrolling through notifications, feeling productive. 

Except something felt... off.

The page loaded, sort of. The feed showed up, but the left sidebar? Just spinning. And spinning. I clicked around, nothing worked. The whole thing was frozen solid.

My first thought? "Ugh, GitHub's probably having issues again" or maybe my internet's acting up. 

---

## The Symptom

Here's what I was dealing with:

- GitHub dashboard loaded but **that sidebar spinner just wouldn't quit**
- Left column (where my repos should've been) — completely blank
- Feed showed some stuff, but not all of it
- Nothing worked — buttons, links, navigation, all dead
- No error messages, no GitHub status alerts, nothing

And honestly? That's the scary part. This wasn't some obvious "YOUR COMPUTER HAS BEEN LOCKED" ransomware nonsense. It just looked like a broken webpage. The kind of thing that makes you hit F5 and move on.

---

## DevTools Investigation

Instead of refreshing like a normal person, I hit F12 and opened DevTools. The Console tab immediately told me this wasn't a "normal" problem.

First thing that jumped out:

```
Access to CSS stylesheet at 'https://infird.com/cdn/primer-react-css.d18392e72f73b5a6.module.css'
from origin 'https://github.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

Wait, what? **`infird.com`**? GitHub's never used that domain. Ever.

GitHub serves all its stuff from `github.githubassets.com`. That's it. Any other domain is immediately suspicious.

And it wasn't just one error. My console was getting absolutely spammed. Hundreds of errors, all the same story — every single JavaScript chunk and CSS file GitHub needed was being requested from `infird.com` instead of the real CDN:

```
Access to script at 'https://infird.com/cdn/chunk-63160-d2b6073aa911a9ad.js'
from origin 'https://github.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.

Access to script at 'https://infird.com/cdn/primer-react-1cbafc24ef5f5e7d.js'
from origin 'https://github.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

After trying and failing multiple times, GitHub's webpack bundler finally gave up and threw fatal errors:

```
css loading:30 Uncaught (in promise) Error: Loading chunk 90225 failed after 3 retries.
    at t.onerror.t.onload (css loading:30:14)

jsonp_chunk_loading:28 Uncaught (in promise) ChunkLoadError: Loading chunk 51591 failed after 3 retries.
    at b.f.j (jsonp_chunk_loading:28:16)
```

At this point, the page was basically dead. React can't do its thing without those chunks, so the UI just froze completely.

Oh, and one more interesting line:

```
Tracking Prevention blocked an XHR request to
https://www.google-analytics.com/mp/collect?measurement_id=G-03XW3FWG7L
```

This wasn't directly related to the attack, but it confirmed something important: whatever was running in my browser was intercepting **everything** — even analytics requests.

---

## Attack Mechanism

### How the Hijack Works

So here's the thing about **Browsec** (and honestly, most VPN extensions): they're not just "privacy tools." They're full-blown browser-level proxies. 

Think about it — they intercept ALL your HTTP/HTTPS traffic **after** TLS termination. That means they can see everything in plaintext, because they're running inside your browser itself. It's the same level of access as running Burp Suite as a proxy or having a corporate firewall inspecting your SSL traffic.

In this case, Browsec (or a compromised version of it) was rewriting GitHub's CDN URLs on the fly. Instead of fetching from `github.githubassets.com`, my browser was being told to fetch from `infird.com`. The attacker's server was supposed to serve modified versions of these JS/CSS chunks — with malicious code injected into GitHub's React frontend.

### Technical Evidence Against Browsec

Independent security analysis from CRXaminer (a browser extension security scanner) rated Browsec VPN as **HIGH RISK**, which perfectly explains the mechanics of this attack:

1. **Excessive Permissions:**
   - `webRequest` + `webRequestAuthProvider` = can intercept and modify ALL web requests.
   - `scripting` + `<all_urls>` = can inject code into any webpage.
   - `content_scripts` running at `document_start` = executes before the page even loads.

2. **Suspicious CSP Policy:**
   Browsec's manifest allows `connect-src` to over 100 suspicious HTTP domains, including:
   - `trafcfy.com` (classified as a "rogue website" by Avast)
   - `fastcache.xyz`, `cdnboost.me`, `datafrenzy.org`
   - Multiple generic CDN-sounding domains that look like drop-zones for injected payloads.

3. **Unencrypted HTTP Traffic:**
   Symantec security research confirmed that Browsec sends telemetry data over plain HTTP, making it vulnerable to MITM attacks. The manifest's allowance for HTTP connections to multiple third-party servers completely undermines the expectation that a VPN tool would always encrypt traffic.

This technical evidence strongly supports the conclusion that Browsec was the exact attack vector for the `infird.com` injection attempt.

### Why CORS Saved Me (Accidentally)

Here's where the attackers messed up: `infird.com` wasn't configured with proper CORS headers.

When GitHub's frontend tried to load resources from `infird.com`, my browser enforced CORS policy and blocked execution. This was an operational mistake on their part. If `infird.com` had just responded with `Access-Control-Allow-Origin: https://github.com`, the injected scripts would have executed silently. No console errors. No broken UI. I'd have been completely compromised without knowing it.

### The Retry Pattern & Partial Infrastructure

One interesting detail: GitHub's webpack bundler doesn't give up easily. It tries to load each chunk up to 3 times. Toward the end of the retry sequence, some requests started returning `404` instead of just `ERR_FAILED`. 

A `404` means the domain resolved, the TCP connection was established, and the server actually responded — it just didn't have those specific files. In other words: **the attack infrastructure was real and operational**, but it only had a subset of GitHub's chunks mirrored. 

---

## Why This Attack Class Is Dangerous

### What Could Have Happened

If `infird.com` had been properly configured, the injected JavaScript would've had access to:

- **Session cookies** and CSRF tokens
- **API responses** — private repo names, SSH keys, email addresses
- **Form submissions** — including 2FA codes and password change forms
- **Full DOM manipulation** — silently adding SSH keys, creating webhooks, authorizing OAuth apps

And here's the kicker: **all of this would've happened while GitHub looked and worked completely normally**. No weird popups, no obvious signs of compromise. Just business as usual — while everything I did was being monitored and modified.

### Silent Failure by Design

This is the most terrifying part: if the attack had succeeded, I would've seen **zero indication** anything was wrong. The frozen page I experienced? That was the attack **failing noisily**. A successful version would be completely invisible.

---

## Section 5 — Incident Response

Once I identified `infird.com` as the injecting domain and Browsec as the culprit, I moved fast:

**1. Remove the extension**
Not disable — **hard remove**. Browsec got deleted from my browser immediately.

**2. Rotate GitHub password**
Changed it right away. I assumed my session might've been compromised before CORS blocked the attack. Better safe than sorry.

**3. Revoke all Personal Access Tokens**
Nuked every single active token. Any token that was visible to page-level JS during an active session could've been exfiltrated.

**4. Audit OAuth Applications**
Went through and revoked anything I didn't recognize or actively use.

**5. Review Security Log**
Filtered for `token`, `session`, `oauth` events. Looking for access from unexpected IPs or user agents. (Thankfully, didn't find anything suspicious.)

**6. Block `infird.com` at DNS level**
Added it to my AdGuard DNS blocklist. This prevents the domain from resolving anywhere on my network.

---

## Detection Methods

If you want to catch this type of attack, here's what to look for:

### 1. DevTools Console — CORS Violations
Any `Access-Control-Allow-Origin` error referencing an unexpected CDN domain is a massive red flag. GitHub loads assets from `github.githubassets.com` **only**.

### 2. ChunkLoadError Cascade
A single `ChunkLoadError`? Network glitch. Multiple simultaneous failures across dozens of chunks? Systematic failure. Someone's intercepting your traffic.

### 3. 404 on Foreign Domain
The clearest confirmation that active attack infrastructure exists. If you see a 404 from a domain that shouldn't be serving your assets, investigate immediately.

### 4. Network Tab — Domain Filtering
Filter by domain. Any domain serving JS or CSS outside of the official list (`github.com`, `github.githubassets.com`, etc.) is a red flag.

### 5. DNS Query Logs
If you're running AdGuard DNS or Pi-hole, check your query logs. The hijack generates **hundreds** of DNS queries for the malicious domain that should never appear in normal traffic.

---

## Lessons

**VPN extensions are MITM proxies, not privacy tools.**
Any extension with `<all_urls>` permission has the same level of access as a network appliance. Treat it accordingly — or don't install it at all.

**Broken UI is not always a bug.**
When a page doesn't load properly, **open DevTools first**. The Console and Network tabs will show you exactly what's happening. This whole investigation took me under 60 seconds once I opened DevTools.

**CORS is not a security control.**
It only stopped this attack because the attackers misconfigured their server. Don't rely on CORS failures to detect injection attempts.

**Supply chain risk is real for extensions.**
A legitimate extension can be acquired, compromised via update, or have a malicious component silently pushed. Audit your extensions regularly.

**Rotate first, investigate second.**
When you suspect an active compromise, revoke credentials **before** you fully understand the attack. You can always reissue tokens; you can't un-exfiltrate data.

---

A broken GitHub page led me to uncover an active browser-level injection attempt. The attack failed because of an operational mistake on the attackers' part — not because of any built-in defense. 

The real lesson isn't that we're safe. It's that we got lucky. Next time, we might not see the console errors at all.

---

*Found this useful? Share it with your team. Browser extension attacks are seriously underreported and underestimated.*

*P.S. After removing Browsec, GitHub loads noticeably faster. Just saying. 😉*

*Tags: #CyberSecurity #MITM #BrowserSecurity #PenetrationTesting #AppSec #GitHub #InfoSec*

---

## References

| # | Source | Type | Direct Link |
|---|--------|------|-------------|
| 1 | Malwarebytes | Threat Intelligence | [Help with the infird.com Trojan](https://forums.malwarebytes.com/topic/323866-help-with-the-infirdcom-trojan/) |
| 2 | cside.com (Simon Wijckmans) | Security Research | [How web extensions can hurt your site (INFIRC.com and INFIRD.com)](https://cside.com/blog/how-web-extensions-can-hurt-your-site-infirc-and-infird) |
| 3 | Greatis Software | Malware Database | [Remove INFIRD.COM virus (Removal Guide)](https://greatis.com/unhackme/help/remove/remove-infird-com-virus.htm) |
| 4 | Cyclonis | Threat Database | [Infird.com: What Is This Pop-Up Menace?](https://www.cyclonis.com/remove-infird-com/) |
| 5 | VirusTotal | Malware Analysis | [infird.com Domain Analysis](https://www.virustotal.com/gui/domain/infird.com) |
| 6 | AlienVault OTX (LevelBlue) | Threat Intelligence | [infird.com Threat Indicator](https://otx.alienvault.com/indicator/domain/infird.com) |
| 7 | Darktrace / Cyberhaven | Security Report | [Cyberhaven Supply Chain Attack: Exploiting Browser Extensions](https://www.darktrace.com/blog/cyberhaven-supply-chain-attack-exploiting-browser-extensions) |
| 8 | The Hacker News | Fake Extensions Farm | [Over 100 malicious Chrome extensions slipped through Google's radar](https://thehackernews.com/2025/02/over-100-malicious-chrome-extensions.html) |
| 9 | GitLab Security | 100+ Fake Extensions | [Malicious browser extensions impacting at least 3.2 million users](https://gitlab-com.gitlab.io/gl-security/security-tech-notes/threat-intelligence-tech-notes/malicious-browser-extensions-feb-2025/) |
| 10 | Georgia Tech Research | Privacy Research | [Study Finds Thousands of Browser Extensions Compromise User Data](https://research.gatech.edu/study-finds-thousands-browser-extensions-compromise-user-data) |
| 11 | GitHub Gist (Th3w33knd) | Adware Detection | [FileCR.com is forcing a malicious extension (CRXLauncher)](https://gist.github.com/Th3w33knd/06b15b0df1b08ab371c392cc75be4f53) |
| 12 | CRXPlorer | Extension Scanner | [Free Chrome Extension Security Scanner & CRX Viewer](https://crxplorer.com/) |
| 13 | vpnMentor | VPN Reviews | [Browsec Review 2026 - Security & Privacy Analysis](https://www.vpnmentor.com/reviews/browsec/) |
| 14 | Joe Sandbox | Malware Analysis | [Automated Malware Analysis Report for infird.com](https://www.joesandbox.com/analysis/1904974/0/html) |
| 15 | Security.com (Symantec) | Technical Analysis | [Chrome Extension Leaks User Data Over Unencrypted HTTP](https://www.security.com/threat-intelligence/chrome-extension-leaks) |
