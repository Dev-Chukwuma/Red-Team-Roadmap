# 🔴 Module 02 — Attack Surface & Threat Modeling

## 📚 What I Learned

The **attack surface** is every possible entry point into a system — not just the ones currently in use, but everything that exists: network services, web applications, people, physical access, third-party software, and external/cloud assets. The bigger the attack surface, the more places an attacker can try. A core part of defense is shrinking that surface — turning off what isn't needed.

**Threat modeling** is the systematic thinking process that comes before any attack or defense. It answers three questions:
1. What are we protecting? (assets)
2. Who might want it, and how could they get it? (threats + attack surface)
3. What happens if they succeed? (impact)

The simplest working version of this: *"If I were an attacker looking at this system, where would I try first, and why?"*

I also learned that **an open port is not the same as an exposed port.** A service can be actively LISTENING, but if the firewall's default policy blocks inbound connections and no rule explicitly allows that port, it's effectively unreachable from the network. Netstat output alone gives an incomplete picture — firewall state has to be checked alongside it.

## 🧠 Key Concepts

- **Attack surface = every entry point that exists, not just the ones being actively used.**
- **Threat modeling reframes "is this open?" into "who could reach it, and what's the realistic impact if they did?"**
- **Binding (0.0.0.0 vs 127.0.0.1) determines who could theoretically reach a service; firewall default policy determines who actually can.** Both must be checked together — one without the other is an incomplete assessment.
- **Default-deny is safer than default-allow.** A firewall set to BlockInbound/AllowOutbound protects every port by default, only exposing what's explicitly permitted.

## 🧪 Practical Lab

### Objective
Build a real threat model of ACME-LAB by combining Module 01's port/binding data with firewall policy — determining actual (not theoretical) exposure.

### Environment
Same as Module 01 — HP EliteBook 840 G3, standalone/authorized personal lab.

### Rules of Engagement
- No exploitation, no scanning outside own machine
- Purely analytical — reasoning from existing data plus new firewall checks

### Tools
Built-in Windows utilities: netsh advfirewall

### Commands
netsh advfirewall show allprofiles state
netsh advfirewall firewall show rule name=all | findstr /i "445"
netsh advfirewall firewall show rule name=all | findstr /i "SMB"
netsh advfirewall firewall show currentprofile

## 🔎 Findings

**Firewall profile state (all profiles):**
- Domain Profile: State ON
- Private Profile: State ON
- Public Profile: State ON

**Rule search for port 445 / SMB:**
No matching named rule returned via findstr for "445," "SMB," or "File and Printer" — Windows manages this via default policy rather than a distinctly named allow-rule in this configuration.

**Current profile default policy:**
- Inbound: **BlockInbound** (unsolicited inbound connections denied unless explicitly allowed)
- Outbound: **AllowOutbound** (outbound traffic permitted freely)

**Combined threat model:**

| Port | Service | Binding | Firewall Default | Real Exposure |
|---|---|---|---|---|
| 445 | SMB | 0.0.0.0 | BlockInbound (no explicit allow found) | Low — blocked by default policy |
| 135 | RPC | 0.0.0.0 | BlockInbound (no explicit allow found) | Low — same default protection |
| 5432 | PostgreSQL | 127.0.0.1 | N/A — not network-reachable regardless of firewall | None |

## 💡 What I Discovered

Before this lab, I would have looked at port 445 being 0.0.0.0 and LISTENING and assumed that meant real risk. What I actually found is that the firewall's default-block posture is doing real protective work here — the port is technically open on the OS level, but the firewall's default policy stands in front of it. This taught me that a single command (netstat) never tells the full story — I needed to pair it with the firewall check to get an accurate picture of actual, not theoretical, attack surface.

## 🛡️ Defensive Perspective

- Default-deny inbound (like this machine has) is a strong baseline — it means new services don't become exposed automatically; something has to explicitly open a hole.
- Even with default-block in place, defenders shouldn't rely on it alone — periodically auditing which specific rules exist (and why) prevents "silent" exceptions from piling up over time.
- The Public profile matters most in practice — it's the one active when joining unfamiliar networks (like a coffee shop or, notably, someone else's hotspot), so its rule set deserves the closest attention.
- Disabling unused services entirely (rather than relying solely on the firewall to block them) is a stronger defense-in-depth practice — fewer things listening means fewer things that could ever be misconfigured into exposure.

## 📝 Lessons Learned

A listening port and an exposed port are two different things, and confusing them leads to either false alarm or false confidence. Real threat modeling requires combining multiple data sources — binding address, firewall state, and default policy — rather than treating any single command's output as the full picture. This is the same instinct as Module 01's Postgres investigation: don't conclude from one data point, verify with another.

## 🏁 Conclusion

Module 02 moved from theory into practice, building a real threat model of ACME-LAB rather than a hypothetical one. It revealed that this machine's network-facing "attack surface" ports (135, 445) are currently low-risk thanks to a default-block firewall policy, while the one truly local service (5432) carries no network exposure at all. The key takeaway carried forward: attack surface analysis is only accurate when binding, service identity, and firewall policy are all checked together — no single command tells the whole story.
