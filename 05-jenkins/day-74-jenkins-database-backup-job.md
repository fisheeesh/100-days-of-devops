# Day 74: Jenkins Database Backup Job

## Question

There is a requirement to create a Jenkins job to automate the database backup. Below you can find
more details to accomplish this task:

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and password `Adm!n321`.

1. Create a Jenkins job named `database-backup`.
2. Configure it to take a database dump of the `kodekloud_db01` database present on the
   **App server (stapp01)** in Stratos Datacenter, the database user is `kodekloud_roy` and
   password is `asdfgdsd`.
3. The dump should be named in `db_$(date +%F).sql` format, where `date +%F` is the current date.
4. Copy the `db_$(date +%F).sql` dump to the **Storage server (ststor01)** under location
   `/home/natasha/db_backups`.
5. Further, schedule this job to run periodically at `/10 * * * *` (please use this exact schedule
   format).

> `Note:`
>
> 1. You might need to install some plugins and restart Jenkins service. So, we recommend clicking
>    on `Restart Jenkins when installation is complete and no jobs are running` on plugin
>    installation/update page i.e `update centre`. Also, Jenkins UI sometimes gets stuck when
>    Jenkins service restarts in the back end. In this case please make sure to refresh the UI
>    page.
> 2. Please make sure to define you cron expression like this `/10 * * * *` (this is just an
>    example to run job every 10 minutes).
> 3. For these kind of scenarios requiring changes to be done in a web UI, please take screenshots
>    so that you can share it with us for review in case your task is marked incomplete. You may
>    also consider using a screen recording software such as loom.com to record and share your
>    work.

## Solution

**1. Plugin.** Manage Jenkins, then **Plugins**, install **SSH Agent** rather than the older SSH
plugin, then restart.

**2. Keys from the Jenkins box to both servers:**

```bash
ssh-keygen

ssh-copy-id tony@stapp01
ssh tony@stapp01          # should land with no password

ssh-copy-id natasha@ststor01
ssh natasha@ststor01

cat ~/.ssh/id_ed25519     # paste this into the credential
```

**3. Credential.** Manage Jenkins, then **Credentials**, add **SSH Username with private key**:
ID `db-backup`, username `tony`, paste the key.

**4. The job.** New Item, name `database-backup`, **Freestyle project**:

- **This project is parameterised** (optional, keeps credentials out of the shell step):
  - String `DB_NAME`, default `kodekloud_db01`
  - String `DB_USER`, default `kodekloud_roy`
  - Password `DB_PASS`, default `asdfgdsd`
- Build Triggers, **Build periodically**: `*/10 * * * *`
- Build Environment, tick **SSH Agent**, pick the `db-backup` credential.
- Build Steps, **Execute shell**:

```bash
FILE_PATH="/tmp/db_$(date +%F).sql"

ssh -o StrictHostKeyChecking=no tony@stapp01 \
    "mysqldump -u ${DB_USER} -p${DB_PASS} ${DB_NAME} > ${FILE_PATH}"

scp -o StrictHostKeyChecking=no \
    tony@stapp01:${FILE_PATH} natasha@ststor01:/home/natasha/db_backups/
```

**5. Build**, check Console Output for `Finished: SUCCESS`, then verify on both servers:

```bash
ssh tony@stapp01 'ls -lah /tmp/db_*.sql'
ssh natasha@ststor01 'ls -lah /home/natasha/db_backups'
```

## Why

| Piece | Why |
|:--|:--|
| SSH Agent instead of the SSH plugin | The SSH plugin runs one command on one host. Here a single shell step has to reach two servers, so the key needs to be in an agent that every command in the step can use. |
| The `>` **inside** the quotes | `ssh host "mysqldump ... > /tmp/x.sql"` writes the dump on stapp01. Move the redirect outside the quotes and the file lands on the Jenkins node instead. |
| `-o StrictHostKeyChecking=no` | A first connection asks you to accept the host key, and a build has no terminal to answer with. |
| `ssh-copy-id` run **twice** | Modern scp routes remote-to-remote copies through the local host by default, so the Jenkins node opens connections to both servers itself. |

## Notes

- Requirement 5 writes `/10 * * * *`, but Jenkins' cron parser wants `*/10 * * * *`. Use the latter if Jenkins rejects the literal string.
- Drop the `-t` from the ssh command. A build has no TTY, so ssh just prints `Pseudo-terminal will not be allocated` and the flag buys nothing.
- `-p` takes no space: `-p${DB_PASS}`. Written as `-p ${DB_PASS}`, mysqldump reads the next word as the database name instead.
- On older OpenSSH (8.x, as on RHEL 9) remote-to-remote scp went **direct** instead, which would need a key from stapp01 to ststor01. `scp -3` forces the route-through-local behaviour if you ever hit that.
- A timer-triggered run of a parameterised job uses the default values, which is why setting sensible defaults matters here.
