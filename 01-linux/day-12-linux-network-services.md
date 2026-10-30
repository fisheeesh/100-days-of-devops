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
| `telnet stapp01 6100` | Raw TCP, no HTTP. If telnet fails the problem is below Apache: service, port or firewall. |
| `netstat -tulnp` | Every listening socket and who owns it. `-t` TCP, `-u` UDP, `-l` listening, `-n` numeric, `-p` PID. |
| `stop` + `disable sendmail` | Two processes cannot hold one port. Without `disable` it restarts at boot and races httpd. |
| `iptables -I INPUT 4` | `-I` inserts at a position. `-A` appends below the REJECT where it is never reached. |

## Notes

- **"No route to host" was the firewall, not routing.** Rule 5 rejects with `icmp-host-prohibited` and the client reports that as `No route to host`.
- **Rule 3 is not "accept everything".** `-L -n` hides the interface column; `-v` shows `lo` under `in`. That is why curl worked on the box but not from the jump host.
- `iptables -I` is lost on reboot. `sudo service iptables save` writes `/etc/sysconfig/iptables`.
- `journalctl -u httpd` names the conflict directly: `(98)Address already in use`.
