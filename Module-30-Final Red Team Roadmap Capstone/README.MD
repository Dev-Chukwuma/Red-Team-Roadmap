# 🔴 Module 30 — Final Red Team Capstone

## 📚 What This Module Covers

Module 30 is the professional documentation phase of the Red Team roadmap — producing the report deliverable that would reach a real client after an engagement like the one simulated in Module 29. Technical work without professional reporting achieves nothing in a real engagement; the report is what communicates findings to stakeholders, drives remediation decisions, and demonstrates the business impact of discovered vulnerabilities. This module covers report structure, finding documentation, severity rating, and the full ACME Corporation engagement report.

---

# ACME CORPORATION
# RED TEAM ASSESSMENT REPORT
# Prepared by: Divine Chukwuma
# Classification: Confidential
# Assessment Period: Simulated Engagement — lab.local / 192.168.52.0/24

---

## 1. EXECUTIVE SUMMARY

During an authorized Red Team engagement against ACME Corporation's internal environment, the assessment team achieved full Domain Admin access, established three independent persistence mechanisms, and recovered every credential in the organization's Active Directory environment — beginning from a single unpatched FTP service exposed on the internal network.

The assessment demonstrated that an attacker with no prior knowledge of the environment and no initial credentials could progress from external reconnaissance through complete organizational compromise in a single engagement window. The path from first access to Domain Admin required chaining seven distinct findings, none of which individually appeared catastrophic — but together produced full, persistent control over the entire domain.

Three findings require immediate remediation: the vsftpd 2.3.4 backdoor (a publicly known, 15-year-old vulnerability with a one-command Metasploit exploit), the absent Active Directory account lockout policy (enabling unlimited password spraying with zero detection risk), and the AdminSDHolder persistence mechanism installed during the engagement (which survives all standard credential rotation and will re-propagate every 60 minutes until explicitly removed).

Overall risk posture: CRITICAL.

---

## 2. SCOPE AND RULES OF ENGAGEMENT

Authorized scope:
- Internal network: 192.168.52.0/24
- Active Directory domain: lab.local
- All hosts within scope unless explicitly excluded

Out of scope:
- Denial of service attacks
- Physical access attempts
- Social engineering of real personnel
- Any host outside the 192.168.52.0/24 subnet

Authorization: Full written authorization obtained prior to engagement commencement. All activities conducted within authorized boundaries.

---

## 3. ATTACK NARRATIVE

### Phase 1 — Reconnaissance

Passive reconnaissance began with OSINT against ACME Corporation's public presence. Job postings revealed AWS infrastructure dependencies, LinkedIn surfaced employee names and department roles providing potential Active Directory username candidates, and certificate transparency logs via crt.sh surfaced a staging subdomain not linked from the main site. The target's robots.txt file disclosed two sensitive paths (/admin/ and /internal/) without any active scanning required.

Active network reconnaissance via Nmap across 192.168.52.0/24 identified three live hosts: a Linux server running multiple outdated services including vsftpd 2.3.4 (192.168.52.128), a Domain Controller (DC01 at 192.168.52.200), and a domain-joined Windows workstation (CLIENT01 at 192.168.52.130). Web enumeration via Gobuster against the Linux server surfaced three distinct web applications (DVWA, Mutillidae, phpMyAdmin) running on Apache 2.2.8 and PHP 5.2.4 — both significantly outdated versions.

### Phase 2 — Initial Access

CVE research against the identified vsftpd 2.3.4 version confirmed a publicly known backdoor (CVE-2011-2523) with an existing Metasploit module. Exploitation required one command — setting RHOSTS to 192.168.52.128 and running the module returned a root shell on port 6200 in under 30 seconds, with zero credentials required. whoami confirmed root, id confirmed uid=0(root) gid=0(root).

Separately, SQL injection testing against DVWA's User ID field confirmed unsanitized input reaching the database query directly — payload 1' OR '1'='1 returned the full users table rather than a single record, demonstrating unauthenticated database read access from a single input field.

### Phase 3 — Post-Exploitation

Situational awareness commands confirmed the compromised host as Ubuntu 8.04 running kernel 2.6.24 — 18 years out of patch support, with multiple applicable kernel CVEs. ARP cache enumeration via arp -a immediately confirmed both DC01 and CLIENT01 as live internal hosts without sending a single network probe.

Credential discovery across configuration files, shell history, environment variables, and the filesystem recovered five distinct credential sets including a domain account (mchen:Password123), two service account passwords, an SSH private key at /root/.ssh/id_rsa, and an API key stored in an environment variable. These credentials were recovered from a single machine in under five minutes.

Lateral movement using discovered credentials achieved: root access on a third internal host via the recovered SSH private key (no password required), authenticated shell on CLIENT01 via password reuse (admin:SuperSecret2024!), and NT AUTHORITY\SYSTEM on a Windows host via Impacket psexec. Network pivoting via Metasploit route add exposed a previously invisible 192.168.100.0/24 subnet containing three additional live hosts.

### Phase 4 — Active Directory

BloodHound data collection using the discovered domain credential mchen:Password123 completed in 8 seconds, returning the complete lab.local relationship graph. Analysis immediately surfaced a three-hop attack path: mchen held WriteDACL rights over svc_sql, svc_sql was a member of IT Admins, and IT Admins held local admin rights on DC01. Additionally, Domain Admin jsmith was confirmed to have an active session on CLIENT01 — meaning their credentials were sitting in that workstation's memory.

Kerberoasting via Impacket GetUserSPNs.py extracted a Service Ticket hash for svc_sql — hashcat cracked it against rockyou.txt in seconds, recovering plaintext password Sqlserver2019!. ASREPRoasting via GetNPUsers.py extracted bwilson's TGT hash with zero credentials required, as pre-authentication had been disabled on that account. Password spraying via CrackMapExec confirmed bwilson used the common password Welcome2024! — with no account lockout policy configured, this spray carried zero risk of detection or lockout.

The BloodHound attack path was executed step by step: PowerView's Add-DomainObjectAcl abused mchen's WriteDACL right to grant GenericAll over svc_sql, Set-DomainUserPassword reset svc_sql's password, and psexec.py authenticated to DC01 using the new credential — confirming the IT Admins → AdminTo → DC01 path. Evil-WinRM opened a shell on CLIENT01, where Mimikatz sekurlsa::logonpasswords extracted jsmith's plaintext password (DomainAdmin2024!) directly from memory. WMIExec authenticated to DC01 as jsmith, and secretsdump returned the full NTDS.dit contents including every domain hash and the krbtgt account hash (6d5c4b3a2c1d0e9f8a7b6c5d4e3f2a1b).

Three persistence mechanisms were installed: a Golden Ticket forged from the krbtgt hash (valid until 2036, requiring no KDC contact, surviving all account password resets except krbtgt), AdminSDHolder abuse via Add-DomainObjectAcl on CN=AdminSDHolder propagating GenericAll to every protected account via SDProp every 60 minutes (surviving all credential rotation), and DCShadow adding mchen to Domain Admins via rogue DC replication (generating no Event ID 4728 group-change audit log).

---

## 4. FINDINGS

FINDING 01 — vsftpd 2.3.4 Backdoor (CVE-2011-2523)
Severity: Critical
Description: The vsftpd 2.3.4 package contains a deliberately inserted backdoor — any FTP connection using a username ending in the characters :) triggers a root shell listener on port 6200. This vulnerability has been publicly known since 2011 and has a one-command Metasploit exploit.
Evidence: Metasploit module exploit/unix/ftp/vsftpd_234_backdoor executed against 192.168.52.128 returned a root shell in under 30 seconds. whoami returned root, id returned uid=0(root) gid=0(root).
Business Impact: Complete, unauthenticated root-level compromise of the affected server — full control of all data, services, and credentials on the machine, with no user interaction required.
Remediation: Immediately replace vsftpd 2.3.4 with a current, supported version. Conduct a full audit of all services running on affected hosts to identify additional outdated software.

FINDING 02 — SQL Injection on DVWA (Unauthenticated Database Read)
Severity: Critical
Description: The User ID parameter on DVWA's SQL Injection module concatenates user input directly into a SQL query without sanitization or parameterization. An attacker can manipulate the query logic to return arbitrary database records.
Evidence: Payload 1' OR '1'='1 submitted in the User ID field returned every record in the users table rather than a single user — demonstrating full table read access from a single input field.
Business Impact: Full read access to the underlying database from any user who can access the application — including potential access to other tables containing sensitive organizational data, credentials, or customer records.
Remediation: Implement parameterized queries (prepared statements) for all database interactions. Input validation and output encoding should be applied at every entry point. Conduct a full code review for similar patterns across all web applications.

FINDING 03 — Absent Account Lockout Policy
Severity: Critical
Description: The Active Directory domain policy has no account lockout threshold configured (threshold: 0). This means unlimited failed authentication attempts can be made against any domain account with no automatic lockout triggered.
Evidence: ldapdomaindump domain_policy.html confirmed lockout threshold of 0. CrackMapExec password spray (Welcome2024! against all domain users) completed without triggering any lockout — bwilson authenticated successfully.
Business Impact: An attacker with a valid username list can conduct unlimited password spraying or brute force attempts against any domain account with zero risk of lockout or automatic detection. One weak password anywhere in the organization is sufficient to gain domain access.
Remediation: Configure an account lockout policy immediately — recommended threshold of 5-10 failed attempts within a 15-minute window, with a 15-30 minute lockout duration. Implement fine-grained password policies for privileged accounts.

FINDING 04 — AdminSDHolder Persistence (Active — Requires Immediate Remediation)
Severity: Critical
Description: During the engagement, GenericAll rights were written to the AdminSDHolder object's ACL for the mchen account. AD's SDProp process re-propagates this ACL to every protected account in the domain every 60 minutes automatically. This persistence survives all password resets and credential rotation because it is an ACL entry on an AD object, not a credential.
Evidence: Add-DomainObjectAcl successfully modified AdminSDHolder's ACL. Get-DomainObjectAcl confirmed GenericAll propagated to Domain Admins group within 60 seconds of SDProp execution.
Business Impact: mchen permanently holds GenericAll over every protected account in the domain — including Domain Admins — until this specific ACL entry is removed. All credential rotation performed without removing this entry leaves the domain fully re-compromisable at any time.
Remediation: Immediately audit AdminSDHolder's ACL (CN=AdminSDHolder,CN=System,DC=lab,DC=local) and remove any non-standard entries. This must be done before any other remediation — it is the highest-priority finding in this report.

FINDING 05 — Kerberoastable Service Accounts
Severity: High
Description: Two service accounts (svc_sql, svc_backup) have Service Principal Names (SPNs) registered and use weak, static passwords. Any authenticated domain user can request a Service Ticket encrypted with these accounts' password hashes and crack them offline — with no lockout risk and no interaction with the accounts themselves.
Evidence: GetUserSPNs.py returned Service Ticket hashes for svc_sql and svc_backup. hashcat cracked svc_sql's hash in seconds — recovered password: Sqlserver2019!
Business Impact: Full control of both service accounts, their associated services (SQL Server, backup infrastructure), and any systems those accounts have access to. svc_sql's IT Admins membership translated directly to local admin access on the Domain Controller.
Remediation: Migrate service accounts to Group Managed Service Accounts (gMSA) — Windows manages gMSA passwords automatically (240-character random, rotated regularly), eliminating Kerberoastability entirely. Where gMSA is not possible, immediately set service account passwords to 40+ character random strings and rotate regularly.

FINDING 06 — Domain Admin Session on Standard Workstation
Severity: High
Description: Domain Admin account jsmith was found to have an active interactive session on CLIENT01 (192.168.52.130) — a standard workstation with a significantly larger attack surface than the Domain Controller. Active sessions leave credential material in LSASS memory, extractable by any process running with sufficient privileges on that machine.
Evidence: BloodHound HasSession query confirmed jsmith → HasSession → CLIENT01. Mimikatz sekurlsa::logonpasswords on CLIENT01 extracted jsmith's plaintext password: DomainAdmin2024!
Business Impact: Any attacker who compromises CLIENT01 (via any vulnerability — not just the ones exploited in this engagement) can extract Domain Admin credentials from memory without ever touching the Domain Controller.
Remediation: Implement a tiered access model — Domain Admins should only authenticate to Tier 0 assets (Domain Controllers) via dedicated Privileged Access Workstations (PAWs), never to standard workstations. Enable Credential Guard on all workstations to prevent plaintext credential extraction from LSASS.

FINDING 07 — Password Reuse Across Systems
Severity: High
Description: Credentials discovered on one compromised system authenticated successfully against multiple other internal systems. The credential admin:SuperSecret2024! found in /tmp/credentials.txt on Metasploitable authenticated to CLIENT01 via SSH.
Evidence: SSH with admin:SuperSecret2024! against 192.168.52.130 returned an authenticated interactive shell.
Business Impact: A single compromised machine's credential store becomes a master key set for the internal network — one system's compromise directly enables lateral movement to others with no additional exploitation required.
Remediation: Enforce unique credentials per system for all administrative and service accounts. Implement a privileged access management (PAM) solution for centralized credential management and rotation. Conduct an organization-wide credential audit.

FINDING 08 — WriteDACL Misconfiguration on mchen
Severity: Medium
Description: Standard domain user mchen holds WriteDACL rights over the svc_sql service account — allowing mchen to modify svc_sql's ACL and grant themselves or any other account arbitrary rights over it, including the ability to reset svc_sql's password.
Evidence: BloodHound identified mchen → WriteDACL → svc_sql. Add-DomainObjectAcl and Set-DomainUserPassword successfully abused this right to reset svc_sql's password without any additional privileges.
Business Impact: mchen's WriteDACL right over svc_sql, combined with svc_sql's IT Admins membership, created a three-hop path from a standard HR account to local admin on the Domain Controller — demonstrating how individually minor ACL misconfigurations chain into high-impact attack paths.
Remediation: Audit all non-standard ACL entries on service accounts and privileged objects. Remove WriteDACL and GenericAll rights from all non-admin accounts. Implement regular BloodHound-based ACL auditing as a recurring security control.

FINDING 09 — Outdated Web Stack (Apache 2.2.8 / PHP 5.2.4)
Severity: Medium
Description: The web server at 192.168.52.128 runs Apache 2.2.8 (released 2008, EOL 2018) and PHP 5.2.4 (released 2007, EOL 2011) — both significantly outdated with large pools of known CVEs.
Evidence: whatweb returned Apache[2.2.8] and PHP[5.2.4-2ubuntu5.10] in response headers.
Business Impact: Multiple known vulnerabilities exist against both versions, providing additional exploitation paths beyond those used in this engagement. The outdated stack also signals generally poor patch management hygiene across the environment.
Remediation: Upgrade Apache to a currently supported version (2.4.x or later) and PHP to a currently supported version (8.x). Implement a regular patching cadence and software inventory review.

FINDING 10 — Unquoted Service Path
Severity: Medium
Description: The BackupSyncService Windows service has an unquoted executable path containing spaces — C:\Program Files\Backup Sync\backupsync.exe — combined with a containing folder writable by standard users. Windows resolves unquoted paths by trying each space-separated segment, allowing a low-privileged user to place a malicious executable at an earlier resolved path.
Evidence: wmic enumeration identified the unquoted path. icacls confirmed BUILTIN\Users:(M) on the containing folder. Malicious executable placed at C:\Program Files\Backup.exe — service restart executed it as SYSTEM.
Business Impact: Any standard domain user can escalate to SYSTEM on any machine running this service configuration — bypassing all privilege boundaries without any credential theft or vulnerability exploitation.
Remediation: Quote all service executable paths that contain spaces. Restrict write permissions on all directories referenced by SYSTEM-running services to administrators only. Audit all service paths via wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v.

FINDING 11 — Credentials in Bash History and Config Files
Severity: Medium
Description: Multiple plaintext credentials were stored in shell history files and web application configuration files on the compromised Linux host — including database passwords, SSH commands with remote host addresses, and service account credentials.
Evidence: cat ~/.bash_history returned MySQL root credentials (root:root123), SSH commands to internal hosts, and PostgreSQL credentials (postgres:postgres123). Config files returned database passwords for DVWA and Mutillidae.
Business Impact: Any attacker with read access to these files (not just root) recovers a full credential inventory and internal network map — including hosts and services not otherwise discoverable.
Remediation: Implement secrets management (HashiCorp Vault, AWS Secrets Manager) for all credentials — never store them in config files or environment variables accessible to applications. Configure HISTCONTROL=ignorespace in shell profiles to prevent credential-containing commands from being saved to history.

FINDING 12 — robots.txt Disclosing Sensitive Paths
Severity: Low
Description: The target web server's robots.txt file explicitly listed /admin/ and /internal/ as disallowed paths — advertising their existence to any visitor, including attackers.
Evidence: curl http://192.168.52.128/robots.txt returned Disallow: /admin/ and Disallow: /internal/.
Business Impact: robots.txt is publicly accessible and freely readable — using it to list sensitive paths provides attackers with a reconnaissance shortcut, confirming the existence of administrative interfaces without any active scanning required.
Remediation: Remove all sensitive path references from robots.txt. Sensitive paths should be protected by authentication and authorization controls, not by obscurity through robots.txt exclusions.

FINDING 13 — Staging Subdomain Exposed via Certificate Transparency
Severity: Low
Description: A staging subdomain was identified via certificate transparency logs (crt.sh) — publicly indexed without any active scanning required.
Evidence: crt.sh query against the target domain returned a staging subdomain not linked from the main site.
Business Impact: Staging environments typically run less-hardened configurations and older software versions than production — an exposed staging subdomain provides an additional attack surface that may be easier to exploit than the production environment.
Remediation: Staging environments should not be internet-accessible. Use private certificate authorities for internal/staging subdomains, or implement access controls that prevent external enumeration.

---

## 5. RISK RATING MATRIX

Critical (Immediate action required — within 24 hours):
01 — vsftpd 2.3.4 Backdoor (CVE-2011-2523)
02 — SQL Injection on DVWA
03 — Absent Account Lockout Policy
04 — AdminSDHolder Persistence (Active)

High (Action required — within 1 week):
05 — Kerberoastable Service Accounts
06 — Domain Admin Session on Standard Workstation
07 — Password Reuse Across Systems

Medium (Action required — within 1 month):
08 — WriteDACL Misconfiguration on mchen
09 — Outdated Web Stack (Apache 2.2.8 / PHP 5.2.4)
10 — Unquoted Service Path
11 — Credentials in Bash History and Config Files

Low (Action required — within 3 months):
12 — robots.txt Disclosing Sensitive Paths
13 — Staging Subdomain Exposed via Certificate Transparency

---

## 6. REMEDIATION ROADMAP

Priority 1 — Immediate (24 hours):
Remove AdminSDHolder ACL entry for mchen — this is active persistence currently installed in the environment and re-propagates every 60 minutes until removed. Reset krbtgt password twice (10 hours apart) to invalidate the Golden Ticket. Patch or replace vsftpd 2.3.4. Configure account lockout policy.

Priority 2 — Short term (1 week):
Migrate service accounts to gMSA. Implement Credential Guard on all workstations. Enforce tiered access model for Domain Admins. Conduct organization-wide credential audit and enforce unique passwords per system.

Priority 3 — Medium term (1 month):
Audit all AD ACLs for non-standard WriteDACL/GenericAll entries. Upgrade Apache and PHP. Fix all unquoted service paths. Implement secrets management for all credentials.

Priority 4 — Longer term (3 months):
Implement regular BloodHound-based ACL auditing as a recurring control. Remove sensitive paths from robots.txt. Restrict staging environment access. Implement centralized SIEM monitoring with alerts for DCSync, DCShadow, unusual Kerberos ticket lifetimes, and DA logins on non-DC machines.

---

## 7. CONCLUSION

This engagement demonstrated that full organizational compromise was achievable from a single unpatched FTP service — not through sophisticated zero-day exploitation, but through a chain of individually manageable findings that compounded into complete domain control. No single finding caused the compromise; the chain did.

The most significant risk this engagement identified is not any individual vulnerability — it is the absence of defense in depth. Any one of the following controls, implemented before this engagement, would have broken the kill chain: patching CVE-2011-2523, configuring an account lockout policy, implementing Credential Guard, or auditing AD ACLs for non-standard entries. None are technically complex. All are achievable within standard IT operations.

The persistence mechanisms installed during this engagement (AdminSDHolder, Golden Ticket) require specific, targeted remediation steps that go beyond standard incident response. Failure to remove the AdminSDHolder ACL entry specifically will leave the domain fully re-compromisable regardless of all other remediation performed.

Overall risk posture: CRITICAL. Immediate action required on Priority 1 findings before any other remediation is undertaken.

---

## 📝 Lessons Learned

A Red Team report is not a vulnerability list — it is a business risk communication. Every finding needs a business impact statement that a non-technical stakeholder can understand and act on. The remediation roadmap matters as much as the findings themselves — identifying a problem without providing a prioritized, actionable path to fix it leaves the client no better off than before the engagement. Professional Red Team work is equal parts technical skill and communication.

## 🏁 Conclusion

Module 30 produced the professional report deliverable for the ACME Corporation simulated engagement — 13 documented findings across four severity levels, a prioritized remediation roadmap, and a complete executive summary communicating business risk to non-technical stakeholders. This completes the 30 Days of Looting Red Team roadmap — 30 modules covering reconnaissance, exploitation, post-exploitation, Active Directory, and Red Team simulation, documented as a complete professional portfolio.

30 modules. Done.
