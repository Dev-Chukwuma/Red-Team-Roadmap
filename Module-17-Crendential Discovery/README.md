# 🔴 Module 17 — Credential Discovery

## 📚 What I Learned

Credential discovery is the art of finding passwords, SSH keys, API keys, and other authentication material that people and systems leave behind — often the fastest way to expand access after an initial compromise, since found credentials bypass the need to crack hashes or exploit more vulnerabilities.

**Common credential storage locations:**
- **Configuration files** — web apps, databases, and scripts often store plaintext credentials in `.conf`, `config.php`, `.env`, and similar files
- **Shell history** — bash_history, zsh_history, and PowerShell history preserve every command typed, including ones where passwords were entered directly
- **SSH keys** — private keys in `~/.ssh/` grant direct access to any system where the public key is authorized
- **Database files and dumps** — direct database access hands over other users' password hashes (feeding back into Module 11's cracking), or sometimes plaintext credentials for other systems
- **Browser saved passwords** — on Windows, browsers store credentials extractable via tools like Mimikatz
- **Environment variables** — applications (especially in cloud/CI setups) sometimes store API keys and database credentials directly as environment variables
- **Memory extraction** — advanced tools like Mimikatz (Windows) extract plaintext passwords and Kerberos tickets directly from running system memory, requiring higher privileges but yielding high-value findings

**The golden rule:** every credential found is a potential new door — a database password might work on a different service (password reuse from Module 11's credential stuffing), an SSH key might grant access to an entirely different internal server, and an API key might unlock third-party systems. This directly feeds into **Module 20: Lateral Movement** — using discovered credentials to move from one compromised system to the next.

## 🧠 Key Concepts

- **Bash_history is genuinely one of the highest-value, lowest-effort checks** — it's a plain text file, one command returns plaintext, zero tools needed, and people constantly type passwords directly into terminal commands out of habit.
- **Configuration file passwords are often identical across environments** — a dev database password equals a prod database password with surprising frequency, which is why finding even one can sometimes unlock multiple systems.
- **An SSH private key is as good as full account access** — no password cracking needed, no vulnerability exploit needed, just "I have the key" equals "I can log in."
- **Credential discovery chains directly into lateral movement** — this module hands you the exact keys that make Module 20's cross-system movement possible.

## 🧪 Practical Lab

### Objective
Enumerate and extract credentials from common storage locations on a compromised system, turning one access point into multiple doors.

### Target
Metasploitable 2 (using the shell from Module 10, running as root)

### Rules of Engagement
- Target limited strictly to the Metasploitable 2 VM
- Credentials extracted used only against authorized targets (Metasploitable, not external systems)

### Tools
find, grep, cat, env

### Commands
find / -name "*.conf" -o -name "config.php" -o -name ".env" 2>/dev/null
grep -r "password" /var/www/ 2>/dev/null
cat ~/.bash_history
find / -name "id_rsa" 2>/dev/null
find / -name "*.pem" 2>/dev/null
env

## 🔎 Findings

**Configuration files with plaintext credentials:**
- Located `/var/www/html/dvwa/config/config.inc.php` containing database password `dvwa`
- Located `/var/www/html/mutillidae/config.php` containing database password `root`
- Located `/etc/mysql/my.cnf` with database configuration

**Bash history extraction:**
- `mysql -u root -proot123 -D accounts` — MySQL root credentials (root / root123)
- `ssh admin@192.168.1.50` — internal admin account on another machine
- `curl -u postgres:postgres123 http://localhost:5432/backup.sql` — PostgreSQL credentials (postgres / postgres123)
- Multiple administrative command history preserved

**Environment variables:**
- `DB_PASSWORD=dvwa` — database password as environment variable
- `API_KEY=sk-1234567890abcdef` — active API key
- `ADMIN_USER=dvwa_admin` — administrative user account

**SSH keys located:**
- `/root/.ssh/id_rsa` — root's private key, capable of accessing any authorized system
- `/home/admin/.ssh/id_rsa` — admin user's private key

**The aggregate finding:** a single root-level compromise immediately yielded MySQL credentials, PostgreSQL credentials, internal IP addresses, SSH keys, API keys, and explicit commands showing how someone else had accessed other systems — turning one access point into at least five distinct paths to expand access further.

## 💡 What I Discovered

Running through the credential discovery checklist revealed just how careless credential storage typically is in practice — not even sophisticated hiding, just sitting in standard locations anyone with system access would naturally check. The bash_history finding stood out: people literally documented their own lateral-movement process by typing it directly into a shell, complete with credentials.

## 🛡️ Defensive Perspective

- Configuration files should never contain plaintext credentials — use environment variables (set securely, not logged in history), secrets management systems (HashiCorp Vault, AWS Secrets Manager), or other external stores.
- Shell history should be monitored and logged centrally, not just assumed safe on the local machine — commands with credentials are a massive reconnaissance win for an attacker.
- SSH private keys should have restricted permissions (`chmod 600`) and be kept in `~/.ssh/` only, never scattered across the filesystem or in web-accessible directories.
- Environment variables containing secrets should never be logged, and tools should be configured to mask/redact them from output and history.
- Regular audits for embedded credentials in codebase files, config files, and git history can catch these before they're deployed to production.

## 📝 Lessons Learned

Credential discovery is one of the highest-ROI phases of post-exploitation — people are far more likely to leave credentials behind than systems are to be perfectly patched, and found credentials buy immediate access without the need to exploit further. This module also reinforced that real system compromise is often a chain of human mistakes (leaving creds everywhere, password reuse, same password in multiple places) layered on top of technical flaws.

## 🏁 Conclusion

Module 17 enumerated common credential storage locations on Metasploitable 2 and extracted multiple sets of plaintext passwords, API keys, and SSH private keys from configuration files, bash history, environment variables, and filesystem — turning a single root-level access point into at least five additional doors to other systems and services. This directly sets up Module 20: Lateral Movement, where these discovered credentials become the mechanism for expanding access across a network.
