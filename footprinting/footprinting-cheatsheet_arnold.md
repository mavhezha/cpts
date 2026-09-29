# Footprinting Cheatsheet

## Enumeration Principles
- Enumeration is active (scans) and passive (third-party sources) information gathering; OSINT is a separate, purely passive procedure.
- Enumeration is a loop: keep gathering based on what you already know or have found.
- Goal is not to break in directly but to find every possible way in, understanding the infrastructure before attacking it.
- Three guiding principles:
  1. There is more than meets the eye. Consider all points of view.
  2. Distinguish between what you see and what you do not see.
  3. There are always ways to gain more information. Understand the target.

## Enumeration Methodology (6 Layers)
| Layer | Focus | Information Categories |
|---|---|---|
| 1. Internet Presence | Externally accessible infrastructure | Domains, subdomains, vHosts, ASN, netblocks, IPs, cloud instances, security measures |
| 2. Gateway | Security measures protecting the infrastructure | Firewalls, DMZ, IPS/IDS, EDR, proxies, NAC, segmentation, VPN, Cloudflare |
| 3. Accessible Services | Externally/internally hosted services | Service type, functionality, configuration, port, version, interface |
| 4. Processes | Internal processes tied to services | PID, processed data, tasks, source, destination |
| 5. Privileges | Internal permissions on accessible services | Groups, users, permissions, restrictions, environment |
| 6. OS Setup | Internal system configuration | OS type, patch level, network config, environment, config files, sensitive files |

## Domain Information (Passive)
- Start with the company's own website, read what services/technologies they describe offering.
- SSL certificate on the main site often lists multiple DNS names.
- crt.sh (Certificate Transparency logs):
```bash
curl -s https://crt.sh/\?q\=<domain>\&output\=json | jq .
curl -s https://crt.sh/\?q\=<domain>\&output\=json | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```
- Identify company-hosted (vs third-party-hosted) IPs before testing:
```bash
for i in $(cat subdomainlist); do host $i | grep "has address" | grep <domain> | cut -d" " -f1,4; done
```
- Shodan lookups on discovered IPs to see open ports/services.
- Full DNS record dump:
```bash
dig any <domain>
```
- TXT records can reveal third-party providers in use (Atlassian, Google Workspace, LogMeIn, Mailgun, Outlook/Office 365, hosting provider IDs like INWX) which hint at attack surface (e.g. open GDrive links, SMB-based Azure file storage, API interfaces to test for IDOR/SSRF).

## Cloud Resources
- Misconfigured S3 buckets (AWS), blobs (Azure), and cloud storage (GCP) can be publicly accessible.
- Google dorks: `intext:<company> inurl:amazonaws.com`, `intext:<company> inurl:blob.core.windows.net`
- Check page source for `dns-prefetch`/`preconnect` links to cloud storage domains.
- domain.glass for infrastructure overview and CDN/WAF status (e.g. Cloudflare).
- GrayHatWarfare to search/filter discovered cloud storage by file type; can expose leaked SSH keys, documents, etc.

## Staff / OSINT
- LinkedIn job postings reveal tech stack: languages, databases, ORMs, web frameworks, CI/CD, version control.
- Employee profiles/GitHub repos can leak personal emails, hardcoded JWTs/secrets, or internal tooling details.
- Cross-reference technical employees (dev + security) to infer likely defenses in place.

---

## Service-by-Service Footprinting

### FTP (21/tcp, data 20/tcp)
```bash
ftp <IP>                                  # try anonymous:anything
nmap -Pn --script ftp-anon <IP>
nmap -sV -p21 -sC -A <IP>
nmap -sV -p21 -sC -A <IP> --script-trace
openssl s_client -connect <IP>:21 -starttls ftp   # for FTPS
```
- Config: `/etc/vsftpd.conf`, blocked users in `/etc/ftpusers`.
- Dangerous settings: `anonymous_enable=YES`, `anon_upload_enable=YES`, `anon_mkdir_write_enable=YES`, `write_enable=YES`, `hide_ids=YES` (masks UID/GID), `ls_recurse_enable=YES`.
- `ftp> ls -R` for recursive listing, `get`/`put` to transfer, `debug`/`trace` for verbose output.
- Bulk download: `wget -m --no-passive ftp://anonymous:anonymous@<IP>`
- TFTP (UDP, no auth): `connect`, `get`, `put`, `status`, `verbose`; no directory listing.

### SMB (139, 445/tcp)
```bash
smbclient -N -L //<IP>/
smbclient //<IP>/<share>
rpcclient -U "" <IP>
smbmap -H <IP>
crackmapexec smb <IP> --shares -u '' -p ''
enum4linux-ng.py <IP> -A
```
- rpcclient queries: `srvinfo`, `enumdomains`, `querydominfo`, `netshareenumall`, `netsharegetinfo <share>`, `enumdomusers`, `queryuser <RID>`, `querygroup <RID>`.
- Brute force RIDs:
```bash
for i in $(seq 500 1100); do rpcclient -N -U "" <IP> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo ""; done
```
- Dangerous smb.conf settings: `browseable = yes`, `read only = no`, `writable = yes`, `guest ok = yes`, `enable privileges = yes`, `create mask = 0777`, `directory mask = 0777`.
- samrdump.py (Impacket) as an alternative to RID brute forcing.

### NFS (111, 2049/tcp, RPC-based)
```bash
nmap 10.129.14.128 -p111,2049 -sV -sC
nmap --script nfs* <IP> -sV -p111,2049
showmount -e <IP>
mkdir target-NFS && sudo mount -t nfs <IP>:/ ./target-NFS/ -o nolock
sudo umount ./target-NFS
```
- Config: `/etc/exports`. Options: `rw`, `ro`, `sync`, `async`, `secure`, `insecure`, `no_subtree_check`, `root_squash`.
- Dangerous: `rw`, `insecure`, `nohide`, `no_root_squash` (root-created files keep UID/GID 0).
- Match UID/GID on your own system to read/write as the mapped user.

### DNS (53/tcp+udp)
```bash
dig ns <zone> @<IP>
dig CH TXT version.bind <IP>
dig any <zone> @<IP>
dig axfr <zone> @<IP>              # zone transfer attempt
for sub in $(cat wordlist); do dig $sub.<domain> @<IP> | grep -v ';\|SOA' | grep $sub; done
dnsenum --dnsserver <IP> --enum -p 0 -s 0 -o subdomains.txt -f <wordlist> <domain>
```
- Config files: `named.conf.local`, `named.conf.options`, zone files, reverse zone files.
- Dangerous options: `allow-query`, `allow-recursion`, `allow-transfer` (misconfigured to a wide subnet/any allows full zone dumps, sometimes revealing internal-only zones).

### SMTP (25/tcp, submission 587, SMTPS 465)
```bash
telnet <IP> 25
nmap <IP> -sC -sV -p25
nmap <IP> -p25 --script smtp-open-relay -v
```
- Commands: `HELO`/`EHLO`, `MAIL FROM`, `RCPT TO`, `DATA`, `RSET`, `VRFY`, `EXPN`, `NOOP`, `QUIT`.
- `VRFY <user>` can enumerate valid users (not always reliable, some servers return 252 for anything).
- Dangerous config: `mynetworks = 0.0.0.0/0` (open relay, spoofable mail).

### IMAP (143/993) / POP3 (110/995)
```bash
nmap <IP> -sV -p110,143,993,995 -sC
curl -k 'imaps://<IP>' --user <user>:<pass>
openssl s_client -connect <IP>:imaps
openssl s_client -connect <IP>:pop3s
```
- IMAP commands: `LOGIN`, `LIST`, `CREATE`, `DELETE`, `RENAME`, `LSUB`, `SELECT`, `UNSELECT`, `FETCH`, `CLOSE`, `LOGOUT`.
- POP3 commands: `USER`, `PASS`, `STAT`, `LIST`, `RETR`, `DELE`, `CAPA`, `RSET`, `QUIT`.
- Dangerous Dovecot settings: `auth_debug`, `auth_debug_passwords`, `auth_verbose_passwords`, `auth_anonymous_username`.

### SNMP (161/udp, traps 162/udp)
```bash
snmpwalk -v2c -c public <IP>
onesixtyone -c <community_wordlist> <IP>
braa <community>@<IP>:.1.3.6.*
```
- v1/v2c have no real security, community string sent in plaintext; v3 adds auth + encryption.
- Config: `/etc/snmp/snmpd.conf`; dangerous: `rwuser noauth`, `rwcommunity`/`rwcommunity6` bound broadly.

### MySQL (3306/tcp)
```bash
nmap <IP> -sV -sC -p3306 --script mysql*
mysql -u root -p -h <IP>
```
- Config: `/etc/mysql/mysql.conf.d/mysqld.cnf`.
- Dangerous: plaintext `user`/`password`/`admin_address` in config, verbose `debug`/`sql_warnings`, weak `secure_file_priv`.
- Useful queries: `show databases;`, `use <db>;`, `show tables;`, `show columns from <table>;`, `select * from <table>;`.

### MSSQL (1433/tcp)
```bash
nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes -sV -p1433 <IP>
impacket-mssqlclient <domain>/<user>:'<pass>'@<IP> [-windows-auth]
```
- Default DBs: master, model, msdb, tempdb, resource.
- Watch for: unencrypted clients, self-signed certs, named pipes enabled, weak/default `sa` credentials.

### Oracle TNS (1521/tcp)
```bash
nmap -p1521 -sV <IP> --open
nmap -p1521 -sV <IP> --open --script oracle-sid-brute
./odat.py all -s <IP>
sqlplus <user>/<pass>@<IP>/<SID>
sqlplus <user>/<pass>@<IP>/<SID> as sysdba
```
- Config: `tnsnames.ora` (client-side), `listener.ora` (server-side).
- Default creds to try: Oracle 9 `CHANGE_ON_INSTALL`, DBSNMP `dbsnmp`.
- After access: `select table_name from all_tables;`, `select * from user_role_privs;`, `select name, password from sys.user$;` for hash extraction, or `utlfile --putFile` for web shell upload if a web root path is known.

### IPMI (623/udp)
```bash
nmap -sU --script ipmi-version -p623 <IP>
msf > use auxiliary/scanner/ipmi/ipmi_version
msf > use auxiliary/scanner/ipmi/ipmi_dumphashes
```
- Default creds: Dell iDRAC `root:calvin`, Supermicro `ADMIN:ADMIN`, HP iLO random 8-char string.
- RAKP protocol flaw in IPMI 2.0 leaks salted password hashes for any valid user; crack offline with Hashcat mode 7300.

### Linux Remote Management
- SSH (22/tcp): `ssh-audit`, banner reveals version; dangerous sshd_config: `PasswordAuthentication yes`, `PermitEmptyPasswords yes`, `PermitRootLogin yes`, `Protocol 1`, `X11Forwarding yes`, `AllowTcpForwarding yes`.
- Rsync (873/tcp): `nc -nv <IP> 873` then `#list`; `rsync -av --list-only rsync://<IP>/<share>`.
- R-services (512/513/514/tcp): `rlogin`, `rsh`, `rexec`; trust controlled by `/etc/hosts.equiv` and `~/.rhosts` (`+` wildcard = any host/user trusted); `rwho`/`rusers` to enumerate logged-in users.

### Windows Remote Management
- RDP (3389/tcp): `nmap -sV -sC <IP> -p3389 --script rdp*`; `xfreerdp /u:<user> /p:<pass> /v:<IP>`; check NLA and encryption level.
- WinRM (5985 HTTP / 5986 HTTPS): `nmap -sV -sC <IP> -p5985,5986`; `evil-winrm -i <IP> -u <user> -p '<pass>'`.
- WMI (135/tcp then dynamic port): `wmiexec.py <domain>/<user>:'<pass>'@<IP> "<cmd>"`.
