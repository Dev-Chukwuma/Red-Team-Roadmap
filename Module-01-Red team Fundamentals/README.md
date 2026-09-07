# 🔴 Module 01 — Red Team Fundamentals

## 📚 What I Learned

Red Teaming is an **authorized** security assessment where security professionals simulate realistic attackers to identify weaknesses in systems, people, and processes. What separates it from a crime isn't the technique used — it's **authorization**. A Red Teamer operates under explicit written permission (Rules of Engagement); anyone doing the same actions without that permission is committing unauthorized access, regardless of intent.

I also learned the distinction between the three core security roles:

- 🔴 **Red Team** — simulates attackers, finds weaknesses, performs controlled attacks, tests security controls
- 🔵 **Blue Team** — defends systems, detects attacks, investigates suspicious activity, responds to incidents
- 🟣 **Purple Team** — a real-time collaboration between Red and Blue, where offensive findings are used immediately to sharpen defensive detection — rather than Red simply handing over a report and walking away

Finally, I learned the general attack lifecycle attackers mentally follow:

Reconnaissance → Enumeration → Initial Access → Exploitation → Privilege Escalation → Credential Discovery → Lateral Movement → Objective → Reporting

## 🧠 Key Concepts

- **Authorization is what defines Red Teaming** — not skill, not technique. No permission, no Red Team.
- **Purple Team is a feedback loop, not a document handoff** — Red attacks while Blue actively watches, and gaps in detection are diagnosed together, in real time.
- **Reconnaissance is passive** (no direct contact with the target — e.g. public records, DNS lookups) while **Enumeration is active** (direct interaction with the target, e.g. probing open ports) — this distinction drives stealth and detection risk throughout an engagement.
- **Binding matters as much as the port itself** — a service bound to 127.0.0.1 is only reachable from the local machine, while a service bound to 0.0.0.0 is reachable from any network interface the machine has active. This is the real line between an internal-only service and actual attack surface.

## 🧪 Practical Lab

### Objective
Determine what an attacker could discover about the lab environment and identify potential attack surfaces — without causing any damage.

### Environment
ACME-LAB — my own local Windows machine (HP EliteBook 840 G3, authorized personal lab).

### Rules of Engagement
- Target limited strictly to my own machine
- No exploitation performed — reconnaissance only
- No damage, no persistence, no data exfiltration

### Tools
Built-in Windows utilities: ipconfig, systeminfo, netstat, tasklist

### Commands
ipconfig
systeminfo
netstat -ano
tasklist | findstr <PID>

## 🔎 Findings

**Network state (ipconfig):**
The Wi-Fi adapter was not connected to any network at time of capture. Instead of a normal DHCP-assigned address, it self-assigned an APIPA address (169.254.4.33, subnet 255.255.0.0) — Windows' fallback when it fails to reach a DHCP server. All other adapters (Ethernet, Bluetooth Network Connection) showed "Media disconnected."

**System information (systeminfo):**
- Host: DIVINE-PC
- OS: Microsoft Windows 11 Pro, Build 22621
- System: HP EliteBook 840 G3, x64-based
- Domain: WORKGROUP (standalone — not joined to any Active Directory domain)
- Total Physical Memory: 16,264 MB (9,456 MB available at time of capture)

**Listening ports (netstat -ano):**
- `0.0.0.0:135` (RPC) — PID 4 (System)
- `0.0.0.0:445` (SMB) — PID 4 (System)
- Several dynamic/high-numbered ports bound to `0.0.0.0` (902, 912, 5040, 5357, 8000, 8089, 8090, 8191, 49664–49669) tied to various background service PIDs
- `127.0.0.1:5432` (PostgreSQL) — PID 9376, localhost-only, with active ESTABLISHED connections from local client ports

**Port → PID → Process:**
Port 5432 traced to a running Postgres process (PID 9376) — origin of the install not yet confirmed (not something I recall installing directly), but bound only to loopback, so it poses no external attack surface regardless.

## 💡 What I Discovered

The most useful thing I took from this lab wasn't a vulnerability — it was realizing that reading `ipconfig` carefully told me my Wi-Fi wasn't actually connected before I even checked manually. I also learned that a port number alone doesn't tell you risk — the binding address (0.0.0.0 vs 127.0.0.1) is what actually determines whether something is reachable by anyone else on a network. Ports 135 and 445 stood out as the classic "attack surface" ports to watch, even though on this machine they're currently only reachable if I were on an active network.

## 🛡️ Defensive Perspective

Even at the reconnaissance stage, defenders have options:
- Restrict unnecessary listening services (e.g. disable SMB/port 445 if not needed on a given host)
- Monitor for unusual outbound reconnaissance-style traffic on the network
- Keep OS patch levels current so systeminfo results don't map to known CVEs
- Use host-based firewalls to limit which ports are even reachable, reducing what netstat-style enumeration would reveal to an attacker on the same network
- Periodically audit "why is this service running and who installed it" — the way I did with the unexplained Postgres process — since unexplained local services are worth tracking down even when they aren't externally reachable

## 📝 Lessons Learned

Reconnaissance seems simple — just a few built-in commands — but it's the foundation everything else in an engagement is built on. Understanding *why* a port or OS version matters (not just listing it) is what separates real analysis from just running commands. I also learned that unexplained findings (like a service I didn't remember installing) are worth documenting and investigating rather than ignoring — that habit is exactly what real security analysis looks like.

## 🏁 Conclusion

Module 01 established the foundational mindset for this journey: Red Teaming is defined by authorization, not skill; Purple Team collaboration closes feedback loops in real time; and the attack lifecycle begins with passive, low-risk reconnaissance before any active interaction with a target. The practical lab on my own machine (ACME-LAB) confirmed this in practice — surfacing a disconnected network state, standard Windows attack-surface ports, and an unexplained local Postgres instance worth further investigation in a future module.
