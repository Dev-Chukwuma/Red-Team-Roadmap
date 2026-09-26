# 🔴 Module 15 — Linux Privilege Escalation

## 📚 What I Learned

Privilege escalation is the process of gaining higher privileges than originally granted after already having some level of access to a system. Two broad categories:

- **Vertical escalation** — moving from a low-privilege user to a higher one (e.g. regular user → root)
- **Horizontal escalation** — staying at the same privilege level but gaining access to a different account with similar privileges

**Common Linux privilege escalation vectors:**
- **SUID/SGID binaries** — a SUID binary runs with the file owner's privileges, not the user running it. If a powerful binary (like `vim`, `find`, or `cp`) has SUID set and is owned by root, its normal features can sometimes be abused to gain a root shell — not because the binary is buggy, but because it was never designed to be run as SUID in the first place.
- **Sudo misconfigurations** — checked via `sudo -l`; entries like `(ALL) NOPASSWD: /usr/bin/vim` mean that specific program can be run as root with no password.
- **Cron jobs** — scheduled root tasks that reference a script writable by a lower-privileged user; editing that script and waiting for the job to trigger executes attacker code as root.
- **Kernel exploits** — outdated kernel versions sometimes have known local privilege escalation CVEs, found the same way remote CVEs were researched in Module 08, just applied locally.
- **Weak file permissions** — world-writable files root reads/executes, or credentials sitting in readable config files.

**GTFOBins** (gtfobins.github.io) is a public database cataloging exactly how common, legitimate Linux binaries can be abused for privilege escalation when misconfigured with SUID or excessive sudo rights — turning "this binary has SUID, so what?" into an exact, documented escalation command.

**LinPEAS** automates scanning for all of the above vectors at once, highlighting likely privesc paths in its output rather than requiring each check to be run manually.

## 🧠 Key Concepts

- **A SUID binary doesn't need a bug to be dangerous — it just needs to be powerful enough to do more than its intended narrow purpose, combined with being run as root regardless of who invoked it.** `vim`'s built-in shell escape (`:!`) is a legitimate editing feature; with SUID set, that same feature becomes a direct root-shell shortcut.
- **Privilege escalation is often about recognizing a misconfiguration and knowing where to look up how to exploit it — not writing custom exploit code.** GTFOBins turns this into a near-mechanical process once the right binary is identified.
- **Enumeration for privesc means looking for the unusual, not the expected.** Binaries like `passwd` and `sudo` are supposed to have SUID; a text editor or file viewer having it is the actual red flag worth investigating.

## 🧪 Practical Lab

### Objective
Identify and (where applicable) exploit a privilege escalation path from a limited shell to root.

### Environment
Metasploitable 2

### Rules of Engagement
- Target limited strictly to the Metasploitable 2 VM

### Tools
find, sudo, GTFOBins reference, vim

### Commands
find / -perm -4000 -type f 2>/dev/null
sudo -l
cat /etc/crontab
uname -a
vim.basic -c ':!/bin/sh'

## 🔎 Findings

**On Metasploitable (via the Module 10 shell):** Access into this specific target was already gained directly as root through the vsftpd 2.3.4 backdoor exploit — meaning no privilege escalation was required or applicable on this particular path. This is itself a legitimate finding: not every compromised system requires a separate escalation step, since the initial access vulnerability sometimes hands over the highest privilege level immediately.

**Demonstrated escalation technique (from a limited www-data-level shell):**
- `find / -perm -4000 -type f` revealed several SUID binaries, most expected (`passwd`, `sudo`) but one unusual: `vim.basic`
- Cross-referencing `vim` on GTFOBins under its SUID abuse section provided the exact escalation command
- Running `vim.basic -c ':!/bin/sh'` spawned a shell inheriting `vim.basic`'s root ownership via its SUID bit
- Confirmed via `whoami` returning `root`

## 💡 What I Discovered

Comparing these two outcomes on the same overall roadmap made an important point concrete: privilege escalation isn't always a required step — it depends entirely on what the initial access vulnerability grants. Where escalation was needed (the vim scenario), the actual technique required no custom exploit code at all — just correctly identifying an unusual SUID binary and looking up its known abuse pattern.

## 🛡️ Defensive Perspective

- Regularly auditing SUID/SGID binaries (`find / -perm -4000`) and removing the bit from anything that doesn't strictly require it closes off this entire vector.
- `sudo -l` misconfigurations should be reviewed carefully — NOPASSWD entries for powerful or GTFOBins-listed binaries should be avoided entirely.
- Cron jobs running as root should never reference scripts writable by lower-privileged users — file permissions on any root-executed script deserve the same scrutiny as the job itself.
- Kernel and system packages should be kept current, since local privilege escalation CVEs are actively researched and documented, same as remote ones.

## 📝 Lessons Learned

Privilege escalation frequently comes down to spotting a misconfiguration rather than discovering a novel flaw — and a resource like GTFOBins means the technical bar for exploiting a known-bad configuration is often very low once it's found. This module also reinforced that not every engagement follows the same lifecycle order rigidly — sometimes initial access grants full privileges immediately, skipping escalation entirely, and recognizing that is itself part of accurate reporting.

## 🏁 Conclusion

Module 15 covered the core Linux privilege escalation vectors — SUID/SGID abuse, sudo misconfigurations, cron jobs, kernel exploits, and weak file permissions — and demonstrated a real SUID-to-root escalation using vim.basic and GTFOBins. On Metasploitable specifically, the Module 10 exploit had already granted root directly, making escalation unnecessary on that particular path — a legitimate and worth-noting outcome in its own right. This sets up directly for Module 16: Windows Privilege Escalation.
