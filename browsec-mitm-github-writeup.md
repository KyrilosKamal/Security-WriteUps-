# How a VPN Extension Hijacked My GitHub Session (and How I Caught It)

*A real-world browser extension MITM attack, caught live through DevTools.*

---

## Background

I was checking my GitHub dashboard — routine activity. The page loaded partially: the feed showed some content, but the left sidebar never populated. Clicks did nothing. The page was frozen.

My first instinct was a slow network or a GitHub outage. I was wrong.

---

## Section 1 — The Symptom

- GitHub dashboard loaded but **sidebar spinner never resolved**
- Left column (repositories list) completely empty
- Feed rendered partially
- No UI interaction worked — buttons, links, navigation all unresponsive
- No error banner, no status page alert

This is the danger of this attack class: it presents as a mundane broken page, not an obvious security incident.

---

## Section 2 — DevTools Investigation

Opening DevTools (`F12`) → **Console tab** immediately revealed the real picture.

The first line that caught my eye:

```
Access to CSS stylesheet at 'https://infird.com/cdn/primer-react-css.d18392e72f73b5a6.module.css'
from origin 'https://github.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

**`infird.com` — GitHub has never used this domain.**

GitHub serves all static assets (JS bundles, CSS modules, fonts) from `github.githubassets.com`. Any other CDN origin is immediately suspicious.

This wasn't a one-off. The console was flooded with hundreds of identical errors — every JS chunk and CSS module GitHub needed was being requested from `infird.com` instead of the legitimate CDN:

```
Access to script at 'https://infird.com/cdn/chunk-63160-d2b6073aa911a9ad.js'
from origin 'https://github.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.

Access to script at 'https://infird.com/cdn/primer-react-1cbafc24ef5f5e7d.js'
from origin 'https://github.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.

Access to script at 'https://infird.com/cdn/octicons-react-ad81b26f4cfaa5de.js'
from origin 'https://github.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

After repeated failures, GitHub's webpack bundler exhausted its retry logic and threw fatal errors:

```
css loading:30 Uncaught (in promise) Error: Loading chunk 90225 failed after 3 retries.
    at t.onerror.t.onload (css loading:30:14)

jsonp_chunk_loading:28 Uncaught (in promise) ChunkLoadError: Loading chunk 51591 failed after 3 retries.
    at b.f.j (jsonp_chunk_loading:28:16)
```

At this point the page is effectively dead — React can't hydrate without its chunks, so the UI freezes completely.

One additional line confirmed full traffic interception was in play:

```
Tracking Prevention blocked an XHR request to
https://www.google-analytics.com/mp/collect?measurement_id=G-03XW3FWG7L
```

Not directly related to the attack, but confirms the extension was intercepting and processing all outbound requests — including analytics pings.

---

## Section 3 — Attack Mechanism

### How the Hijack Works

**Browsec** is a VPN browser extension. Like all VPN extensions, it operates as a browser-level proxy — it intercepts all HTTP/HTTPS traffic *after TLS termination* (because it runs inside the browser process, post-decryption). This gives it full plaintext visibility into every request and response.

In this case, Browsec (or a compromised version of it) was rewriting GitHub's asset CDN URLs. Instead of:

```
https://github.githubassets.com/assets/chunk-63160-d2b6073aa911a9ad.js
```

Requests were being redirected to:

```
https://infird.com/cdn/chunk-63160-d2b6073aa911a9ad.js
```

The attacker's server at `infird.com` was meant to serve modified versions of these JS/CSS chunks — with injected malicious code — to GitHub's React frontend.

### Why CORS Saved Me (Accidentally)

The attack failed because `infird.com` was not configured with proper CORS headers. GitHub's frontend, running at `https://github.com`, tried to load cross-origin resources from `infird.com`. The browser enforced CORS policy and blocked execution:

```
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

This is an operational failure on the attacker's part. Had `infird.com` responded with:

```
Access-Control-Allow-Origin: https://github.com
```

...the injected scripts would have executed silently. GitHub's React app would have loaded the attacker's modified chunks with no console errors and no visible indication of compromise.

### The Retry Pattern

GitHub's webpack bundler implements retry logic for failed chunk loads. The console log shows each chunk being attempted up to 3 times with incrementing retry parameters:

```
GET https://infird.com/cdn/chunk-63160-d2b6073aa911a9ad.js        → ERR_FAILED (CORS)
GET https://infird.com/cdn/chunk-63160-d2b6073aa911a9ad.js?&r=1   → ERR_FAILED (CORS)
GET https://infird.com/cdn/chunk-63160-d2b6073aa911a9ad.js?&r=2   → ERR_FAILED (CORS)
GET https://infird.com/cdn/chunk-63160-d2b6073aa911a9ad.js?&r=3   → ERR_FAILED (CORS)
```

After 3 retries, the bundler throws `ChunkLoadError` and stops trying. This happened for 50+ chunks simultaneously, explaining the complete UI freeze.

### Partial Infrastructure — A Telling Detail

Toward the end of the retry sequence, some requests started returning `404` instead of just `ERR_FAILED`:

```
GET https://infird.com/cdn/41328-c0eb6926392c50ed.js?&r=3  net::ERR_FAILED 404 (Not Found)
GET https://infird.com/cdn/7463-c4bd1db840718ba5.js?&r=3   net::ERR_FAILED 404 (Not Found)
GET https://infird.com/cdn/17864-2863f28639789b81.js?&r=3  net::ERR_FAILED 404 (Not Found)
```

This is significant. A `404` means `infird.com` resolved, the TCP connection was established, and the server responded — it just didn't have those specific files. This tells us:

1. The attack infrastructure is real and operational
2. The attacker's server only mirrored a subset of GitHub's chunks
3. The attack was partially deployed — not fully operational at the time

---

## Section 4 — Why This Attack Class Is Dangerous

### The Extension Trust Model Problem

Browser extensions that request `<all_urls>` permission are granted full proxy-level access to all browser traffic. This is functionally equivalent to:

- Running Burp Suite as a local proxy
- Installing a corporate SSL inspection firewall
- A network-level MITM appliance

The critical difference: users install extensions casually, often without reading permissions. The same user who would never configure a manual proxy will install a VPN extension in seconds.

### What a Successful Attack Could Achieve

If `infird.com` had served properly CORS-headered responses, the injected JavaScript would have had access to:

- **Session cookies** — GitHub auth cookies accessible to page-level JS (unless `HttpOnly`, but `__Host-user_session_same_site` and similar can be read)
- **CSRF tokens** embedded in page HTML
- **API responses** — private repo names, SSH keys, email addresses
- **Form submissions** — intercepting 2FA codes, password change forms
- **DOM manipulation** — silently adding SSH keys, creating webhooks, authorizing OAuth apps

All of this while GitHub's UI looks and feels completely normal to the user.

### Silent Failure by Design

The most dangerous aspect: if the attack had succeeded, there would be **zero visible indication**. No console errors. No broken UI. GitHub loads, works, and the user continues normally — while every page interaction is being monitored and modified.

The frozen page I experienced was actually the attack failing noisily. A successful version would be completely invisible.

---

## Section 5 — Incident Response

Upon identifying `infird.com` as the injecting domain and Browsec as the vector, I took the following steps immediately:

**1. Remove the extension**
Hard remove, not disable. Browsec deleted from browser.

**2. Rotate GitHub password**
Settings → Password → change immediately. Assume session may have been compromised before CORS blocked execution.

**3. Revoke all Personal Access Tokens**
Settings → Developer Settings → Personal access tokens → revoke all active tokens. Any token visible to page-level JS during an active session could have been exfiltrated.

**4. Audit OAuth Applications**
Settings → Applications → Authorized OAuth Apps → review and revoke any unrecognized or suspicious app authorizations.

**5. Review Security Log**
Settings → Security → Log → filtered for `token`, `session`, `oauth` events. Look for access from unexpected IPs or user agents.

**6. Block `infird.com` at DNS level**
Added `infird.com` to AdGuard DNS blocklist. This prevents any future resolution of the domain network-wide, not just in the browser.

---

## Section 6 — Detection Methods

How to catch this class of attack before or during exploitation:

### 1. DevTools Console — CORS Violations

The most reliable signal. Open Console and filter for your target domain. Any `Access-Control-Allow-Origin` error referencing an unexpected CDN domain is an immediate red flag:

```
Access to script at 'https://infird.com/cdn/chunk-51591-c57a938d95d8d8e6.js'
from origin 'https://github.com' has been blocked by CORS policy
```

GitHub loads assets from `github.githubassets.com` only. Deviation = investigation required.

### 2. ChunkLoadError Cascade

A single `ChunkLoadError` is a network blip. Multiple simultaneous failures across dozens of chunks:

```
ChunkLoadError: Loading chunk 51591 failed after 3 retries.
ChunkLoadError: Loading chunk 63160 failed after 3 retries.
ChunkLoadError: Loading chunk 78975 failed after 3 retries.
```

...is a systematic failure. Either CDN is down (check status page) or traffic is being intercepted.

### 3. 404 on Foreign Domain

The clearest confirmation of active infrastructure:

```
GET https://infird.com/cdn/26802-30fc156c4abd69b3.js?&r=3 net::ERR_FAILED 404 (Not Found)
```

`404` means the domain resolved, server responded. Not a DNS block, not a firewall drop. An actual server exists at `infird.com` that GitHub's browser code is hitting. That server should not exist in any legitimate GitHub flow.

### 4. Network Tab — Domain Filtering

DevTools → Network → filter by domain. All GitHub asset requests should originate from:
- `github.com`
- `github.githubassets.com`
- `avatars.githubusercontent.com`
- `api.github.com`

Any other domain serving JS or CSS = investigate.

### 5. DNS Query Logs

If running AdGuard DNS or Pi-hole, check query logs during a GitHub session. The hijack generates hundreds of DNS queries for `infird.com` that should never appear in normal GitHub traffic. This is detectable even before opening DevTools.

---

## Section 7 — Lessons

**VPN extensions are MITM proxies, not privacy tools.**
Any extension with `<all_urls>` permission has the same traffic access as a network appliance. The fact that it's labeled "VPN" or "privacy" is irrelevant to its capability. Treat it accordingly — or don't install it.

**Broken UI is not always a bug.**
The instinct when a page doesn't load is to refresh, clear cache, blame the network. Open DevTools first. Console and Network tabs show exactly what the browser is doing. This investigation took under 60 seconds once DevTools was open.

**CORS is not a security control — it's a browser enforcement mechanism.**
It failed to stop the attack only because the attacker misconfigured their server. A correctly configured `infird.com` would have delivered a fully functional attack with zero user-visible symptoms. Don't rely on CORS failures to detect injection.

**Supply chain risk is real for extensions.**
A legitimate extension can be acquired, compromised via update, or have a malicious component silently pushed. Browsec may have been the vector without the original developers' intent. Audit your extensions regularly. Remove anything you don't actively use.

**Rotate first, investigate second.**
In an active compromise scenario, the priority is limiting damage. Revoke credentials before you fully understand the attack. You can always reissue tokens; you can't un-exfiltrate data.

---

## Summary

| | Detail |
|---|---|
| **Attack vector** | Browsec VPN browser extension |
| **Mechanism** | CDN URL rewriting → MITM injection via `infird.com` |
| **Target** | GitHub JS/CSS chunk delivery |
| **Detection** | DevTools Console → CORS violations on foreign domain |
| **Failure point** | Missing `Access-Control-Allow-Origin` on attacker server |
| **Response** | Extension removed, credentials rotated, DNS blocked |

A broken GitHub page led to uncovering an active browser-level injection attempt. The attack failed due to an operational mistake on the attacker's side — not any built-in defense. The lesson isn't that we're safe. It's that we got lucky, and next time we might not see the console errors at all.

---

*Found this useful? Share it with your team. Browser extension attacks are underreported and underestimated.*

*Tags: #CyberSecurity #MITM #BrowserSecurity #PenetrationTesting #AppSec #GitHub #InfoSec*
