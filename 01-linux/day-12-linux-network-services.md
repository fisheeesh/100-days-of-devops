# Day 12: Linux Network Services

## Question

Our monitoring tool has reported an issue in `Stratos Datacenter`. One of our app servers has an
issue, as its Apache service is not reachable on port `6100` (which is the Apache port). The
service itself could be down, the firewall could be at fault, or something else could be causing
the issue.

Use tools like `telnet`, `netstat`, etc. to find and fix the issue. Also make sure Apache is
reachable from the jump host without compromising any security settings.

Once fixed, you can test the same using command `curl http://stapp01:6100` command from jump
host.

> **Note:** Please do not try to alter the existing `index.html` code, as it will lead to task
> failure.

## Solution

Reproduce from the jump host, where the task tests it:

```bash
curl http://stapp01:6100
# curl: (7) Failed to connect to stapp01 port 6100: No route to host

telnet stapp01 6100
# telnet: connect to address 10.244.247.223: No route to host
```

**Problem 1: Apache is not running.**

```bash
ssh tony@stapp01

# Apache is packaged as httpd on CentOS
sudo systemctl status httpd
# disabled, failed

sudo yum install -y net-tools
sudo netstat -tulnp | grep 6100
# tcp  0  0 127.0.0.1:6100  0.0.0.0:*  LISTEN  21838/sendmail: acc
# sendmail already holds 6100, so httpd cannot bind to it

sudo systemctl stop sendmail
sudo systemctl disable sendmail

sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
# active (running), enabled

sudo netstat -tulnp | grep 6100
# tcp6  0  0 :::6100  :::*  LISTEN  51468/httpd

curl http://stapp01:6100
# page loads, but only from stapp01 itself
```

**Problem 2: the firewall blocks it from outside.**

```bash
sudo iptables -L -n --line-numbers
# Chain INPUT (policy ACCEPT)
# num  target  prot opt source     destination
# 1    ACCEPT  all  --  0.0.0.0/0  0.0.0.0/0   state RELATED,ESTABLISHED
# 2    ACCEPT  icmp --  0.0.0.0/0  0.0.0.0/0
# 3    ACCEPT  all  --  0.0.0.0/0  0.0.0.0/0
# 4    ACCEPT  tcp  --  0.0.0.0/0  0.0.0.0/0   state NEW tcp dpt:22
# 5    REJECT  all  --  0.0.0.0/0  0.0.0.0/0   reject-with icmp-host-prohibited
# only ssh gets in, everything else hits rule 5

# insert ABOVE the reject rule
sudo iptables -I INPUT 4 -p tcp --dport 6100 -j ACCEPT

sudo iptables -L -n --line-numbers
# 4    ACCEPT  tcp  --  0.0.0.0/0  0.0.0.0/0   tcp dpt:6100
# ...
# 6    REJECT  all  --  0.0.0.0/0  0.0.0.0/0   reject-with icmp-host-prohibited
```

Back on the jump host:

```bash
curl http://stapp01:6100
telnet stapp01 6100
```

## Why

| Command | Why |
|:--|:--|
| `telnet stapp01 6100` | Tests the raw TCP connection with no HTTP involved. If telnet fails, the problem sits below Apache: the service, the port, or the firewall. |
| `systemctl status httpd` | `failed` is not the same as `inactive`. Something tried to start it and it crashed, so find out why before just starting it again. |
| `netstat -tulnp` | Lists every listening socket and which process owns it. `-t` TCP, `-u` UDP, `-l` listening only, `-n` numeric (no DNS or service-name lookups), `-p` owning PID and program. `-p` needs sudo to see other users' processes. |
| `stop` + `disable sendmail` | Two processes cannot listen on the same port. `disable` matters too: otherwise sendmail starts again at the next boot and races httpd for 6100. Whichever binds first wins, so you could end up back where you started. |
| `start` + `enable httpd` | It was `failed` and `disabled`. Starting fixes it now, enabling makes it survive a reboot. |
| `iptables -L -n --line-numbers` | Rules are checked top to bottom and the first match wins. You need the numbers to insert in the right spot. |
| `iptables -I INPUT 4` | `-I` inserts at a position. `-A` appends to the end, below the REJECT, where the rule never gets reached. `-A` also does not take a position, so `-A INPUT 4` is a syntax error. |
| `--dport 6100` only | "Without compromising security" means opening the one port. Flushing the rules or deleting the REJECT would also make curl work, and would defeat the point of the task. |

## Notes

- **"No route to host" was not a routing problem.** Rule 5 rejects with `icmp-host-prohibited`, and the client reports that ICMP reply as `No route to host`. The real cause was the firewall. For comparison, `Connection refused` usually means the host answered but nothing is listening, and a hang or timeout usually means a DROP rule.
- **Rule 3 looks like "accept everything" but is not.** `-L -n` hides the interface columns. `sudo iptables -L -n -v` adds `in` and `out` columns, and rule 3 shows `lo` under `in`, so it matches loopback only. That is why curl worked on stapp01 but failed from the jump host.
- **`iptables -I` does not survive a reboot**, or by default even `systemctl restart iptables`, which reloads the saved file. Save it with `sudo service iptables save`, which writes `/etc/sysconfig/iptables`.
- httpd's own log names the port conflict directly: `sudo journalctl -u httpd` shows `(98)Address already in use: AH00072: make_sock: could not bind to address [::]:6100`. netstat then tells you who holds it.
- sendmail only bound `127.0.0.1:6100`, while httpd wanted `[::]:6100` (every address, IPv4 included). They overlap, so the bind still fails.
- `ss -tulnp` does the same job as netstat and is installed by default. netstat comes from `net-tools`, which is deprecated.
- `systemctl disable --now sendmail` and `enable --now httpd` do each pair in one command.
