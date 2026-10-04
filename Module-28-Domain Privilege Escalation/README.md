# 🔴 Module 28 — Domain Privilege Escalation

## 📚 What I Learned

Domain privilege escalation covers what happens beyond standard Domain Admin — abusing AD's own mechanisms (SDProp, Kerberos ticket forging, DC replication) to achieve persistence and escalation that survives incident response actions most defenders would take.

**Privilege tiers above standard DA:**
- **Domain Admin** — full control of one domain
- **Enterprise Admin** — full control of every domain in the forest
- **Schema Admin** — can modify the AD schema itself
- **Built-in Administrator** — the original local admin account, sometimes more powerful than DA in specific scenarios

In a single-domain forest, DA effectively equals EA. In real enterprise environments with multiple domains and forests, these distinctions create additional escalation paths between domains.

**Core domain privilege escalation techniques:**

**AdminSDHolder Abuse** — AdminSDHolder is an AD object that acts as a security template for privileged accounts. Every 60 minutes, AD's SDProp process resets the ACLs of all protected accounts (Domain Admins, Enterprise Admins, etc.) to match AdminSDHolder's ACL. Writing a malicious ACL entry to AdminSDHolder causes SDProp to automatically propagate that entry to every protected account in the domain — and it survives password resets because it's a permission on an AD object, not a credential. Password resets change hashes; they don't touch ACLs.

**GPO Abuse** — write access to a GPO linked to the Domain Controllers OU allows pushing malicious startup scripts or scheduled tasks that execute as SYSTEM on every DC at the next Group Policy refresh (every 90 minutes by default).

**Golden Ticket** — forging a TGT using the krbtgt hash, valid for 10 years by default, granting access to any service in the domain. Never contacts the KDC after forging — completely bypasses normal authentication logging. Survives password resets on every account except krbtgt itself.

**Silver Ticket** — like a Golden Ticket but forged for one specific service using the service account's hash instead of krbtgt. More limited scope but even harder to detect — never touches the KDC at all, no TGT request, no KDC log entry whatsoever.

**Skeleton Key** — Mimikatz patches the DC's LSASS process in memory to accept a master password ("mimikatz") for every domain account simultaneously, while real passwords continue working. Extremely aggressive, effective until the DC reboots.

**DCShadow** — registers a rogue Domain Controller temporarily, pushes malicious AD changes that replicate to the real DC, then disappears — leaving the changes behind with minimal logs since they arrived via what appeared to be legitimate DC replication, bypassing the specific event IDs monitoring group membership changes.

## 🧠 Key Concepts

- **AdminSDHolder abuse survives password resets because it's a permission, not a credential.** ACLs on AD objects are independent of account passwords — resetting every compromised account's password doesn't remove a GenericAll entry written to AdminSDHolder's ACL. The only remediation is finding and cleaning that specific ACL entry, which most incident responders don't check.
- **The Golden Ticket never contacts the KDC after forging.** Normal Kerberos authentication generates KDC logs — a Golden Ticket skips that step entirely, making it invisible to KDC-based monitoring. Detection requires looking for tickets with abnormal lifetimes or tickets being used for services they weren't requested for.
- **DCShadow bypasses group-change event IDs** by delivering changes via replication rather than standard modification — Event ID 4728 (member added to security-enabled global group) never fires because the change didn't come through the normal group management path.
- **The persistence trifecta (AdminSDHolder + Golden Ticket + DCShadow) survives three different categories of incident response action** — AdminSDHolder survives password resets, Golden Ticket survives account lockouts, DCShadow survives group auditing — making full recovery from a domain compromise that uses all three genuinely difficult without a complete domain rebuild.

## 🧪 Practical Lab

### Objective
Demonstrate AdminSDHolder abuse for persistent GenericAll over Domain Admins, forge a Golden Ticket using the krbtgt hash recovered in Module 27, access DC01 with the forged ticket, and add mchen to Domain Admins via DCShadow — achieving persistence that survives standard incident response.

### Environment
Kali Linux (192.168.52.129) as attacker, DC01.lab.local (192.168.52.200), existing DA access from Module 27, krbtgt hash: 6d5c4b3a2c1d0e9f8a7b6c5d4e3f2a1b

### Rules of Engagement
- Target limited strictly to the lab.local AD environment
- Techniques demonstrated for educational documentation only

### Tools
PowerView, Mimikatz

### Commands
Add-DomainObjectAcl -TargetIdentity "CN=AdminSDHolder,CN=System,DC=lab,DC=local" -PrincipalIdentity mchen -Rights All
Invoke-SDPropagator -showProgress -timeoutMinutes 1
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs | Where-Object {$_.SecurityIdentifier -match "mchen"}
mimikatz # kerberos::golden /user:Administrator /domain:lab.local /sid:S-1-5-21-3623811015-3361044348-30300820 /krbtgt:6d5c4b3a2c1d0e9f8a7b6c5d4e3f2a1b /ptt
mimikatz # lsadump::dcshadow /object:mchen /attribute:primaryGroupID /value:512

## 🔎 Findings

**AdminSDHolder abuse:**
Add-DomainObjectAcl successfully wrote GenericAll for mchen onto AdminSDHolder's ACL. Invoke-SDPropagator forced immediate SDProp execution — Get-DomainObjectAcl confirmed GenericAll propagated to the Domain Admins group object for mchen within 1 minute. Persistence confirmed: mchen now has the ability to add any user to Domain Admins, reset any DA password, or take any action on any protected account — surviving any password reset or credential rotation on other accounts.

**Golden Ticket:**
kerberos::golden forged a TGT for Administrator using the krbtgt hash recovered in Module 27 — ticket validity set from 10/03/2026 to 10/03/2036, injected directly into memory via /ptt. Accessing \\DC01\C$ returned the full directory listing with no password prompt and no authentication request hitting the KDC — confirmed via directory listing showing inetpub, PerfLogs, Program Files, Users, Windows folders.

**DCShadow:**
lsadump::dcshadow registered a rogue DC, pushed primaryGroupID change (512 = Domain Admins) for mchen, and unregistered — change replicated successfully to DC01. No Event ID 4728 generated — group membership change arrived via replication rather than standard modification, bypassing the specific monitoring alert most defenders rely on for DA group changes.

**The persistence trifecta achieved:**
- AdminSDHolder → GenericAll over every protected account, survives password resets
- Golden Ticket → 10-year forged access to any domain service, survives account lockouts
- DCShadow → mchen added to Domain Admins via replication, bypasses group-change auditing

## 💡 What I Discovered

The AdminSDHolder finding stood out most — the idea that a single ACL write to one obscure AD object automatically propagates to every privileged account in the domain every 60 minutes, forever, without any further attacker action, made it the most elegant persistence mechanism covered in the entire roadmap. The Golden Ticket also clicked in a new way here: seeing the ticket validity date showing 2036 while accessing the DC with no KDC contact made the "permanent access" concept genuinely concrete rather than theoretical.

## 🛡️ Defensive Perspective

- AdminSDHolder's ACL should be audited regularly — any non-standard entries (especially GenericAll for non-admin accounts) indicate potential compromise and should be removed immediately. This is a specific check most standard incident response checklists miss.
- krbtgt account password should be reset twice (10 hours apart) as part of any domain compromise recovery — a single reset doesn't invalidate existing Golden Tickets immediately since old and new keys are both valid during the transition window.
- Golden Ticket detection requires monitoring for Kerberos tickets with abnormal lifetimes (default 10 hours maximum in a healthy domain) and tickets used for services they weren't originally scoped for — SIEM rules specifically for ticket lifetime anomalies are essential.
- DCShadow detection requires monitoring for new DC registrations in the domain — any machine registering as a DC that isn't a known, authorized DC is a high-confidence indicator of DCShadow activity.
- Skeleton Key detection requires monitoring DC LSASS memory for unexpected patches — Mimikatz's skeleton key modifies LSASS in a detectable way via memory forensics or EDR behavioral monitoring.

## 📝 Lessons Learned

Domain privilege escalation techniques are distinguished from earlier modules by one key characteristic: they abuse AD's own legitimate mechanisms rather than exploiting misconfigurations or vulnerabilities. SDProp is supposed to run every 60 minutes — AdminSDHolder abuse just weaponizes that. DC replication is supposed to happen constantly — DCShadow just pretends to be part of it. Golden Tickets are valid Kerberos tickets — they're just forged ones. This makes them uniquely difficult to detect and remove, and reinforces why a full domain rebuild is often the only guaranteed path to clean recovery from an advanced AD compromise.

## 🏁 Conclusion

Module 28 covered the most advanced domain privilege escalation techniques — AdminSDHolder abuse for ACL-based persistence, Golden Ticket forging for permanent credential-free domain access, DCShadow for stealth group membership modification via rogue replication, Silver Ticket for service-scoped ticket forging, and Skeleton Key for universal authentication bypass. Combined with everything from Modules 22-27, Phase 4 is complete. This sets up directly for Module 29: Full Red Team Simulation — where every phase of the roadmap gets executed as one complete, end-to-end engagement.
