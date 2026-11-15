# Security Writeups & Research

A technical portfolio documenting hands-on security labs, CTF challenges, and security investigations.

**Author:** [Kyrillos Kamal](https://github.com/KyrilosKamal)  
**Focus:** Network security, penetration testing, web application security, digital forensics, and security research.

> These writeups document learning and authorized lab activity. Techniques should only be used on systems you own or are explicitly authorized to assess. Target IPs, credentials, tokens, and flags are redacted where appropriate.

## Contents

| Area | What you'll find |
|---|---|
| [TryHackMe](#tryhackme) | Room walkthroughs, enumeration, exploitation, and technical analysis |
| [InitCTF](#initctf) | Challenge writeups across OSINT, cryptography, forensics, miscellaneous, and web |
| [Security investigations](#security-investigations) | Investigation notes based on observed behavior and collected evidence |

## TryHackMe

- **[Brr](Brr-Writeup-room)** — ICS/OT lab involving ScadaBR, a JSP upload path, Docker network discovery, and Modbus TCP register reads.
- **[Recruit](Recruit-TryHackMe-Writeup.md)** — Web application lab covering exposed information, server-side file retrieval, and SQL injection.

## InitCTF

- [OSINT](CTFs/InitCTF/01.%20OSINT/)
- [Cryptography](CTFs/InitCTF/02.%20Crypto/)
- [Forensics](CTFs/InitCTF/03.%20Forensics/)
- [Miscellaneous](CTFs/InitCTF/04.%20Misc/)
- [Web](CTFs/InitCTF/04.%20Web/)

Topics include geolocation and metadata analysis, cryptography, disk and network forensics, steganography, archive analysis, SQL injection, and signed-cookie security.

## Security investigations

- **[Browsec / unexpected GitHub asset requests](browsec-mitm-github-writeup.md)** — Investigation of unexpected third-party asset requests observed while using GitHub. The report separates browser observations from hypotheses; attribution to a specific extension and a confirmed session compromise are not established by the currently documented evidence.

## Methodology and evidence

Where possible, writeups include:

1. Scope and lab context.
2. Enumeration and attack-surface discovery.
3. Evidence supporting each finding.
4. Validation and impact.
5. Attack-chain summary.
6. Lessons learned and defensive recommendations.

Commands and screenshots are included as learning evidence. Challenge environments and software versions may change, so results should be independently validated.

## Responsible use

Exploitation examples are intended for intentionally vulnerable training environments, CTFs, or systems where explicit authorization has been granted. Do not reuse payloads against third-party systems without permission.

## About

I'm an IT and Network Specialist developing deeper practical skills in network security and penetration testing. This repository records hands-on learning and technical documentation.

- GitHub: [KyrilosKamal](https://github.com/KyrilosKamal)
- Portfolio: [KyrilosKamal Portfolio](https://kyriloskamal.github.io/KyrilosKamal/)
