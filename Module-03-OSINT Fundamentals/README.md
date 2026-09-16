# 🔴 Module 03 — OSINT Fundamentals

## 📚 What I Learned

OSINT (Open Source Intelligence) means gathering information about a target using only publicly available sources — nothing hacked, nothing stolen, nothing requiring special access. It's the equivalent of a detective building a profile before ever knocking on a door: public records, social media, job postings, news, company filings.

Attackers use OSINT to learn who works at a company (phishing targets), what technology they use (job postings often reveal tech stacks), what domains/subdomains exist, what email format is used, and whether anyone has accidentally leaked something. Defenders should be doing the exact same searches on their own organization first — you can't protect what you don't know is exposed.

The golden rule: **you never touch the target directly.** No pinging, no logging in, no visiting non-public areas. Purely observing what's already public. This is why OSINT is the first recon phase — it carries essentially zero risk of detection.

I also learned where the line sits between OSINT and social engineering: finding a public LinkedIn profile is OSINT (passive observation). Contacting that person while pretending to be someone else to extract information crosses into social engineering/phishing — an active deception technique requiring its own, far more sensitive authorization.

## 🧠 Key Concepts

- **OSINT = passive observation of public information only.** The moment you contact or deceive someone, it becomes social engineering — a different category entirely.
- **Job postings are an underrated OSINT source** — companies often list exact internal tools/frameworks without realizing it maps their attack surface for anyone reading.
- **"Clean" OSINT results are still data.** A company whose public presence gives away nothing specific (no named internal tools, no leaked credentials) demonstrates real security discipline — the absence of findings is itself a finding.
- **Source freshness matters.** Third-party aggregator data (like tech-stack listing sites) can be years out of date; treating it as current in a real assessment would be a mistake.

## 🧪 Practical Lab

### Objective
Practice passive OSINT technique against a real, well-known public company — map what's publicly discoverable about their tech stack and attack surface without touching their systems.

### Target
Flutterwave (fintech, payment processing — Lagos, Nigeria)

### Rules of Engagement
- Purely passive — search engines, public job boards, public documentation only
- No contact with Flutterwave, no login attempts, no access to non-public pages

### Tools
Web search, public job boards (Indeed, Glassdoor, Built In), third-party tech-stack aggregators (StackShare), public API/integration documentation (Pipedream)

## 🔎 Findings

**Careers pages:** Job listings reference building secure tools for Operations, Risk, and Customer Support, plus core payment flow and settlement/chargeback systems. Notably, postings avoid naming specific internal frameworks — a disciplined, low-leakage approach.

**Tech-stack aggregator (StackShare):** Lists jQuery, PHP, Gmail, Google Analytics, and NGINX — but this entry is dated and likely reflects an older stack rather than current infrastructure.

**Public integration docs (Pipedream):** Shows Flutterwave's API supports webhook-based payment initiation/verification, and demonstrates real-world pairing with AWS services (e.g. CloudWatch) — suggesting AWS involvement somewhere in their or their integrators' ecosystem.

**Third-party GitHub repo:** A community-built microservice integrating Flutterwave's payment API shows typical implementation patterns (idempotency, webhook handling) — useful context on integration architecture, though it's third-party code, not Flutterwave's own.

## 💡 What I Discovered

No hardcoded credentials, internal hostnames, or careless leaks turned up in this pass. The most interesting realization was that a well-run OSINT pass on a security-conscious company often comes back "clean" — and that's meaningful data in itself, not a failed search. The one real technical signal (AWS + webhook architecture) is public by necessity for any payments API business, not evidence of a leak.

## 🛡️ Defensive Perspective

- Job postings should be reviewed before publishing for any mention of specific internal tools, versions, or naming conventions that narrow an attacker's guesswork.
- Payment/webhook APIs are inherently public-facing by design — the real defensive focus belongs on webhook signature validation and endpoint authentication, not on hiding the fact that an API exists.
- Organizations should periodically run OSINT on themselves (the same way I just did on Flutterwave) to catch accidental exposure before an attacker finds it first.

## 📝 Lessons Learned

OSINT isn't about finding something dramatic every time — it's about accurately separating what's genuinely exposed from what merely looks interesting. Source freshness and distinguishing a company's own claims from third-party assumptions about them (like the AWS inference) are both critical to avoid overstating findings.

## 🏁 Conclusion

Module 03 introduced passive reconnaissance through OSINT — the foundation the entire recon phase of the attack lifecycle builds on. The practical lab against Flutterwave demonstrated that disciplined public-facing communication (careful job postings, no leaked internals) significantly narrows what passive OSINT alone can reveal, while still surfacing legitimate architectural signals (AWS, webhook-based payment API) that are expected and necessary for a fintech platform to operate publicly.
