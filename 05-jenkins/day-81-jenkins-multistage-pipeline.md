# Day 81: Jenkins Multistage Pipeline

## Question

The development team of xFusionCorp Industries is working on to develop a new static website and
they are planning to deploy the same on Nautilus App Server using Jenkins pipeline. They have
shared their requirements with the DevOps team and accordingly we need to create a Jenkins
pipeline job. Please find below more details about the task:

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and password `Adm!n321`.

Similarly, click on the `Gitea` button on the top bar to access the Gitea UI. Login using username
`sarah` and password `Sarah_pass123`.

There is a repository named `sarah/web` in Gitea that is already cloned on **App Server 1** under
`/var/www/html` directory.

1. Update the content of the file `index.html` under the same repository to
   `Welcome to xFusionCorp Industries` and push the changes to the origin into the `master`
   branch.
2. Apache is already installed on the app server and is running on port `8080`.
3. Add **App Server 1** as a Jenkins agent (slave) node: name `App Server 1`, label `stapp01`,
   remote root directory `/home/sarah/jenkins_agent`, launch via SSH with host `stapp01` and
   credentials for user `sarah`. Install `java-17-openjdk` on App Server 1 if needed.
4. Create a Jenkins pipeline job named `deploy-job` (it must not be a `Multibranch pipeline` job)
   and pipeline should have two stages `Deploy` and `Test` ( names are case sensitive ). Configure
   these stages as per details mentioned below.

   a. The `Deploy` stage should deploy the code from `web` repository under `/var/www/html` on
   **App Server 1**, as this is the document root of the app server.

   b. The pipeline should run on the **App Server 1** node (e.g. use label `stapp01`).

   c. The `Test` stage should just test if the app is working fine and website is accessible. Its
   up to you how you design this stage to test it out, you can simply add a `curl` command as well
   to run a curl against the LBR URL (`http://stlb01:8091`) to see if the website is working or
   not. Make sure this stage fails in case the website/app is not working or if the `Deploy` stage
   fails.

Click on the `App` button on the top bar to see the latest changes you deployed. Please make sure
the required content is loading on the main URL `http://stlb01:8091` i.e there should not be a
sub-directory like `http://stlb01:8091/web` etc.

> `Note:`
>
> 1. You might need to install some plugins and restart Jenkins service. So, we recommend clicking
>    on `Restart Jenkins when installation is complete and no jobs are running` on plugin
>    installation/update page i.e `update centre`. Also, Jenkins UI sometimes gets stuck when
>    Jenkins service restarts in the back end. In this case, please make sure to refresh the UI
>    page.
> 2. For these kind of scenarios requiring changes to be done in a web UI, please take screenshots
>    so that you can share it with us for review in case your task is marked incomplete. You may
>    also consider using a screen recording software such as loom.com to record and share your
>    work.

## Solution

**1. Plugins.** Manage Jenkins, then **Plugins**: install **Pipeline**, **SSH Build Agents** and
**Git**, tick restart on completion, then log back in.

**2. Prepare App Server 1** as the sudo user:

```bash
ssh tony@stapp01

sudo yum install -y java-17-openjdk

sudo mkdir -p /home/sarah/jenkins_agent
sudo chown sarah:sarah /home/sarah/jenkins_agent

# the pipeline runs as sarah, so sarah must own the doc root to pull into it
sudo chown -R sarah:sarah /var/www/html
exit
```

**3. Push the content change** as sarah:

```bash
ssh sarah@stapp01
cd /var/www/html

echo "Welcome to xFusionCorp Industries" > index.html
git add index.html
git commit -m "index.html update"
git push origin master
exit
```

**4. Credential.** Manage Jenkins, then **Credentials**: **Username with password**, username
`sarah`, password `Sarah_pass123`, ID `sarah`.

**5. Node.** Manage Jenkins, then **Nodes**, then **New Node**, **Permanent Agent**:

- Name `App Server 1`, label `stapp01`
- Remote root directory `/home/sarah/jenkins_agent`
- Launch method **Launch agents via SSH**, host `stapp01`, credential `sarah`
- Host Key Verification Strategy: **Non verifying Verification Strategy**

Wait for the node to come online.

**6. The job.** New Item, `deploy-job`, **Pipeline**:

```groovy
pipeline {
    agent { label 'stapp01' }

    stages {
        stage('Deploy') {
            steps {
                sh '''
                    cd /var/www/html
                    git -c safe.directory=/var/www/html pull origin master
                '''
            }
        }
        stage('Test') {
            steps {
                script {
                    def ok = false
                    for (int i = 1; i <= 5; i++) {
                        def code = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://stlb01:8091", returnStdout: true).trim()
                        echo "Attempt ${i}: HTTP ${code}"
                        if (code == '200') { ok = true; break }
                        sleep 3
                    }
                    if (!ok) { error('App not working after deployment') }
                    echo 'App is working'
                }
            }
        }
    }
}
```

**7. Build it two or three times**, all green, then check the **App** button.

## Why

| Piece | Why |
|:--|:--|
| `agent { label 'stapp01' }` | Requirement 4b. Without it the pipeline runs on the controller, where `/var/www/html` belongs to a different machine entirely. |
| **Launch agents via SSH** | The controller connects out, so there is no `agent.jar` to download and no `screen` session to keep alive, unlike the inbound agents on Day 77 and 78. |
| `chown -R sarah:sarah /var/www/html` | The pipeline runs as sarah through that SSH agent, so sarah has to own the doc root to pull into it. |
| Retry loop on the HTTP code | Apache may not be serving the new content the instant the pull returns. Five attempts with a gap makes `Test` reliable rather than flaky. |

## Notes

- **Do not grep for the welcome text in `Test`.** The checker pushes its own random string to `index.html` and re-runs the job, so a text match fails against content you never wrote. Check the HTTP status instead.
- A declarative pipeline stops at the first failing stage, so a failed `Deploy` means `Test` never runs. That alone satisfies the second half of requirement 4c.
- `git -c safe.directory=...` scopes the exception to that single command, which is what the build user needs. Same trick as Day 80.
- Stage names are case sensitive: `Deploy` and `Test`.
