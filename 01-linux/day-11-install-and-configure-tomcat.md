# Day 11: Install and Configure Tomcat Server

## Question

The `Nautilus` application development team recently finished the beta version of one of their
Java-based applications, which they are planning to deploy on one of the app servers in
`Stratos DC`. After an internal team meeting, they have decided to use the `tomcat` application
server. Based on the requirements mentioned below complete the task:

a. Install `tomcat` server on `App Server 3`.

b. Configure it to run on port `8086`.

c. There is a `ROOT.war` file on `Jump host` at location `/tmp`. Deploy it on this tomcat server
and make sure the webpage works directly on base URL i.e `curl http://stapp03:8086`

## Solution

Get the war onto the right box first, from the jump host:

```bash
ls -lah /tmp/ROOT.war
scp /tmp/ROOT.war banner@stapp03:~/
```

Install and set the port:

```bash
ssh banner@stapp03

sudo yum install -y tomcat

sudo vi /etc/tomcat/server.xml
# search /8080 and change the HTTP Connector only:
#   <Connector port="8086" protocol="HTTP/1.1" ... />

sudo systemctl status tomcat
# disabled, inactive (dead)

sudo systemctl enable --now tomcat

curl localhost:8086
```

Deploy:

```bash
ls -lah /usr/share/tomcat
# webapps -> /var/lib/tomcat/webapps

sudo mv ~/ROOT.war /usr/share/tomcat/webapps/

ls -lah /usr/share/tomcat/webapps/
# ROOT.war   and   ROOT/     <- tomcat exploded it on its own

curl localhost:8086
curl http://stapp03:8086
```

## Why

| Command | Why |
|:--|:--|
| `scp /tmp/ROOT.war banner@stapp03:~/` | The war sits on the jump host, tomcat runs on stapp03. Home directory first, because `banner` cannot scp straight into a root-owned path. |
| `vi /etc/tomcat/server.xml` | The HTTP `Connector` port is what `curl` hits. That file also has `8005` (shutdown) and `8009` (AJP). Changing those does nothing for this task and breaks other things. |
| `systemctl enable --now tomcat` | It ships stopped and disabled. `--now` starts it and enables it for boot in one step. Do this after the edit, since the port is only read at startup. |
| `mv ROOT.war .../webapps/` | Tomcat's `Host` runs with `autoDeploy="true"` and `unpackWARs="true"`, so it spots the new war and unpacks it into `ROOT/` without a restart. |
| The name **ROOT** | This is the whole point of part c. A war deployed as `ROOT.war` serves at `/`. Name it `app.war` and it serves at `/app`, so the base URL would still 404. |

## Notes

- `/usr/share/tomcat/webapps` is a symlink to `/var/lib/tomcat/webapps`. RHEL keeps static files under `/usr/share` and changing data under `/var/lib`. Either path works.
- If a `ROOT/` directory is already there from the stock install, remove it and the old `ROOT.war` before dropping yours in, otherwise Tomcat can keep serving the old content.
- `mv` as root leaves the war owned by `root:root`. Mode 644 is world-readable so Tomcat can still read it, but `sudo chown tomcat:tomcat` on the war is tidier.
- If the page does not appear, read `/var/log/tomcat/catalina.*.log`. Deployment failures land there, not in `systemctl status`, which will happily say `active (running)` while the app failed to deploy.
- `curl localhost:8086` proves Tomcat is up. `curl http://stapp03:8086` from the jump host is what the task actually checks, because it also proves nothing is blocking the port from outside.
