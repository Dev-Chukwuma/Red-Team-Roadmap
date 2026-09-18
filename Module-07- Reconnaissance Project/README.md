# 🔴 Module 07 — Reconnaissance Project

## 📚 What I Learned

This module didn't introduce new theory — it's a capstone that chains everything from Modules 03–06 into a single, realistic reconnaissance workflow. Real engagements don't run OSINT, passive recon, network recon, and enumeration as isolated exercises — they build on each other, with each phase narrowing the picture until there's a concrete list of targets worth investigating further.

The combined flow:
1. **OSINT** — public-facing intel (job postings, tech stack signals, general presence)
2. **Passive Recon** — WHOIS, DNS records, Certificate Transparency logs
3. **Network Recon** — host discovery, port/version scanning
4. **Service Enumeration** — deep dive into whatever services scanning revealed

## 🧠 Key Concepts

- **Recon isn't a checklist of isolated commands — it's a narrowing funnel.** OSINT casts the widest net (any public info at all), passive recon narrows to infrastructure specifics, network recon narrows further to exactly what's live and listening, and enumeration narrows all the way down to what's actually accessible right now.
- **Passive and active recon require different authorization boundaries within the same project.** OSINT/passive recon (Modules 03–04) can responsibly target a real, named organization when anonymized in writeups. Network recon/enumeration (Modules 05–06) require direct authorization — meaning a combined project has to keep those two halves on separate, appropriately-scoped targets.
- **A real recon report tells a story, not just a list of findings.** The value isn't "here are 10 facts" — it's showing how each finding led to the next question, which is what actually demonstrates understanding to anyone reading the portfolio.

## 🧪 Practical Lab

### Objective
Combine OSINT, passive reconnaissance, network reconnaissance, and service enumeration into one unified reconnaissance report.

### Targets
- **External/passive phase:** Target Company A (fintech, payment processing) — OSINT and passive recon only
- **Internal/active phase:** My own Kali Linux machine — network recon and service enumeration only

### Rules of Engagement
- External target: passive techniques only, no direct contact, findings anonymized
- Internal target: active scanning/enumeration permitted, strictly limited to my own machine

### Tools
Web search, job boards, whois, dig, curl (crt.sh), nmap, smbclient, enum4linux

## 🔎 Findings

**Phase 1 — OSINT (external, passive):**
Job postings for Target Company A avoided naming specific internal frameworks — a disciplined, low-leakage public presence. A dated third-party tech-stack aggregator listed older technologies unlikely to reflect current infrastructure. Public integration documentation confirmed a webhook-based payment API paired with cloud infrastructure (AWS).

**Phase 2 — Passive Recon (external, passive):**
WHOIS showed enterprise-grade registration with redacted registrant info. DNS records pointed to a major CDN/proxy (masking the real backend IP), a hosted email provider via MX records, and TXT records confirming third-party service verification. Certificate Transparency logs surfaced expected subdomains (api, checkout, dashboard, www) plus one less-obvious "staging" subdomain — a lower-scrutiny environment worth flagging.

**Phase 3 — Network Recon (internal, active):**
Host discovery confirmed my Kali machine reachable via loopback. Version-detection scanning identified open services with specific software versions attached — turning bare port numbers into actionable, CVE-searchable data points.

**Phase 4 — Service Enumeration (internal, active):**
Null-session SMB access was permitted, revealing a real username and the machine's password policy (including a weak minimum length) without any credentials required.

## 💡 What I Discovered

Chaining these phases together made the funnel effect obvious in a way that doing each module separately didn't. OSINT gave broad context, passive recon narrowed to specific infrastructure and one flagged subdomain, and the active phase on my own machine showed exactly how enumeration converts open services into concrete, actionable intelligence (a username, a password policy) with zero exploitation. Seeing all four phases in one report made it clear why real reconnaissance is described as a funnel, not a checklist.

## 🛡️ Defensive Perspective

- Organizations should periodically run this exact same combined recon chain against themselves — OSINT, passive infra mapping, and (with proper internal authorization) active scanning — to see their environment the way an attacker would, end-to-end rather than in isolated checks.
- The "staging" subdomain finding reinforces that non-production environments need the same security posture as production, since they're discoverable through the same public sources.
- Disabling null-session SMB access alone would have prevented the most concrete finding in the entire combined report — a strong argument for prioritizing that single fix.

## 📝 Lessons Learned

Real reconnaissance work rarely stays inside one technique's lane — it's a continuous narrowing process where each phase's output becomes the next phase's input. Authorization boundaries matter even within a single combined project: passive and active techniques carry different risk levels and require different scoping, which is why this project deliberately split its targets rather than running active scans against a real company.

## 🏁 Conclusion

Module 07 closed out the Reconnaissance phase of the roadmap by combining OSINT, passive recon, network recon, and service enumeration into one end-to-end assessment. The result demonstrates the full funnel: broad public intelligence narrowing down to specific infrastructure, then to live services, then to concrete, actionable findings — a real username and password policy, extracted without a single exploit. This closes Phase 1 of the roadmap and sets up directly for Phase 2: Exploitation.
