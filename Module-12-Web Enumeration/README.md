# 🔴 Module 12 — Web Enumeration

## 📚 What I Learned

Web enumeration applies the same core idea as Module 06's service enumeration, but focused on a single web application — finding hidden directories, files, endpoints, and technologies that aren't obviously linked from the site's visible pages.

**Core techniques:**
- **Directory/file brute-forcing** — tools like Gobuster and ffuf take a wordlist of common directory/file names, request each one against the target, and report what responds with something other than a 404. This surfaces pages and folders that exist but aren't linked anywhere visible.
- **Technology fingerprinting** — tools like WhatWeb identify exactly what's powering a site (CMS, framework, server software, language version) from response headers, page structure, and known signatures, telling you which known vulnerabilities might apply.
- **robots.txt and sitemap.xml** — ironically, sites sometimes reveal their own hidden paths by telling search engines not to index them. A `Disallow: /admin/` line is a site operator accidentally advertising exactly where sensitive paths live.
- **HTTP response codes** — 200 means the page loaded normally, 301/302 is a redirect (often revealing where a login page actually points), 403 means the resource exists but access is forbidden, and 404 means it genuinely doesn't exist.

## 🧠 Key Concepts

- **A 403 Forbidden is still a valuable finding, not a dead end.** It confirms the resource actually exists on the server and that a permission rule is actively being enforced against it — very different from a 404, which reveals nothing. A 403 tells an attacker exactly where to keep investigating (misconfigured permissions, alternate access paths).
- **A site's own robots.txt can hand over reconnaissance for free.** Directories explicitly disallowed for search engines are, ironically, directories a human just confirmed exist.
- **Web enumeration follows the same funnel principle as Module 07** — a handful of quick commands can turn "there's a web server" into a concrete list of distinct applications, technologies, and versions running on the target.

## 🧪 Practical Lab

### Objective
Run directory brute-forcing and technology fingerprinting against a web application to map what's actually running beneath the surface.

### Target
Metasploitable 2 (192.168.52.128)

### Rules of Engagement
- Target limited strictly to the Metasploitable 2 VM
- No enumeration of any other host without separate authorization

### Tools
whatweb, gobuster, curl

### Commands
whatweb http://192.168.52.128
gobuster dir -u http://192.168.52.128 -w /usr/share/wordlists/dirb/common.txt
curl http://192.168.52.128/robots.txt

## 🔎 Findings

**Technology fingerprint (whatweb):** Apache 2.2.8 running on Ubuntu, PHP 5.2.4-2ubuntu5.10 — both significantly outdated versions, a strong signal for follow-up CVE research.

**Directory brute-force (gobuster):**
- `/dvwa` (301 redirect) — Damn Vulnerable Web App, an intentionally vulnerable application
- `/mutillidae` (301 redirect) — a second intentionally vulnerable web application
- `/phpMyAdmin` (301 redirect) — a database administration panel, historically a high-value target since it's often left with default or weak credentials
- `/dav` (403 Forbidden) — confirmed to exist but access-restricted; WebDAV misconfigurations are a known vulnerability class worth further investigation
- `/test` (200 OK) — a directly accessible page

**robots.txt:** Explicitly disallows `/dvwa/` and `/mutillidae/` — confirming both paths without needing brute-force at all.

## 💡 What I Discovered

In just three commands, this went from "a web server is running" to a concrete map: three distinct web applications, one of them a database admin panel, all sitting on notably outdated software versions. The robots.txt finding stood out — a security-relevant file, meant to guide search engines, ended up handing over exactly the paths I was trying to brute-force in the first place.

## 🛡️ Defensive Perspective

- robots.txt should never be relied on to "hide" sensitive paths — it's a public file, and listing a path there openly advertises its existence to anyone who checks it, including attackers.
- Directory brute-forcing is detectable — a burst of requests for hundreds of common paths in a short window is a recognizable pattern that a properly tuned WAF or IDS can flag.
- Outdated software versions (Apache 2.2.8, PHP 5.2.4) should be prioritized for patching or upgrading, since older versions accumulate a much larger pool of known, often well-documented CVEs.
- Administrative panels like phpMyAdmin should never be left reachable on default paths with default credentials — restricting access by IP or requiring additional authentication significantly reduces this attack surface.

## 📝 Lessons Learned

Web enumeration turns a single URL into a real map of an application's structure remarkably quickly. The combination of automated brute-forcing and a simple manual check like robots.txt often reveals more together than either would alone — and outdated software versions surfaced here set up directly for deeper vulnerability research in the next modules.

## 🏁 Conclusion

Module 12 mapped Metasploitable 2's web attack surface — uncovering multiple distinct applications, an exposed administrative panel, and outdated core software, all through a handful of enumeration commands. This sets up directly for Module 13: Web Vulnerabilities, where the specific weaknesses in these discovered applications (starting with DVWA) get investigated in depth.
