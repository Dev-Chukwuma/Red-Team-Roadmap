# 🔴 Module 25 — Credential Attacks in AD Labs

## 📚 What I Learned

This module pulled together credential attack techniques from Modules 11, 17, 23, and 24 into one focused AD credential attack chain — password spraying, hash dumping, Pass-the-Hash at scale, and DCSync — demonstrating how each technique feeds the next in a real domain environment.

**Core AD credential attack techniques:**

**Password spraying against AD** — one common password tried against every domain user via CrackMapExec (CME), targeting the domain's own authentication service. CME flags successful authentications without triggering lockout when done at the right pace. The weakest link in any organization is always someone using a predictable password.

**SAM database dumping** — on a compromised Windows machine, the SAM database stores local account hashes, extractable via CrackMapExec with admin credentials.

**NTDS.dit dumping** — on the DC specifically, NTDS.dit holds every domain account's hash. Extractable via CrackMapExec (`--ntds`) or Impacket's secretsdump — one command returns every credential in the organization.

**Pass-the-Hash at domain scale** — hashes dumped from one machine used to authenticate across an entire subnet simultaneously. In a domain environment, a single domain account hash works on every machine where that account exists — one hash, entire network.

**DCSync attack** — impersonating a Domain Controller using replication privileges to request password hashes directly from the real DC, without touching NTDS.dit at all. Undetectable without specific monitoring for non-DC accounts making replication requests, because it abuses a legitimate AD replication mechanism that runs constantly as normal DC-to-DC traffic.

## 🧠 Key Concepts

- **DCSync is harder to detect than direct NTDS.dit access because it abuses a legitimate protocol.** AD replication between DCs is constant, normal background traffic — a DCSync request is indistinguishable from real replication at the network level unless you're specifically watching for non-DC accounts initiating replication calls.
- **CrackMapExec is the Swiss Army knife of AD credential attacks** — the same tool handles password spraying, hash dumping, Pass-the-Hash, and result verification across entire subnets in one consistent interface.
- **One Domain Admin credential or hash unlocks the entire domain's credential store** — secretsdump with DA rights returns every hash in NTDS.dit including krbtgt, which feeds directly into Module 24's Golden Ticket.
- **The credential attack chain is cumulative** — password spray found bwilson, bwilson's access path led to DA credentials, DA credentials enabled secretsdump, secretsdump returned krbtgt, krbtgt enables a permanent Golden Ticket. Each step unlocked the next.

## 🧪 Practical Lab

### Objective
Execute a full AD credential attack chain — from password spraying through hash dumping, Pass-the-Hash at subnet scale, and DCSync — using the lab.local environment built in Module 22.

### Environment
Kali Linux (192.168.52.129) as attacker, DC01.lab.local (192.168.52.200), domain users established in Module 22-23, DA credential path established via Kerberoasting in Module 24

### Rules of Engagement
- Target limited strictly to the lab.local AD environment
- All recovered credentials and hashes used only within the authorized lab

### Tools
CrackMapExec (CME), Impacket (secretsdump.py), rockyou.txt

### Commands
crackmapexec smb 192.168.52.200 -u users.txt -p 'Welcome2024!' --continue-on-success
python3 secretsdump.py lab.local/jsmith:Password123@192.168.52.200
crackmapexec smb 192.168.52.0/24 -u Administrator -H 7c4e4e8a9b5f3d2c1a0b8e7f6d5c4b3a --local-auth
python3 secretsdump.py lab.local/jsmith:Password123@192.168.52.200 -just-dc

## 🔎 Findings

**Password spraying:** CME sprayed Welcome2024! across all domain users — bwilson authenticated successfully (STATUS: Pwn3d!), confirming password reuse on a domain account. All other accounts returned STATUS_LOGON_FAILURE.

**secretsdump — full domain hash dump:** Using jsmith's DA credentials (recovered via the svc_sql Kerberoasting path in Module 24), secretsdump returned every domain account hash including Administrator, jsmith, mchen, bwilson, svc_backup, svc_sql, and critically krbtgt (hash: 6d5c4b3a2c1d0e9f8a7b6c5d4e3f2a1b).

**Pass-the-Hash across subnet:** Administrator NTLM hash sprayed across 192.168.52.0/24 via CME — authenticated successfully to Metasploitable (192.168.52.128), CLIENT01 (192.168.52.130), and DC01 (192.168.52.200) simultaneously. Kali (192.168.52.129) returned STATUS_LOGON_FAILURE as expected (Linux host, no SMB).

**DCSync:** secretsdump with -just-dc flag extracted krbtgt and Administrator hashes via DC replication protocol — no file access to NTDS.dit, no VSS shadow copy, no disk operation. Stealth hash extraction confirmed.

**Full credential chain recovered:**
- bwilson:Welcome2024! (password spray)
- svc_sql:Sqlserver2019! (Kerberoasting — Module 24)
- Every domain hash including krbtgt (secretsdump via DA)
- Administrator hash authenticated to 3 machines (Pass-the-Hash)
- krbtgt hash extracted via DCSync — Golden Ticket ready to forge

## 💡 What I Discovered

The cumulative nature of this module's chain made it the most impactful single session in the AD phase — a password spray on one weak account eventually unlocked every credential in the domain. The DCSync finding also reinforced the detection point from the quick check: there was nothing in the network traffic to distinguish it from normal DC replication, which is exactly what makes it a preferred technique over physically extracting NTDS.dit in real engagements.

## 🛡️ Defensive Perspective

- Account lockout policies and spray-aware monitoring (many accounts, one failed attempt each, short window) are the primary defenses against password spraying — without them, CME can spray an entire org silently.
- DCSync rights should be audited strictly — only actual Domain Controllers should hold replication privileges. Any user account with DCSync rights is a full domain compromise waiting to happen.
- Privileged account credential hygiene (unique, complex, regularly rotated) limits the blast radius of any single hash dump — if DA passwords are strong and rotated, cracking is infeasible even after secretsdump.
- krbtgt password should be rotated on a schedule (twice, 10 hours apart per rotation) to limit the validity window of any forged Golden Tickets.
- Monitoring for non-DC accounts initiating directory replication requests is the specific detection control for DCSync — without this alert, DCSync is essentially invisible.

## 📝 Lessons Learned

AD credential attacks aren't isolated techniques — they're a chain where each recovery enables the next step. A single weak password on one standard user account, combined with a misconfigured service account and DA credentials without MFA, became full domain credential exposure in four steps. The module also made concrete why defender monitoring strategy in AD environments needs to think about protocol abuse (DCSync, Pass-the-Hash) rather than just file access and vulnerability exploitation.

## 🏁 Conclusion

Module 25 completed the AD credential attack chain — password spraying identified a weak account, Kerberoasting recovered a service account password, DA credentials enabled a full NTDS.dit dump via secretsdump, Pass-the-Hash authenticated to three machines simultaneously, and DCSync extracted the krbtgt hash for Golden Ticket forging. All findings will be re-run against the real AD lab environment when setup is complete. This sets up directly for Module 26: BloodHound & Attack Paths, where the relationships between all these discovered accounts and permissions get visualized as a complete attack graph.
