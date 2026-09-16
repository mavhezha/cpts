# Privilege Escalation Checklist (CPTS Prep)

## General Approach
- [ ] Confirm current user context and privilege level (`whoami`, `id`)
- [ ] Run thorough enumeration before attempting any exploit
- [ ] Reference HackTricks and PayloadsAllTheThings checklists for both Linux and Windows
- [ ] Keep in mind that automated scripts generate noise and may trigger AV or monitoring tools; consider manual enumeration when stealth matters

## Enumeration Scripts
- [ ] Linux: LinEnum, linuxprivchecker, LinPEAS
- [ ] Windows: Seatbelt, JAWS, WinPEAS
- [ ] PEASS suite covers both Linux and Windows and stays actively maintained

## Kernel Exploits
- [ ] Identify OS and kernel version
- [ ] Check for known CVEs against that version (searchsploit, Google, exploit-db)
- [ ] Confirm the box is unpatched before assuming a kernel exploit will work
- [ ] Test kernel exploits in a lab environment first; kernel exploits can crash production systems
- [ ] Get explicit client approval before running kernel exploits on production systems

## Vulnerable Software
- [ ] List installed software (`dpkg -l` on Linux, `C:\Program Files` on Windows)
- [ ] Check installed versions against public exploit databases
- [ ] Pay special attention to outdated or unpatched software

## User Privileges

### Sudo (Linux)
- [ ] Run `sudo -l` to check current sudo privileges
- [ ] If `(ALL : ALL) ALL` is present, escalate directly with `sudo su -`
- [ ] Check for NOPASSWD entries on specific binaries
- [ ] Use `sudo -u user /bin/echo ...` style syntax if privilege is scoped to a specific user, not root
- [ ] Cross reference any allowed binary against GTFOBins for a known escalation path

### SUID (Linux)
- [ ] Search for SUID binaries (`find / -perm -4000 2>/dev/null`)
- [ ] Cross reference discovered binaries against GTFOBins

### Windows Token Privileges
- [ ] Enumerate token privileges (`whoami /priv`)
- [ ] Cross reference against LOLBAS for exploitation techniques

## Scheduled Tasks / Cron Jobs
- [ ] Check for write access to cron related paths:
  - [ ] `/etc/crontab`
  - [ ] `/etc/cron.d`
  - [ ] `/var/spool/cron/crontabs/root`
- [ ] Check for Windows scheduled tasks that run as a higher privileged user
- [ ] Look for opportunities to add a new cron job or scheduled task
- [ ] Look for opportunities to trick an existing task into executing malicious code
- [ ] If write access to a script called by a cron job is found, insert a reverse shell payload

## Exposed Credentials
- [ ] Search configuration files for hardcoded credentials (e.g. `config.php`, `web.config`)
- [ ] Search log files for leaked passwords
- [ ] Check user history files:
  - [ ] `.bash_history` on Linux
  - [ ] PSReadLine history on Windows
- [ ] Test password reuse against other local users (`su`) and services (SSH, databases)

## SSH Keys
- [ ] Check for read access to `.ssh` directories, particularly `/root/.ssh/id_rsa` or `/home/user/.ssh/id_rsa`
- [ ] If a private key is readable, copy it locally and set correct permissions before use:
  - [ ] `chmod 600 id_rsa`
  - [ ] `ssh user@target -i id_rsa`
- [ ] If write access to a user's `.ssh` directory is found, plant a public key:
  - [ ] Generate a new key pair with `ssh-keygen -f key`
  - [ ] Append the public key to `authorized_keys` on the target
  - [ ] Log in using the matching private key with `ssh user@target -i key`
- [ ] Remember this only works once you already control that user account; the SSH server will not accept keys written by another user

## Notes
- Deep dives on each of these techniques live in the Linux Privilege Escalation and Windows Privilege Escalation modules.
- Treat this as a living document; add new techniques and tool notes as you work through more machines.
