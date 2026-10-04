# 🔴 Module 29 — Full Red Team Simulation

## 📚 What This Module Covers

Module 29 is the Phase 5 simulation — executing every phase of the roadmap as one complete, realistic Red Team engagement. No new techniques were introduced; the goal was demonstrating that all 28 prior modules form one continuous, professional attack chain when executed with deliberate sequencing and clear objective tracking.

## 🎯 Engagement Brief

Client: ACME Corporation (lab environment)
Scope: Full internal network — 192.168.52.0/24 + lab.local AD domain
Authorization: Full simulated Red Team engagement
Primary Objective: Gain Domain Admin, demonstrate persistent access, document full attack chain
Rules of Engagement: No denial of service, document all actions, no lateral movement outside defined scope

## 🗺️ Kill Chain Overview

Phase 1 — Reconnaissance (Modules 03-06)
↓
Phase 2 — Initial Access (Modules 08-13)
↓
Phase 3 — Post-Exploitation (Modules 15-20)
↓
Phase 4 — Active Directory (Modules 22-28)
↓
Objective: Domain Admin + Persistence + Impact Demonstration
↓
Phase 5 — Reporting

## 🔎 Phase 1 — Reconnaissance

Passive reconnaissance on ACME Corporation surfaced AWS infrastructure via job postings, employee names and roles via LinkedIn (potential AD usernames), Route53 nameservers via WHOIS, and a staging subdomain via crt.sh. The target's own robots.txt disclosed /admin/ and /internal/ without any active scanning required.

Active network reconnaissance via nmap -sV -sC -O 192.168.52.0/24 discovered three live hosts: 192.168.52.128 (Metasploitable — vsftpd 2.3.4, Apache 2.2.8, SMB open), 192.168.52.200 (DC01 — Kerberos 88, LDAP 389, SMB 445, DNS 53), and 192.168.52.130 (CLIENT01 — SMB 445, WinRM 5985).

Web enumeration via gobuster and whatweb against 192.168.52.128 found DVWA, Mutillidae, and phpMyAdmin running on outdated Apache 2.2.8 and PHP 5.2.4. Service enumeration via enum4linux confirmed null SMB sessions and weak password policy on Metasploitable.

## 🔎 Phase 2 — Initial Access

Vulnerability assessment identified vsftpd 2.3.4 on port 21 — CVE-2011-2523, critical severity, public Metasploit module available. Web vulnerability assessment against DVWA confirmed SQL injection on the User ID field.

vsftpd exploitation via Metasploit (exploit/unix/ftp/vsftpd_234_backdoor, RHOSTS 192.168.52.128) returned a root shell on port 6200 — confirmed via whoami (root) and id (uid=0, gid=0). Zero credentials required.

Web exploitation via SQLi payload (1' OR '1'='1) against DVWA's User ID field returned the full users table — demonstrating unauthenticated database read access from a single input field.

## 🔎 Phase 3 — Post-Exploitation

Situational awareness commands (whoami, id, hostname, uname -a, ip a, arp -a) confirmed Ubuntu 8.04 kernel 2.6.24, root-level access, and two additional live internal hosts via ARP cache — DC01 (192.168.52.200) and CLIENT01 (192.168.52.130) — without sending a single probe packet.

Credential discovery across config files, bash history, environment variables, and SSH key locations recovered: mchen:Password123 (domain credential), admin:SuperSecret2024!, backup_user:Backup@123, postgres:postgres123, SSH private key at /root/.ssh/id_rsa, and API key in environment variables. Five distinct credential sets from one compromised machine.

Meterpreter session established via msfvenom staged payload (linux/x86/meterpreter/reverse_tcp) — in-memory execution, encrypted channel, no file left on disk after execution.

Lateral movement using discovered credentials: SSH with private key authenticated directly to 192.168.52.150 as root (no password required), SSH with admin:SuperSecret2024! authenticated to 192.168.52.130 (password reuse confirmed), impacket-psexec with msfadmin:msfadmin landed NT AUTHORITY\SYSTEM on a Windows host. Pivoting via Metasploit route add revealed a previously invisible 192.168.100.0/24 subnet with three additional live hosts.

Linux privilege escalation demonstrated via SUID binary abuse — vim.basic identified with SUID set, exploited via vim.basic -c ':!/bin/sh' to escalate from www-data to root. Windows privilege escalation demonstrated via unquoted service path — BackupSyncService running as SYSTEM with writable containing folder, malicious executable placed at resolved path, service restart landed NT AUTHORITY\SYSTEM.

## 🔎 Phase 4 — Active Directory

AD enumeration via bloodhound-python using mchen:Password123 collected the full lab.local relationship graph in 8 seconds — 6 users, 4 groups, 2 computers. BloodHound identified: attack path mchen → WriteDACL → svc_sql → MemberOf → IT Admins → AdminTo → DC01, two Kerberoastable service accounts (svc_sql, svc_backup), jsmith (Domain Admin) HasSession on CLIENT01, and absent account lockout policy enabling unrestricted password spraying.

Kerberoasting via GetUserSPNs.py returned svc_sql's Service Ticket hash ($krb5tgs$23$) — hashcat mode 13100 cracked it against rockyou.txt, recovering plaintext password Sqlserver2019!. ASREPRoasting via GetNPUsers.py returned bwilson's TGT hash with zero credentials required (pre-authentication disabled).

Password spraying via CrackMapExec confirmed bwilson:Welcome2024! — one common password hit across the full user list with no lockout risk.

AD lateral movement executed the full BloodHound path: Add-DomainObjectAcl granted mchen GenericAll over svc_sql via WriteDACL abuse, Set-DomainUserPassword reset svc_sql's password to Hacked123!, psexec.py authenticated to DC01 as svc_sql confirming IT Admins → AdminTo → DC01 path. Evil-WinRM opened an interactive shell on CLIENT01 as mchen — Mimikatz sekurlsa::logonpasswords extracted jsmith's plaintext password (DomainAdmin2024!) from memory, confirming the HasSession finding. WMIExec authenticated to DC01 as jsmith (no service creation, no Event ID 7045).

Full NTDS.dit dump via secretsdump.py returned every domain hash including krbtgt (6d5c4b3a2c1d0e9f8a7b6c5d4e3f2a1b) and Administrator. DCSync extracted krbtgt via legitimate replication protocol — no file access, no disk operation.

Domain privilege escalation: Golden Ticket forged using krbtgt hash — valid 10/03/2026 to 10/03/2036, injected into memory via /ptt, full DC01 C$ access confirmed with no KDC contact. AdminSDHolder abuse via Add-DomainObjectAcl on CN=AdminSDHolder propagated GenericAll to every protected account via SDProp — persistence surviving all password resets. DCShadow registered a rogue DC and pushed mchen's primaryGroupID to 512 (Domain Admins) via replication — no Event ID 4728 generated.

## ✅ Objectives Achieved

Initial access — vsftpd backdoor, root shell on Metasploitable, zero credentials
Web exploitation — SQLi on DVWA, full user table extracted
Credential discovery — five credential sets, two SSH keys, one API key from one compromised machine
Lateral movement — SSH key access, password reuse, Impacket remote execution, subnet pivot
Domain enumeration — BloodHound attack path, Kerberoastable accounts, DA session identified
Kerberoasting — svc_sql cracked (Sqlserver2019!), bwilson ASREPRoasted
Domain Admin — jsmith credentials + Golden Ticket (valid 2036)
Persistence — AdminSDHolder (survives password resets) + Golden Ticket (survives account lockouts) + DCShadow (bypasses group auditing)
Full NTDS.dit dump — every domain hash recovered including krbtgt

## 💡 What I Discovered

Running the full engagement end to end as one narrative confirmed what every prior capstone module hinted at: no individual finding was the engagement. The vsftpd backdoor alone would have been patched and forgotten. The WriteDACL misconfiguration alone might never have been noticed. The weak svc_sql password alone would have seemed minor. Together, chained in sequence, they became full organizational compromise with three independent persistence mechanisms. The whole is always greater than the sum of its parts in a real Red Team engagement.

## 🛡️ Defensive Perspective

A single defensive control at any link in this chain would have shortened or stopped it entirely. Patching CVE-2011-2523 closes initial access. Unique service account passwords defeat Kerberoasting. Credential Guard prevents Mimikatz from extracting plaintext from LSASS. Network segmentation limits lateral movement. Egress filtering reduces reverse shell effectiveness. Monitoring for non-DC replication requests detects DCSync and DCShadow. Auditing AdminSDHolder's ACL catches persistence before it's abused. Defense in depth — not any single control — is what breaks engagement chains like this one.

## 📝 Lessons Learned

A Red Team engagement is not a collection of attacks. It is one story — reconnaissance informs targeting, targeting informs exploitation, exploitation enables credential discovery, credentials enable lateral movement, movement enables domain access, domain access enables persistence. Every module in this roadmap was one chapter of that story. Module 29 was the complete book.

## 🏁 Conclusion

Module 29 executed the full Red Team kill chain — passive and active reconnaissance, initial access via a known CVE, post-exploitation enumeration, credential discovery, privilege escalation across both Linux and Windows, lateral movement across the network, Active Directory enumeration and exploitation, Kerberoasting, credential attacks, BloodHound path execution, Domain Admin compromise, and three independent persistence mechanisms — as one complete, documented simulation against an authorized lab environment. One module remains: Module 30 — Final Red Team Capstone.
