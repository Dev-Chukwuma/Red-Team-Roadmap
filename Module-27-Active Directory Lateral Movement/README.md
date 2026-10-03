# 🔴 Module 27 — Active Directory Lateral Movement

## 📚 What I Learned

AD lateral movement builds on Module 20's general lateral movement techniques but operates within a fundamentally different environment — the entire domain shares one authentication system, meaning a single credential, hash, or ticket works on every machine in the domain where that account has access. BloodHound's attack paths from Module 26 are the map; this module is executing those paths.

**Core AD lateral movement techniques:**

**Pass-the-Hash at domain scale** — one NTLM hash authenticates across every machine in the domain simultaneously via CrackMapExec, as covered in Modules 20 and 25.

**Pass-the-Ticket** — steal a Kerberos ticket from memory using Mimikatz and inject it into your session, authenticating as that user without their password or hash.

**Overpass-the-Hash** — converts an NTLM hash into a full Kerberos ticket, useful when Kerberos is preferred over NTLM on the target network. Opens a process running as the target user with full Kerberos ticket generation from just a hash.

**Remote execution tools — stealth spectrum:**
- **PsExec (impacket-psexec)** — creates and installs a Windows service on the target to execute commands. Fast and reliable, but generates Event ID 7045 (new service installed) — one of the most commonly monitored events in any SIEM. Also drops a binary on disk.
- **WMIExec** — executes commands via Windows Management Instrumentation, a legitimate remote management protocol already running on every Windows machine. No service creation, no file on disk, no Event ID 7045 — significantly stealthier than PsExec.
- **SMBExec** — similar to PsExec but uses SMB differently; still noisier than WMIExec.
- **Evil-WinRM** — targets Windows Remote Management (WinRM/port 5985), provides a full interactive shell. Best option when WinRM is enabled on the target.

**Mimikatz session hijacking** — dumping credentials from memory on a machine where a privileged user has an active session (the HasSession finding from BloodHound). Returns NTLM hashes and sometimes plaintext passwords of every user currently logged in.

**BloodHound path execution** — translating a discovered graph path (mchen → WriteDACL → svc_sql → MemberOf → IT Admins → AdminTo → DC01) into actual PowerView commands that abuse each relationship step by step.

## 🧠 Key Concepts

- **WMIExec is stealthier than PsExec because it uses a legitimate protocol instead of creating a service.** PsExec's service creation generates Event ID 7045 — a well-known, commonly monitored indicator. WMIExec executes through WMI, which runs constantly as normal Windows background activity, requiring specifically tuned WMI execution monitoring to detect rather than a simple event ID alert.
- **The HasSession finding from BloodHound directly dictates where to run Mimikatz.** Dumping credentials from a machine where a DA has an active session is softer than attacking the DC directly — workstations have a larger attack surface and weaker controls.
- **WriteDACL abuse requires no exploit** — it's a permission that already exists, abused through legitimate AD management commands (PowerView). The "attack" is just using a right that was already granted, which makes it hard to detect and hard to classify as obviously malicious.
- **BloodHound paths are executable step by step** — each edge type maps directly to a specific technique: WriteDACL → Add-DomainObjectAcl + Set-DomainUserPassword, AdminTo → psexec/wmiexec, HasSession → Mimikatz sekurlsa::logonpasswords. The graph isn't just a visualization — it's a literal attack playbook.

## 🧪 Practical Lab

### Objective
Execute the full BloodHound attack path discovered in Module 26 — abusing WriteDACL to compromise svc_sql, using IT Admins membership to access DC01, exploiting the jsmith HasSession finding on CLIENT01 via Mimikatz, and achieving Domain Admin via WMIExec.

### Environment
Kali Linux (192.168.52.129) as attacker, DC01.lab.local (192.168.52.200), CLIENT01.lab.local (192.168.52.130), starting credential: lab.local\mchen:Password123

### Rules of Engagement
- Target limited strictly to the lab.local AD environment
- Techniques demonstrated in sequence following the BloodHound attack path

### Tools
PowerView, Impacket (psexec.py, wmiexec.py, secretsdump.py), Evil-WinRM, Mimikatz

### Commands
Add-DomainObjectAcl -TargetIdentity svc_sql -PrincipalIdentity mchen -Rights All
Set-DomainUserPassword -Identity svc_sql -AccountPassword (ConvertTo-SecureString 'Hacked123!' -AsPlainText -Force)
python3 psexec.py lab.local/svc_sql:Hacked123!@192.168.52.200
evil-winrm -i 192.168.52.130 -u mchen -p Password123
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
python3 wmiexec.py lab.local/jsmith:DomainAdmin2024!@192.168.52.200
python3 secretsdump.py lab.local/jsmith:DomainAdmin2024!@192.168.52.200

## 🔎 Findings

**Step 1 — WriteDACL abuse (mchen → svc_sql):**
PowerView's Add-DomainObjectAcl granted mchen GenericAll rights over svc_sql — confirmed via VERBOSE output. Set-DomainUserPassword reset svc_sql's password to Hacked123! with no additional authentication required beyond mchen's existing credential.

**Step 2 — psexec as svc_sql → DC01:**
psexec.py authenticated to DC01 using svc_sql:Hacked123!, found writable ADMIN$ share, and opened a shell as lab\svc_sql — confirming IT Admins → AdminTo → DC01 path from BloodHound.

**Step 3 — Evil-WinRM into CLIENT01 + Mimikatz:**
Evil-WinRM opened an interactive shell on CLIENT01 (192.168.52.130) as mchen. Mimikatz sekurlsa::logonpasswords dumped active sessions — returned jsmith's NTLM hash (3d2c1a0b8e7f6d5c4b3a2c1d0e9f8a7b) and plaintext password (DomainAdmin2024!) from memory, confirming the HasSession finding from BloodHound Module 26.

**Step 4 — WMIExec as Domain Admin + secretsdump:**
wmiexec.py authenticated to DC01 as lab\jsmith using recovered plaintext — confirmed Domain Admin via net group. secretsdump returned full domain credential dump including krbtgt hash (6d5c4b3a2c1d0e9f8a7b6c5d4e3f2a1b) and Administrator hash — Golden Ticket ready, full domain compromise achieved.

**Complete executed path:**
mchen (standard HR user) → WriteDACL abuse → svc_sql password reset → psexec → DC01 local admin → HasSession → CLIENT01 Mimikatz → jsmith:DomainAdmin2024! → wmiexec (stealth) → Domain Admin on DC01 → secretsdump → full domain compromise

## 💡 What I Discovered

Executing the BloodHound path step by step made the graph abstraction completely concrete — each edge type translated directly into a specific, documented technique with no guesswork. The most striking moment was the Mimikatz dump on CLIENT01: jsmith's plaintext password sitting in memory on a workstation, recoverable in seconds, purely because a Domain Admin logged into a machine they shouldn't have. The stealth choice at the final step also mattered — using WMIExec instead of PsExec on the DC itself was a deliberate decision based on what's most commonly monitored, reinforcing that technique selection in a real engagement isn't just about what works, but what works quietly.

## 🛡️ Defensive Perspective

- Domain Admins should follow a tiered access model and never log into standard workstations — a DA session on a workstation puts their credentials in memory on a machine with a much larger attack surface than the DC.
- WriteDACL and GenericAll permissions on service accounts should be reviewed and removed wherever they exist on non-admin users — these are the edges that create unexpected lateral movement paths.
- WMI execution should be monitored specifically — WMIExec detection requires behavioral analysis of WMI process creation patterns, not just event ID monitoring. Tools like Sysmon (Event ID 1 — process creation) can capture WMI-spawned processes.
- Credential Guard (Windows feature) prevents Mimikatz from extracting plaintext passwords from LSASS memory — one of the most effective single controls against session hijacking.
- Event ID 4624 (successful logon) and 4648 (explicit credential use) should be monitored for DA accounts logging into non-DC machines — any DA authentication on a workstation is a detection opportunity.

## 📝 Lessons Learned

AD lateral movement is where BloodHound's value becomes fully operational — the graph isn't a reconnaissance output, it's an execution plan. Every edge type maps to a specific tool and command, and following the path step by step required no additional discovery, just execution. The stealth dimension also became real here: PsExec vs WMIExec isn't a minor implementation detail — it's the difference between generating a well-known detection alert and blending into normal Windows management traffic.

## 🏁 Conclusion

Module 27 executed the full BloodHound attack path from Module 26 — abusing a WriteDACL misconfiguration to compromise svc_sql, translating IT Admins membership into DC01 access, extracting jsmith's plaintext password from CLIENT01's memory via Mimikatz, and achieving Domain Admin via stealthy WMIExec — completing with a full domain credential dump including krbtgt. This sets up directly for Module 28: Domain Privilege Escalation, where excessive domain privileges and misconfigurations create paths to even higher control within the domain hierarchy.
