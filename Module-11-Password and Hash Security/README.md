# 🔴 Module 11 — Password & Hash Security

## 📚 What I Learned

A hash is a one-way function: plaintext goes in, a fixed-length scrambled output comes out, and it cannot be reversed directly back to the original. Passwords are stored as hashes rather than plaintext so that even a leaked database doesn't hand attackers passwords outright — it hands them hashes that must be cracked.

**Cracking techniques:**
- **Dictionary attack** — hash every word in a wordlist (e.g. rockyou.txt) and compare to the target hash, looking for a match
- **Brute force** — systematically try every possible character combination; guaranteed to eventually succeed, but impractical for longer/complex passwords. Search space grows exponentially with length and character variety — a 4-character lowercase-only password has roughly 450,000 combinations, while an 8-character password mixing upper/lower/numbers/symbols has over 6 quadrillion. This is why password length matters more than almost any other single factor.
- **Rainbow tables** — precomputed hash-to-plaintext lookup tables, fast but defeated by salting
- **Hybrid/rule-based attacks** — apply mutations to dictionary words (capitalization, appended numbers, character substitution) since real people often build passwords this way

**Salting** — random, per-user data added to a password before hashing. Identical passwords produce completely different hashes when salted differently, which defeats precomputed rainbow tables entirely.

**Attacks against live services (distinct from cracking a stolen hash):**
- **Online brute force login** — many passwords tried against one account. Slower than offline brute force (network round-trip per guess) and more likely to trigger lockout policies.
- **Password spraying** — one common password tried against many usernames, specifically to avoid per-account lockout thresholds. Trades brute force's "guaranteed eventual success" for a much lower detection risk — the logic being that among hundreds or thousands of accounts, at least one person likely uses a common weak password.
- **Credential stuffing** — reusing leaked username/password pairs from one breach against a different service, betting on password reuse

**Hydra** is the standard tool for live-service password attacks — the same tool handles both brute force (`-l` single user, `-P` password wordlist) and password spraying (`-L` username list, `-p` single password), just by swapping which side holds the list.

## 🧠 Key Concepts

- **Password spraying evades detection by exploiting how lockout policies count failures — per account, not per password across accounts.** A single account never accumulates enough failed attempts to trigger a lockout, even if hundreds of accounts are sprayed with the same password.
- **A dictionary attack failing is itself a meaningful result.** It demonstrates the password wasn't present in that specific leaked-password dataset — a real signal of relative strength, not just an inconclusive test.
- **rockyou.txt is large but finite.** At roughly 14 million entries from a real 2009 breach, it's a strong baseline test, but a password can still resist it without being maximally strong — a hybrid/rule-based attack or brute force could still succeed where a pure dictionary attack fails.
- **Offline vs online brute force are fundamentally different in cost.** Offline (against a stolen hash) is limited only by computing power. Online (against a live service) is limited by network latency and defensive controls like rate-limiting and lockouts — which is exactly why techniques like password spraying evolved as a workaround.

## 🧪 Practical Lab

### Objective
Generate a hash from a self-chosen password and attempt to crack it using a dictionary attack, applying the same John the Ripper workflow from a defensive exercise but framed offensively this time.

### Environment
My own Kali Linux machine

### Rules of Engagement
- Hash generated and cracked locally, self-contained — no real-world credentials involved

### Tools
md5sum, John the Ripper, rockyou.txt wordlist

### Commands
echo -n "Activeboy001" | md5sum
echo "414fd3167c3db3211cc4af4dc6f0e509" > hash.txt
john --format=Raw-MD5 hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john --show hash.txt

## 🔎 Findings

- Generated an MD5 hash from a self-chosen password ("Activeboy001"): `414fd3167c3db3211cc4af4dc6f0e509`
- Ran John the Ripper against the hash using the full rockyou.txt wordlist
- Result: **0 hashes cracked** — the dictionary attack completed a full pass through rockyou.txt without finding a match
- Confirmed via `john --show hash.txt`: 0 password hashes cracked

## 💡 What I Discovered

This was a genuinely useful contrast to Module 09's vsftpd example, where the "vulnerability" was a hardcoded backdoor with no real strength to test. Here, testing my own chosen password against a real 14-million-entry breach dataset and having it resist cracking gave a concrete, practical sense of what "not being in a common password list" actually means in practice. It also clarified that dictionary attacks are only one tool among several — brute force, hybrid/rule-based attacks, and live-service techniques like password spraying each target a different weakness, and a real assessment would layer more than one.

## 🛡️ Defensive Perspective

- Choosing passwords that aren't dictionary words or simple leaked-password variants meaningfully increases resistance to the most common cracking technique.
- Salting remains essential regardless of password strength — without it, even a strong individual password is vulnerable to rainbow-table-style attacks at scale.
- Password length is the single highest-leverage factor against brute force, since search space grows exponentially rather than linearly with each added character.
- Organizations should monitor for password spraying patterns specifically — many accounts, each with exactly one or two failed logins in a short window, is a distinct signature from traditional brute force and requires different detection logic (this connects directly to MITRE ATT&CK T1110.003 from my Blue Team work).
- Rate-limiting and account lockout policies should account for both attack shapes — a policy tuned only to catch many-attempts-on-one-account will miss a well-executed spray entirely.

## 📝 Lessons Learned

A failed dictionary attack is not a null result — it's a legitimate finding demonstrating relative password strength. This module also reinforced that offensive and defensive password security are two sides of the same coin: the same John the Ripper workflow used here to test a password's strength is exactly what a defender would use to audit their own organization's password hygiene. Understanding brute force and password spraying conceptually — even before running them live — clarified why real credential attacks rarely rely on just one technique.

## 🏁 Conclusion

Module 11 applied password and hash security concepts hands-on, testing a self-chosen password against a real dictionary attack using John the Ripper and rockyou.txt. The password resisted cracking, demonstrating the practical difference between weak, commonly-leaked passwords and ones that fall outside common breach datasets. Extending into brute force and password spraying theory rounded out the picture — showing that offline hash-cracking, online brute force, and spraying each exploit a different weakness, and a real credential attack strategy typically layers more than one.
