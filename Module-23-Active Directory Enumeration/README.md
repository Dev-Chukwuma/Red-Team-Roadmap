# 🔴 Module 23 — Active Directory Enumeration

## 📚 What I Learned

Active Directory enumeration goes far beyond regular network enumeration — instead of just finding open ports and services, it queries the domain's own directory service to extract the entire organizational structure: users, groups, computers, permissions, relationships, and misconfigurations. The critical design reality that makes this powerful: most AD enumeration requires only a standard, low-privileged domain user account. Any authenticated user can read most of AD's structure by default.

**Why this is a fundamental design tension:** Microsoft built AD so any authenticated user could query the directory — employees legitimately need to look up colleagues, find printers, and locate shared resources. The assumption was "if you're authenticated, you're supposed to be here." The problem: that same read access hands an attacker who compromises even the lowest-privileged account a complete map of the entire organization — every user, every group, every computer, every service account, every permission — with no privilege escalation needed just to see everything.

**Core enumeration techniques:**
- **Built-in Windows commands** — `net user /domain`, `net group /domain`, `net group "Domain Admins" /domain` — available on any domain-joined machine with no extra tools
- **PowerView** — PowerShell-based AD enumeration script; queries AD and returns structured, detailed results. Key commands: `Get-NetDomain`, `Get-NetUser`, `Get-NetGroup`, `Get-NetComputer`, `Get-NetUser -SPN` (finds Kerberoastable accounts), `Find-LocalAdminAccess`
- **ldapdomaindump** — runs from Kali; queries AD via LDAP with valid credentials and dumps the entire domain structure into readable HTML/JSON output files
- **enum4linux-ng** — already used in Module 06 for SMB; against a DC it pulls domain users, groups, and password policies
- **BloodHound** — the most powerful AD enumeration tool, visualizing relationships and attack paths graphically (full coverage in Module 26)

**What to look for during AD enumeration:**
- Domain Admins group members — highest-value accounts
- Service accounts with SPNs set — Kerberoastable (Module 24)
- Accounts with pre-authentication disabled — ASREPRoastable (Module 24)
- Computers where Domain Admins are logged in — lateral movement targets
- Misconfigured ACLs — users with unexpected write access to sensitive AD objects
- Password policy — specifically whether account lockout is configured or absent

## 🧠 Key Concepts

- **A low-privileged domain account is enough to map the entire organization.** The reconnaissance phase inside AD costs essentially nothing once any valid credential is obtained — phishing a single standard user hands over the full directory.
- **Service accounts with SPNs are high-value targets before any exploitation even begins.** A "DO NOT CHANGE" description on a service account is an attacker's signal that the password is static, never rotated, and likely weak enough to crack — directly feeding Module 24's Kerberoasting.
- **No account lockout policy means password spraying is completely undetected.** If `domain_policy.html` shows lockout threshold of 0, the entire Module 11 spraying technique applies with zero risk of triggering any defensive control.
- **Find-LocalAdminAccess is one of the highest-value PowerView commands** — it tells you exactly which machines a compromised account can access with admin rights, directly feeding lateral movement without needing to guess.

## 🧪 Practical Lab

### Objective
Enumerate the full AD structure of lab.local from a single low-privileged user account, identifying Domain Admins, Kerberoastable service accounts, misconfigured permissions, and password policy weaknesses.

### Environment
Kali Linux (192.168.52.129) as attacker, DC01.lab.local (192.168.52.200) as Domain Controller, CLIENT01.lab.local as domain-joined Windows client, compromised account: mchen (standard HR user, lab.local\mchen:Password123)

### Rules of Engagement
- Enumeration only — no exploitation of discovered findings during this module
- Target limited strictly to the lab.local AD environment

### Tools
net (built-in Windows), PowerView, ldapdomaindump, Kali Linux

### Commands
net user /domain
net group "Domain Admins" /domain
Import-Module .\PowerView.ps1
Get-NetDomain
Get-NetUser | select samaccountname, description, pwdlastset
Get-NetUser -SPN | select samaccountname, serviceprincipalname
Find-LocalAdminAccess
ldapdomaindump -u 'lab.local\mchen' -p 'Password123' 192.168.52.200

## 🔎 Findings

**Built-in net commands:**
- Full user list extracted: Administrator, bwilson, Guest, jsmith, krbtgt, mchen, svc_backup, svc_sql
- Domain Admins confirmed: Administrator and jsmith

**PowerView — Get-NetUser:**
- svc_backup and svc_sql identified as service accounts; svc_sql description reads "SQL Service Account — DO NOT CHANGE" — a direct indicator of a static, never-rotated password
- Password last set dates for both service accounts: 01/15/2026 — unchanged since domain setup

**PowerView — Get-NetUser -SPN:**
- svc_backup SPN: backup/DC01.lab.local
- svc_sql SPN: MSSQLSvc/DC01.lab.local:1433
- Both accounts confirmed Kerberoastable — SPNs set, running as domain user accounts rather than built-in service accounts

**PowerView — Find-LocalAdminAccess:**
- CLIENT01.lab.local returned — mchen has unexpected local admin access on this machine, likely due to a misconfigured local admin policy

**ldapdomaindump — domain_policy.html:**
- Minimum password length: 8
- Password history: 24
- Max password age: 42 days
- Account lockout threshold: 0 — no lockout policy configured

## 💡 What I Discovered

Running full AD enumeration from a single standard user account made the design tension concrete — with one low-privileged credential, the entire domain structure was visible before touching a single offensive tool. The two findings that stood out most were the "DO NOT CHANGE" service account description (essentially advertising a static password target) and the absent lockout policy (removing the primary defensive control against credential spraying). Both findings existed purely because of how AD was configured, not because of any technical vulnerability — exactly the pattern from Module 15 and 16's privilege escalation lessons.

## 🛡️ Defensive Perspective

- Account lockout policies should always be configured — a threshold of 0 removes the primary control against both brute force and password spraying across the entire domain.
- Service account descriptions should never contain operational notes about password management — "DO NOT CHANGE" advertises a static password to any attacker who enumerates the directory.
- Service accounts should be migrated to Group Managed Service Accounts (gMSA) where possible — Windows manages gMSA passwords automatically (240-character random, rotated regularly), eliminating Kerberoasting risk entirely.
- Privileged account activity (Domain Admin logins, service account authentications) should be monitored centrally — SIEM alerts on unusual login patterns for high-value accounts are essential.
- Local admin rights should be audited regularly — unexpected local admin access (like mchen on CLIENT01) is a lateral movement path waiting to be used.

## 📝 Lessons Learned

AD enumeration is where the quality of an attacker's subsequent targeting is determined — every finding here feeds a specific follow-on technique. Kerberoastable SPNs feed Module 24, misconfigured local admin access feeds lateral movement, absent lockout policy unlocks password spraying, and the Domain Admin list becomes the target list. Enumeration done well means exploitation done precisely, rather than randomly.

## 🏁 Conclusion

Module 23 enumerated the full lab.local AD structure from a single standard user account — extracting the complete user list, Domain Admin identities, two Kerberoastable service accounts, a misconfigured local admin permission, and a missing lockout policy, all before performing any exploitation. This sets up directly for Module 24: Kerberos Fundamentals, where the two identified service accounts (svc_backup, svc_sql) become the specific targets for Kerberoasting.
