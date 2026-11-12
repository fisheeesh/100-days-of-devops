# Day 71: Configure Jenkins Job for Package Installation

## Question

Some new requirements have come up to install and configure some packages on the Nautilus
infrastructure under Stratos Datacenter. The Nautilus DevOps team installed and configured a new
Jenkins server so they wanted to create a Jenkins job to automate this task. Find below more
details and complete the task accordingly:

1. Access the Jenkins UI by clicking on the `Jenkins` button in the top bar. Log in using the
   credentials: username `admin` and password `Adm!n321`.

2. Create a new Jenkins job named `install-packages` and configure it with the following
   specifications:

- Add a string parameter named `PACKAGE`.
- Configure the job to install a package specified in the `$PACKAGE` parameter on the
  `storage server` (Stratos Datacenter).
- Build the job at least once (e.g. with parameter `PACKAGE=vim-enhanced`) so the package is
  installed on the Storage server and can be verified.

> `Note`:
>
> 1. Ensure to install any required plugins and restart the Jenkins service if necessary. Opt for
>    `Restart Jenkins when installation is complete and no jobs are running` on the plugin
>    installation/update page. Refresh the UI page if needed after restarting the service.
> 2. Verify that the Jenkins job runs successfully on repeated executions to ensure reliability.
> 3. Capture screenshots of your configuration for documentation and review purposes.
>    Alternatively, use screen recording software like `loom.com` for comprehensive documentation
>    and sharing.

## Solution

**1. Prepare the storage server.** A build has no terminal, so neither ssh nor sudo may prompt:

```bash
ssh-keygen
ssh-copy-id natasha@ststor01

# now it should land with no password
ssh natasha@ststor01

sudo yum install curl
# still asks for a password, which a build cannot answer

sudo visudo
# add, or edit the existing NOPASSWD line:
#   natasha ALL=(ALL) NOPASSWD: ALL

sudo ls
# no password now
```

Keep the private key handy for the credential:

```bash
cat ~/.ssh/id_ed25519
```

**2. Install the SSH plugin.** Manage Jenkins, then **Plugins**, then **Available plugins**,
search `SSH`, install, and tick **Restart Jenkins when installation is complete and no jobs are
running**.

**3. Add the credential.** Manage Jenkins, then **Credentials**, then **Add Credentials**:

- Kind: **SSH Username with private key**
- ID: `storage-ssh`
- Username: `natasha`
- Private key: **Enter directly**, paste the key from step 1

**4. Add the remote host.** Manage Jenkins, then **System**, find the **SSH remote hosts**
section, then **Add**:

- Hostname `ststor01`, port `22`, credentials `natasha`

**5. Create the job.** New Item, name `install-packages`, **Freestyle project**:

- Tick **This project is parameterised**, add a **String Parameter** named `PACKAGE`.
- Build Steps, **Execute shell script on remote host using ssh**, pick the `ststor01` site, and
  set the command:

```bash
sudo yum -y install $PACKAGE
```

**6. Build.** **Build with Parameters**, enter `vim-enhanced`, build. Run it a second time to
confirm it is still green.

## Why

| Step | Why |
|:--|:--|
| `ssh-copy-id natasha@ststor01` | Jenkins authenticates with a key, never a password. The public key has to sit in natasha's `authorized_keys` on the storage server. |
| `NOPASSWD: ALL` in sudoers | A build has no terminal to type a sudo password into. Without this the step hangs and then fails. |
| `-y` on `yum` | Same reason. Nothing is there to answer the confirmation prompt. |
| The SSH site under **System** | The build step only offers hosts that are defined globally. A credential on its own gives the step nothing to pick. |

## Notes

- Order matters: credential, then SSH site, then job. The site form has no credential to choose until the credential exists, and the build step has no host until the site does.
- Jenkins uses the private key you paste into the credential, so it does not matter which user generated the pair. What matters is that the matching public key is in natasha's `authorized_keys`.
- Re-running is safe: `yum install` on a package that is already present exits `0`. That is exactly what note 2 is asking you to confirm.
- `$PACKAGE` is the parameter name reaching the remote shell as an environment variable. Change the parameter name and the command has to change with it.
