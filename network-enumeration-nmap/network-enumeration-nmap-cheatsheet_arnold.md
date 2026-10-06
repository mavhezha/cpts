# Network Enumeration with Nmap Cheatsheet

## Core Philosophy
Enumeration is the most critical phase of any engagement. Tools simplify the process but cannot replace knowledge of how services work. Always look for two things: functions/resources that allow interaction with the target, and information that reveals additional attack paths. Manual enumeration remains essential since scanners can mismark ports and miss opportunities.

Always save every scan. Use `-oA <name>` to store results in all formats at once for later comparison and documentation.

---

## Nmap Syntax
```bash
nmap <scan types> <options> <target>
```

---

## Host Discovery

### Scan a network range (ping sweep, no port scan)
```bash
sudo nmap 10.129.2.0/24 -sn -oA tnet
sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5   # extract IPs only
```

### Scan from a host list
```bash
sudo nmap -sn -oA tnet -iL hosts.lst
```

### Scan multiple IPs or a range
```bash
sudo nmap -sn -oA tnet 10.129.2.18 10.129.2.19 10.129.2.20
sudo nmap -sn -oA tnet 10.129.2.18-20
```

### Single host, confirm alive with ICMP (bypass ARP)
```bash
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping
sudo nmap 10.129.2.18 -sn -oA host -PE --reason                    # show why host is marked up
```

---

## Port States
| State | Meaning |
|---|---|
| open | Connection established (TCP/UDP/SCTP) |
| closed | RST received from target |
| filtered | No response or error received, firewall likely dropping packets |
| unfiltered | Port accessible but open/closed undetermined (ACK scan only) |
| open\|filtered | No response, firewall or packet filter may be protecting it |
| closed\|filtered | Only in IP ID idle scans |

---

## Port Scanning

### Default SYN scan (requires root)
```bash
sudo nmap 10.129.2.28                          # top 1000 TCP ports, SYN scan
sudo nmap 10.129.2.28 --top-ports=10           # top 10 ports only
sudo nmap 10.129.2.28 -p-                      # all 65535 ports
sudo nmap 10.129.2.28 -p 22,80,443            # specific ports
sudo nmap 10.129.2.28 -p 22-445               # port range
sudo nmap 10.129.2.28 -F                       # fast scan, top 100 ports
```

### Full connect scan (TCP three-way handshake, no root needed)
```bash
nmap 10.129.2.28 -sT -p 443
```
More accurate but noisier than SYN scan; creates logs on target.

### UDP scan (slow, stateless)
```bash
sudo nmap 10.129.2.28 -sU -F
sudo nmap 10.129.2.28 -sU -p 137              # specific UDP port
```
No response = `open|filtered`. ICMP type 3 code 3 = closed.

### ACK scan (useful for firewall rule mapping)
```bash
sudo nmap 10.129.2.28 -sA -p 21,22,25
```
Returns `unfiltered` if RST received (firewall allows it), `filtered` if packet dropped or ICMP unreachable returned.

### Packet tracing and debugging
```bash
--packet-trace        # show every packet sent and received
--reason              # show why a port is in its state
-Pn                   # disable ICMP echo requests (treat host as up)
-n                    # disable DNS resolution
--disable-arp-ping    # disable ARP pings
```

---

## Service and Version Detection
```bash
sudo nmap 10.129.2.28 -p- -sV                 # version detection on all ports
sudo nmap 10.129.2.28 -p- -sV -v              # verbose (shows open ports as found)
sudo nmap 10.129.2.28 -p- -sV --stats-every=5s  # progress every 5 seconds
```
Press `Space Bar` during a scan to see live progress.

### Banner grabbing manually
```bash
nc -nv <IP> 25                                 # grab SMTP banner manually
sudo tcpdump -i eth0 host <attacker_IP> and <target_IP>   # capture alongside
```
Nmap's `-sV` may miss information visible in raw banners (e.g. OS in SMTP greeting).

---

## OS Detection
```bash
sudo nmap 10.129.2.28 -O                      # OS detection
sudo nmap 10.129.2.28 -A                      # aggressive: -sV + -O + traceroute + -sC
```
TTL clues from ping: ~128 = Windows, ~64 = Linux.

---

## Nmap Scripting Engine (NSE)

### Script categories
auth, broadcast, brute, default, discovery, dos, exploit, external, fuzzer, intrusive, malware, safe, version, vuln

### Running scripts
```bash
sudo nmap <target> -sC                                  # default scripts
sudo nmap <target> --script <category>                  # whole category
sudo nmap <target> --script <script-name>,<script-name> # specific scripts
sudo nmap <target> -p 25 --script banner,smtp-commands  # example
sudo nmap <target> -p 80 -sV --script vuln              # vuln scan on port 80
```

### Aggressive scan (combines everything)
```bash
sudo nmap 10.129.2.28 -p 80 -A
```
Returns: service version, OS guess, traceroute, and default script results.

---

## Saving Results
```bash
sudo nmap 10.129.2.28 -p- -oA target     # all formats (.nmap, .gnmap, .xml)
sudo nmap 10.129.2.28 -p- -oN target     # normal output only (.nmap)
sudo nmap 10.129.2.28 -p- -oG target     # grepable output only (.gnmap)
sudo nmap 10.129.2.28 -p- -oX target     # XML output only (.xml)
```

### Convert XML to HTML report
```bash
xsltproc target.xml -o target.html
```

---

## Performance Tuning

### Timing templates (0 = slowest/stealthiest, 5 = fastest/noisiest)
```bash
-T0    # paranoid
-T1    # sneaky
-T2    # polite
-T3    # normal (default)
-T4    # aggressive
-T5    # insane
```

### Manual performance options
```bash
--min-rate 300              # send at least 300 packets/sec
--max-retries 0             # no retries (faster but may miss ports)
--initial-rtt-timeout 50ms  # aggressive RTT start
--max-rtt-timeout 100ms     # cap RTT timeout
--min-parallelism <n>       # minimum parallel probes
```
Reducing retries and RTT timeout speeds things up but can cause missed hosts and ports. Use `-T4` or `--min-rate` for a good balance during white-box tests.

---

## Firewall and IDS/IPS Evasion

### Identify filtered vs rejected ports
- **Dropped** (filtered): no response, Nmap retries (takes ~2 seconds)
- **Rejected** (filtered): ICMP type 3 / code 3 returned quickly

### ACK scan to map firewall rules
```bash
sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```
Useful because ACK packets often pass through firewalls that block SYN packets.

### Decoys (hide real IP among spoofed sources)
```bash
sudo nmap 10.129.2.28 -p 80 -sS -D RND:5     # 5 random decoy IPs
sudo nmap 10.129.2.28 -p 80 -sS -D <decoy_IP>,<decoy_IP>,ME,<decoy_IP>
```
Decoys must be alive or SYN-flood protection may trigger. Spoofed packets can be filtered by ISPs/routers.

### Spoof source IP (test from allowed subnet)
```bash
sudo nmap 10.129.2.28 -p 445 -O -S 10.129.2.200 -e tun0 -Pn -n
```

### Use DNS source port to bypass firewall rules
```bash
sudo nmap 10.129.2.28 -p 50000 -sS -Pn -n --disable-arp-ping --source-port 53
ncat -nv --source-port 53 10.129.2.28 50000   # verify with manual connection
```
Firewalls often trust DNS traffic on TCP/UDP 53; IDS/IPS may filter these less strictly too.

### DNS proxying (route queries through trusted internal DNS)
```bash
sudo nmap 10.129.2.28 --dns-server <internal_dns_IP>
```

---

## Common Scan Combinations (Reference)

### Standard pentest starting scan
```bash
nmap -Pn -sV -sC -p- --min-rate 5000 -oA scan <IP>
```

### Quick top 1000 check
```bash
nmap -Pn -sV -sC --min-rate 5000 -oA quick_scan <IP>
```

### Stealth host discovery (avoid noisy port scan)
```bash
sudo nmap 10.129.2.0/24 -sn -PE --disable-arp-ping -oA hosts
```

### Full verbose scan with version detection
```bash
sudo nmap 10.129.2.28 -p- -sV -v --stats-every=5s -oA full
```

### Vuln scan on specific port
```bash
sudo nmap 10.129.2.28 -p 80 -sV --script vuln
```

### OS detection with spoofed source
```bash
sudo nmap 10.129.2.28 -p 445 -O -S 10.129.2.200 -e tun0 -Pn -n
```
