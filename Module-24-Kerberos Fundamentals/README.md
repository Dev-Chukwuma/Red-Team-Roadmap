# 🔴 Module 24 — Kerberos Fundamentals

## 📚 What I Learned

Kerberos is the authentication protocol Active Directory uses — and it has deeply exploitable design characteristics that make it one of the most targeted attack surfaces in enterprise environments.

**How Kerberos works:**
1. User authenticates to the KDC (Key Distribution Center, running on the DC) with an encrypted timestamp — proving identity
2. KDC returns a TGT (Ticket Granting Ticket) — the user's proof of identity, valid for 10 hours by default
3. User presents the TGT to the KDC and requests access to a specific service
4. KDC returns a Service Ticket — encrypted with the service account's NTLM hash — granting access to that specific service
5. User presents the Service Ticket to the service and gets access

Three core players: KDC (issues all tickets, lives on the DC), TGT (identity proof), Service Ticket (service-specific access grant).

**The four major Kerberos attacks:**

**Kerberoasting** — any authenticated user can request a Service Ticket for any SPN-registered service. The KDC encrypts it using the service account's NTLM hash and hands it over. That encrypted ticket can be taken offline and cracked — completely silent from AD's perspective, no lockout risk, no unusual behavior. The design assumption was that the encryption would be unbreakable — reasonable in the 1980s when Kerberos was designed, completely wrong against modern GPU cracking speeds combined with weak or static service account passwords.

**ASREPRoasting** — accounts with Kerberos pre-authentication disabled don't require proving identity before receiving a TGT. Anyone can request an encrypted TGT for those accounts without knowing their password, then crack it offline. Doesn't even require a valid domain credential to attempt.

**Pass-the-Ticket (PtT)** — steal a valid Kerberos ticket from memory using Mimikatz and inject it into your own session, authenticating as that user without ever knowing their password.

**Golden Ticket** — forge a TGT using the krbtgt account's hash (obtained after compromising the DC). Valid for 10 years by default, grants access to any service in the domain, and survives even password resets on other accounts. Essentially permanent domain control.

## 🧠 Key Concepts

- **Kerberoasting is silent by design** — requesting a Service Ticket is normal, expected network behavior. The KDC has no way to distinguish a legitimate request from an attacker requesting a ticket to crack offline. Detection requires monitoring for unusual SPN request patterns, not blocking the requests themselves.
- **Service account password strength is everything against Kerberoasting** — the attack is unstoppable at the protocol level, but a 40-character random password makes offline cracking computationally infeasible. Weak or static passwords (like "Sqlserver2019!") are cracked in seconds.
- **ASREPRoasting doesn't even need a credential** — an attacker with just a username list can attempt ASREPRoasting against any account with pre-authentication disabled, making it a powerful unauthenticated attack vector.
- **The Golden Ticket is the endgame** — forging tickets from the krbtgt hash is the closest thing to permanent, irrevocable domain control that exists. Recovering from a Golden Ticket attack requires resetting the krbtgt account password twice (each reset takes 10 hours to propagate), and even then any existing forged tickets remain valid until they expire.
- **Kerberoasting directly exploits the same service account weaknesses identified in Module 23** — the "DO NOT CHANGE" svc_sql account from enumeration became the specific cracking target here, confirming that enumeration quality directly determines exploitation precision.

## 🧪 Practical Lab

### Objective
Kerberoast the svc_sql service account identified during Module 23's enumeration, crack the extracted hash offline, and verify authenticated access using the recovered plaintext credential. Also identify and ASREPRoast any accounts with pre-authentication disabled.

### Environment
Kali Linux (192.168.52.129) as attacker, DC01.lab.local (192.168.52.200), domain credential: lab.local\mchen:Password123 (standard user, no elevated rights)

### Rules of Engagement
- Target limited strictly to the lab.local AD environment
- Cracked credentials used only for access verification within the lab

### Tools
Impacket (GetUserSPNs.py, GetNPUsers.py, psexec.py), hashcat, rockyou.txt

### Commands
python3 GetUserSPNs.py lab.local/mchen:Password123 -dc-ip 192.168.52.200 -request
echo '$krb5tgs$23$*svc_sql$LAB.LOCAL$...' > svc_sql.hash
hashcat -m 13100 svc_sql.hash /usr/share/wordlists/rockyou.txt
python3 psexec.py lab.local/svc_sql:Sqlserver2019!@192.168.52.200
python3 GetNPUsers.py lab.local/ -dc-ip 192.168.52.200 -usersfile users.txt -no-pass

## 🔎 Findings

**Kerberoasting — svc_sql:**
- GetUserSPNs confirmed svc_sql registered SPN: MSSQLSvc/DC01.lab.local:1433, password last set 2026-01-15 (static since domain setup)
- Service Ticket hash extracted: $krb5tgs$23$*svc_sql$LAB.LOCAL$MSSQLSvc/DC01.lab.local:1433*
- Hashcat (mode 13100) cracked the hash against rockyou.txt — status: Cracked
- Recovered plaintext password: Sqlserver2019!
- Verified via psexec.py — authenticated to DC01 as lab\svc_sql, confirmed Domain Admins (Administrator, jsmith) via net group

**ASREPRoasting — bwilson:**
- GetNPUsers identified bwilson with DONT_REQ_PREAUTH flag set — pre-authentication disabled
- Encrypted TGT extracted without providing any credential for the account
- Hash returned: $krb5asrep$23$bwilson@LAB.LOCAL — ready for offline cracking

**Combined impact:** two accounts compromised or partially compromised from a single standard user credential — svc_sql fully cracked and authenticated, bwilson's hash extracted without even needing mchen's credential.

## 💡 What I Discovered

The connection between Module 23's enumeration and this module's exploitation was the clearest example yet of how precisely good enumeration targets exploitation. The "DO NOT CHANGE" service account description from Module 23 wasn't just an interesting note — it directly predicted a crackable password, which hashcat confirmed in seconds. The ASREPRoasting finding also stood out: bwilson's hash was extracted without using any credential at all, which means an attacker with only a username list — no valid domain account — could have found and cracked that account entirely unauthenticated.

## 🛡️ Defensive Perspective

- Service account passwords should be long, complex, and randomly generated — 40+ characters makes Kerberoasting computationally infeasible regardless of how long the attacker runs hashcat. Group Managed Service Accounts (gMSA) handle this automatically.
- Kerberos pre-authentication should be enabled on every account — there is no legitimate reason to disable it for standard user accounts, and its absence directly enables ASREPRoasting.
- SPN assignments should be audited regularly — service accounts should only have SPNs they actually need, minimizing the Kerberoastable attack surface.
- Monitoring for unusual Service Ticket request patterns (a single account requesting tickets for many SPNs in a short window, or requests for SPNs of rarely-used services) can detect Kerberoasting in progress.
- The krbtgt account password should be rotated regularly (twice, 10 hours apart) to limit the window in which any stolen or forged tickets remain valid.

## 📝 Lessons Learned

Kerberos attacks are uniquely dangerous because they abuse legitimate, expected protocol behavior — there's nothing inherently "wrong" about requesting a Service Ticket, which is why Kerberoasting is so hard to detect and so widely used in real engagements. The fix isn't patching a vulnerability; it's configuration hygiene (strong passwords, pre-auth enabled, gMSA adoption). This module also reinforced that the quality of Module 23's enumeration directly determined the quality of this module's exploitation — precise targeting beats random attempts every time.

## 🏁 Conclusion

Module 24 covered the four major Kerberos attack techniques — Kerberoasting, ASREPRoasting, Pass-the-Ticket, and Golden Ticket — and demonstrated Kerberoasting and ASREPRoasting hands-on against the lab.local environment. svc_sql's password was cracked from a Service Ticket using only a standard user credential, and bwilson's TGT hash was extracted with no credential at all. This sets up directly for Module 25: Credential Attacks in AD Labs, where these recovered credentials feed into a broader credential attack chain across the domain.
