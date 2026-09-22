# 🔴 Module 10 — Metasploit Fundamentals

## 📚 What I Learned

Metasploit Framework (MSF) is the industry-standard exploitation framework — a large, organized library of known exploits, payloads, and auxiliary tools, all accessible through one consistent interface. Instead of writing exploit code from scratch for every vulnerability, Metasploit packages thousands of known ones so they can be selected, configured, and launched through a repeatable workflow. It comes pre-installed on Kali Linux.

Core structure:
- Exploits — modules that trigger a specific vulnerability
- Payloads — what runs after a successful exploit (a shell, a Meterpreter session, etc.)
- Auxiliary modules — scanners and non-exploit tools, e.g. checking whether a target is vulnerable without actually exploiting it
- Encoders — modify payload code, historically for evasion purposes (largely limited against modern AV/EDR)
- NOPs — padding used in certain buffer-overflow techniques

Basic workflow:
msfconsole
search vulnerability-name
use exploit-path
show options
set RHOSTS target-ip
show payloads
set payload payload-name
run

## 🧠 Key Concepts

- Metasploit's multi-step workflow (search → use → set payload → run) directly reflects the vulnerability/exploit/payload separation from Module 09. The exploit (trigger) and the payload (outcome) are chosen independently, because the same exploit can be paired with different payloads depending on the goal.
- A single "hack the target" command would break this modularity. It would force exploit and payload together, removing the flexibility real engagements need — sometimes the goal is stealth, sometimes a full interactive session, sometimes just proving a vulnerability exists without gaining a shell at all.
- search matters because multiple exploit modules can exist for the same vulnerability — picking the right one for the exact target/version is part of the skill.

## 🧪 Practical Lab

### Objective
Run a real Metasploit exploit against a target running the vsftpd 2.3.4 backdoor, confirming the vulnerability/exploit/payload chain hands-on.

### Environment
Kali Linux (attacker) + Metasploitable 2 (intentionally vulnerable target), both on an isolated VMware host-only/NAT network.

### Rules of Engagement
- Target strictly limited to the Metasploitable 2 VM
- Network isolated from any external/production network

### Tools
Metasploit Framework (msfconsole)

### Commands
msfconsole
search vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor
show options
set RHOSTS 192.168.1.150
run

## 🔎 Findings

- search vsftpd returned the correct module: exploit/unix/ftp/vsftpd_234_backdoor
- show options confirmed RHOSTS and RPORT (21) as the required parameters, with the payload hardcoded rather than selectable
- Setting RHOSTS to the Metasploitable 2 VM's IP (192.168.1.150) and running the exploit successfully opened a shell on port 6200
- Confirmed access with whoami and id, both returning root — full root-level access gained via the backdoor, with no credentials required

## 💡 What I Discovered

Seeing the exploit actually succeed made the vulnerability/exploit/payload framework from Module 09 fully concrete. The backdoor's payload being hardcoded (always root shell, no selection) stood out as a contrast to how most modern Metasploit exploits let you choose the payload independently — this one skipped that step entirely because of how the original malicious code was written.

## 🛡️ Defensive Perspective

- Metasploit's module database is public — meaning any vulnerability with an existing module is trivially exploitable by anyone with basic tool familiarity. This is exactly why patching known CVEs quickly matters so much.
- Auxiliary scanner modules exist specifically so defenders/testers can confirm a vulnerability exists without risking a live exploit — worth using during authorized assessments before committing to a full exploit attempt.
- Unusual msfconsole-style connection patterns (specific exploit signatures, known payload callback behavior) are detectable by IDS/IPS systems tuned for them — a reminder that using well-known tools carries its own detection footprint.
- Verifying package integrity before installation would have prevented the vsftpd 2.3.4 backdoor from ever being deployed in the first place.

## 📝 Lessons Learned

Metasploit isn't a "magic hack button" — its structure enforces the same disciplined separation of concepts introduced in Module 09. Understanding why the workflow is split into distinct steps matters more than memorizing the commands themselves, and seeing a real exploit succeed reinforced just how dangerous a supply-chain-style backdoor can be once it's live in the wild.

## 🏁 Conclusion

Module 10 translated the vulnerability/exploit/payload framework from theory into a real, working exploitation workflow — running the vsftpd 2.3.4 backdoor exploit against Metasploitable 2 and gaining root access confirmed the entire chain discussed conceptually in Module 09. This sets up directly for Module 11: Password & Hash Security.
