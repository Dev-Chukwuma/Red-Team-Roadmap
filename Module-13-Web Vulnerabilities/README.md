# 🔴 Module 13 — Web Vulnerabilities

## 📚 What I Learned

**SQL Injection (SQLi)** happens when a web app builds a database query by directly inserting unsanitized user input into it. A classic example: a login checks `SELECT * FROM users WHERE username='$user' AND password='$pass'`. Entering `' OR '1'='1` as input turns the query's logic always-true, potentially bypassing authentication or, as demonstrated in the practical lab, dumping an entire table's worth of data.

**Cross-Site Scripting (XSS)** occurs when an app displays user input back on a page without properly sanitizing it, so a browser executes it as code instead of displaying it as text. Three types:
- **Reflected** — malicious input is part of the request itself (e.g. a URL parameter), executes immediately for whoever clicks the crafted link
- **Stored** — malicious input is saved (e.g. in a comment) and executes for every visitor who later views that page
- **DOM-based** — the vulnerability lives entirely in client-side JavaScript, never touching the server at all

**IDOR (Insecure Direct Object Reference)** happens when an app exposes an internal reference (a database ID, a filename) directly in a URL without verifying the logged-in user is actually authorized to access that specific object — e.g. changing `?id=1002` to `?id=1001` and getting someone else's data.

**Access control issues** are the broader category IDOR belongs to — any failure to properly check who should be allowed to do what. Unlike IDOR, these don't require manipulating an object reference at all — e.g. a regular user simply navigating directly to an admin-only URL and getting in because the server never checks their role.

**Authentication weaknesses** cover weak password policies, missing rate-limiting/lockouts (ties directly to Module 11's brute force/spraying), session tokens that don't expire, and predictable "remember me" tokens.

**CSRF (Cross-Site Request Forgery)** is a distinct vulnerability class from IDOR — it's about tricking a logged-in user's browser into submitting a request they never intended to make (e.g. a hidden form on a malicious site silently submitting a transfer request using the victim's existing session). CSRF tokens defend against this by ensuring a request could only have originated from the legitimate page. CSRF protection does not prevent IDOR, and fixing IDOR does not prevent CSRF — they address entirely different problems.

## 🧠 Key Concepts

- **IDOR is a subset of the broader access control category** — IDOR specifically involves manipulating a direct object reference (an ID, a filename); access control failures more broadly cover any missing permission check, including ones that don't involve object references at all.
- **CSRF and IDOR are unrelated vulnerabilities with unrelated fixes.** CSRF is about forging who sent a request; IDOR is about the app failing to verify authorization on a request the real user genuinely intended to send.
- **A single unsanitized input field can compromise an entire database.** The SQLi lab demonstrated that one text box, with no other access required, can escalate from "view one record" to "dump every record in the table."
- **Always-true SQL logic (`'1'='1'`) is the core mechanism behind most classic SQLi exploitation** — whether the goal is bypassing a login or extracting data, the underlying trick is manipulating the query's WHERE clause to match more than it should.

## 🧪 Practical Lab

### Objective
Demonstrate SQL injection hands-on against an intentionally vulnerable application, confirming the vulnerability class discussed in theory.

### Target
DVWA (Damn Vulnerable Web App) on Metasploitable 2 — discovered during Module 12's web enumeration

### Rules of Engagement
- Target limited strictly to DVWA on the Metasploitable 2 VM
- Security level set to "low" specifically for learning purposes, per DVWA's built-in difficulty setting

### Tools
DVWA's SQL Injection module (browser-based)

### Steps
1. Logged into DVWA
2. Set security level to low
3. Navigated to the SQL Injection module
4. Submitted normal input (User ID: 1) to establish a baseline
5. Submitted injected input (User ID: 1' OR '1'='1)

## 🔎 Findings

**Baseline (User ID: 1):** Returned exactly one record — the expected user.

**Injected input (User ID: 1' OR '1'='1):** Returned every user in the table — including accounts not otherwise visible through normal use of the form. The underlying query was very likely structured as `SELECT first_name, last_name FROM users WHERE user_id = '1' OR '1'='1'` — since `'1'='1'` is always true, the WHERE clause matched every row instead of just one.

This confirms unsanitized user input reaching the SQL query directly, with no input validation or parameterization in place at this security level.

## 💡 What I Discovered

Seeing a single text field escalate from "show me user 1" to "show me every user in the database" made the severity of SQL injection concrete in a way the theory alone didn't. It also reinforced the CSRF/IDOR distinction — none of what happened here involved forging a request or guessing an object ID; it was purely about unsanitized input reaching a query directly, a completely separate failure mode from either of those.

## 🛡️ Defensive Perspective

- **Parameterized queries / prepared statements** are the standard fix for SQL injection — user input is passed as data, never concatenated directly into query logic, so `' OR '1'='1` is treated as a literal string, not executable SQL.
- **Input validation and sanitization** should be applied at every entry point, not assumed to be someone else's responsibility further down the stack.
- **IDOR and access control failures** require explicit server-side ownership/permission checks on every object access — never trusting that a user only requests their own data just because the UI doesn't offer a way to request anyone else's.
- **CSRF tokens** should be present on all state-changing requests (transfers, password changes, account updates) — unique per session/request, and validated server-side before the action is processed.
- **Session and authentication hardening** (token expiry, rate-limiting, lockouts) closes the gap between weak passwords and full account compromise, tying directly back to Module 11's credential attack techniques.

## 📝 Lessons Learned

Web vulnerabilities cluster into a few core failure patterns — trusting user input too much (SQLi, XSS), trusting user-supplied identifiers too much (IDOR), and trusting requests too much (CSRF) — and confusing similarly-named concepts (like CSRF and IDOR) can lead to applying the wrong fix entirely. A single unsanitized field is enough to compromise an entire dataset, which is why input handling deserves disproportionate attention relative to how simple the fix usually is.

## 🏁 Conclusion

Module 13 moved from Module 12's web enumeration findings into actual exploitation — demonstrating real SQL injection against DVWA and dumping a full user table from a single vulnerable input field. Alongside SQLi, this module built a clear conceptual map of XSS, IDOR, broader access control failures, authentication weaknesses, and CSRF — distinct vulnerability classes that are often confused but require different, specific defenses. This sets up directly for Module 14: Web Security Project, where these techniques get combined into one full assessment.
