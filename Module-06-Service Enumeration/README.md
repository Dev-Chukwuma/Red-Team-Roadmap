# 🔴 Module 06 — Service Enumeration

## 📚 What I Learned

Service enumeration is the step between "a port is open" (Module 05) and "here's exactly what I can already do with that service" — it's the difference between confirming a door exists and actually inspecting whether the lock is even engaged.

**Core enumeration techniques covered:**
- **HTTP/HTTPS** — grabbing response headers (`curl -I`) to reveal server software/version directly
- **SSH** — confirming version and checking whether weaker configurations (password auth, root login) are allowed
- **FTP** — testing for anonymous login, a shockingly common misconfiguration; also historically a plaintext protocol vulnerable to credential sniffing
- **SMB** — using `smbclient -L` and `enum4linux` to enumerate shares, usernames, and password policy, sometimes without any credentials at all (null session access)
- **DNS** — attempting zone transfers (`dig axfr`) against a target's own DNS server, which if misconfigured hands over the entire internal DNS record set

## 🧠 Key Concepts

- **A port being open only confirms a service exists. Enumeration reveals what that service actually lets you do without credentials.** "Port 21 is open" is far less significant than "anonymous FTP login is allowed" — the second one means access is already possible, no exploit required.
- **Null/anonymous SMB sessions can leak real intelligence** — usernames, share names, and password policy — with zero authentication.
- **Enumeration sits directly between Recon and Exploitation in the attack lifecycle.** It's what hands exploitation its actual target list — you can't attack a specific account or service until enumeration identifies it.
- **A leaked password policy is actionable intel on its own.** Knowing a minimum password length (e.g. 5 characters) tells you roughly how weak the passwords protecting an account might be, before any password attack is even attempted.

## 🧪 Practical Lab

### Objective
Enumerate services identified in Module 05 on my own machine — moving from "what's open" to "what's actually configured and accessible."

### Target
My own Kali Linux machine

### Rules of Engagement
- Target limited strictly to my own machine
- No enumeration of any other host without separate authorization

### Tools
smbclient, enum4linux, curl, nmap

### Commands
nmap -sV -p 22 127.0.0.1
curl -I http://127.0.0.1
smbclient -L //127.0.0.1/ -N
enum4linux 127.0.0.1
nmap -p 21 127.0.0.1

## 🔎 Findings

- **SMB (`smbclient -L`, `-N` flag):** Null session access succeeded — anonymous, credential-free listing of shares was possible, including a share worth further investigation.
- **enum4linux:** Confirmed anonymous session access explicitly, and extracted a real username along with the workgroup's password policy — notably a weak minimum password length.
- **SSH/HTTP:** Version information already captured via Module 05's `-sV` scan; isolated checks here confirmed the same findings.

## 💡 What I Discovered

The real shift in this module was realizing how much can be extracted with zero credentials and zero exploitation — just by asking a misconfigured service politely. Two commands (`smbclient -L` and `enum4linux`) turned "SMB port is open" into a real username and a concrete password policy. That's the entire value of enumeration: it does more of the real reconnaissance work than I expected before any actual exploit attempt happens.

## 🛡️ Defensive Perspective

- Disable null/anonymous SMB sessions wherever they aren't explicitly required — this single misconfiguration handed over usernames and password policy for free.
- Enforce strong minimum password length policies (12+ characters is standard guidance) — a leaked policy showing a weak minimum is itself a finding an attacker can act on.
- Disable anonymous FTP login unless there's a specific, intentional reason for it (e.g. a public file drop with no sensitive contents).
- Monitor for and alert on SMB enumeration attempts and DNS zone transfer requests — both are recognizable, loggable patterns that indicate active reconnaissance against internal services.

## 📝 Lessons Learned

Enumeration is the quiet, unglamorous step that does most of the real intelligence-gathering work in an engagement — long before any exploit is attempted. A weak configuration (anonymous SMB access, weak password policy) can hand over more useful information than an actual vulnerability sometimes does. This module reinforced why the attack lifecycle orders enumeration directly before exploitation: you can't target what you haven't first identified.

## 🏁 Conclusion

Module 06 moved from confirming services exist (Module 05) to understanding exactly what those services expose without any credentials — usernames, shares, and password policy, extracted through null-session SMB access alone. This sets up directly for the exploitation phase of the roadmap: enumeration is what hands the next modules their actual target list.
