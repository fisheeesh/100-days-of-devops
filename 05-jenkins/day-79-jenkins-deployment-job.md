# Day 79: Jenkins Deployment Job

## Question

The Nautilus development team had a meeting with the DevOps team where they discussed automating
the deployment of one of their apps using Jenkins (the one in `Stratos Datacenter`). They want to
auto deploy the new changes in case any developer pushes to the repository. As per the
requirements mentioned below configure the required Jenkins job.

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and `Adm!n321` password.

Similarly, you can access the `Gitea UI` using `Gitea` button. Username and password for Git are
`sarah` and `Sarah_pass123`. Under user `sarah` you will find a repository named `web` that is
already cloned on **App Server 1** under sarah's home (`/home/sarah/web`). `sarah` is a developer
who is working on this repository.

1. `httpd` is already installed and configured on the app server (listening on port `8080`).
   Ensure the `httpd` service is running on App Server 1 (e.g. start it manually if needed). You
   can make starting/restarting httpd part of your Jenkins job if you prefer.

2. Create a Jenkins job named `datacenter-app-deployment` and configure it so that if anyone
   pushes any new change to the origin repository in `master` branch, the job should auto build
   and deploy the latest code on **App Server 1** under `/var/www/html` directory.

   Before deployment, ensure that the ownership of the `/var/www/html` directory is set to user
   `sarah`, so that Jenkins can successfully deploy files to that directory.

3. SSH into **App Server 1** using `sarah` user credentials mentioned above. Under sarah user's
   home (`/home/sarah/web`) you will find a cloned Git repository named `web`. Under this
   repository there is an `index.html` file, update its content to
   `Welcome to the xFusionCorp Industries`, then push the changes to the `origin` into `master`
   branch. This push must trigger your Jenkins job and the latest changes must be deployed on the
   server, also make sure it deploys the entire repository content not only `index.html` file.

Click on the `App` button on the top bar to access the app. Please make sure the required content
is loading on the main URL (e.g. http://stlb01:8091) i.e there should not be any sub-directory
like http://stlb01:8091/web etc.

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

**1. Plugin and URL.** Manage Jenkins, then **Plugins**, install **Git**, restart. Then Manage
Jenkins, then **System**, set the Jenkins URL to `http://jenkins:8080`.

**2. Gitea credential.** Manage Jenkins, then **Credentials**, add **Username with password**: ID
and username `sarah`, password `Sarah_pass123`. This is what the job uses to reach the repo.

**3. Node.** Manage Jenkins, then **Nodes**, then **New Node**: name `App Server 1`, label
`stapp01`, launch method **Launch agent by connecting it to the controller**. Copy the secret.

**4. On App Server 1, fix ownership first:**

```bash
ssh tony@stapp01
sudo yum install -y java-17-openjdk screen
sudo systemctl enable --now httpd

ls -lah /var/www/html
# drwxr-xr-x 1 root  root   .      <- the directory itself is root's
# -rw-r--r-- 1 sarah sarah  index.html

sudo chown -R sarah:sarah /var/www/html
exit
```

**5. Start the agent as `sarah`:**

```bash
ssh sarah@stapp01

screen -S jk
curl -sO http://jenkins:8080/jnlpJars/agent.jar

java -jar agent.jar -url http://jenkins:8080/ \
  -secret <SECRET> -name "App Server 1" -webSocket \
  -workDir "/var/www/html"
# Ctrl-A then D to detach
```

**6. The job.** New Item, `datacenter-app-deployment`, **Freestyle project**:

- **Restrict where this project can be run**: `stapp01`
- **Source Code Management**: Git, the Gitea repo URL, credential `sarah`, branch `*/master`
- **Build Triggers**: tick **Poll SCM**, schedule `* * * * *`
- **Build Steps**, **Execute shell**:

```bash
cd /var/www/html
git pull
```

**7. Trigger it.** Still on App Server 1 as `sarah`:

```bash
cd ~/web
echo "Welcome to the xFusionCorp Industries" > index.html
git add index.html
git commit -m "First deployment"
git push
```

**8. Verify.** The job builds within a minute. Then:

```bash
cd /var/www/html
git log --oneline     # the new commit is here
cat index.html        # Welcome to the xFusionCorp Industries
```

## Why

| Piece | Why |
|:--|:--|
| Jenkins URL set correctly | The controller hands this address to agents as the place to register. Point it at the wrong host and an inbound agent has nowhere to call back to, so the node never comes online. |
| `chown -R sarah:sarah /var/www/html` | The files inside already belonged to sarah, but the **directory** was root's. The agent runs as sarah and needs to write in the directory, not just edit files in it. |
| **Poll SCM** `* * * * *` | Gitea is not set up to call Jenkins, so Jenkins asks Gitea every minute instead. A build only starts when the commit hash has actually changed. |
| **Restrict where this project can be run** | Without it the job runs on the controller, where `/var/www/html` is a different machine's directory entirely. |

## Notes

- Run the agent as `sarah`, not under sudo. Same lesson as Day 78: a root-owned `.git` breaks the validation that runs as sarah.
- `git pull` is fine while history stays linear. `git fetch origin` plus `git reset --hard origin/master` is the version that survives a diverged branch, which matters because note 2 says repeated builds must pass.
- The clone already sitting at `/var/www/html` is what keeps the site on `/` rather than `/web`. The job's own SCM checkout lands in the agent workspace and is not what Apache serves.
- httpd only needs to be running here. Its port is already configured at 8080, so requirement 1 is just `systemctl enable --now httpd`.
