# Investigation: Unexpected Third-Party Asset Requests on GitHub

**Type:** Browser-side troubleshooting / security investigation  
**Status:** Cause not conclusively attributed  
**Scope:** The author's own browser session and observations described below

> **Important:** This report documents unexpected requests to `infird.com` while loading GitHub. The available evidence does **not** establish that Browsec caused the requests, that TLS was intercepted, that malicious JavaScript executed, or that a GitHub session was compromised. Those remain hypotheses unless supported by reproducible tests and preserved artifacts.

## Summary

While browsing GitHub, parts of the interface failed to load. DevTools showed CSS and JavaScript asset requests to `infird.com`, rather than the expected GitHub asset host. The browser reported cross-origin loading errors and JavaScript chunk-load failures.

This is anomalous and warrants investigation. A third-party hostname appearing in a resource URL is evidence of an unexpected resource reference; by itself, it does not identify which component introduced that reference or prove malicious behavior.

## Observed symptoms

The reported symptoms were:

- GitHub dashboard content loaded incompletely.
- The left-side repository area remained blank or displayed a spinner.
- Some controls and navigation did not work.
- DevTools showed requests for CSS and JavaScript resources hosted at `infird.com`.
- The browser reported CORS-related failures and webpack chunk-load errors.
- Some requests reportedly returned HTTP 404.

Example console output captured during the incident:

```text
Access to CSS stylesheet at
https://infird.com/cdn/primer-react-css.d18392e72f73b5a6.module.css
from origin https://github.com has been blocked by CORS policy.
```

A failed cross-origin stylesheet or script request explains why dependent UI components may fail to render. It does not, on its own, prove that an attacker attempted to inject code.

## What the evidence establishes

| Observation | What it supports | What it does not establish |
|---|---|---|
| Resource URL uses `infird.com` | The page or browser attempted to load a resource from that host | Which extension, script, proxy, cache, or page component introduced the URL |
| CORS error | The browser blocked a cross-origin response under the applicable resource-loading rules | That CORS prevented all possible malicious activity or that an attack was underway |
| Chunk-load error | A required frontend chunk failed to load | The cause of the failed request |
| HTTP 404 | The responding server indicated that the requested resource was not found | That the server was attacker-controlled or that an attack was operational |
| GitHub UI partially failed | Some frontend resources or dependencies did not load successfully | That credentials, cookies, or tokens were accessed |

CORS is not a general-purpose defense against all script injection. Browser behavior depends on the resource type, request mode, response headers, and applicable content-security policy.

## Hypotheses to test

Possible explanations include:

1. A browser extension modified or redirected resource URLs.
2. A local proxy, VPN client, DNS/security product, or filtering component affected requests.
3. A cached or injected page resource referenced an unexpected host.
4. A transient network, CDN, or browser-state issue caused the unexpected behavior.

Browsec was installed at the time and was therefore one component worth testing. Its presence is not proof of attribution.

## Reproduction and evidence collection

The following checks would help establish causality. Record browser version, extension IDs and versions, timestamps, and the exact URL/request details. Redact cookies, authorization headers, tokens, and personal data before sharing HAR files.

### 1. Preserve browser evidence

- Save a HAR file with request and response headers where possible.
- Save relevant DevTools Console and Network screenshots.
- Record the initiator shown for each unexpected request.
- Record DNS resolution and remote IP information with timestamps.
- Preserve the extension ID, version, manifest, and installed-source details.

### 2. Compare controlled browser states

Repeat the same GitHub page load under controlled conditions:

1. Existing browser profile with extensions enabled.
2. Same profile with Browsec disabled.
3. Same profile after Browsec is removed.
4. A clean browser profile with no extensions.
5. If possible, a second browser or device on the same network.

Keep the URL, network, browser version, and test timing as consistent as possible. If the unexpected requests disappear only when one extension is disabled, repeat the test to rule out coincidence and inspect the request initiator and extension behavior.

### 3. Inspect the extension

Inspect the installed extension's actual manifest and code for relevant permissions, content scripts, declarative network rules, proxy settings, and update provenance. Permissions such as `webRequest`, `scripting`, or host access indicate capabilities; they do not prove that a particular request was modified.

### 4. Inspect the network path

Check browser proxy settings, VPN state, DNS answers, local filtering software, and any system-level proxy configuration. Compare DNS and connection details across the controlled tests.

## Impact assessment

A malicious extension or other component with sufficient access could potentially alter page behavior or access data available to its execution context. However, this investigation has not documented evidence that:

- A malicious script from `infird.com` successfully executed in the GitHub origin.
- GitHub session cookies or authentication tokens were read or transmitted.
- Credentials or private repository data were exfiltrated.
- Browsec was the component that changed the asset URLs.

Do not describe these impacts as confirmed without request-level, extension-level, or endpoint evidence.

## Precautionary response

The following actions were reported as having been taken:

- Removed the Browsec extension.
- Changed the GitHub password.
- Revoked active personal access tokens.
- Reviewed authorized OAuth applications.
- Reviewed GitHub security activity.
- Added `infird.com` to a DNS blocklist.

These are precautionary containment steps. They do not demonstrate that credentials or tokens were compromised. If repeating this response, preserve relevant evidence before clearing browser state where practical, and verify account activity through GitHub's security settings.

## Detection notes

When investigating unexpected third-party asset requests:

- Inspect the request's **Initiator** and call stack in DevTools.
- Compare the requested URL with the site's documented or observed asset hosts.
- Capture response status, headers, remote address, and timing.
- Check extension activity and proxy configuration.
- Reproduce with extensions disabled and in a clean profile.
- Treat CORS and chunk-load errors as symptoms to investigate, not proof of an attack.

## Conclusion

The incident involved unexpected requests to `infird.com` and failures loading parts of GitHub's frontend. This is a valid security anomaly and a useful starting point for browser-extension and supply-chain investigation.

**The root cause remains unconfirmed in the evidence currently documented.** A defensible attribution to Browsec would require a repeatable controlled test or direct evidence connecting the extension to the URL modification. A claim of session theft would require separate evidence of credential or token access and exfiltration.

## References

These references provide background and investigation leads; they do not independently prove what happened in this browser session.

- [cside: How web extensions can hurt your site (INFIRC.com and INFIRD.com)](https://cside.com/blog/how-web-extensions-can-hurt-your-site-infirc-and-infird)
- [Malwarebytes forum: Help with the infird.com Trojan](https://forums.malwarebytes.com/topic/323866-help-with-the-infirdcom-trojan/)
- [VirusTotal: infird.com domain analysis](https://www.virustotal.com/gui/domain/infird.com)
- [AlienVault OTX: infird.com](https://otx.alienvault.com/indicator/domain/infird.com)
- [Chrome Extensions documentation](https://developer.chrome.com/docs/extensions/)
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
