# Day 10: Linux Bash Scripts

## Question

The production support team of `xFusionCorp Industries` is working on developing some bash
scripts to automate different day to day tasks. One is to create a bash script for archiving
website content files. They have a static website running on `App Server 2` in
`Stratos Datacenter`, and they need to create a bash script named `official_archive.sh` which
should accomplish the following tasks. (Also remember to place the script under the `/scripts`
directory on `App Server 2`).

a. Create a zip archive named `xfusioncorp_official.zip` of `/var/www/html/official` directory.

b. Save the archive in the `/archives/` directory on the `App Server 2`. This is a temporary
storage, as archives from this location will be cleaned on a weekly basis. Therefore, the
archive should also be copied to the `Nautilus Storage Server` so it can be retrieved later for
validation purposes.

c. Copy the created archive to the `Nautilus Storage Server` server in the `/archives/`
location.

d. Please make sure script won't ask for password while copying the archive file. Additionally,
the respective server user (for example, `tony` in case of `App Server 1`) must be able to run
it.

e. Do not use sudo inside the script.

> **Note:** The zip package must be installed on given App Server before executing the script.
> This package is essential for creating the zip archive of the website files. Install it
> manually outside the script.

## Solution

Setup on `stapp02`, all outside the script:

```bash
ssh steve@stapp02

# the script cannot use sudo, so anything needing root happens here
sudo yum install -y zip

# password-less copy, set up as steve because steve is who runs the script
ssh-keygen -t ed25519            # empty passphrase
ssh-copy-id natasha@ststor01     # natasha's password, once

# steve has to be able to write /archives without sudo
ls -ld /archives
sudo chown -R steve:steve /archives
```

Create the script:

```bash
cd /scripts
touch official_archive.sh
chmod +x official_archive.sh
ls -lah
# -rwxr-xr-x 1 steve steve 0 Sep 11 03:08 official_archive.sh

vi official_archive.sh
```

```bash
#!/bin/bash

ARCH_FILE="xfusioncorp_official.zip"
SOURCE_DIR="/var/www/html/official"
ARCH_DIR="/archives"

SSH_USER="natasha"
SSH_HOST="ststor01"
SRV_DIR="/archives/"

# package everything under SOURCE_DIR into ARCH_DIR/ARCH_FILE
zip -r "${ARCH_DIR}/${ARCH_FILE}" "${SOURCE_DIR}"

# copy to the storage server, no password prompt thanks to the key
scp "${ARCH_DIR}/${ARCH_FILE}" "${SSH_USER}@${SSH_HOST}:${SRV_DIR}"
```

Run and verify on both ends:

```bash
./official_archive.sh
ls -lah /archives/
ssh natasha@ststor01 ls -lah /archives/
```

## Why

| Command | Why |
|:--|:--|
| `yum install -y zip` | Not installed by default, and the script is not allowed to use sudo, so it has to happen beforehand. |
| `ssh-keygen` + `ssh-copy-id` as `steve` | The key must belong to whoever **runs** the script. A key set up for `thor` on the jump host does nothing for `steve` on stapp02. |
| `chown -R steve:steve /archives` | No sudo inside the script, so `steve` needs write access to `/archives` on his own. |
| `#!/bin/bash` | Pins the interpreter. Without it, which shell runs the file depends on who calls it, and cron can end up using `sh`. |
| `zip -r` | `-r` means recursive. Without it, zip grabs only the empty directory and skips everything inside. With it, zip goes into the directory and picks up every file and subfolder. |

## Notes

- `${SSH_HOST}`, not `{$SSH_HOST}`. The second one expands to the literal `{ststor01}` and scp fails with `hostname contains invalid characters`. One character apart, easy to miss.
- `chown` takes a user, `chmod` takes a mode. `chmod -R steve /archives` just errors with `invalid file mode`.
- If the script asks for a password, the key is not set up for the user actually running it.
- Running it again does not replace the zip, it updates the existing archive in place. Files deleted from the source stay inside. Remove the old archive first if you need a clean one each run.
