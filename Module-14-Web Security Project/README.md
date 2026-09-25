# 🔴 Module 14 — Web Security Project

## 📚 What I Learned

This module didn't introduce new theory — it's a capstone that chains Modules 12 (Web Enumeration) and 13 (Web Vulnerabilities) into one complete, real assessment narrative. Real web security engagements follow this exact flow: map the application's structure first, then test what was found for actual exploitable weaknesses. Each phase feeds directly into the next — enumeration without follow-up testing just produces a list; testing without enumeration means guessing at targets blindly.

## 🧠 Key Concepts

- **A full web assessment is enumeration + exploitation working together, not two separate exercises.** Module 12 found DVWA existed as a discoverable path; Module 13 proved that path was actually exploitable. Neither step alone tells the complete story.
- **Technology fingerprinting directly informs what vulnerabilities are worth testing for.** Knowing the stack was outdated Apache/PHP going into Module 13 meant the SQL injection test wasn't a random guess — it was a logical next step given what enumeration already revealed.
- **A real report tells the story in order** — discovery, then confirmation, then impact — which is exactly the structure that makes a portfolio writeup demonstrate genuine understanding rather than a disconnected list of commands.

## 🧪 Practical Lab

### Objective
Chain web enumeration and web vulnerability testing into one continuous assessment of a single target, from initial discovery through confirmed exploitation.

### Target
DVWA on Metasploitable 2 (192.168.52.128/dvwa/)

### Rules of Engagement
- Target limited strictly to DVWA on the Metasploitable 2 VM
- Security level set to "low" for learning purposes

### Tools
whatweb, gobuster, curl, DVWA's SQL Injection module

## 🔎 Findings

**Phase 1 — Enumeration:**
Technology fingerprinting identified Apache 2.2.8 and PHP 5.2.4-2ubuntu5.10 — both significantly outdated. Directory brute-forcing surfaced three distinct web applications (DVWA, Mutillidae, phpMyAdmin), with `/dvwa` and `/mutillidae` also confirmed independently via the site's own robots.txt file. A `/dav` path returned 403 Forbidden, confirming its existence without granting access.

**Phase 2 — Vulnerability testing:**
Within DVWA specifically, the SQL Injection module was tested using the payload `1' OR '1'='1` in the User ID field. A baseline request (User ID: 1) returned exactly one record. The injected payload returned every user in the underlying table — confirming the application concatenates user input directly into its SQL query without sanitization or parameterization.

**Combined narrative:** Enumeration identified DVWA as reachable and running on outdated software; targeted testing within DVWA confirmed a critical SQL injection vulnerability capable of dumping the entire user table from a single unauthenticated-adjacent input field.

## 💡 What I Discovered

Seeing the two modules connect end-to-end — from a directory scan result to an actual data dump — made the funnel effect from Module 07 click again, this time entirely within the web layer. The outdated software versions found during enumeration weren't just a side note; they set the expectation that vulnerabilities would likely be present, which the SQL injection test then confirmed directly.

## 🛡️ Defensive Perspective

- Outdated software versions surfaced during enumeration should be treated as a leading indicator, not just a footnote — they correctly predicted a real, high-severity vulnerability in this case.
- Parameterized queries would have fully prevented the SQL injection finding regardless of how thorough the enumeration phase was — input handling remains the actual root-cause fix.
- robots.txt should never be used as a security control — it directly handed over two of the three discovered application paths in this assessment.
- Administrative panels like phpMyAdmin, found during enumeration, deserve the same scrutiny as any other discovered path — the assessment's next logical step in a real engagement.

## 📝 Lessons Learned

A full assessment is genuinely more valuable as a connected story than as isolated technique demonstrations — the enumeration phase didn't just find things, it correctly pointed toward where the real vulnerability would be. This reinforced that reconnaissance and exploitation are not separate skills but one continuous process, at the web layer just as much as the network layer covered back in Module 07.

## 🏁 Conclusion

Module 14 closed out the Web phase of the roadmap by combining enumeration and exploitation into one complete assessment — from discovering DVWA's existence through outdated technology fingerprinting and directory brute-forcing, to confirming a critical SQL injection vulnerability capable of dumping the full user table. This closes Phase 2 (Exploitation) of the roadmap's web-focused modules and sets up directly for Phase 3: Post-Exploitation, starting with Module 15's Linux Privilege Escalation.
