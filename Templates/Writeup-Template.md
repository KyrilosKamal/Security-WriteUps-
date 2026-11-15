# <Challenge or Assessment Name>

**Platform / Client:** <Platform or authorized client>  
**Category:** <Web / Network / Linux / Windows / AD / Forensics / Other>  
**Difficulty:** <Difficulty, if applicable>  
**Date:** <YYYY-MM-DD>  
**Scope:** <Target identifier or lab scope>

> **Authorization:** State that this work was performed in an authorized lab or under written permission. Redact credentials, session tokens, personal data, and sensitive target details.

## Executive summary

Summarize the objective, the main finding(s), and the outcome in a few sentences. Do not claim impact that was not demonstrated.

## Scope and rules of engagement

- In-scope targets:
- Out-of-scope targets:
- Constraints:
- Testing environment:

## Environment

| Item | Value |
|---|---|
| Attacker OS | |
| Target OS / application | |
| Relevant versions | |
| Tools | |

## 1. Reconnaissance

Explain what was discovered and why the next test was selected.

```bash
# Command
```

**Evidence:** Add a sanitized screenshot or relevant output.

## 2. Enumeration

Document ports, services, routes, users, shares, endpoints, or other attack-surface details.

| Finding | Evidence | Notes |
|---|---|---|
| | | |

## 3. Vulnerability analysis

For each suspected weakness, document:

- **Observation:** What did you see?
- **Hypothesis:** What might explain it?
- **Validation:** What controlled test confirmed or rejected the hypothesis?
- **Evidence:** Request/response, output, screenshot, or log.
- **Limitations:** What was not tested or cannot be concluded?

## 4. Exploitation / proof of concept

Describe the minimum steps needed to reproduce the finding in the authorized scope. Include sanitized requests and commands.

```text
<sanitized request or output>
```

Explain why the technique worked. Distinguish tool output from your own interpretation.

## 5. Impact

Describe only impact demonstrated by the evidence. Separate confirmed impact from potential impact.

## 6. Attack chain

```text
Initial observation
    |
    v
Validated weakness
    |
    v
Demonstrated impact
```

## 7. Remediation

| Finding | Recommended fix | Validation after remediation |
|---|---|---|
| | | |

## 8. Lessons learned

- What did you learn?
- What assumption did you have to revise?
- What would you test next time?

## References

- [Official documentation or advisory](<URL>)
- [Relevant research](<URL>)

## Evidence checklist

- [ ] Scope and authorization are clear.
- [ ] Commands and requests are reproducible.
- [ ] Screenshots support the claims.
- [ ] Secrets and personal data are redacted.
- [ ] Observations are separated from hypotheses.
- [ ] Limitations and untested assumptions are stated.
- [ ] Remediation is actionable.
