# Day 14: Linux Process Troubleshooting

## Question

The production support team of `xFusionCorp Industries` has deployed some of the latest monitoring tools to keep an eye on every service, application, etc. running on the systems. One of the monitoring systems reported about Apache service unavailability on one of the app servers in `Stratos DC`.

Identify the faulty app host and fix the issue. Make sure Apache service is up and running on all app hosts. They might not have hosted any code yet on these servers, so you don't need to worry if Apache isn't serving any pages. Just make sure the service is up and running. Also, make sure Apache is running on port `3003` on all app servers.

## Solution

### 1. Identify the Faulty Host

From the jump host, probe port `3003` across all three app servers:

```bash
curl -m 5 http://stapp01:3003
curl -m 5 http://stapp02:3003
curl -m 5 http://stapp03:3003
```

`stapp02` and `stapp03` respond properly. `stapp01` drops/refuses the connection.

---

### 2. Troubleshoot and Fix `stapp01`

Log in to the failing host:

```bash
ssh tony@stapp01
```

Check the status of the Apache service:

```bash
sudo systemctl status httpd
# Active: inactive (dead) / failed
```

Attempt to start it and inspect errors:

```bash
sudo systemctl start httpd
# Job for httpd.service failed because the control process exited with error code.
# See "systemctl status httpd.service" and "journalctl -xeu httpd.service" for details.

sudo journalctl -xeu httpd.service --no-pager
# (98)Address already in use: AH00072: make_sock: could not bind to address [::]:3003
# (98)Address already in use: AH00072: make_sock: could not bind to address 0.0.0.0:3003
# no listening sockets available, shutting down
```

Identify which process is holding port `3003`:

```bash
sudo ss -tulnp | grep 3003
# tcp  LISTEN  0  128  0.0.0.0:3003  0.0.0.0:* users:(("sendmail",pid=1234,fd=4))
```

Free the port and start Apache:

```bash
# stop the rogue process and prevent it from starting on boot
sudo systemctl stop sendmail
sudo systemctl disable sendmail

# enable and start apache
sudo systemctl enable --now httpd
sudo systemctl status httpd
# Active: active (running)
```

Verify the port binding locally and from the jump host:

```bash
# locally on stapp01
sudo ss -tulnp | grep 3003
# tcp  LISTEN  0  511  *:3003  *:* users:(("httpd",pid=5678,fd=4))

curl -I http://localhost:3003

# exit back to jump host and verify externally
exit
curl -I http://stapp01:3003
```

## Why

| Command | Why |
|:--|:--|
| `curl -m 5` | External triage. Identifies the broken host instantly without manually logging into all three nodes. |
| `journalctl -xeu httpd.service` | Displays targeted systemd unit logs. Surfaces `(98)Address already in use`, pinpointing a port binding collision. |
| `ss -tulnp \| grep 3003` | Lists listening TCP/UDP sockets with process names and PIDs (`-p`). Discovers `sendmail` holding port `3003`. |
| `systemctl disable sendmail` | Stopping the service releases the port immediately; disabling prevents it from reclaiming `3003` across reboots. |
| `systemctl enable --now httpd` | Starts Apache immediately and registers it to boot targets in one command. |

## Notes

- `Address already in use` (`EADDRINUSE`, errno 98) is a standard socket binding conflict. Always check listening sockets before assuming corrupted configurations or broken binaries.
- `ss` flags breakdown:
  - `-t`: TCP sockets
  - `-u`: UDP sockets
  - `-l`: Listening sockets only
  - `-n`: Numeric IP/ports (prevents slow DNS resolution)
  - `-p`: Associated process and PID (requires root privileges)
- If a rogue process is not managed by systemd, retrieve its PID using `sudo ss -tulnp` or `sudo lsof -i :3003` and terminate it directly with `sudo kill -15 <PID>` (or `sudo kill -9 <PID>` if unresponsive).