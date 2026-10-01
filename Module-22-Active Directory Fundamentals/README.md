# 🔴 Module 22 — Active Directory Fundamentals

## 📚 What I Learned

Active Directory (AD) is Microsoft's centralized identity and access management system — the backbone of almost every corporate network, controlling who can log into what, on which machines, with what permissions, across an entire organization from one central place. Instead of every machine managing its own users and permissions locally, AD handles everything centrally through a Domain Controller.

**Core components:**
- **Domain** — a logical grouping of users, computers, and resources sharing the same AD database (e.g. `lab.local`). Every object in the domain is managed centrally.
- **Domain Controller (DC)** — the server running Active Directory. Authenticates users, enforces policies, and stores the entire AD database (NTDS.dit). Compromising the DC means compromising the entire domain — every credential, every permission, every machine.
- **Organizational Units (OUs)** — containers organizing objects (users, computers, groups) inside the domain, like folders inside a filing cabinet.
- **Users** — every person in the organization has an AD account. Service accounts (used by applications/services rather than humans) also live here and are frequently misconfigured, making them high-value targets.
- **Groups** — collections of users with shared permissions. Security groups control access to resources; distribution groups are email lists only with no security relevance.
- **Computers** — every domain-joined machine has a computer account in AD, just like users do.
- **Group Policy Objects (GPOs)** — rules pushed from the DC to users and computers automatically: password policies, software installation, security configurations. GPOs apply to entire OUs at once.
- **Forest and trust relationships** — a forest is a collection of multiple domains that trust each other. Trust misconfigurations are a major attack path in advanced AD engagements (Module 28 territory).

## 🧠 Key Concepts

- **Compromising the DC is the ultimate objective in an AD engagement** — NTDS.dit holds every user account, every password hash, and every permission in the entire domain. Extracting it gives every credential in the organization at once, the ability to forge authentication tickets (Golden Ticket — Module 24-25), and the ability to push malicious GPOs to every machine simultaneously. Any other machine gives you one machine; the DC gives you the organization.
- **Service accounts are disproportionately high-value targets** — they're created by applications, often given excessive privileges, rarely have their passwords rotated, and are frequently configured with weak or default credentials. This connects directly to Kerberoasting in Module 24.
- **GPOs are both a powerful administrative tool and a powerful attack tool** — a domain admin can push any configuration or script to any machine in the domain via GPO, which means GPO control equals code execution across the entire organization.
- **AD is a trust system at its core** — every authentication decision, every permission check, every access control is built on trusting AD's database. Corrupting or abusing that trust (via forged tickets, stolen hashes, or privilege escalation within the domain) is what every Phase 4 module builds toward.

## 🧪 Practical Lab

### Objective
Set up a functional Active Directory lab environment — a Domain Controller with test users, groups, OUs, and a domain-joined client machine — to serve as the target for Modules 23-28.

### Environment
- Windows Server 2019 (Domain Controller) — hostname: DC01, domain: lab.local
- Windows 10 (domain-joined client) — hostname: CLIENT01
- Kali Linux (attacker) — 192.168.52.129

### Setup Steps
1. Installed Windows Server 2019 in VMware Workstation
2. Installed the AD DS (Active Directory Domain Services) role via Server Manager
3. Promoted the server to a Domain Controller, creating domain lab.local
4. Created test OUs: IT, HR, Finance
5. Created test users: jsmith (IT, Domain Admin), mchen (HR, standard user), bwilson (Finance, standard user)
6. Created service account: svc_backup (misconfigured with weak password for Kerberoasting lab in Module 24)
7. Domain-joined the Windows 10 client machine (CLIENT01) to lab.local
8. Applied a basic GPO (password policy: minimum 8 characters) to demonstrate GPO enforcement

## 🔎 Findings

**Domain structure confirmed via Active Directory Users and Computers (ADUC):**
- Domain: lab.local
- Domain Controller: DC01.lab.local (192.168.52.200)
- OUs created: IT, HR, Finance
- Users created: jsmith (Domain Admin), mchen, bwilson, svc_backup (service account)
- Computer accounts: DC01, CLIENT01

**GPO confirmed:**
- Default Domain Policy applied at domain level — minimum password length: 8 characters, password history: 24, maximum password age: 42 days

**Trust relationships:** Single domain, single forest — no cross-domain trusts configured in this lab environment.

## 💡 What I Discovered

Building the AD structure made the relationship between components concrete in a way the theory alone didn't. The service account standing out as a deliberate misconfiguration target (weak password, excessive privileges) previewed exactly why Kerberoasting is such an effective real-world technique — AD is full of service accounts that nobody pays attention to, and they're exactly the ones attackers go for first.

## 🛡️ Defensive Perspective

- Domain Controllers should be treated as the highest-priority assets on any network — physical security, patching cadence, monitoring, and access restrictions should all reflect that the DC is the single point of organizational control.
- Service accounts should follow the principle of least privilege, have long/complex randomly generated passwords, and be rotated regularly — the combination of excessive privilege and weak/static passwords is exactly what makes them Kerberoastable.
- GPOs should be audited regularly — unauthorized GPO modifications are a high-impact attack that can push malicious configurations to every machine in the domain simultaneously.
- NTDS.dit should be protected at the file system level and its access monitored — any process attempting to read or copy it outside of backup operations is a strong indicator of credential dumping activity.

## 📝 Lessons Learned

Active Directory fundamentally changes the scope of what "compromise" means — on a standalone machine, compromise affects one machine. In an AD environment, a single misconfigured service account or a single stolen hash can cascade into full organizational control. Every subsequent module in Phase 4 builds on this foundation, and understanding the trust model at the center of AD is what makes all of those techniques make sense.

## 🏁 Conclusion

Module 22 established the foundational concepts of Active Directory — domains, Domain Controllers, OUs, users, groups, computers, GPOs, and forest trusts — and set up the lab environment that Modules 23-28 will target. The AD lab (Windows Server 2019 Domain Controller + Windows 10 client, domain lab.local) is the new target environment for the remainder of Phase 4, replacing Metasploitable 2 as the primary lab target. Module 23: Active Directory Enumeration starts next.
