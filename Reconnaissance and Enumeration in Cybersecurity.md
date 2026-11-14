# Reconnaissance and Enumeration in Cybersecurity: A Complete Guide

*From first footprint to a full attack-surface map — concepts, methodology, and hands-on tool usage with examples at every step.*

**What this guide covers:** what recon and enumeration actually are and where they sit in a real engagement; every major passive and active technique; protocol-by-protocol enumeration (SMB, DNS, SNMP, LDAP, web apps, Active Directory, cloud); a worked walkthrough tying it all together; the defensive side; the legal basics; and a cheat sheet to come back to.

**Ground rule:** every technique below should only be run against a system you own, a lab (TryHackMe, HackTheBox, your own VMs), or a target you have explicit written authorization to test (a signed pentest scope, a bug bounty program's listed scope). Section 13 covers this properly — keep it in mind as you read.

\---

## 1\. What Recon and Enumeration Actually Are

**Reconnaissance ("recon")** is the information-gathering phase: building a picture of a target before you touch it, or while barely touching it. Domains it owns, IP ranges, employees, technology stack, public-facing services — the goal is a map, not an exploit.

**Enumeration** is recon zoomed all the way in. Once you know a service exists (say, SMB on port 445), enumeration is the process of actively querying it to pull out specific, usable details: share names, usernames, group memberships, software versions, configuration. Where recon answers "what's out there," enumeration answers "exactly what is this, and what can I do with it." In practice the two blend into one continuous phase, which is why they're almost always taught and performed together.

### Where this fits in a real engagement

Every major methodology puts information gathering first, because everything downstream depends on it:

* **Cyber Kill Chain** (Lockheed Martin): Reconnaissance is stage 1 of 7.
* **MITRE ATT\&CK**: Reconnaissance (TA0043) and Discovery (TA0007) are dedicated tactics with dozens of documented techniques.
* **PTES** (Penetration Testing Execution Standard): Intelligence Gathering is phase 2, right after pre-engagement, and feeds directly into Threat Modeling.
* **OSCP-style methodology**: Information Gathering → Enumeration → Vulnerability Analysis → Exploitation → Post-Exploitation → Reporting. Enumeration is so central here that the community shorthand is "enumeration is the key."

In a real-world paid pentest, it's common for 30–50% of total engagement time to go into recon and enumeration alone. Rushing this phase is the single most common reason testers (and students) miss things that are sitting in plain sight.

### Passive vs. Active

This distinction matters because it changes your risk of detection and, in some cases, your legal footing:

||Passive Recon|Active Recon|
|-|-|-|
|Definition|Gathering info without directly interacting with the target's systems|Directly interacting with the target's systems|
|Detection risk|Essentially zero — you're querying third parties (search engines, WHOIS registries, certificate logs)|Real — packets hit the target, may appear in logs, trip an IDS/IPS or WAF|
|Examples|WHOIS, Google dorking, Shodan, certificate transparency, social media|Port scanning, banner grabbing, sending crafted requests to a login page|
|Typical tools|whois, theHarvester, crt.sh, Shodan|nmap, gobuster, enum4linux|

A full engagement almost always starts passive and gets progressively more active as the picture sharpens.

\---

## 2\. The Methodology at a Glance

|Phase|Goal|Typical Activities|Example Tools|
|-|-|-|-|
|1. OSINT / Passive Recon|Learn about the target without touching it|WHOIS, DNS history, employee info, leaked secrets, tech-stack hints|whois, theHarvester, Shodan, crt.sh|
|2. Active Host Discovery|Confirm which hosts are actually alive|Ping/ARP sweeps|nmap -sn, arp-scan, fping|
|3. Port Scanning|Find open ports and running services|TCP/UDP scans, service/version detection|nmap, masscan, rustscan|
|4. Service Enumeration|Extract concrete, usable detail from each open service|Protocol-specific queries|enum4linux, snmpwalk, ldapsearch|
|5. Web Enumeration|Map the web attack surface|Subdomains, directories, parameters, APIs|gobuster, ffuf, Amass|
|6. (Next phase, out of scope here)|Match findings to known weaknesses|Vulnerability scanning/analysis|Nessus, Nuclei|

Don't picture this as a strict straight line — it's a loop. A subdomain found during web enumeration sends you back to DNS recon. A username found via SMB null session sends you back into SNMP or SMTP enumeration to validate it. A tech-stack hint from job postings tells you which CVE databases to check later. Good recon is iterative: each phase feeds the next and sometimes feeds back into an earlier one.

\---

## 3\. Passive Reconnaissance (OSINT)

### 3.1 WHOIS Lookups

WHOIS records show domain registration data: registrar, creation/expiry dates, and nameservers (registrant contact info is usually redacted today behind privacy proxies, but the rest is still valuable).

```bash
whois example.com
```

What to actually look for: the **nameservers** often reveal the hosting/DNS provider (e.g., `ns1.cloudflare.com` tells you Cloudflare sits in front of the site); the **creation date** tells you how much historical footprint might exist to dig through; and if privacy protection isn't enabled, the admin/tech contact can hand you an employee name or internal email format directly.

You can also WHOIS an IP or ASN to map an organization's full address space before scanning anything:

```bash
whois 93.184.216.34
whois -h whois.radb.net -- '-i origin AS15133'
```

### 3.2 Passive DNS Recon

Before brute-forcing anything, just ask DNS directly what it already knows:

```bash
dig example.com MX
dig example.com TXT
dig example.com NS
```

* **MX records** reveal the mail provider (e.g., seeing Google's or Microsoft's mail servers tells you the org likely uses Google Workspace or Microsoft 365 — useful later for phishing-simulation or password-spray engagements).
* **TXT records** hold SPF/DKIM entries and domain-verification tokens, which sometimes leak the names of internal SaaS tools (e.g., a Stripe or Salesforce verification string).
* **NS records** confirm the authoritative name servers and, again, often the hosting provider.

### 3.3 Search Engine Recon (Google Dorking)

Search operators let you ask search engines to do your discovery work for you:

```text
site:example.com filetype:pdf
site:example.com inurl:admin
site:example.com intitle:"index of"
site:example.com ext:sql | ext:env | ext:log
```

`site:` restricts to a domain, `filetype:`/`ext:` finds specific file types (PDFs and spreadsheets often carry useful metadata; stray `.env` or `.sql` files are gold if indexed), `intitle:"index of"` finds exposed directory listings, and `inurl:` hunts for path patterns like `/admin` or `/backup`. The **Google Hacking Database (GHDB)**, hosted on Exploit-DB, is a curated, searchable list of these dorks if you want ready-made queries rather than building your own. Bing and DuckDuckGo index differently and occasionally surface pages Google has dropped, so it's worth repeating a search across more than one engine.

### 3.4 Shodan, Censys, and ZoomEye

These are internet-wide scanning engines: they continuously scan the entire IPv4 (and growing IPv6) space and index service banners, so you can search without sending a single packet to the target yourself.

Shodan examples:

```text
hostname:example.com
org:"Example Corp"
port:3389 country:"PK"
```

Censys uses a similar query language (`services.port:3389`, `services.service\\\_name:RDP`). These engines are especially good at surfacing **forgotten or shadow-IT assets**: a dev server someone spun up and never took down, an exposed database, an IoT device on the corporate network — all things that would never show up in official DNS records.

### 3.5 Certificate Transparency Logs

Every publicly trusted TLS certificate issued today is logged in public Certificate Transparency (CT) logs — a browser requirement, not optional. Since certificates list every hostname they cover (the Subject Alternative Names field), CT logs are one of the single best sources for subdomain discovery. **crt.sh** is the standard searchable front end:

```bash
curl -s "https://crt.sh/?q=%25.example.com\\\&output=json" | jq -r '.\\\[].name\\\_value' | sort -u
```

This one command alone often returns dozens of subdomains a target never intended to advertise.

### 3.6 theHarvester

An aggregator that pulls emails, subdomains, hosts, and employee names from search engines, PGP key servers, Shodan, and more — all through one tool instead of visiting each source by hand.

```bash
theHarvester -d example.com -b all -l 500
```

`-b` picks the data source (`google`, `bing`, `crtsh`, `shodan`, or `all`); `-l` caps how many results to pull per source. Some sources need a free API key configured first.

### 3.7 Historical / Archived Content (Wayback Machine)

archive.org's Wayback Machine stores historical snapshots of web pages — a great way to find endpoints, parameters, or whole pages that have since been removed (sometimes precisely because they were sensitive).

```bash
waybackurls example.com | sort -u
```

(`waybackurls` by tomnomnom, or the similar `gau`/"get all URLs" tool.) It's common to pipe this output into a secrets-hunting grep, since old archived JS bundles occasionally still contain hardcoded API keys.

### 3.8 Code Repository Recon (GitHub/GitLab)

Developers accidentally commit secrets — API keys, credentials, internal hostnames, `.env` files — to public repositories constantly. Beyond manual searches like `org:examplecorp password` on GitHub, dedicated scanners automate this:

```bash
trufflehog github --org=examplecorp
```

`trufflehog` and `gitleaks` both scan full commit history (not just the current file state), which matters because a secret removed in a later commit often still lives in the git log.

### 3.9 Social Media \& People OSINT

LinkedIn is the obvious one — it reveals employee names and job titles, which lets you guess the organization's email/username convention (`firstname.lastname@`, `flastname@`, etc.) for later phases like password spraying. Job postings are an underrated source too: a listing asking for "experience with Jenkins, GitLab CI, AWS EKS" just told you the internal tech stack for free. For hunting a specific person's footprint across platforms, `Sherlock` checks username availability/existence across hundreds of sites at once:

```bash
sherlock johndoe123
```

### 3.10 Metadata Analysis

Publicly published documents (PDFs, Word files, images) often carry metadata: author names, software versions, sometimes internal file paths or usernames.

```bash
exiftool document.pdf
```

`metagoofil` automates the whole workflow — it searches a domain for public documents, downloads them, and extracts metadata in one pass.

### 3.11 Technology Fingerprinting

Knowing exactly what a site runs on shapes everything downstream.

```bash
whatweb -v example.com
```

This (and the Wappalyzer browser extension, or BuiltWith's web lookup) reveals CMS platform (WordPress, Drupal), web server software (nginx, Apache, IIS), frameworks, analytics scripts, and CDN/WAF presence (e.g., Cloudflare) — all of which inform which tools and techniques make sense later.

### 3.12 Breach \& Credential Exposure Awareness

Services like Have I Been Pwned let you check whether a company's domain or specific email addresses have appeared in known public breaches. This is normally used for awareness (e.g., flagging password-reuse risk as a finding) within an authorized engagement — not as a shortcut to directly try leaked passwords against live systems without that being explicitly within scope.

### 3.13 OSINT Automation Frameworks

When you want more than one tool doing the work:

* **Recon-ng** — a modular CLI recon framework with a marketplace of installable modules and workspace-based organization, similar in feel to Metasploit but for OSINT.
* **SpiderFoot** — an automated OSINT aggregator with a web UI and 200+ modules; point it at a domain and let it correlate results across sources.
* **Maltego** — graph-based OSINT visualization; "transforms" turn one piece of data (a domain, a person, an IP) into connected entities, which is extremely good for mapping relationships rather than just listing facts.
* **OSINT Framework** (osintframework.com) — not a tool, but a curated, categorized directory of OSINT resources; useful as a map of "what exists" when you're stuck.
* **Amass** (passive mode) — bridges OSINT and DNS enumeration; covered in depth in Section 5.3.

Example Recon-ng session:

```bash
recon-ng
\\\[recon-ng]\\\[default] > marketplace install all
\\\[recon-ng]\\\[default] > workspaces create example-corp
\\\[recon-ng]\\\[example-corp] > modules load recon/domains-hosts/hackertarget
\\\[recon-ng]\\\[example-corp]\\\[hackertarget] > options set SOURCE example.com
\\\[recon-ng]\\\[example-corp]\\\[hackertarget] > run
```

\---

## 4\. Active Reconnaissance

This is where you start touching the target directly. Everything from here on can appear in logs.

### 4.1 Host Discovery

Before scanning ports, confirm which hosts in a range are actually alive. On a local network, ARP-based discovery works even when ICMP (ping) is firewalled off, because ARP operates below IP:

```bash
nmap -sn 192.168.1.0/24
arp-scan --localnet
fping -a -g 192.168.1.0/24
```

For external targets, first establish the IP space an organization actually owns — via the WHOIS/ASN lookups from Section 3.1, or a quick check on bgp.he.net — so you're not scanning blind.

### 4.2 Port Scanning with Nmap

Nmap is the standard here, and knowing its scan types properly matters more than memorizing flags:

|Flag|Scan Type|Notes|
|-|-|-|
|`-sS`|TCP SYN scan|"Half-open"/stealthier, default when run as root, fast|
|`-sT`|TCP connect scan|Full 3-way handshake; used when raw sockets aren't available|
|`-sU`|UDP scan|Slower, but essential — catches SNMP, DNS, NTP, TFTP|
|`-sA`|ACK scan|Maps firewall rules, doesn't report open/closed|
|`-sV`|Service/version detection|Identifies exact software + version behind a port|
|`-O`|OS fingerprinting|Guesses the target's OS from TCP/IP stack behavior|
|`-sC`|Default script scan|Runs Nmap's safe default NSE scripts|

Useful modifiers: `-p-` scans all 65535 ports (vs. the default top-1000), `--top-ports 100` scans just the 100 most common, `-T0` through `-T5` control timing (paranoid → insane; `-T4` is the common choice in labs/CTFs with decent bandwidth), and `-oA filename` saves output in all three formats (normal, XML, grepable) at once.

A realistic full scan:

```bash
nmap -sS -sV -sC -O -p- -T4 -oA full\\\_scan 10.10.10.10
```

A realistic UDP sweep (don't skip this — UDP services are frequently missed):

```bash
nmap -sU --top-ports 20 -T4 10.10.10.10
```

**NSE (Nmap Scripting Engine)** is where Nmap goes from "port scanner" to "enumeration tool." Scripts are grouped by category (`safe`, `vuln`, `discovery`, `auth`, `brute`, `intrusive`, etc.) and can be run broadly or precisely:

```bash
nmap -p445 --script smb-enum-shares,smb-enum-users 10.10.10.10
nmap --script vuln 10.10.10.10
```

### 4.3 Faster Scanning at Scale: Masscan \& RustScan

Nmap is thorough but slow across huge ranges or all 65535 ports. **Masscan** uses its own TCP/IP stack to send packets asynchronously and can scan the entire IPv4 address space in minutes:

```bash
masscan -p1-65535 10.10.10.0/24 --rate 10000
```

**RustScan** takes a different approach — it does fast initial port discovery, then automatically hands the open ports to Nmap for the detailed scripted scan, giving you speed and depth in one command:

```bash
rustscan -a 10.10.10.10 -- -sV -sC
```

### 4.4 Banner Grabbing

Manually connecting to a service and reading what it says back often reveals the exact software and version without needing a scripted scan:

```bash
nc -nv 10.10.10.10 21
curl -sI http://10.10.10.10
```

The `curl -I` form is especially useful on the web — the `Server` and `X-Powered-By` response headers frequently name the exact web server and backend framework.

### 4.5 OS Fingerprinting

`nmap -O` fingerprints the OS from subtle TCP/IP stack behavior differences. `p0f` does the same thing passively, just by observing traffic rather than sending probes. A quick manual heuristic: default TTL values hint at the OS — roughly 64 for Linux/Unix, 128 for Windows, and 255 for a lot of networking gear — though this is easily spoofed and only a rough signal.

### 4.6 Traceroute / Network Path Mapping

```bash
traceroute 10.10.10.10
```

Useful for understanding network topology — how many hops away the target is, and whether a load balancer or WAF sits in front of it (often visible as the path terminating early or showing an unexpected final hop).

\---

## 5\. DNS Enumeration

### 5.1 Basic DNS Query Tools

```bash
dig example.com A
dig -x 93.184.216.34
host -t MX example.com
nslookup -type=TXT example.com
```

`dig -x` performs a reverse lookup (IP → hostname). Between `dig`, `host`, and `nslookup`, `dig` is generally preferred for its clearer, more scriptable output.

### 5.2 Zone Transfer (AXFR) Attempts

A DNS zone transfer, if misconfigured to allow any requester, hands over the **entire zone file** — every record for the domain — in a single request. It's a critical misconfiguration, and it costs nothing to test:

```bash
dig axfr example.com @ns1.example.com
```

A properly configured DNS server restricts AXFR to authorized secondary name servers only, so this will usually just fail — which is the expected, healthy outcome. When it works, it's one of the single highest-value findings in recon.

### 5.3 Subdomain Enumeration

This is arguably the single highest-value activity in external recon — every subdomain found is a new piece of attack surface.

**Passive sources** (no direct interaction with the target): crt.sh (Section 3.5), DNSdumpster, SecurityTrails.

**Active tools**, combining passive aggregation with brute-forcing:

|Tool|What it does|
|-|-|
|`Sublist3r`|Aggregates multiple passive sources in one pass|
|`Amass`|The gold standard — combines passive sources, active brute force, and name permutation/alteration scanning|
|`subfinder`|Fast passive aggregator, popular for chaining into other tools|
|`assetfinder`|Lightweight passive aggregator|
|`dnsenum`, `fierce`|Classic brute-force tools that also attempt zone transfers automatically|
|`gobuster dns`|Brute-forces subdomains against a wordlist|
|`massdns` / `puredns`|Extremely fast resolvers built for brute-forcing huge wordlists at scale|

A realistic chained workflow:

```bash
amass enum -passive -d example.com -o passive\\\_subs.txt
subfinder -d example.com -o subfinder\\\_subs.txt
cat passive\\\_subs.txt subfinder\\\_subs.txt | sort -u > all\\\_subs.txt
gobuster dns -d example.com -w subdomains.txt -o brute\\\_subs.txt
cat all\\\_subs.txt brute\\\_subs.txt | sort -u | httpx -silent -o live\\\_subs.txt
```

That last step — piping every discovered subdomain through `httpx` — is standard practice: it quickly tells you which subdomains actually have a live web server responding, so you're not wasting time investigating dead DNS entries.

### 5.4 Key DNS Record Types

|Record|Reveals|
|-|-|
|A / AAAA|IPv4 / IPv6 address|
|MX|Mail provider — hints at Google Workspace, Microsoft 365, etc.|
|TXT|SPF/DKIM entries, domain-verification tokens, sometimes internal tool names|
|NS|Authoritative name servers, hosting-provider hints|
|CNAME|Aliases — a prime indicator for subdomain takeover (see below)|
|SRV|Service-location records (e.g., SIP, XMPP, game servers)|
|PTR|Reverse DNS|

**Subdomain takeover** is worth flagging here: if a CNAME points to a cloud resource that's since been deprovisioned (an old Heroku app, an unclaimed S3 bucket, a removed GitHub Pages site), an attacker can often claim that resource and serve their own content under the victim's trusted subdomain. Tools like `subjack` or Nuclei's takeover templates (`nuclei -t http/takeovers/`) automate checking a list of subdomains for this exact condition. It's covered again in Section 9.3 since it shows up constantly in cloud recon.

\---

## 6\. Network \& Service Enumeration

This is the core of "enumeration" proper: once Nmap has told you a port is open, each protocol has its own enumeration playbook for pulling out specific, actionable detail.

### 6.1 SMB (Ports 139 / 445)

SMB is Windows file and printer sharing, and it is consistently one of the richest sources of information on a network — share names, usernames, groups, OS/build version, and sometimes full **null sessions** (unauthenticated access, a legacy misconfiguration still found surprisingly often).

```bash
enum4linux -a 10.10.10.10
smbclient -L //10.10.10.10 -N
smbmap -H 10.10.10.10
rpcclient -U "" -N 10.10.10.10
```

The modern all-in-one tool for this is **NetExec** (`nxc`), the actively maintained successor to the now-archived CrackMapExec:

```bash
nxc smb 10.10.10.10 -u '' -p '' --shares
nxc smb 10.10.10.10 -u '' -p '' --users
```

(If you see older write-ups using `crackmapexec` / `cme`, know that the project is archived — `netexec`/`nxc` is its direct, actively developed continuation and uses near-identical syntax.)

### 6.2 NetBIOS (Port 137)

```bash
nbtscan 10.10.10.0/24
nmblookup -A 10.10.10.10
```

Quick way to grab a Windows host's NetBIOS name and workgroup/domain across a whole subnet in one pass.

### 6.3 SNMP (Port 161/UDP)

SNMP is used for network device monitoring and management, authenticated by a "community string" that functions like a password. Default strings — `public` (read-only) and `private` (read-write) — are left unchanged far more often than you'd expect, and a readable SNMP service can leak process lists, routing tables, installed software, and sometimes usernames.

```bash
onesixtyone -c community\\\_strings.txt 10.10.10.10
snmpwalk -c public -v1 10.10.10.10
snmp-check 10.10.10.10 -c public
```

### 6.4 SMTP (Port 25)

The `VRFY` and `EXPN` SMTP commands can confirm whether a username/mailbox exists on misconfigured mail servers — effectively free username enumeration:

```bash
smtp-user-enum -M VRFY -U userlist.txt -t 10.10.10.10
```

Manually:

```bash
nc 10.10.10.10 25
VRFY root
```

### 6.5 FTP (Port 21)

Always check for anonymous login — still a common finding:

```bash
ftp 10.10.10.10
# Name: anonymous
# Password: (blank, or anonymous@example.com)
```

Or automated:

```bash
nmap --script ftp-anon 10.10.10.10
```

### 6.6 SSH (Port 22)

```bash
nc -nv 10.10.10.10 22
```

The banner alone reveals the SSH daemon and version, which matters for version-specific issues (for example, OpenSSH's CVE-2018-15473 allowed username enumeration via a timing side-channel in certain versions — a good illustration of why version detection is never "just metadata").

### 6.7 LDAP (Port 389 / 636)

LDAP backs most Active Directory deployments. An anonymous/unauthenticated bind, where allowed, can expose the entire directory structure:

```bash
ldapsearch -x -H ldap://10.10.10.10 -b "dc=example,dc=com"
```

`windapsearch` and `ldapdomaindump` wrap this into friendlier, more structured output — covered further in Section 8.

### 6.8 NFS (Port 2049)

```bash
showmount -e 10.10.10.10
mount -t nfs 10.10.10.10:/share /mnt/nfs
```

`showmount -e` lists exported shares; misconfigured exports sometimes allow mounting without any authentication at all.

### 6.9 RPC (Port 135)

```bash
rpcclient -U "" -N 10.10.10.10
rpcinfo -p 10.10.10.10
```

Inside an `rpcclient` null session, commands like `enumdomusers` and `querydominfo` can enumerate domain users and basic domain info without credentials.

### 6.10 Databases (MySQL 3306, MSSQL 1433, etc.)

```bash
nmap -p3306 --script mysql-info,mysql-enum 10.10.10.10
mysql -h 10.10.10.10 -u root -p
```

For MSSQL, Impacket's `mssqlclient.py` is the standard tool once any credentials (even low-privilege) are known, and `nmap --script ms-sql-info` works pre-auth.

### 6.11 Other Commonly-Found Exposed Services

```bash
nmap -p6379 --script redis-info 10.10.10.10      # Redis — historically no auth by default
nmap -p9200 10.10.10.10                           # Elasticsearch — check for open REST API
nmap -p2375 --script docker-version 10.10.10.10   # Docker API exposed without TLS
```

RDP (3389), VNC (5900), and Telnet (23) round out the list of services worth an automatic banner/version check on every engagement — they show up far more often than they should.

\---

## 7\. Web Application Recon \& Enumeration

Web applications deserve their own section because they're usually the largest and most dynamic part of the attack surface — and because enumeration here keeps going well past the initial port scan.

### 7.1 Technology Fingerprinting (Revisited)

```bash
whatweb -v example.com
httpx -u example.com -tech-detect
```

Knowing the CMS, framework, and server software up front tells you which tool-specific sections below are even worth running.

### 7.2 Virtual Host Enumeration

A single IP can quietly host multiple sites, distinguished only by the `Host` header — and "hidden" vhosts that never made it into public DNS are a classic way to find a staging or admin panel that was never meant to be public.

```bash
gobuster vhost -u http://10.10.10.10 -w vhosts\\\_wordlist.txt --append-domain
ffuf -u http://10.10.10.10 -H "Host: FUZZ.example.com" -w vhosts\\\_wordlist.txt -fs 123
```

`-fs 123` in the `ffuf` example filters out responses matching the default/baseline page size (123 bytes, as an example) — without this, every guess looks like a "hit" because the server just returns its default site for unmatched hosts.

### 7.3 Directory \& File Brute-Forcing

```bash
gobuster dir -u http://example.com -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html -t 50
ffuf -u http://example.com/FUZZ -w wordlist.txt -mc 200,301,302,403
feroxbuster -u http://example.com -w wordlist.txt --depth 3
```

The **SecLists** repository (`/usr/share/seclists/` on Kali) is the standard wordlist collection for all of this — it's worth getting familiar with its layout rather than hunting for wordlists ad hoc. `feroxbuster` is worth knowing specifically for its recursive behavior: it automatically dives into discovered directories rather than requiring a fresh command for each one.

### 7.4 robots.txt, sitemap.xml, and security.txt

```bash
curl http://example.com/robots.txt
curl http://example.com/sitemap.xml
curl http://example.com/.well-known/security.txt
```

`robots.txt` is meant to tell search crawlers what *not* to index — which ironically makes it a developer-provided map of the paths they considered sensitive enough to hide. It's not a security control, just a convention crawlers are supposed to respect, so treat anything listed there as a lead, not an obstacle.

### 7.5 Parameter Discovery

Hidden or undocumented request parameters are a common source of access-control and injection bugs, and they're easy to miss by just reading the visible UI.

```bash
arjun -u http://example.com/endpoint
```

`Arjun` and Burp Suite's Param Miner extension both work by sending a large batch of candidate parameter names and watching for any response difference.

### 7.6 API Endpoint Discovery

Look for exposed API documentation first — it often does the enumeration for you:

```bash
curl -s http://example.com/swagger.json
curl -s http://example.com/api-docs
```

For GraphQL, check whether introspection is enabled (it frequently is, by default, in development builds that made it to production) — a successful introspection query dumps the entire schema, every type and field included. For REST APIs with no documentation, grepping bundled JavaScript for endpoint patterns is a reliable fallback:

```bash
curl -s http://example.com/app.js | grep -oE "/api/\\\[a-zA-Z0-9\\\_/-]+"
```

Tools like `LinkFinder` automate exactly this kind of JS-endpoint extraction at scale.

### 7.7 CMS-Specific Enumeration

Once fingerprinting (7.1) identifies a CMS, a dedicated scanner goes much deeper than generic tools can:

```bash
wpscan --url http://example.com --enumerate vp,vt,u
```

`wpscan` (WordPress) enumerates vulnerable plugins (`vp`), vulnerable themes (`vt`), and usernames (`u`) in one pass, cross-referencing a vulnerability database along the way. `joomscan` and `droopescan` serve the same role for Joomla and Drupal respectively.

### 7.8 Nikto

```bash
nikto -h http://example.com
```

A general-purpose web server scanner that checks for thousands of known misconfigurations, outdated software signatures, and dangerous files in one run — it sits right on the boundary between enumeration and light vulnerability scanning, and it's a reasonable first command to run against any web target.

\---

## 8\. Active Directory \& Windows Domain Enumeration

AD environments get their own methodology because the goal shifts from "what services exist" to "what *relationships* exist between users, groups, and computers" — and those relationships are often where the real attack path hides.

### 8.1 Null Sessions \& Anonymous Enumeration

The SMB, RPC, and LDAP null-session techniques from Section 6 are usually the very first thing tried against a Domain Controller — a surprising number of real environments still permit some level of unauthenticated enumeration (`enum4linux -a`, `rpcclient -U "" -N`, anonymous `ldapsearch`), and when they do, it's often enough to pull a full user list before you have a single credential.

### 8.2 BloodHound

BloodHound maps every AD relationship — group memberships, who's a local admin of what, who can reset whose password, trust relationships between domains — as a graph, and then lets you query it for non-obvious attack paths (a classic example: "user X can RDP into a box where a Domain Admin has a cached logon session").

The current version is **BloodHound CE (Community Edition)** — a full rewrite from the legacy 4.x tool, now running on Docker Compose with a PostgreSQL + Neo4j backend and a modernized web UI. Data is collected separately and then imported:

```bash
# Collector (from a Windows box, domain-joined context):
SharpHound.exe -c All

# Or cross-platform, with just valid domain credentials:
bloodhound-python -u username -p password -d example.com -ns 10.10.10.10 -c All
```

The resulting JSON gets uploaded through BloodHound CE's web UI (File Ingest), after which you can run its built-in queries — "Shortest Path to Domain Admins" is the one almost everyone runs first — or write custom Cypher queries directly.

### 8.3 PowerView

For enumeration from an existing foothold on a domain-joined machine, PowerView (PowerShell) is the classic toolkit:

```powershell
Get-NetUser
Get-NetGroup "Domain Admins"
Get-NetComputer
```

### 8.4 Kerberos Enumeration

**Kerbrute** enumerates valid usernames by abusing Kerberos pre-authentication responses — notably, this does *not* trigger a failed-logon event the way an LDAP bind attempt would, making it considerably stealthier for username enumeration:

```bash
kerbrute userenum -d example.com --dc 10.10.10.10 userlist.txt
```

Worth flagging here because it starts as an enumeration activity: accounts configured with "Do not require Kerberos preauthentication" set can have their ticket requested and hashed offline (**AS-REP Roasting**) using Impacket's `GetNPUsers.py`:

```bash
GetNPUsers.py example.com/ -usersfile userlist.txt -no-pass
```

### 8.5 LDAP Enumeration of AD Specifically

```bash
windapsearch -d example.com --dc-ip 10.10.10.10 -U
ldapdomaindump -u 'EXAMPLE\\\\username' -p password 10.10.10.10
```

`ldapdomaindump` is particularly useful because it exports the entire directory dump as clean, browsable HTML/JSON — much easier to review than raw LDAP output.

### 8.6 AD Certificate Services (ADCS) — Advanced Mention

Misconfigured certificate templates (the well-known "ESC1" through "ESC8"-style issues) have become one of the most common real-world AD privilege-escalation paths in recent years. **Certipy** is the standard tool for finding them:

```bash
certipy find -u username@example.com -p password -dc-ip 10.10.10.10
```

This is a deep topic in its own right — worth knowing it exists and that it starts with an enumeration step, but a full treatment belongs in a dedicated AD attack-paths guide.

### 8.7 NetExec for AD at Scale

Beyond SMB, NetExec (Section 6.1) speaks LDAP, WinRM, MSSQL, SSH, RDP, and more — making it the practical way to enumerate an entire subnet of domain machines in one or two commands rather than one host at a time:

```bash
nxc smb 10.10.10.0/24 -u '' -p '' --shares
nxc ldap 10.10.10.10 -u username -p password --users
nxc smb 10.10.10.10 -u username -p password --bloodhound -c All
```

That last flag is worth noting — recent NetExec versions can run a BloodHound-compatible collection directly, skipping a separate SharpHound/bloodhound-python step entirely.

\---

## 9\. Cloud Reconnaissance \& Enumeration

As organizations move infrastructure to AWS, Azure, and GCP, a growing share of real attack surface lives in cloud-specific misconfigurations that traditional network recon simply won't find.

### 9.1 S3 Bucket Enumeration

AWS S3 buckets are one of the most common real-world misconfiguration sources — publicly listable or publicly readable buckets, often with predictable names tied to the company or project name.

```bash
aws s3 ls s3://examplecorp-backups --no-sign-request
s3scanner scan --bucket-file bucket\\\_names.txt
```

Bucket-name guessing typically follows predictable patterns (`examplecorp-prod`, `examplecorp-dev`, `examplecorp-backup`, `examplecorp-assets`), which is exactly what tools like `s3scanner` and `cloud\\\_enum` automate at scale. **Grayhat Warfare** maintains a searchable index of already-discovered open buckets, useful as a passive first check.

### 9.2 Azure \& GCP Storage

The same misconfiguration class exists outside AWS: Azure Blob Storage (`<account>.blob.core.windows.net`) and GCP Buckets (`storage.googleapis.com/<bucket>`). `cloud\\\_enum` covers all three providers together in one tool:

```bash
cloud\\\_enum -k examplecorp
```

### 9.3 Subdomain Takeover (Cloud Context)

As flagged in Section 5.4, this is especially common in cloud setups: a CNAME still pointing at a deprovisioned Heroku app, an unclaimed S3 bucket referenced by a custom domain, or a removed Azure/GitHub Pages site. Whoever claims that resource next can serve content under the victim's trusted subdomain.

```bash
subjack -w subdomains.txt -t 50 -o takeovers.txt -ssl
nuclei -l subdomains.txt -t http/takeovers/
```

### 9.4 Cloud Metadata Service Recon

Cloud instances (AWS EC2, and equivalents elsewhere) expose an internal-only metadata endpoint that can hand over IAM role credentials, instance identity documents, and user-data scripts — a critical target the moment you get **SSRF** or any form of local access to an instance:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

This is exactly the kind of target worth keeping in mind alongside SSRF testing — a blind SSRF that can reach `169.254.169.254` from inside a cloud VPC is a direct line to credential theft. AWS's hardening response is **IMDSv2**, which requires a session token obtained via a PUT request first, specifically to make this harder to reach through a simple SSRF.

### 9.5 Cloud Security Posture Tools

Once basic enumeration surfaces cloud infrastructure, broader configuration-auditing tools are the natural next step for an authorized cloud assessment: **ScoutSuite** (multi-cloud auditing) and **Prowler** (AWS-focused, maps findings to CIS benchmarks) both scan an entire account for misconfigurations well beyond what manual enumeration would catch.

\---

## 10\. Automation Frameworks \& All-in-One Platforms

A quick consolidated look at the broader tools that tie multiple recon phases together, several of which were introduced earlier:

|Tool|Role|
|-|-|
|**Recon-ng**|Modular CLI OSINT framework — workspaces, a module marketplace, structured output|
|**Amass**|Combines passive source aggregation, active DNS brute force, and permutation scanning in one engine|
|**SpiderFoot**|Automated OSINT aggregator with a web UI and 200+ correlating modules|
|**Maltego**|Graph-based OSINT visualization — "transforms" expand one entity (domain, person, IP) into its connections|
|**Nuclei**|Template-based scanner; technically bridges into vulnerability scanning, but its `technologies/`, `exposed-panels/`, and `misconfiguration/` template sets are pure recon value|

Nuclei example, used purely for recon-style checks rather than active exploitation:

```bash
nuclei -u http://example.com -t technologies/ -t exposed-panels/
```

The practical pattern most testers converge on: **Amass or subfinder** for subdomains → **httpx** to find live web hosts → **Nuclei** with recon-focused templates for a fast first pass → manual, protocol-specific enumeration (Sections 6–9) wherever something interesting turns up.

\---

## 11\. Worked Example: A Full Methodology Walkthrough

This ties every phase above together against a single hypothetical, authorized target — `example.com`, internal range `10.10.10.0/24`. Each step shows how a finding at one stage feeds directly into the next.

**Step 1 — OSINT.** `whois example.com` shows Cloudflare as the nameserver. `theHarvester -d example.com -b all` turns up a handful of employee emails in the `first.last@example.com` format, plus two subdomains. `crt.sh` (Section 3.5) adds another six subdomains from certificate SANs that never appeared in search engine results.

**Step 2 — DNS enumeration.** Running `amass enum -d example.com` against the full subdomain list confirms the known subdomains and brute-forces three more. A zone-transfer attempt (`dig axfr`) fails, as expected — one box checked off, no finding.

**Step 3 — Live host triage.** Piping the full subdomain list through `httpx -silent` shows which of the \~11 discovered subdomains actually serve a live web application right now — says 7 of them.

**Step 4 — Port scanning.** `nmap -sS -sV -sC -p- -T4` against the resolved IPs reveals the expected 80/443, plus an unexpected 445 (SMB) and 22 (SSH) open on one host — likely a misconfigured internal box that ended up with a public IP.

**Step 5 — SMB enumeration.** `enum4linux -a` against that host reveals a null session is permitted. `smbclient -L //<ip> -N` lists a share called `backups$`. Connecting to it turns up an old configuration file referencing a service account name and a password hint in a comment.

**Step 6 — Web enumeration.** `whatweb` against the main site identifies WordPress. `wpscan --enumerate vp,vt,u` finds an outdated plugin with a known CVE and confirms two valid usernames. Separately, `gobuster dir` against the same host turns up `/admin` and `/api/v1/` — neither linked from the visible site.

**Step 7 — Where recon ends.** At this point the enumeration phase has done its job: a full map of live subdomains, open ports and service versions, a set of valid usernames, an exposed SMB share with a credential hint, a vulnerable plugin, and two hidden endpoints. What comes next — matching the outdated plugin to its CVE, trying the discovered credential hint, deciding whether `/api/v1/` is worth probing further — is vulnerability analysis and exploitation, deliberately out of scope for this recon-and-enumeration-focused guide. The quality of everything that follows depends entirely on how thorough this phase was.

\---

## 12\. Defensive Perspective: Detecting \& Preventing Recon/Enumeration

Everything above has an equal and opposite blue-team version. Understanding both sides makes you better at each.

**Detection \& monitoring.** IDS/IPS platforms like Snort and Suricata ship with signatures for classic scan patterns (sequential SYN scans, NSE script fingerprints). SIEM rules alerting on repeated failed binds/authentications catch brute-force-style enumeration (SMTP VRFY sprays, SNMP community-string guessing, LDAP bind attempts) long before any single attempt would look suspicious on its own. `fail2ban` and equivalent tools handle this automatically at the host level.

**Reducing passive exposure.** WHOIS privacy protection hides registrant contact details. Being deliberate about what goes into job postings and public documentation limits free tech-stack intel. Stripping metadata before publishing documents closes off the ExifTool angle. Monitoring Certificate Transparency logs for unexpected certificate issuance (several free services do this) catches attacker subdomain discovery — and sometimes catches a rogue internal team spinning up unsanctioned infrastructure — before it's abused. Monitoring GitHub/GitLab continuously for leaked secrets (tools like GitGuardian exist specifically for this) closes the single most common real-world leak vector.

**Hardening live services.** Disable anonymous/null SMB sessions. Restrict DNS zone transfers to authorized secondary servers only. Change default SNMP community strings, or better, move to SNMPv3 and drop v1/v2c entirely. Use generic service banners and error pages — this is security-through-obscurity and shouldn't be relied on as a primary control, but it does raise the cost and time of reconnaissance. Enforce rate-limiting and account-lockout policies (balanced carefully against the risk of enabling a denial-of-service via forced lockouts). Disable unused legacy protocols (NetBIOS, LLMNR) where feasible. A WAF in front of a web app meaningfully slows directory brute-forcing. Remember that `robots.txt` is a convention, not a control — never rely on it to actually hide something sensitive. On AWS, enforce IMDSv2 so a simple SSRF can't reach instance credentials.

**Canaries and honeytokens.** Decoy assets — a fake admin panel, a planted canary credential, a honeytoken AWS key that alerts the moment it's used — turn passive attacker reconnaissance into an active, attributable alert the moment it crosses from "looking" to "touching."

\---

## 13\. Legal \& Ethical Considerations

The techniques in this guide are standard, widely taught, and legal to *learn* — what makes using them legal or illegal is authorization, not the technique itself.

* **Get it in writing.** A professional pentest runs under a signed scope document and rules of engagement (RoE), typically alongside a "get-out-of-jail" authorization letter. A bug bounty program defines its scope explicitly — assets outside that scope are off-limits even if they're owned by the same company running the program.
* **Unauthorized scanning carries real legal risk.** Laws like the U.S. Computer Fraud and Abuse Act (CFAA), the UK Computer Misuse Act, and similar legislation elsewhere criminalize unauthorized access to computer systems — and this can apply even to "just" active scanning or enumeration against a system you don't have explicit permission to test. Passive OSINT against genuinely public information carries far lower risk than anything active, but "is this actually public and actually fine to collect" is itself worth thinking through rather than assuming.
* **When in doubt, practice somewhere unambiguous.** TryHackMe, HackTheBox, PortSwigger's Web Security Academy, OWASP Juice Shop, DVWA, and a home lab built from vulnerable VMs are all built exactly for this, with zero ambiguity about authorization.

This isn't a substitute for actual legal advice on a specific engagement — just the baseline every security professional is expected to operate from.

\---

## 14\. Quick-Reference Cheat Sheet

|Category|Tool|Example Command|Purpose|
|-|-|-|-|
|OSINT|whois|`whois example.com`|Registrar, nameservers, creation date|
|OSINT|theHarvester|`theHarvester -d example.com -b all`|Emails, subdomains, hosts|
|OSINT|crt.sh|`curl -s "crt.sh/?q=%25.example.com\\\&output=json"`|Subdomains via cert transparency|
|OSINT|Shodan|`hostname:example.com` (web query)|Internet-wide exposed service search|
|OSINT|whatweb|`whatweb -v example.com`|Tech-stack fingerprinting|
|Host discovery|nmap|`nmap -sn 192.168.1.0/24`|Find live hosts|
|Port scanning|nmap|`nmap -sS -sV -sC -p- -T4 10.10.10.10`|Full TCP scan + version + default scripts|
|Port scanning|masscan|`masscan -p1-65535 10.10.10.0/24 --rate 10000`|Very fast large-range scanning|
|DNS|dig|`dig axfr example.com @ns1.example.com`|Zone transfer test|
|Subdomains|Amass|`amass enum -d example.com -active`|Passive + active subdomain discovery|
|SMB|enum4linux|`enum4linux -a 10.10.10.10`|Shares, users, OS info, null sessions|
|SMB/AD|NetExec|`nxc smb 10.10.10.10 -u '' -p '' --shares`|Modern all-in-one SMB/AD enumeration|
|SNMP|onesixtyone|`onesixtyone -c strings.txt 10.10.10.10`|Community-string brute force|
|LDAP|ldapsearch|`ldapsearch -x -H ldap://10.10.10.10 -b "dc=example,dc=com"`|Anonymous directory dump|
|Web dirs|gobuster|`gobuster dir -u http://example.com -w wordlist.txt`|Directory/file brute force|
|Web dirs|ffuf|`ffuf -u http://example.com/FUZZ -w wordlist.txt`|Fast fuzzing, vhosts, params|
|CMS|wpscan|`wpscan --url http://example.com --enumerate vp,vt,u`|WordPress plugins/themes/users|
|AD graph|BloodHound CE|`bloodhound-python -u user -p pass -d example.com -c All`|AD relationship/attack-path mapping|
|Kerberos|Kerbrute|`kerbrute userenum -d example.com --dc 10.10.10.10 users.txt`|Stealthy username enumeration|
|Cloud|cloud\_enum|`cloud\\\_enum -k examplecorp`|AWS/Azure/GCP storage enumeration|

\---

## 15\. Further Learning Resources

**Hands-on practice:**

* TryHackMe — the "Pre Security" and "Jr Penetration Tester" learning paths, plus standalone rooms like "Nmap," "Network Services," and "Enumeration"
* HackTheBox Academy — the "Footprinting" module specifically, and the beginner-friendly "Starting Point" machines
* PortSwigger Web Security Academy — less focused on network-level recon, but the strongest free resource for web-specific attack classes once enumeration hands you a target endpoint
* OWASP Juice Shop and DVWA — safe, intentionally vulnerable web apps for practicing web enumeration end to end

**Certifications that lean heavily on this material:**

* OSCP (Offensive Security Certified Professional) — enumeration depth is arguably the single biggest predictor of success
* eJPT, PNPT, CEH — all cover recon/enumeration methodology as a core module

**Reference material:**

* *Nmap Network Scanning* by Gordon "Fyodor" Lyon — the official Nmap book, still the deepest single resource on port scanning
* *RTFM: Red Team Field Manual* — a dense command-reference booklet, good as a physical/PDF companion to this guide
* **SecLists** (GitHub, `danielmiessler/SecLists`) — the wordlist collection referenced throughout this guide
* **PayloadsAllTheThings** (GitHub) — has a dedicated, actively maintained enumeration section covering most of the protocols above in more exhaustive, flag-by-flag detail

That combination — a lab to practice in, a structured path to follow, and a command reference to fall back on — is really the whole game from here. Everything in this guide is a starting map; the depth comes from running these commands against real (authorized) targets often enough that the methodology in Section 2 stops being a checklist and starts being instinct.

