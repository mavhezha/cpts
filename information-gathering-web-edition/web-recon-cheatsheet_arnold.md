# Information Gathering - Web Edition Cheatsheet

## Recon Types
- Active: direct interaction with the target (port scanning, vuln scanning, banner grabbing, web spidering). Higher detection risk.
- Passive: relies on public sources (search engines, WHOIS, DNS, web archives, social media, code repos). Very low detection risk.

## WHOIS
```bash
sudo apt install whois -y
whois <domain>
```
Reveals registrar, registrant/admin/tech contacts, creation/expiry dates, name servers, domain status flags. Combine with other sources since GDPR-era privacy services often mask registrant details.

## DNS

### Manual lookups with dig
```bash
dig <domain>                  # default A record
dig <domain> A
dig <domain> AAAA
dig <domain> MX
dig <domain> NS
dig <domain> TXT
dig <domain> CNAME
dig <domain> SOA
dig @1.1.1.1 <domain>          # query a specific name server
dig +trace <domain>            # show full resolution path
dig -x <IP>                    # reverse lookup
dig +short <domain>            # concise answer
dig +noall +answer <domain>    # answer section only
dig <domain> ANY               # many servers ignore this per RFC 8482
```

### Record types worth knowing
A, AAAA, CNAME, MX, NS, TXT, SOA, SRV, PTR

### Hosts file (manual override, bypasses DNS)
- Linux/macOS: `/etc/hosts`
- Windows: `C:\Windows\System32\drivers\etc\hosts`
```
<IP Address>    <Hostname> [<Alias> ...]
```
Add discovered vhosts/subdomains here so your browser and tools resolve them correctly during an engagement.

## Subdomain Enumeration

### Brute force with dnsenum
```bash
dnsenum --enum <domain> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r
```
`-r` enables recursive brute-forcing of discovered subdomains.

Other brute-force tools: fierce, dnsrecon, amass, assetfinder, puredns, gobuster, ffuf.

### DNS zone transfer (AXFR)
```bash
dig axfr @<name_server> <domain>
```
If misconfigured, dumps the entire zone file (subdomains, IPs, NS records) in one shot. Rarely works against hardened targets but always worth trying.

### Certificate Transparency logs (passive)
```bash
curl -s "https://crt.sh/?q=<domain>&output=json" | jq -r '.[] | .name_value' | sort -u
```
Filter for a keyword:
```bash
curl -s "https://crt.sh/?q=<domain>&output=json" | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | sort -u
```
CT logs give a historical, definitive record of issued certs, including subdomains no wordlist would guess, and can surface old/expired certs tied to outdated, vulnerable hosts.

## Virtual Hosts (vHosts)
Web servers can host multiple sites on one IP, distinguished by the HTTP `Host` header rather than DNS. A vHost with no DNS record still resolves once added to your local hosts file.

### Fuzzing vHosts with gobuster
```bash
gobuster vhost -u http://<target_IP> -w <wordlist> --append-domain
```
- `-t` more threads
- `-k` ignore TLS cert errors
- `-o` save output to file

Other vHost fuzzers: Feroxbuster, ffuf.

## Fingerprinting

### Banner grabbing / headers
```bash
curl -I <domain>
curl -I https://<domain>
```
Look at `Server`, `X-Powered-By`, `X-Redirect-By`, and any `wp-json` style links, they reveal CMS, framework, and version info.

### WAF detection
```bash
pip3 install git+https://github.com/EnableSecurity/wafw00f
wafw00f <domain>
```

### Nikto (software identification only)
```bash
nikto -h <domain> -Tuning b
```

Other fingerprinting tools: Wappalyzer, BuiltWith, WhatWeb, Nmap (`-sV`, `-O`, NSE scripts), Netcraft.

## Crawling / Spidering
Breadth-first (map the whole site) vs depth-first (chase one path deep). Extract: internal/external links, comments, metadata, and sensitive files (backups, configs, logs).

### ReconSpider (Scrapy-based)
```bash
pip3 install scrapy
wget -O ReconSpider.zip https://cdn.services-k8s.prod.aws.htb.systems/content/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip
python3 ReconSpider.py http://<domain>
```
Outputs `results.json` with emails, links, external_files, js_files, form_fields, images, videos, audio, and comments.

Other crawlers: Burp Suite Spider, OWASP ZAP Spider, Apache Nutch.

## robots.txt
Located at `<domain>/robots.txt`. Not enforced, but legitimate bots respect it.
```
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /public/

User-agent: Googlebot
Crawl-delay: 10

Sitemap: https://example.com/sitemap.xml
```
Disallowed paths are a map of what the owner wants hidden from search engines, check them manually.

## Well-Known URIs (RFC 8615)
Standard metadata location at `<domain>/.well-known/`.

| URI Suffix | Purpose |
|---|---|
| security.txt | Contact info for reporting vulnerabilities |
| change-password | Standard password change URL |
| openid-configuration | OpenID Connect provider metadata (authorization/token/userinfo endpoints, JWKS URI, supported scopes) |
| assetlinks.json | Verifies ownership of linked digital assets |
| mta-sts.txt | SMTP MTA Strict Transport Security policy |

## Search Engine Discovery / Google Dorking

| Operator | Purpose |
|---|---|
| site: | Limit to a domain |
| inurl: | Term in the URL |
| filetype: | Specific file extension |
| intitle: | Term in the page title |
| intext: / inbody: | Term in body text |
| cache: | Cached version of a page |
| link: | Pages linking to a URL |
| " " | Exact phrase |
| - | Exclude a term |
| AND / OR / NOT | Combine or exclude terms |

Common dork patterns:
```
site:example.com inurl:login
site:example.com filetype:pdf
site:example.com inurl:config.php
site:example.com inurl:backup
site:example.com filetype:sql
```

## Wayback Machine
`https://web.archive.org/web/*/<domain>`

Reveals old pages, directories, subdomains, and configurations no longer linked from the live site. Fully passive, no direct interaction with target infrastructure.

## Automated Recon Frameworks
- FinalRecon: headers, WHOIS, SSL info, crawling, DNS/subdomain enum, directory enum, Wayback URLs, port scan
- Recon-ng, theHarvester, SpiderFoot, OSINT Framework

### FinalRecon quick use
```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py
./finalrecon.py --headers --whois --url http://<domain>
./finalrecon.py --full --url http://<domain>
```
