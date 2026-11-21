# Day 80: Jenkins Chained Builds

## Question

The DevOps team was looking for a solution where they want to restart Apache service on all app
servers if the deployment goes fine on these servers in Stratos Datacenter. After having a
discussion, they came up with a solution to use Jenkins chained builds so that they can use a
downstream job for services which should only be triggered by the deployment job. So as per the
requirements mentioned below configure the required Jenkins jobs.

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and `Adm!n321` password.

Similarly you can access `Gitea UI` on port `3000` (or click the `Gitea` button) and username and
password for Git is `sarah` and `Sarah_pass123` respectively. Under user `sarah` you will find a
repository named `web`.

Apache is already installed and configured on the app server. The doc root `/var/www/html` on
**App Server 1** is a local git repository tracking the origin `web` repository.

1. Create a Jenkins job named `devops-app-deployment` and configure it to pull changes from the
   `master` branch of the `web` repository on **App Server 1** under `/var/www/html` directory.

2. Create another Jenkins job named `manage-services` and make it a `downstream job` for
   `devops-app-deployment`. Things to take care about this job are:

   a. This job should restart `httpd` service on the app server (App Server 1).

   b. Trigger this job only if the `upstream job` i.e `devops-app-deployment` is stable.

The LB server is already configured. Click on the `App` button on the top bar to access the app.
Please make sure the required content is loading on the main URL (e.g. http://stlb01:8091) i.e
there should not be a sub-directory like http://stlb01:8091/web etc.

> `Note:`
>
> 1. You might need to install some plugins and restart Jenkins service. So, we recommend clicking
>    on `Restart Jenkins when installation is complete and no jobs are running` on plugin
>    installation/update page i.e `update centre`. Also some times Jenkins UI gets stuck when
>    Jenkins service restarts in the back end so in such case please make sure to refresh the UI
>    page.
> 2. Make sure Jenkins job passes even on repetitive runs as validation may try to build the job
>    multiple times.
> 3. Deployment related tasks should be done by `sudo` user on the destination server to avoid any
>    permission issues so make sure to configure your Jenkins job accordingly.
> 4. For these kind of scenarios requiring changes to be done in a web UI, please take screenshots
>    so that you can share it with us for review in case your task is marked incomplete. You may
>    also consider using a screen recording software such as loom.com to record and share your
>    work.

## Solution

**1. Key and plugin.** On the Jenkins box:

```bash
ssh-keygen
ssh-copy-id tony@stapp01
ssh tony@stapp01          # should land with no password
```

Manage Jenkins, then **Plugins**, install **SSH Agent**, then restart.

**2. Check the repo on the app server:**

```bash
ssh tony@stapp01
cd /var/www/html
git status
# if it complains about ownership:
git config --global --add safe.directory /var/www/html
git remote -v
exit
```

**3. Credential.** `cat ~/.ssh/id_ed25519`, then Manage Jenkins, then **Credentials**, add
**SSH Username with private key**: ID and username `tony`, paste the key.

**4. Upstream job.** New Item, `devops-app-deployment`, **Freestyle project**:

- Build Environment, tick **SSH Agent**, pick the `tony` credential.
- Build Steps, **Execute shell**:

```bash
ssh -o StrictHostKeyChecking=no tony@stapp01 \
  "cd /var/www/html && sudo git -c safe.directory=/var/www/html checkout master && sudo git -c safe.directory=/var/www/html pull origin master"
```

**5. Downstream job.** New Item, `manage-services`, **Freestyle project**:

- Build Environment, tick **SSH Agent**, pick `tony`.
- Build Triggers, tick **Build after other projects are built**, project to watch
  `devops-app-deployment`, and select **Trigger only if build is stable**.
- Build Steps, **Execute shell**:

```bash
ssh -o StrictHostKeyChecking=no tony@stapp01 "sudo systemctl restart httpd"
```

**6. Link it from the upstream side too.** Back in `devops-app-deployment`, **Configure**, then
**Post-build Actions**, **Build other projects**: `manage-services`, **Trigger only if build is
stable**.

**7. Build** `devops-app-deployment`. Both jobs should go green, the downstream one starting on
its own.

## Why

| Piece | Why |
|:--|:--|
| **Trigger only if build is stable** | The whole point of the chain. A failed deployment must not restart Apache on top of a half-updated document root. |
| `sudo git -c safe.directory=...` | `-c` sets the config for that one command, which reaches the `sudo` process. `git config --global` as tony would not, because sudo runs as root. |
| `-o StrictHostKeyChecking=no` | A build has no terminal to accept a host key with on first connection. |
| Both jobs restricted to the same credential | Each job opens its own SSH connection to stapp01, so each needs the key in its own build environment. |

## Notes

- You can set the chain from either end. **Post-build Actions, Build other projects** on the upstream, or **Build after other projects are built** on the downstream. Setting one fills in the other, so doing both is harmless but not required.
- `checkout master` before the pull makes repeated runs safe. The doc root may be left on another branch, and a bare `git pull` would then update the wrong one, which is what note 2 is checking.
- The deployment runs against the clone already at `/var/www/html`, which keeps the site on `/` rather than `/web`.
- "Stable" means the previous build succeeded. A build marked unstable (tests failed but did not error) does **not** trigger the downstream job.
