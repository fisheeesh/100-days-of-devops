# Day 77: Jenkins Deploy Pipeline

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
   `/var/www/html`; the agent uses a separate directory so it does not pollute the repo).
2. We have already cloned repository on **App Server 1** under `/var/www/html`.
3. Apache is already installed on the app server and is running on port `8080`.
4. Create a Jenkins pipeline job named `datacenter-webapp-job` (it must not be a
   `Multibranch pipeline`) and configure it to:
   - Deploy the code from `web_app` repository under `/var/www/html` on **App Server 1**, as this
     is the document root of the app server. The pipeline should have a single stage named
     `Deploy` ( which is case sensitive ) to accomplish the deployment.

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

**1. Plugin.** New Item offers no **Pipeline** type, so Manage Jenkins, then **Plugins**, install
**Pipeline**, then restart.

**2. Jenkins URL.** Manage Jenkins, then **System**, set the Jenkins URL to `http://jenkins:8080/`
and save. An inbound agent dials this address, so it has to be correct.

**3. The node.** Manage Jenkins, then **Nodes**, then **New Node**:

- Name `App Server 1`, label `stapp01`
- Remote root directory `/home/sarah/jenkins_agent`
- Launch method: **Launch agent by connecting it to the controller**

The node page then shows the connection instructions, including a secret.

**4. Start the agent on stapp01:**

```bash
ssh tony@stapp01

sudo yum install -y java-17-openjdk
sudo yum install -y screen

echo <secret-from-the-node-page> > secret-file
curl -sO http://jenkins:8080/jnlpJars/agent.jar

# keep it alive after you log out
screen -S jenkins

sudo java -jar agent.jar \
  -url http://jenkins:8080/ \
  -secret @secret-file \
  -name "App Server 1" \
  -webSocket \
  -workDir "/home/sarah/jenkins_agent"

# Ctrl-A then D to detach, leaving it running
```

The node should now show as connected in Jenkins.

**5. The pipeline job.** New Item, name `datacenter-webapp-job`, **Pipeline**, then paste:

```groovy
pipeline {
    agent {
        label 'stapp01'
    }

    stages {
        stage('Deploy') {
            steps {
                echo 'Starting deployment for web_app'
                sh '''
                    git config --global --add safe.directory /var/www/html
                    cd /var/www/html
                    git pull
                '''
                echo 'Finished deployment for web_app'
            }
        }
    }
}
```

Build it, then check the site through the **App** button.

## Why

| Piece | Why |
|:--|:--|
| Inbound launch, not SSH | Day 75 used SSH mode, where the controller dials out. Here the node connects **back**, so the Jenkins URL must be right and the agent process runs on the app server. |
| `-webSocket` | Carries the agent connection over the same HTTP port, so no separate agent port needs opening. |
| `screen` | The agent is a foreground process. Without it, logging out kills the agent and the node drops offline. |
| `safe.directory` **inside** the `sh` step | The agent was started with sudo, so it runs as root. `git config --global` only writes the config of the user who runs it, so setting it as `tony` never reached root. |

## Notes

- Use `http://jenkins:8080/` everywhere, not `https`. The agent.jar download and the Jenkins URL field both need the same scheme, and this instance serves plain HTTP.
- Running the agent under sudo is the quick way past the write permission on `/home/sarah/jenkins_agent`, and it makes every pipeline step run as root. The tidier option is to own that directory as the user who starts the agent, or to start it as `sarah`.
- The repo is already cloned at the document root, so `Deploy` is just a `git pull` in place. That is what keeps the site on `/` instead of `/web_app`.
- The stage name is case sensitive: `Deploy`, not `deploy`.
