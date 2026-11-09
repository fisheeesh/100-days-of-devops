# Day 68: Set Up Jenkins Server

## Question

The DevOps team at xFusionCorp Industries is initiating the setup of CI/CD pipelines and has
decided to utilize Jenkins as their server. Execute the task according to the provided
requirements:

1. Install `Jenkins` on the jenkins server using the `apt` utility only, and start it using the
   `service` command.

- If you face a timeout issue while starting the Jenkins service, first check the service status
  with `service jenkins status`
- Then review the logs in `/var/log/jenkins/jenkins.log` to identify the cause.

2. Jenkin's admin user name should be `theadmin`, password should be `Adm!n321`, full name should
   be `James` and email should be `james@jenkins.stratos.xfusioncorp.com`.

> `Note:`
>
> 1. To access the `jenkins` server, connect from the jump host using the `root` user with the
>    password `S3curePass`.
> 2. After Jenkins server installation, click the `Jenkins` button on the top bar to access the
>    Jenkins UI and follow on-screen instructions to create an admin user.

## Solution

```bash
# from the jump host, password: S3curePass
ssh root@jenkins

# confirm the distro family before reaching for apt
cat /etc/os-release

apt update

# Jenkins is a Java app and does not pull a JRE in itself
apt install -y openjdk-21-jre-headless
java -version

apt install -y jenkins

service jenkins start
service jenkins status
```

If the start times out, the reason is in the log, not in the status output:

```bash
service jenkins status
tail -50 /var/log/jenkins/jenkins.log
```

Then the admin user, which is UI work:

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

1. Click the **Jenkins** button on the top bar.
2. Paste that password to unlock.
3. Install the suggested plugins.
4. Create the first admin user exactly as the task specifies:
   - Username `theadmin`
   - Password `Adm!n321`
   - Full name `James`
   - Email `james@jenkins.stratos.xfusioncorp.com`

## Why

| Command | Why |
|:--|:--|
| `apt install openjdk-21-jre-headless` | Jenkins runs on the JVM and the package does not install one for you. Current Jenkins LTS wants Java 17 or 21. |
| `service jenkins start` | The task says `service`. These lab boxes often run without systemd, where `systemctl` fails with `System has not been booted with systemd`. |
| `tail /var/log/jenkins/jenkins.log` | `service jenkins status` reports that it timed out without saying why. The log names the actual cause, usually a missing or wrong Java. |
| `cat .../initialAdminPassword` | The one-time unlock token the setup wizard asks for. Readable by root only. |

## Notes

- Requirement 2 is entirely in the UI, and the email is part of it. The check reads all four fields, so do not skip `james@jenkins.stratos.xfusioncorp.com`.
- `-headless` is the JRE without GUI libraries. Smaller install, and all a server needs.
- If `apt install jenkins` reports the package is not found, the Jenkins apt repository is not configured on the box. Add the signing key and the `pkg.jenkins.io/debian-stable` source, then `apt update` again.
- Jenkins serves on port `8080` by default.
