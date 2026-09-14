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
| `iptables-services` | Ships `iptables.service`, the thing that loads `/etc/sysconfig/iptables` at boot. The `iptables` package alone gives you the command, but nothing that restores rules after a reboot. |
| `systemctl enable --now iptables` | `enable` is half of requirement 3. At boot the service loads the saved rules file. |
| `iptables -F` | The service starts with the stock rules, which end in `REJECT all`. That blocks 6400 for the LBR as well, plus every other port. Flush and build exactly what the task asks for. |
| `ACCEPT` the LBR **before** `DROP` | First match wins. The LBR's packets hit rule 4 and are accepted before rule 5 ever sees them. Swap the order and the LBR is blocked too. `-A` appends, so the order you type the commands is the order of the rules. |
| `-j DROP` | Silently discards the packet, so the client hangs until it times out. `REJECT` would refuse immediately. Both count as "block". |
| `iptables-save \| sudo tee` | Writes the live rules into the file the service loads at boot. `tee` because in `sudo iptables-save > file` the redirect runs as your user, not root, and fails with permission denied. |
| `systemctl restart iptables` | Tests persistence without a reboot. A restart throws away the live rules and reloads the file, so if the rules are still there, the save worked. |

## Notes

- With the INPUT policy at ACCEPT (check the `Chain INPUT (policy ACCEPT)` header), rules 1 to 3 change nothing: anything unmatched is accepted anyway. Only rules 4 and 5 do the work. Rules 1 to 3 still earn their place, because they stop you locking yourself out if someone later sets `-P INPUT DROP`.
- `-F` flushes rules but does not reset policies. If the policy were already DROP, flushing over ssh would cut your own session. Check the header before flushing a remote box.
- `-s stlb01` is resolved to an IP once, when the rule is added. That is why the listing shows `10.244.49.48`, and the saved file stores the IP, not the name. If the LBR's IP ever changes, the rule is stale.
- Rule 1 shows as `all 0.0.0.0/0` because `-L -n` hides the interface column. `-v` shows `lo` under `in`. Same trap as Day 12.
- `-m state` is the older name for `-m conntrack --ctstate`. Both work.
- Persistence needs both halves: the saved file **and** the enabled service. Either one alone and the rules are gone after a reboot.
- `firewalld` and `iptables.service` conflict, so starting one stops the other. If firewalld is on the box, `sudo systemctl disable --now firewalld` so it does not come back at boot.
- The task says each app host, so this is three boxes.
