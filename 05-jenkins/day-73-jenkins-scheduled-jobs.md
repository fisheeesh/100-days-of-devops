# Day 73: Jenkins Scheduled Jobs

## Question

The devops team of xFusionCorp Industries is working on to setup centralised logging management
system to maintain and analyse server logs easily. Since it will take some time to implement, they
wanted to gather some server logs on a regular basis. At least one of the app servers is having
issues with the Apache server. The team needs Apache logs so that they can identify and
troubleshoot the issues easily if they arise. So they decided to create a Jenkins job to collect
logs from the server. Please create/configure a Jenkins job as per details mentioned below:

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and password `Adm!n321`

1. Create a Jenkins jobs named `copy-logs`.

2. Configure it to periodically build `every 2 minutes` to copy the Apache logs (both `access_log`
   and `error_log`) from **App Server 1** (stapp01) from the default logs location to location
   `/usr/src/dba` on the Storage Server.

3. Build the job at least once so that the logs are copied and can be verified.

> `Note:`
>
> 1. You might need to install some plugins and restart Jenkins. We recommend selecting
>    `Restart Jenkins when installation is complete and no jobs are running` in the update centre.
>    Refresh the page if the UI gets stuck after a restart.
> 2. Define the cron expression as required (e.g. `*/10 * * * *` to run every 10 minutes).
> 3. For scenarios that require web UI changes, take screenshots or record your work (e.g. using
>    loom.com) so you can share it for review if the task is marked incomplete.

## Solution

There are **two** SSH hops here: Jenkins reaches stapp01, then stapp01 reaches ststor01. Each one
needs its own key.

**1. SSH plugin.** Manage Jenkins, then **Plugins**, then **Installed plugins**, check for `SSH`.
Install it from **Available plugins** and restart if it is missing.

**2. Jenkins to stapp01.** On the Jenkins box:

```bash
ssh-keygen
ssh-copy-id tony@stapp01
cat ~/.ssh/id_ed25519
```

- Manage Jenkins, then **Credentials**, add **SSH Username with private key**: ID `stapp01`,
  username `tony`, paste that private key.
- Manage Jenkins, then **System**, **SSH remote hosts**, add hostname `stapp01` with the `tony`
  credential.

**3. stapp01 to ststor01.** On stapp01 itself, as `tony`:

```bash
ssh-keygen
ssh-copy-id natasha@ststor01

# prove the copy works by hand before asking Jenkins to do it
scp /var/log/httpd/access_log /var/log/httpd/error_log natasha@ststor01:/usr/src/dba
# access_log   100%   41   120.2KB/s   00:00
# error_log    100%  733     2.4MB/s   00:00
```

**4. The job.** New Item, name `copy-logs`, **Freestyle project**:

- Build Triggers, tick **Build periodically**, schedule:

  ```
  */2 * * * *
  ```
- Build Steps, **Execute shell script on remote host using ssh**, site `stapp01`, command:

  ```bash
  scp /var/log/httpd/access_log /var/log/httpd/error_log natasha@ststor01:/usr/src/dba
  ```

**5. Build now**, then check Console Output for `Finished: SUCCESS`, and confirm on the storage
server:

```bash
ls -l /usr/src/dba
```

## Why

| Piece | Why |
|:--|:--|
| Two key setups, not one | Jenkins authenticates to stapp01, and then stapp01 authenticates to ststor01 as a separate connection. Setting up only the first leaves the scp prompting for a password with no terminal to answer it. |
| `*/2 * * * *` | Requirement 2 says every 2 minutes. The `*/10` in note 2 is only showing the format. |
| Running the scp by hand first | Separates "the copy works" from "Jenkins can run it". If the manual run fails, the job was never the problem. |
| Build once manually | A schedule proves nothing until a run has actually moved the files. Requirement 3 asks for exactly this. |

## Notes

- Jenkins warns "Do you really mean every 2 minutes?" and suggests `H/2 * * * *`. The `H` form spreads load across jobs, but the task asks for every 2 minutes, so `*/2 * * * *` is right and the warning is advisory.
- Apache logs on CentOS live in `/var/log/httpd/`, named `access_log` and `error_log` with no extension. Debian uses `/var/log/apache2/` and `.log` names.
- If the scp comes back `Permission denied`, `/var/log/httpd` is root-only on that box. Copy the files out with sudo first, then scp from there.
- The destination is `/usr/src/dba`. `/usr/src/sysops` belongs to a different variant of this task, and the check reads the path in the requirement.
