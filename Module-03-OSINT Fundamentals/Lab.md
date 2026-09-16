# 🔎 Module 03 — OSINT Practical Lab Report

## Target
Flutterwave (fintech, payment processing — Lagos, Nigeria)

## Method
Passive OSINT only — job boards, StackShare, public blog, third-party integration docs. No contact made with Flutterwave, no logins attempted, no non-public pages accessed.

## Findings

**1. Careers pages (Indeed, Glassdoor, Built In, company job board)**
Job listings reference building "secure, reliable tools" for Operations, Risk, and Customer Support teams, and mention core payment flow systems, settlement processing, and chargeback dispute handling. Roles are consistently framed as full-stack engineering — no single language/framework was explicitly named across postings, which itself is a finding: Flutterwave's public job listings are relatively disciplined about not leaking specific internal tooling.

**2. Third-party tech-stack aggregator (StackShare)**
A public StackShare profile lists jQuery, PHP, Gmail, Google Analytics, and NGINX among Flutterwave's tools — though this entry is dated and may reflect an older iteration of their stack rather than current infrastructure.

**3. Public integration/API documentation (Pipedream)**
Third-party workflow-automation docs show Flutterwave's API supports webhook-style integration for initiating and verifying payments and managing transactions, and demonstrate real integrations pairing Flutterwave's API with AWS services like CloudWatch — suggesting AWS is a cloud provider used by at least some parts of their ecosystem or by common integrators.

**4. Public GitHub (third-party developer repo, not Flutterwave's own)**
A community-built project referencing Flutterwave payment integration is described as a dedicated microservice handling payment initialization, transaction verification, and webhook processing with idempotency and cart management — useful context on how developers structure Flutterwave integrations, though this is a third party's code, not Flutterwave's internal system.

## Attack Surface Implications

- **No hardcoded credentials or internal hostnames were found** in this pass — a good sign; nothing careless surfaced in a quick search.
- **Job postings avoid naming specific internal frameworks** — this is actually a defensive strength. Compare this to companies whose postings say "must know [specific internal tool]," which hands an attacker a head start.
- **Payment/webhook architecture is publicly documented** (expected and necessary for a payments API business) — the real risk here isn't the documentation itself, it's ensuring webhook endpoints properly validate signatures, since that's a common real-world attack vector against payment APIs generally.
- **StackShare data is stale** — a reminder that OSINT sources have freshness dates; treating old aggregator data as current would be a mistake in a real assessment.

## Lesson

A well-run OSINT pass on a security-conscious company often comes back "clean" — that itself is data. The goal isn't to always find something dramatic; it's to accurately map what's actually exposed versus assumed. The one real technical signal here (AWS + webhook-based API architecture) is public by necessity for a payments platform, not a leak.
