# Day 78: Jenkins Conditional Pipeline

## Question

The development team of xFusionCorp Industries is working on to develop a new static website and
they are planning to deploy the same on Nautilus App Server using Jenkins pipeline. They have
shared their requirements with the DevOps team and accordingly we need to create a Jenkins
pipeline job. Please find below more details about the task:

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and password `Adm!n321`.

Similarly, click on the `Gitea` button on the top bar to access the Gitea UI. Login using username
`sarah` and password `Sarah_pass123`. There under user `sarah` you will find a repository named
`web_app` that is already cloned on **App Server 1** under `/var/www/html`. sarah is a developer
who is working on this repository.

1. Add a slave node named `App Server 1`. It should be labeled as `stapp01` and its remote root
   directory should be `/home/sarah/jenkins_agent` (the repository is cloned under
   `/var/www/html`).
2. We have already cloned repository on **App Server 1** under `/var/www/html`.
3. Apache is already installed on the app server and is running on port `8080`.
4. Create a Jenkins pipeline job named `xfusion-webapp-job` (it must not be a
   `Multibranch pipeline`) and configure it to:
   - Add a string parameter named `BRANCH`.
   - It should conditionally deploy the code from `web_app` repository under `/var/www/html` on
     **App Server 1**, as this is the document root of the app server. The pipeline should have a
     single stage named `Deploy` ( which is case sensitive ) to accomplish the deployment.
   - The pipeline should be conditional, if the value `master` is passed to the `BRANCH` parameter
     then it must deploy the `master` branch, on the other hand if the value `feature` is passed
     to the `BRANCH` parameter then it must deploy the `feature` branch.

LB server is already configured. You should be able to see the latest changes you made by clicking
on the `App` button. Please make sure the required content is loading on the main URL
`https://<LBR-URL>` i.e there should not be a sub-directory like `https://<LBR-URL>/web_app` etc.

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

**1. Same setup as Day 77:** Pipeline plugin, Jenkins URL `http://jenkins:8080/`, then a node
named `App Server 1`, label `stapp01`, remote root `/home/sarah/jenkins_agent`, launch method
**Launch agent by connecting it to the controller**. Copy the secret from the node page.

**2. On stapp01, with the one change that matters: run the agent as `sarah`, not root.**

```bash
ssh tony@stapp01
sudo yum install -y java-17-openjdk screen

# fix ownership rather than escalating past it
sudo chown -R sarah:sarah /var/www/html
sudo mkdir -p /home/sarah/jenkins_agent
sudo chown -R sarah:sarah /home/sarah/jenkins_agent

# /tmp so sarah can read it
cd /tmp
curl -sO http://jenkins:8080/jnlpJars/agent.jar

screen -S jenkins

sudo -u sarah java -jar /tmp/agent.jar -url http://jenkins:8080/ \
  -secret <SECRET> -name "App Server 1" -webSocket \
  -workDir "/home/sarah/jenkins_agent"
# wait for Connected, then Ctrl-A then D to detach
```

**3. The job.** New Item, `xfusion-webapp-job`, **Pipeline**:

```groovy
pipeline {
    agent { label 'stapp01' }

    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to deploy')
    }

    stages {
        stage('Deploy') {
            steps {
                script {
                    if (params.BRANCH == 'master' || params.BRANCH == 'feature') {
                        sh """
                            whoami
                            git config --global --add safe.directory /var/www/html
                            cd /var/www/html
                            git fetch origin
                            git checkout -f ${params.BRANCH}
                            git reset --hard origin/${params.BRANCH}
                        """
                    } else {
                        error "Unsupported BRANCH: ${params.BRANCH}"
                    }
                }
            }
        }
    }
}
```

**4. Build** with `BRANCH=master`, then `feature`, then `master` again so the app ends there.
`whoami` in the log should print `sarah`.

**5. Verify before submitting:**

```bash
cd /var/www/html
ls -la        # owned by sarah, not root
git branch    # * master
```

## Why

| Piece | Why |
|:--|:--|
| `sudo -u sarah`, not plain `sudo` | Running as root leaves root-owned files in `/var/www/html` and `.git`. The checker works as `sarah`, so it cannot write its validation file and the task fails with "Validation file not found". |
| `chown` instead of escalating | The permission error on the agent directory was the signal to fix ownership. The task naming the root `/home/sarah/jenkins_agent` already said which user it wanted. |
| `agent { label 'stapp01' }` | No colon. `label: 'x'` gives `No agent type specified`. |
| `fetch` + `checkout -f` + `reset --hard` | `git pull` fails on a diverged branch or local changes. This lands exactly on the remote branch every time, which is what makes repeated builds reliable. |

## Notes

- Get the Jenkins URL right: if the server URL is wrong, the registration URLs it hands out to agents are wrong too, and nothing connects.
- `git config --global` only writes the config of the user running it, so it belongs inside the `sh` step where the build user actually picks it up.
- Use the exact job name from the task. Day 77 wanted `datacenter-webapp-job`, this one wants `xfusion-webapp-job`, and a near miss fails silently.
- The `else` branch with `error` is what makes it genuinely conditional rather than just parameterised. Any value other than the two named ones fails the build instead of deploying something unexpected.
