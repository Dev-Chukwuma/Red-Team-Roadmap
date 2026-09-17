# 🔴 Module 04 — Passive Reconnaissance

## 📚 What I Learned

Passive reconnaissance extends OSINT into more technical territory — using specific tools to query public third-party databases about a target's infrastructure, without ever sending traffic to the target's own servers.

**Key techniques:**
- **WHOIS** — public domain registration records: registrar, creation/expiry dates, name servers, and sometimes registrant info (often redacted now under privacy rules).
- **DNS lookups (A, MX, NS, TXT records)** — asking a DNS resolver (not the target) about a domain's infrastructure. A records reveal the IP address; MX records reveal the email provider; NS records reveal who manages DNS; TXT records often reveal third-party service verification tokens and email-authentication policy.
- **Certificate Transparency logs (crt.sh)** — every SSL certificate ever issued for a domain is publicly logged. Searching these logs often surfaces subdomains the company never intended to advertise (e.g. staging/dev/internal environments).

The reason all of this still counts as "passive": every query goes to a **third-party public database** (a registrar, a DNS resolver, a certificate log) — never directly to the target's own infrastructure. This is what separates Module 04 from Module 05 (Network Reconnaissance), where scanning the target directly begins.

## 🧠 Key Concepts

- **An unlinked subdomain found via crt.sh matters because it's often less protected than the main production site** — dev/staging environments tend to receive less security attention and can leak more (verbose errors, weaker auth, exposed test data) than the polished public-facing site.
- **IP ranges can reveal infrastructure providers.** Certain IP blocks are publicly known to belong to specific CDNs/proxies (e.g. Cloudflare) — recognizing these tells you the real server is hidden behind a proxy layer.
- **TXT records are an underrated OSINT source.** Domain-verification strings for third-party services (Google, Microsoft, etc.) confirm exactly which platforms an organization uses, without ever contacting them directly.
- **Passive recon can map an organization's cloud provider, CDN, and email platform — entirely from public records, without a single packet sent to their actual servers.**

## 🧪 Practical Lab

### Objective
Use WHOIS, DNS record lookups, and Certificate Transparency logs to passively map a real organization's infrastructure — without touching their servers directly.

### Target
Target Company A (fintech, payment processing)

### Rules of Engagement
- Purely passive — all queries directed at third-party public databases only (registrars, DNS resolvers, crt.sh)
- No direct contact with the target's own infrastructure

### Tools
whois, dig, curl (for crt.sh's JSON API)

### Commands
whois targetdomain.com
dig targetdomain.com A
dig targetdomain.com MX
dig targetdomain.com NS
dig targetdomain.com TXT
curl -s "https://crt.sh/?q=targetdomain.com&output=json" | grep -o '"name_value":"[^"]*' | sed 's/"name_value":"//' | sort -u

## 🔎 Findings

- **WHOIS:** Domain registered through an enterprise-grade registrar, name servers pointing to a major cloud DNS provider, registrant information redacted under standard privacy protection.
- **A record:** Resolves to an IP range associated with a known CDN/reverse-proxy provider — indicating the real backend server IP is not directly exposed.
- **MX record:** Points to a major hosted email provider, identifying which platform handles the organization's email.
- **NS record:** Confirms the same cloud DNS provider seen in WHOIS.
- **TXT record:** SPF entry confirms the email provider; a site-verification string confirms the organization has registered with a search/analytics platform.
- **crt.sh:** Several expected subdomains (api, checkout, dashboard, www) plus one notable subdomain following a "staging" naming pattern — a less-watched environment worth flagging as a potential weaker point, though not tested or accessed.

## 💡 What I Discovered

This module was more eye-opening than I expected — I hadn't fully appreciated how much infrastructure detail (cloud provider, CDN, email platform, and even hidden subdomains) can be mapped without sending a single packet to the target's actual servers. The crt.sh technique especially stood out — finding a "staging" subdomain through certificate logs alone, with zero direct contact, made the earlier theoretical discussion about "why an unlinked subdomain matters" click into place.

## 🛡️ Defensive Perspective

- Organizations should run crt.sh against their own domains periodically — it reveals exactly what an attacker would find, including forgotten subdomains.
- Non-production environments (staging, dev, internal) should receive the same security scrutiny as production, since they're discoverable through public certificate logs regardless of whether they're linked anywhere.
- TXT and MX records reveal real information necessarily (they need to function), but organizations should be aware these confirm exact third-party platforms in use — useful for an attacker building a targeted phishing pretext.
- WHOIS privacy redaction should always be enabled — it's a simple, effective step that's now standard practice.

## 📝 Lessons Learned

Passive reconnaissance can build a surprisingly detailed infrastructure map — cloud provider, CDN, email platform, and hidden subdomains — using only public, third-party sources. The distinction between passive (querying public databases) and active (touching the target directly) recon isn't just theoretical; it's what keeps this entire phase of an engagement low-risk and legally uncomplicated.

## 🏁 Conclusion

Module 04 demonstrated the real power of passive reconnaissance: WHOIS, DNS records, and Certificate Transparency logs together can reveal an organization's cloud provider, CDN, email platform, and even non-public subdomains — all without a single interaction with their actual infrastructure. This module reinforces the attack lifecycle's opening principle: gather everything possible before ever making contact.
