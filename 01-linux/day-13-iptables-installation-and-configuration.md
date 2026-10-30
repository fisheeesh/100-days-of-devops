# Day 13: IPtables Installation And Configuration

## Question

We have one of our websites up and running on our `Nautilus` infrastructure in `Stratos DC`. Our
security team has raised a concern that right now Apache's port i.e `6400` is open for all since
there is no firewall installed on these hosts. So we have decided to add some security layer for
these hosts and after discussions and recommendations we have come up with the following
requirements:

1. Install `iptables` and all its dependencies on each app host.

2. Block incoming port `6400` on all apps for everyone except for LBR host.

3. Make sure the rules remain, even after system reboot.

## Solution

Repeat on `stapp01`, `stapp02`, `stapp03` (`tony`, `steve`, `banner`):

```bash
ssh tony@stapp01

sudo yum install -y iptables iptables-services

sudo systemctl enable --now iptables
sudo systemctl status iptables
```

Build the rules:

```bash
# start clean: the stock rules end in "REJECT all", which would block the LBR too
sudo iptables -F

# traffic the box sends to itself
sudo iptables -A INPUT -i lo -j ACCEPT

# replies to connections this box started
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# keep ssh open so you do not lock yourself out
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# the LBR may reach 6400
sudo iptables -A INPUT -p tcp -s stlb01 --dport 6400 -j ACCEPT

# everyone else on 6400 is dropped
sudo iptables -A INPUT -p tcp --dport 6400 -j DROP

sudo iptables -L -n --line-numbers
# num  target  prot opt source        destination
# 1    ACCEPT  all  --  0.0.0.0/0     0.0.0.0/0
# 2    ACCEPT  all  --  0.0.0.0/0     0.0.0.0/0    state RELATED,ESTABLISHED
# 3    ACCEPT  tcp  --  0.0.0.0/0     0.0.0.0/0    tcp dpt:22
# 4    ACCEPT  tcp  --  10.244.49.48  0.0.0.0/0    tcp dpt:6400
# 5    DROP    tcp  --  0.0.0.0/0     0.0.0.0/0    tcp dpt:6400
```

Make them survive a reboot:

```bash
sudo iptables-save | sudo tee /etc/sysconfig/iptables

# test it without rebooting: restart drops the live rules and reloads the file
sudo systemctl restart iptables
sudo iptables -L -n --line-numbers
# same five rules
```

Test both sides:

```bash
# from the jump host: blocked, hangs until the timeout
curl -m 5 http://stapp01:6400

# from the LBR: allowed
ssh loki@stlb01
curl http://stapp01:6400
```

## Why

| Command | Why |
|:--|:--|
| `iptables-services` | Ships `iptables.service`, which loads `/etc/sysconfig/iptables` at boot. The `iptables` package alone does not. |
| `iptables -F` | The stock rules end in `REJECT all`, which would block the LBR too. Flush and build exactly what the task wants. |
| `ACCEPT` the LBR **before** `DROP` | First match wins. `-A` appends, so the order you type is the order of the rules. |
| `iptables-save \| sudo tee` | `sudo iptables-save > file` runs the redirect as you, not root, and fails. |

## Notes

- With the INPUT policy at ACCEPT, rules 1 to 3 change nothing. They matter only if someone later sets `-P INPUT DROP`.
- `-s stlb01` resolves to an IP once, when the rule is added. The saved file stores the IP, so a changed LBR address leaves it stale.
- Persistence needs both halves: the saved file **and** the enabled service.
- Test it without rebooting: `systemctl restart iptables` drops the live rules and reloads the file.
