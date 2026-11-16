# Day 75: Jenkins Slave Nodes

## Question

The Nautilus DevOps team has installed and configured new Jenkins server in Stratos DC which they
will use for CI/CD and for some automation tasks. There is a requirement to add all app servers as
slave nodes in Jenkins so that they can perform tasks on these servers using Jenkins. Find below
more details and accomplish the task accordingly.

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and password `Adm!n321`.

1. Add all app servers as SSH build agent/slave nodes in Jenkins. Slave node name for
   `app server 1`, `app server 2` and `app server 3` must be `App_server_1`, `App_server_2`,
   `App_server_3` respectively.

2. Add labels as below:

   `App_server_1 : stapp01`

   `App_server_2 : stapp02`

   `App_server_3 : stapp03`

3. Remote root directory for `App_server_1` must be `/home/tony/jenkins`, for `App_server_2` must
   be `/home/steve/jenkins` and for `App_server_3` must be `/home/banner/jenkins`.

4. Make sure slave nodes are online and working properly.

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

## How an agent connects

A node reaches the Jenkins controller in one of two directions:

| Mode | Who starts the connection | Setup |
|:--|:--|:--|
| **Agent mode** (inbound) | The node connects **back** to the controller | Install an agent on the node |
| **SSH mode** (outbound) | The controller connects **out** to the node | Configure SSH on the node |

This task is SSH mode, which is why everything below is keys and credentials rather than
installing anything agent-shaped on the app servers.

## Solution

**1. Plugin.** Manage Jenkins, then **Plugins**, install **SSH Build Agents**, then restart.

**2. Keys to all three servers.** On the Jenkins box:

```bash
ssh-keygen

ssh-copy-id tony@stapp01
ssh-copy-id steve@stapp02
ssh-copy-id banner@stapp03

ssh tony@stapp01          # each should land with no password
```

**3. Java on every app server.** An SSH agent is a Java process running on the node:

```bash
java --version            # probably missing
sudo yum install -y java-17-openjdk
java --version
```

Repeat on all three.

**4. Credentials.** `cat ~/.ssh/id_ed25519`, then Manage Jenkins, then **Credentials**. Add three
**SSH Username with private key** entries, same key each time, usernames `tony`, `steve`,
`banner`.

**5. The nodes.** Manage Jenkins, then **Nodes**, then **New Node**, **Permanent Agent**:

| Node name | Label | Remote root directory | Host | Credential |
|:--|:--|:--|:--|:--|
| `App_server_1` | `stapp01` | `/home/tony/jenkins` | `stapp01` | `tony` |
| `App_server_2` | `stapp02` | `/home/steve/jenkins` | `stapp02` | `steve` |
| `App_server_3` | `stapp03` | `/home/banner/jenkins` | `stapp03` | `banner` |

For each: **Launch method** is **Launch agents via SSH**, and **Host Key Verification Strategy**
is **Non verifying Verification Strategy**. Create the first one, then copy it for the other two
and change the values.

**6. Test.** New job, tick **Restrict where this project can be run** with label `stapp01`, add an
**Execute shell** step running `hostname`, then build. The output should name the app server.

## Why

| Piece | Why |
|:--|:--|
| Java on each app server | The agent is a Java process that the controller starts over SSH. No JRE means the node connects and then immediately goes offline. |
| Three credentials, one key | The key is the same, but the login user differs per server. A credential pairs a username with a key, so you need one per user. |
| **Non verifying** host key strategy | The default checks the controller's `known_hosts`, which has never seen these hosts, so the launch fails before it starts. Fine for a lab, not for production. |
| Label separate from node name | Jobs target **labels**, not node names. The task deliberately gives each node a different name and label so you have to keep them straight. |

## Notes

- The remote root directory is the agent's workspace on that server. It has to be somewhere the login user can write, which is why each one sits under that user's own home.
- If a node shows offline, open its log from the node page. Missing Java and host key refusals both show up there in plain text.
- Jenkins renamed this in the UI: nodes are "agents" now, though the task still says slave.
- The test job's checkbox reads **Restrict where this project can be run**, and it takes the label `stapp01`, not the node name `App_server_1`.
