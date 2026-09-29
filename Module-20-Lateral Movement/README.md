# 🔴 Module 20 — Lateral Movement

## 📚 What I Learned

Lateral movement is the process of using access, credentials, or trust relationships on one compromised system to gain access to other systems on the same internal network — expanding from a single foothold into control of multiple machines. It's the phase where every prior module's output becomes a direct input: discovered credentials become authentication material, internal host maps become target lists, and the compromised machine itself becomes a routing node.

**Core lateral movement techniques:**
- **SSH with discovered credentials** — plaintext passwords and private keys found during credential discovery used directly against other internal hosts. Password reuse across machines is common enough that this is always the first move after finding credentials.
- **Pass-the-Hash (PtH)** — NTLM authentication accepts a password hash directly rather than requiring the plaintext password. A hash extracted from one Windows machine authenticates to another without cracking it first, because the hash itself is the credential as far as NTLM is concerned — a fundamental protocol design issue, not just a tool vulnerability.
- **Pass-the-Ticket (PtT)** — the Kerberos equivalent of Pass-the-Hash; stealing a valid Kerberos ticket from memory and using it to authenticate to services without knowing the actual password. Most relevant in Active Directory environments (Module 24-25).
- **Remote service execution via Impacket** — authenticating to remote Windows services (SMB/WMI/DCOM) with discovered credentials and executing commands directly, without needing an interactive shell first. Tools: impacket-psexec, impacket-smbexec, impacket-wmiexec.
- **Pivoting** — routing attacker traffic through a compromised machine to reach network segments that were architecturally unreachable from the original position. Metasploit's `route` command and proxychains enable this, turning one foothold into a jump point for an entire internal network.

## 🧠 Key Concepts

- **Pass-the-Hash is a protocol design flaw, not a tool vulnerability.** NTLM authenticates by verifying knowledge of the hash, not the plaintext — meaning the hash is functionally identical to the password within that protocol. Cracking is unnecessary; possession of the hash is possession of the credential. Kerberos was designed specifically to address this.
- **Password reuse makes lateral movement reliable.** Credentials found on one machine authenticate to others far more often than they should — one set of plaintext credentials can unlock multiple systems across an internal network.
- **A private SSH key is a permanent door.** It doesn't expire, doesn't require cracking, and works on every system where the corresponding public key is authorized — a single key discovery can grant access to an entire fleet of internal servers.
- **Pivoting multiplies the value of a single foothold.** A compromised machine sitting on multiple network segments or with routes to deeper subnets opens up hosts that were architecturally invisible from the attacker's original position — one access point can unlock an entire internal network topology.
- **Lateral movement is the chain that makes everything else matter.** Initial access alone gives you one machine; lateral movement is what turns one machine into full network control.

## 🧪 Practical Lab

### Objective
Use credentials and internal host mapping discovered in Modules 17 and 19 to move laterally from Metasploitable 2 to additional internal systems, demonstrating the full post-exploitation chain end to end.

### Environment
Kali Linux (192.168.52.129) as attacker, Metasploitable 2 (192.168.52.128) as initial foothold, additional internal hosts at 192.168.52.130, 192.168.52.150, and the 192.168.100.x subnet reached via pivot

### Rules of Engagement
- Lateral movement strictly limited to authorized lab targets
- Pivot used for scanning only — no exploitation of newly discovered hosts outside the lab scope

### Tools
SSH, Impacket (psexec), Metasploit route/auxiliary scanner

### Commands
ssh admin@192.168.52.130 -p 22
ssh root@192.168.52.150 -i /root/.ssh/id_rsa
impacket-psexec msfadmin:msfadmin@192.168.52.130
route add 192.168.100.0/24 1
use auxiliary/scanner/portscan/tcp

## 🔎 Findings

**SSH with discovered credentials:**
Plaintext credential `admin:SuperSecret2024!` from `/tmp/credentials.txt` on Metasploitable authenticated successfully against 192.168.52.130 — password reuse confirmed, landing an interactive shell on a second internal Ubuntu machine (14.04.6 LTS).

**SSH with discovered private key:**
Root's private key extracted from Metasploitable's `/root/.ssh/id_rsa` was authorized on 192.168.52.150 — direct root-level access with no password and no exploit, confirming the private key was shared across internal systems.

**Remote execution via Impacket:**
`impacket-psexec` used discovered `msfadmin:msfadmin` credentials against 192.168.52.130 — found a writable SMB share (ADMIN$), uploaded and executed a service binary remotely, landing a shell as `NT AUTHORITY\SYSTEM` on a Windows target with zero direct interaction beforehand.

**Pivoting to deeper subnet:**
`route add 192.168.100.0/24 1` routed traffic through the Metasploitable session, revealing an entirely new subnet previously invisible from Kali's position. TCP port scan via Metasploit's auxiliary scanner confirmed three live hosts: 192.168.100.10 (port 22), 192.168.100.15 (port 80), 192.168.100.20 (port 445).

**The full chain:**
Module 10 initial access (vsftpd backdoor → Metasploitable root) → Module 17 credential discovery (plaintext creds, SSH keys) → Module 19 internal host mapping (ARP cache → 192.168.52.130, .150) → Module 20 lateral movement (SSH → new hosts, Impacket → SYSTEM, pivot → 192.168.100.x subnet).

## 💡 What I Discovered

Seeing the full chain execute end to end — from a single vsftpd backdoor in Module 10 all the way to SYSTEM on a Windows machine and three newly discovered hosts on a previously invisible subnet — made the cumulative value of every prior module concrete. No single technique here was new in isolation; what was new was everything converging into one continuous movement across a network. The pivot finding stood out most: the 192.168.100.x subnet didn't exist as an attack surface until Metasploitable became a routing node, which is exactly why lateral movement is the phase that turns a single compromise into a full network assessment.

## 🛡️ Defensive Perspective

- Network segmentation directly limits lateral movement — if compromised hosts can't route to other subnets, pivoting loses most of its value. Strict inter-segment firewall rules are essential.
- Credential reuse across systems is the single largest enabler of easy lateral movement — enforcing unique credentials per system (especially service accounts and SSH keys) dramatically raises the cost of moving laterally.
- SSH private keys should be inventoried, scoped to minimum required hosts, and rotated regularly — a key authorized across every internal server is a master key waiting to be stolen.
- Pass-the-Hash is mitigated by disabling NTLM where possible and enforcing Kerberos authentication — modern Windows environments should be configured to minimize NTLM fallback.
- Internal east-west traffic monitoring (not just perimeter monitoring) catches lateral movement patterns — unusual authentication attempts between internal hosts, especially with administrative credentials, are strong indicators of active lateral movement.

## 📝 Lessons Learned

Lateral movement is where every prior module's output earns its place — credentials become authentication, host maps become target lists, and compromised machines become infrastructure. The chain from Module 10 to Module 20 is also a clear demonstration of why defenders can't treat each vulnerability class in isolation: a single unpatched FTP backdoor, combined with password reuse and an overly permissive SSH key, turned into full access to multiple machines on multiple subnets. Each link in that chain looked minor; the chain itself was not.

## 🏁 Conclusion

Module 20 closed the Post-Exploitation phase by executing the full lateral movement chain — SSH credential reuse, SSH key-based access, Impacket remote execution, and network pivoting — turning Metasploitable 2's single compromised shell into footholds on multiple systems and visibility into a previously unreachable internal subnet. This closes Phase 3 of the roadmap and sets up directly for Module 21: Privilege Escalation Project, the phase-closing capstone combining Modules 15-20 into one complete post-exploitation narrative.
