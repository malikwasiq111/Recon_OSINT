# OSINT Tools — The Complete Reference (Sept 2026)

Built GitHub-first, cross-checked against Bellingcat's toolkit, the OSINT Newsletter's tools library, and current 2026 comparison write-ups. Where two tools do the same job, only the better one gets a full entry — the other is named inline so you know it exists and why it lost. 50+ tools total across 14 categories.

---

## 1. All-in-One Frameworks
#
### SpiderFoot
[smicallef/spiderfoot](https://github.com/smicallef/spiderfoot)
The heaviest hitter in the "point it at a target and let it run" category. 200+ modules cover IPs, domains, emails, usernames, and names, feeding each other's output in a publisher/subscriber pipeline so one finding automatically triggers the next round of lookups. Ships both a web UI and a CLI, integrates Shodan/HaveIBeenPwned/SecurityTrails/GreyNoise/etc., includes a YAML correlation engine that flags patterns across results automatically, and can route dark-web lookups through Tor. Python 3, actively developed since 2012, MIT-licensed (a paid SpiderFoot HX cloud tier exists for teams).

### theHarvester
[laramies/theHarvester](https://github.com/laramies/theHarvester)
The standard opening move in almost every recon checklist. Feed it a domain or company name and it pulls emails, subdomains, IPs, names, and URLs from 50+ sources at once — search engines, Shodan, Censys, crt.sh, VirusTotal, GitHub code search, and more. Recent versions added a REST API and moved to `uv` for setup. Python, lightweight, and effectively the baseline every other framework here tries to beat.

### Recon-ng
[lanmaster53/recon-ng](https://github.com/lanmaster53/recon-ng)
A Metasploit-styled interactive console built for web recon instead of exploitation. Modules are grouped by function (recon, discovery, reporting, import) and share one database, so a result from one module becomes the input for the next without manual copy-pasting. If you already know Metasploit's workflow, this is the fastest framework here to pick up. Python.

### Maltego
Community Edition free, paid tiers for teams — [maltego.com](https://www.maltego.com)
For when you need to see the relationships, not just a list of hits. Entities (people, domains, emails, IPs, companies) become nodes on a graph, and "Transforms" — small connectors to outside data sources — auto-expand each node with newly linked entities. The free Community Edition ships ~50 transforms; the real value is the ecosystem around it (Shodan, HaveIBeenPwned, WhatsMyName, and hundreds of community transforms on GitHub). Steeper learning curve than anything else on this list, which is also why cybercrime units (Ukraine's Cyber Police among them, publicly) use it for case-building. Owned by Charlesbank Capital Partners since 2023.

### sn0int
[kpcyrd/sn0int](https://github.com/kpcyrd/sn0int)
Its own README calls it "recon-ng and Maltego's flexibility, but fully open source." Sandboxed modules harvest subdomains from certificate transparency and passive DNS, pull emails from PGP keyservers, enrich IPs with ASN/geoIP, check breach exposure, do passive ARP enumeration on local networks, and scrape Instagram data (with built-in nudity detection on the images). Rust, modules are shareable via a package registry. A bit more of a learning curve than theHarvester, less GUI than Maltego.

### IntelOwl
[intelowlproject/IntelOwl](https://github.com/intelowlproject/IntelOwl)
Different job than the others: less "find things about a person," more "enrich an indicator I already have." Send it a file hash, IP, domain, or URL and it fans the query out to dozens of analyzers (VirusTotal, AbuseIPDB, GreyNoise, Shodan, MISP, and more) in parallel and returns one correlated report via REST API or dashboard. Built by the Italian MDR firm Certego, aimed at SOC/DFIR pipelines rather than one-off investigations. Django/Python, still shipping regular releases (v6.x line as of 2026).

### MISP
[MISP/MISP](https://github.com/MISP/MISP)
The tool IntelOwl often feeds into rather than a competitor to it: MISP stores, tags, and shares indicators of compromise (IPs, hashes, domains) across a trust group of organizations, with a taxonomy system, MITRE ATT&CK linking via "MISP Galaxy," and STIX import/export for interoperability with other threat-intel platforms. Less about one-off personal investigations, more about an organization or CERT running its own long-term threat-intel database. PHP, free and open source, widely deployed across national CERTs.

---

## 2. Username & Social Media Identity

### Maigret
[soxoj/maigret](https://github.com/soxoj/maigret)
The best all-around username tool available. It's a fork of the far more famous **Sherlock** ([sherlock-project/sherlock](https://github.com/sherlock-project/sherlock), still fine and actively maintained if you only want a fast check against ~400 sites) — but Maigret checks 3,000+ sites, does recursive search (a found account pointing to a *different* username triggers a follow-up search on that one too), parses profile pages for bio/links/metadata, and outputs HTML, PDF, or graph reports, plus an experimental AI summary mode. Since it does everything Sherlock does and considerably more, it gets the single spot here. Python, fetches an updated site list automatically.

### Blackbird
[p1ngul1n0/blackbird](https://github.com/p1ngul1n0/blackbird)
Worth running alongside Maigret rather than instead of it, because it covers different ground: Blackbird searches both usernames *and* email addresses (Maigret is username-only) across 600+ sites, pulls its site list from the same WhatsMyName dataset, and has an opt-in AI step that turns the raw hit list into a behavioral/technical profile summary. Clean PDF/CSV export. Python, actively maintained, built with Spain's Cyber Hunter Lab.

### Social-Analyzer
[qeeqbox/social-analyzer](https://github.com/qeeqbox/social-analyzer)
Goes further than a hit/miss list: assigns a 0–100 confidence rating per site using layered detection (plain HTTP checks, headless-browser rendering, and OCR on profile screenshots) across 1,000+ platforms, and can screenshot each confirmed profile automatically. Available as CLI, API, or a self-hosted web app. Heavier setup than Maigret (needs Firefox + Tesseract), but the confidence scoring cuts down on false positives — reportedly used by resource-limited law-enforcement units for exactly that reason.

### WhatsMyName
[WebBreacher/WhatsMyName](https://github.com/WebBreacher/WhatsMyName)
Not really a tool you run yourself — it's the dataset half the tools above quietly run on. A community-maintained, continuously verified JSON file of 700+ sites with exact detection logic (specific status codes, specific page strings) for confirming a username genuinely exists rather than guessing off a redirect. Maigret, Blackbird, SpiderFoot, Recon-ng, sn0int, and Maltego's WhatsMyName transforms all pull from this one file. Worth knowing about directly — browse it at whatsmyname.app — if you ever want to write your own checker.

---

## 3. Email Address Investigation

### Holehe
[megadose/holehe](https://github.com/megadose/holehe)
Point it at an email and it checks 120+ services (Twitter/X, Instagram, Spotify, Adobe, etc.) using each site's own "forgot password" flow to silently confirm an account exists — without a reset email ever reaching the target. Occasionally recovers a partially masked backup email or phone number in the process. Python, simple to import as a module in your own scripts.

### h8mail
[khast3x/h8mail](https://github.com/khast3x/h8mail)
Built for breach/password hunting rather than account discovery. Takes one or more emails, regex-matches related addresses out of raw text/HTML dumps, and queries breach APIs (or a local copy of compilations like "Collection #1") to surface exposed passwords. Its "chase" feature feeds hunter.io results back in to auto-expand the target list. Ships in BlackArch, CSI Linux, and the Trace Labs OSINT VM by default.

### GHunt
[mxrch/GHunt](https://github.com/mxrch/GHunt)
The tool for what's publicly reachable from a Gmail/Google account: name, Google ID, which services are active, a possible YouTube channel, public Google Maps reviews (a real travel/location-history leak), and public Photos/Drive links. Needs your own authenticated Google session (via a companion browser extension) to hit Google's internal endpoints, which makes it more fragile than everything else here — Google's anti-abuse systems periodically break specific lookups (an open issue as of mid-2026 affects the email-lookup path specifically). Still the most capable option for the job; Bellingcat has used it for attribution work, including tracking cartel figures via linked Maps reviews.

---

## 4. Phone Number Investigation

### PhoneInfoga
[sundowndev/PhoneInfoga](https://github.com/sundowndev/PhoneInfoga)
The standard here, rewritten in Go for v2. Validates format, identifies country/carrier/line type, then tries to fingerprint the VoIP provider or turn up footprints via search-engine scanning. Ships as CLI, REST API, and a small web client. No paid keys needed for the basics, though the OSINT-scanning step can get rate-limited by Google.

---

## 5. Domain, Subdomain & DNS Reconnaissance

### Amass
[owasp-amass/amass](https://github.com/owasp-amass/amass)
The most thorough subdomain/attack-surface mapper around, and an official OWASP flagship project. Combines DNS brute-forcing, zone-transfer attempts, reverse-DNS sweeps, scraping dozens of search engines and archives, and pulling from 50+ APIs (Shodan, Censys, SecurityTrails, VirusTotal, GitHub, and more) into a graph database you can query for relationships between subdomains, IPs, and ASNs afterward. Slower than Subfinder because it resolves actively by default.

### Subfinder
[projectdiscovery/subfinder](https://github.com/projectdiscovery/subfinder)
The speed pick. Purely passive — never touches the target directly — pulls from dozens of curated sources, and pipes cleanly into other ProjectDiscovery tools (`subfinder -d target.com | httpx`). Reach for this first when you just need a subdomain list fast; bring in Amass afterward for the deeper, active pass.

### crt.sh
Search interface: [crt.sh](https://crt.sh)
A free, no-signup search engine over Certificate Transparency logs — every publicly trusted SSL/TLS certificate ever issued, searchable by domain, organization, or fingerprint. Because certificates list every subdomain they cover, this routinely surfaces staging/dev/admin subdomains that a normal crawl misses entirely. Add `?output=json` to any query for a free, unrestricted API. It's also one of the core sources theHarvester, Amass, and Subfinder all pull from directly.

### dnsrecon
[darkoperator/dnsrecon](https://github.com/darkoperator/dnsrecon)
The record-level DNS tool, as opposed to the subdomain-name tools above: pulls every record type (A, AAAA, MX, TXT, SOA, SRV), attempts zone transfers, reverse-looks-up a whole CIDR range, and brute-forces against a wordlist. Python. (**DNSDumpster**, a free web-based alternative, does a lighter version of the same job with a visual domain map if you want something browser-based instead.)

### dnstwist
[elceef/dnstwist](https://github.com/elceef/dnstwist)
A different angle: instead of finding what's already registered under a target, it generates every plausible typo, homoglyph, and permutation of a domain name and checks which are registered — the standard way to catch phishing/typosquatting infrastructure being staged against an organization before it's used. Can fuzzy-hash the resulting pages to flag ones visually cloning the real site. Python; a hosted version exists at dnstwist.it if you don't want to install anything.

### pagodo
[opsdisk/pagodo](https://github.com/opsdisk/pagodo)
Automates Google dorking. Pulls the current Google Hacking Database — the standard library of dork queries for finding exposed files, login pages, and misconfigured services — and runs it against a target domain, throttled to avoid getting the querying IP blocked. Python.

---

## 6. Internet-Wide Scanning (Search Engines)

Hosted platforms rather than something you `git clone`, but no OSINT kit is complete without at least one.

### Shodan
[shodan.io](https://www.shodan.io)
The original, and still the most widely integrated — theHarvester, Amass, SpiderFoot, and Recon-ng all ship a native Shodan module. Indexes banners, open ports, and service metadata for internet-connected devices: the standard way to find exposed cameras, industrial control systems, databases, and forgotten test servers. Free tier is metered to roughly 100 query credits a month.

### Censys
[censys.io](https://censys.io)
More research-oriented than Shodan: stronger TLS/certificate data, a SQL-style query language, and results generally considered more precise for deep technical or academic work. Treat it as Shodan's second opinion rather than a straight replacement.

### ZoomEye
[zoomeye.org](https://www.zoomeye.org)
Shodan's Chinese counterpart, run by Knownsec 404 Team, with the best coverage of infrastructure across China and East Asia specifically, plus built-in web-component and vulnerability search. (**FOFA**, **Netlas**, and **Criminal IP** are newer entrants worth a look for a fourth or fifth angle — Criminal IP in particular layers phishing/threat scoring on top of the raw scan data.)

---

## 7. Metadata & Document Intelligence

### ExifTool
[exiftool.org](https://exiftool.org)
The reference tool for reading and writing metadata across 200+ file formats, not just images. For OSINT: camera model and GPS coordinates baked into photos, the software used to build a document, and author/editor names sitting in "Last Modified By" fields. Perl-based, with bindings in nearly every language.

### Metagoofil
[opsdisk/metagoofil](https://github.com/opsdisk/metagoofil) (originally laramies/metagoofil)
Automates the step before ExifTool: searches a target domain for public documents (PDF, DOCX, XLSX, PPTX), downloads them, and extracts the metadata for you — a fast way to build a list of real employee usernames and internal software versions from documents an organization published without realizing what was embedded. The Linux CLI counterpart to FOCA below.

### FOCA
[ElevenPaths/FOCA](https://github.com/ElevenPaths/FOCA)
The Windows GUI version of the same idea: finds and downloads documents via Google/Bing/DuckDuckGo, then extracts metadata, with point-and-click ease and support for a wider document range (InDesign, SVG included). Worth having if you prefer a GUI walkthrough over Metagoofil's command line, or need something for a client-facing report.

---

## 8. Platform-Specific Social Media OSINT

### Osintgram
[Datalux/Osintgram](https://github.com/Datalux/Osintgram)
An interactive shell for digging into one Instagram account at a time: followers/following, hashtags used, engagement totals, tagged users, photo/story/profile-picture downloads. Needs its own logged-in Instagram account to query with, and — like every Instagram scraper — periodically breaks when Instagram changes its API; check open issues before depending on it for anything time-sensitive.

### Instaloader
[instaloader/instaloader](https://github.com/instaloader/instaloader)
The better pick if what you actually need is bulk archiving rather than an interactive shell: downloads full profiles, hashtags, stories, and saved posts along with captions, comments, and geotags, with resumable downloads and automatic handling of username changes. More consistently maintained than Osintgram — use this for "grab everything from this account," Osintgram for "walk through this account interactively."

### Toutatis
[megadose/toutatis](https://github.com/megadose/toutatis)
Narrower and sharper: exploits an Instagram API quirk to pull the partially-masked email and phone number tied to a public account, plus join date and follower/following counts. Had a rough patch when Instagram broke it, but a 2025 refactor fixed the outstanding issues — a good complement to Osintgram/Instaloader rather than a replacement.

### CrossLinked
[m8sec/CrossLinked](https://github.com/m8sec/CrossLinked)
Solves LinkedIn employee enumeration without ever touching LinkedIn or needing an API key — it scrapes Google/Bing's cached LinkedIn profile pages instead, then formats the extracted names into a target company's likely email/username convention (`first.last`, `flast`, etc.). The standard pre-step to building a password-spraying or phishing target list. Python.

---

## 9. Web Crawling & Historical Data

### gau (GetAllUrls)
[lc/gau](https://github.com/lc/gau)
Pulls every known URL for a domain by combining the Wayback Machine, Common Crawl, AlienVault OTX, and URLScan into one query — useful for surfacing old or forgotten endpoints (admin panels, deprecated API versions, parameter names) that a fresh crawl won't show. (**waybackurls** by tomnomnom is the simpler, Wayback-only tool this one extends; use gau unless you specifically want the smaller dependency footprint.)

### Photon
[s0md3v/Photon](https://github.com/s0md3v/Photon)
A crawler purpose-built for OSINT extraction rather than general scraping: while crawling normally, it pulls out in-scope/out-of-scope URLs, parameterized URLs, emails, social media handles, exposed API/auth keys, JS endpoints, and files (PDF, PNG, XML). Has a "Ninja Mode" that routes requests through several public archive services at once for speed and a bit of cover. Python, GPL-3.0.

---

## 10. Web Technology Fingerprinting

### Wappalyzer
[wappalyzer/wappalyzer](https://github.com/wappalyzer/wappalyzer)
Identifies what a site is built with — CMS, JS frameworks, analytics/ad trackers, ecommerce platform, server software — by matching HTML, headers, cookies, and JS variables against a large signature database. Available as a browser extension, CLI, and library. (**WhatWeb**, a Ruby CLI that ships in Kali by default, is the go-to if you want something scriptable without a Node dependency.)

---

## 11. Dark Web / Tor Investigation

### OnionScan
[s-rah/onionscan](https://github.com/s-rah/onionscan)
Audits a .onion hidden service for the operational-security mistakes that actually deanonymize operators in practice — exposed EXIF data, SSH host keys or TLS certs reused across the clear web and Tor, misconfigured servers leaking a real IP. Not a general vulnerability scanner; it's specifically built for the deanonymization angle.

### TorBot
[DedSecInside/TorBot](https://github.com/DedSecInside/TorBot)
A dark-web crawler and link indexer: give it a starting .onion URL and it maps outbound links, extracts emails found on the page, and checks whether linked sites are actually reachable. More general-purpose than OnionScan — use TorBot to map and explore, OnionScan to audit one specific service you already care about.

### Ahmia
[ahmia.fi](https://ahmia.fi)
The search engine that gets you to a starting point for either of the above: indexes and searches .onion hidden services from the clearnet, filtering out abuse material rather than returning everything a crawler finds. Tor Project–endorsed since 2014, open source, and the one dark-web search engine that shows up consistently across journalism and OSINT workflows rather than just cybercrime forums — use it to find the .onion address worth pointing OnionScan or TorBot at next.

---

## 12. Code Repositories & Secrets Exposure

### TruffleHog
[trufflesecurity/trufflehog](https://github.com/trufflesecurity/trufflehog)
Scans full git history (not just the latest commit) for accidentally committed secrets, then actually verifies whether each one is still live by pinging the relevant provider's API — cutting false positives dramatically. Also scans beyond git into S3 buckets, Docker images, and Slack. Go, heavier to run than Gitleaks because of the live-verification step.

### Gitleaks
[gitleaks/gitleaks](https://github.com/gitleaks/gitleaks) (originally zricethezav/gitleaks)
The lighter, faster counterpart: pure regex/entropy pattern matching, no live verification, built to run in milliseconds as a pre-commit hook rather than a deep CI pass. 17k+ stars, one of the most-used secret scanners around. Most teams run both — Gitleaks locally for instant feedback, TruffleHog in CI for the deeper, verified pass.

### GitTools
[internetwache/GitTools](https://github.com/internetwache/GitTools)
A different failure mode than TruffleHog/Gitleaks: those scan a repo you already have. GitTools finds and reconstructs one you *don't* — misconfigured web servers that deployed via `git clone` and left the `.git` folder itself publicly reachable. Its three pieces (Finder, Dumper, Extractor) locate exposed `.git` directories across a list of domains, download them, and rebuild the repository locally, source history and all. (A faster modern rewrite exists at [arthaud/git-dumper](https://github.com/arthaud/git-dumper) if you just need the Dumper piece.)

---

## 13. Cryptocurrency & Blockchain Intelligence

### GraphSense
[github.com/graphsense](https://github.com/graphsense) (documentation at [graphsense.github.io](https://graphsense.github.io))
The open-source end of blockchain forensics — everything else in this space (Chainalysis, TRM Labs, Elliptic) is commercial and closed. Ingests Bitcoin/Ethereum/Litecoin/Zcash/Bitcoin-Cash chain data and lets you traverse transaction graphs, cluster addresses likely controlled by the same entity, and attach public attribution tags (exchange addresses, known scams) to trace where funds moved. Self-hostable for full data control, with a Maltego transform available if you'd rather work from a graph UI you already know. MIT-licensed, maintained jointly by Iknaio and Austria's Complexity Science Hub — heavier to stand up than anything else on this list (needs a Cassandra cluster), but there's no lighter open-source equivalent that does the same job.

---

## 14. Meta-Resource

### OSINT Framework
[osintframework.com](https://osintframework.com)
Not a tool — a browsable, categorized map of hundreds of OSINT tools and resources (many of which are above), maintained as a static reference site. The right starting point when you know the *type* of information you need (a phone number, a leaked credential, a piece of metadata) but haven't yet picked which specific tool covers it.

---

## Also Worth Knowing (brief mentions)

- **Nmap** — not passive OSINT, but active port/service scanning is the natural next step after recon, and both SpiderFoot's and Amass's own docs call out to it directly.
- **GitGot** ([BishopFox/GitGot](https://github.com/BishopFox/GitGot)) — searches an entire GitHub org's repos for sensitive filenames and patterns, complementary to TruffleHog/Gitleaks, which focus on one repo's history at a time.
- **BuiltWith** / **Netcraft** — hosted services for site tech stack + hosting/company history; useful when Wappalyzer's live scan isn't enough and you want historical data instead.

---

## Notes

- Almost everything above is a CLI/Python/Go tool with no GUI dependency — straightforward to wrap as a callable tool if you're extending something like an MCP server rather than running each one by hand.
- This space moves fast. Site-detection lists (Sherlock/Maigret/Blackbird), platform scrapers (Osintgram, Toutatis), and anything relying on an authenticated session (GHunt) break the moment the target platform changes its frontend or API. Check a repo's recent issues before leaning on it for time-sensitive work — several tools above (Osintgram, GHunt) have open, unresolved breakage as of this writing.
- All of this works on information that's already public or exposed via a service's own account-recovery flow. Whether you're authorized to point it at a given target is a separate question the tools themselves don't answer.
