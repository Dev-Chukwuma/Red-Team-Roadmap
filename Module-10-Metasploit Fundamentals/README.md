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
- Two isolated VMs need to be on the same virtual network subnet to reach each other — a networking prerequisite that has nothing to do with Metasploit itself, but blocks the entire exercise if misconfigured.

## 🧪 Practical Lab

### Objective
Run a real Metasploit exploit against a target running the vsftpd 2.3.4 backdoor, confirming the vulnerability/exploit/payload chain hands-on.

### Environment
Kali Linux (192.168.52.129) and Metasploitable 2 (192.168.52.128), both VMs in VMware Workstation set to NAT networking on the same subnet.

### Rules of Engagement
- Target strictly limited to the Metasploitable 2 VM
- Network isolated from any external/production network via VMware NAT

### Tools
Metasploit Framework (msfconsole)

### Commands
msfconsole
search vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.52.128
run
whoami
id

## 🔎 Findings

- search vsftpd correctly returned exploit/unix/ftp/vsftpd_234_backdoor
- Setting RHOSTS to 192.168.52.128 and running the exploit produced: Banner confirmed as vsFTPd 2.3.4, USER request accepted, and the backdoor service spawned successfully
- Metasploit reported UID: uid=0(root) gid=0(root) directly in its output, confirming root-level access before any manual verification
- A command shell session opened automatically (192.168.52.129:45505 → 192.168.52.128:6200), matching the backdoor's known behavior of listening on port 6200
- Manually ran whoami inside the shell — returned root
- Ran id inside the shell — returned uid=0(root) gid=0(root)

## 💡 What I Discovered

Seeing the exploit succeed for real — not as a sample — made the vulnerability/exploit/payload framework from Module 09 fully concrete. The payload here was hardcoded (always a root shell, no selection step), which stood in contrast to the flexible payload-selection workflow described in the theory section. Getting the lab environment working was its own real challenge — VMware networking issues (disabled NAT/DHCP services, a corrupted virtual network configuration) had to be diagnosed and fixed before the exploit itself could even be attempted, which was a legitimate troubleshooting exercise in its own right.

## 🛡️ Defensive Perspective

- Metasploit's module database is public — meaning any vulnerability with an existing module is trivially exploitable by anyone with basic tool familiarity. This is exactly why patching known CVEs quickly matters so much.
- Auxiliary scanner modules exist specifically so defenders/testers can confirm a vulnerability exists without risking a live exploit — worth using during authorized assessments before committing to a full exploit attempt.
- Unusual msfconsole-style connection patterns (specific exploit signatures, known payload callback behavior) are detectable by IDS/IPS systems tuned for them — a reminder that using well-known tools carries its own detection footprint.
- Verifying package integrity before installation would have prevented the vsftpd 2.3.4 backdoor from ever being deployed in the first place.

## 📝 Lessons Learned

Metasploit isn't a "magic hack button" — its structure enforces the same disciplined separation of concepts introduced in Module 09. Understanding why the workflow is split into distinct steps matters more than memorizing the commands themselves, and getting a real exploit to succeed reinforced just how dangerous a supply-chain-style backdoor can be once it's live in the wild. Equally important: real lab work involves real infrastructure problems (networking, VM configuration) that have nothing to do with the exploit itself, but block progress just as effectively.

## 🏁 Conclusion

Module 10 translated the vulnerability/exploit/payload framework from theory into a real, working exploitation workflow — running the vsftpd 2.3.4 backdoor exploit against Metasploitable 2 and gaining confirmed root access, verified via whoami and id. This sets up directly for Module 11: Password & Hash Security.
