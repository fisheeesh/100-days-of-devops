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
| `vi /etc/tomcat/server.xml` | The HTTP `Connector` port is what curl hits. The file also has `8005` (shutdown) and `8009` (AJP). |
| `systemctl enable --now tomcat` | Ships stopped and disabled. The port is only read at startup, so do this after the edit. |
| `mv ROOT.war .../webapps/` | The Host runs `autoDeploy="true"` and `unpackWARs="true"`, so it unpacks into `ROOT/` with no restart. |
| The name **ROOT** | A war named `ROOT.war` serves at `/`. Name it `app.war` and the base URL still 404s. |

## Notes

- `/usr/share/tomcat/webapps` is a symlink to `/var/lib/tomcat/webapps`. Either path works.
- If a `ROOT/` directory already exists from the stock install, remove it first or Tomcat keeps serving the old content.
- `systemctl status` says `active (running)` even when the app failed to deploy. Real errors are in `/var/log/tomcat/catalina.*.log`.
