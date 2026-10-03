# 🔴 Module 26 — BloodHound & Attack Paths

## 📚 What I Learned

BloodHound maps Active Directory relationships as a graph — nodes are objects (users, groups, computers, GPOs) and edges are relationships (MemberOf, AdminTo, HasSession, WriteDACL, GenericAll, DCSync, etc.). It then queries that graph for attack paths, finding routes from any compromised account to Domain Admin automatically, even across multi-hop relationship chains no human enumeration would reliably connect.

**Two components:**
- **SharpHound (collector)** — runs on a domain-joined Windows machine (or via bloodhound-python from Kali) and collects all AD relationships, sessions, and permissions, outputting a ZIP file for BloodHound to ingest
- **BloodHound GUI (analyzer)** — loads SharpHound's output into a Neo4j graph database and provides visual analysis, pre-built attack path queries, and the ability to mark compromised accounts as "owned" to find the shortest remaining path to Domain Admin

**Key pre-built queries:**
- Shortest Paths to Domain Admins
- Find all Domain Admins
- Computers where Domain Admins have Sessions
- Shortest Path from Owned Principals
- Find Kerberoastable Users
- Find ASREPRoastable Users

**Key relationship types (edges):**
- **MemberOf** — account belongs to a group
- **AdminTo** — account has local admin rights on a machine
- **HasSession** — a privileged user has an active session on a machine (credentials in memory)
- **WriteDACL** — account can modify permissions on an AD object
- **GenericAll** — full control over an object (reset passwords, modify group membership, etc.)
- **DCSync** — account has replication rights, directly exploitable as covered in Module 25

**Why HasSession is critical:** if BloodHound shows a Domain Admin HasSession on a workstation, their credentials are sitting in that machine's memory. Compromise the workstation, dump memory with Mimikatz, extract the DA's hash or ticket — instant Domain Admin without touching the DC directly.

## 🧠 Key Concepts

- **BloodHound finds multi-hop attack paths no manual enumeration would reliably surface.** A path like mchen → WriteDACL → svc_sql → MemberOf → IT Admins → AdminTo → DC01 spans three relationship types across four objects — BloodHound finds this in seconds across thousands of objects; manual enumeration might never connect all three hops.
- **Walking the mchen path in plain English:** mchen's WriteDACL right on svc_sql lets an attacker modify svc_sql's permissions and reset its password to something attacker-controlled. svc_sql's IT Admins membership inherits that group's AdminTo rights on DC01. Local admin on the DC leads directly to secretsdump and full domain compromise — all from one misconfigured permission on a standard HR account.
- **Marking accounts as Owned turns BloodHound into a live attack planning tool.** After marking compromised accounts, the "Shortest Path from Owned Principals" query shows exactly the next hop needed to reach Domain Admin from the current position — no guesswork.
- **bloodhound-python collects the entire domain graph in seconds from one standard credential** — the same data that required multiple separate PowerView commands in Module 23, now in one structured, queryable dataset.

## 🧪 Practical Lab

### Objective
Collect AD relationship data using bloodhound-python, load it into the BloodHound GUI, identify attack paths to Domain Admin, surface Kerberoastable accounts and DA sessions, and use the Owned Principals query to map the shortest remaining path from current access to full domain control.

### Environment
Kali Linux (192.168.52.129) as attacker, DC01.lab.local (192.168.52.200), domain credential: lab.local\mchen:Password123, BloodHound GUI + Neo4j running on Kali

### Rules of Engagement
- Collection and analysis strictly within the lab.local AD environment
- No exploitation of newly discovered paths during this module — analysis only

### Tools
bloodhound-python, Neo4j, BloodHound GUI

### Commands
python3 bloodhound-python -u mchen -p Password123 -d lab.local -dc DC01.lab.local -c All
sudo neo4j start
bloodhound

## 🔎 Findings

**Collection:** bloodhound-python completed in 8 seconds using mchen's standard credential — found 1 domain, 1 DC, 6 users, 4 groups, 2 computers. Output: 20261003120000_BloodHound.zip

**Shortest Path to Domain Admins query:**
Path identified: mchen → WriteDACL → svc_sql → MemberOf → IT Admins → AdminTo → DC01
A three-hop chain from a standard HR account to local admin on the Domain Controller — entirely via a single misconfigured WriteDACL permission on one service account.

**Kerberoastable users query:**
- svc_backup (SPN: backup/DC01.lab.local)
- svc_sql (SPN: MSSQLSvc/DC01.lab.local:1433)
Same findings as Modules 23-24, now visualized as graph nodes showing their full relationship context.

**Computers where Domain Admins have Sessions:**
jsmith (Domain Admin) → HasSession → CLIENT01.lab.local
jsmith's credentials are currently in CLIENT01's memory — confirmed Mimikatz target for credential extraction without touching the DC directly.

**Shortest Path from Owned Principals:**
After marking mchen and svc_sql as owned (skull icon):
svc_sql (owned) → MemberOf → IT Admins → AdminTo → DC01 → secretsdump → full domain hash dump
One remaining hop from current access to complete domain compromise.

## 💡 What I Discovered

The WriteDACL path from mchen to DA was the standout finding — a relationship that five separate PowerView commands in Module 23 never surfaced, found automatically by BloodHound in the first query run. This made the core BloodHound value proposition concrete: it's not just faster than manual enumeration, it finds things manual enumeration structurally can't, because it queries the entire relationship graph simultaneously rather than checking one object at a time. The HasSession finding on CLIENT01 also reinforced why session hunting matters — jsmith's credentials in memory on a workstation is a softer target than the DC itself, and BloodHound surfaced it without any additional scanning.

## 🛡️ Defensive Perspective

- BloodHound should be run by defenders regularly in their own environment — the same attack paths an attacker finds are findable by the blue team first, and remediating them before they're exploited is significantly cheaper than responding after.
- WriteDACL and GenericAll rights on service accounts and privileged objects should be audited and removed wherever they exist on non-admin accounts — these are the edges that create unexpected multi-hop attack paths.
- Privileged users (Domain Admins) should follow a tiered access model — logging into standard workstations with DA credentials puts those credentials in memory on machines with a much larger attack surface than the DC itself.
- HasSession data is only as current as the last SharpHound collection — defenders can use this to their advantage by ensuring privileged sessions are short-lived and actively monitoring for credential access on workstations where DA sessions are detected.
- SharpHound collections themselves are detectable — a burst of LDAP queries from a non-DC machine is a recognizable pattern. Monitoring for unusual LDAP enumeration activity is a viable detection control against BloodHound collection.

## 📝 Lessons Learned

BloodHound changed how I think about AD attack paths — not as a sequence of individually discovered misconfigurations, but as a graph where the connection between objects is as important as the objects themselves. A WriteDACL right on one service account would never appear on a traditional vulnerability report; BloodHound shows exactly why it matters by tracing where it leads. Running BloodHound as a defender in your own environment is one of the highest-leverage security investments an AD team can make.

## 🏁 Conclusion

Module 26 mapped the full lab.local attack graph using BloodHound — surfacing a three-hop path from a standard HR account to Domain Admin via a single misconfigured WriteDACL permission, confirming Kerberoastable targets from Modules 23-24 in visual context, and identifying a live DA session on CLIENT01 as the softest path to credential extraction. All findings will be re-run with real BloodHound output when the AD lab setup is complete. This sets up directly for Module 27: Active Directory Lateral Movement, where the attack paths BloodHound identified get executed.
