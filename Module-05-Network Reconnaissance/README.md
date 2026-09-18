# 🔴 Module 05 — Network Reconnaissance

## 📚 What I Learned

Network Reconnaissance is where passive recon ends and active recon begins — this is the first point in the attack lifecycle where packets are actually sent directly to the target's systems, rather than querying third-party public databases.

**Core concepts:**
- **Host discovery** — confirming whether a target is even online/reachable before scanning further (e.g. Nmap's `-sn` ping sweep).
- **Port scanning** — checking which ports are open on a live host. An open port means a service is listening and willing to accept connections.
- **TCP vs UDP** — TCP is connection-oriented and reliable, using a handshake (SYN → SYN-ACK → ACK) before data flows; most common services (web, SSH, databases) run on TCP. UDP is connectionless and faster but less reliable, used for things like DNS queries.
- **Nmap** — the industry-standard scanning tool, with flags for version detection (`-sV`), OS detection (`-O`), default vulnerability-adjacent scripts (`-sC`), aggressive all-in-one scanning (`-A`), timing control (`-T0` to `-T5`), and saving output (`-oN`).

## 🧠 Key Concepts

- **Active recon crosses a real legal/technical line that passive recon doesn't.** Passive recon only queries third-party databases; active recon sends packets directly to the target's own systems — this is why explicit authorization becomes mandatory from this module onward.
- **A port number alone is much less useful than a version number.** Version detection (`-sV`) turns "port 22 is open" into "OpenSSH 9.2p1 is running" — the second one can actually be cross-referenced against known vulnerabilities, directly setting up Module 08 (Vulnerability Assessment).
- **Recon findings are a snapshot in time, not a permanent truth.** Services get started, stopped, installed, and removed — a scan from last week can look completely different today. This is why real assessments always timestamp their scan results.
- **Scan timing is a direct tradeoff between speed and detection risk.** Fast scans (`-T4`/`-T5`) send bursts of traffic that IDS/IPS systems can easily fingerprint as a scan pattern. Slow scans (`-T0`/`-T1`) spread probes out to blend into normal background traffic — a real attacker patient enough to avoid triggering Blue Team's alerts entirely.

## 🧪 Practical Lab

### Objective
Perform host discovery and port scanning against an authorized target to understand how to read Nmap's output and reason about what it reveals.

### Environment
My own Kali Linux machine (authorized personal lab).

### Rules of Engagement
- Target limited strictly to my own machine (127.0.0.1)
- No scanning of any other host on the network without separate authorization

### Tools
nmap, ip

### Commands
ip a
sudo nmap -sV 127.0.0.1
sudo nmap -p- -sV 127.0.0.1
sudo nmap -A 127.0.0.1
sudo nmap -O 127.0.0.1
sudo nmap -sC 127.0.0.1
sudo nmap -sV -sC -T4 -oN scan_results.txt 127.0.0.1

## 🔎 Findings
- Host confirmed up and reachable via loopback
- Sample open ports identified: SSH (with version), HTTP (with web server version), and a database service — each demonstrating how version detection turns a bare port number into something actionable
- Understood the difference between a default top-1000 port scan and a full `-p-` scan across all 65,535 ports

## 💡 What I Discovered

The biggest shift in thinking this module gave me: a port scan isn't just "what's open" — it's "what's open, what version, and how loud was I while checking." Comparing this to Module 01's netstat findings also reinforced that recon results are time-sensitive; a system's exposed services can genuinely differ between two scans taken at different moments, which is why real engagements timestamp everything and often scan more than once.

## 🛡️ Defensive Perspective

- IDS/IPS systems should be tuned to detect burst-pattern scanning (many ports, one source, short window) — this is exactly what fast Nmap scans (`-T4`/`-T5`) produce.
- Slow, low-and-slow scanning (`-T0`/`-T1`) is much harder to detect through volume-based rules alone — defenders need behavioral/statistical detection, not just threshold alerts, to catch patient attackers.
- Regularly re-scanning your own infrastructure (not just once) catches configuration drift — services that got installed or left running without anyone noticing.
- Version-banner exposure (e.g. Nmap's `-sV` showing exact software versions) should be minimized where possible, since it directly hands an attacker the information needed to look up matching CVEs.

## 📝 Lessons Learned

Active reconnaissance fundamentally changes the risk profile of an engagement — every packet sent is a potential detection event, which is why authorization and timing strategy both matter far more here than in the passive phase. I also learned that a scan result is only meaningful with a timestamp attached; treating any single scan as a permanent picture of a system is a mistake.

## 🏁 Conclusion

Module 05 marked the transition from passive to active reconnaissance — the point where an engagement starts genuinely interacting with a target's systems, and where authorization stops being a formality and becomes the entire legal foundation of the work. Nmap's core flags (version detection, OS fingerprinting, scripting, timing control) turn simple port visibility into a real, actionable picture of a target's exposed services — setting up directly for Module 06's deeper service enumeration.
