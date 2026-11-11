# Day 70: Configure Jenkins User Access

## Question

The Nautilus team is integrating Jenkins into their CI/CD pipelines. After setting up a new
Jenkins server, they're now configuring user access for the development team, Follow these steps:

1. Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login with username
   `admin` and password `Adm!n321`.

2. Create a jenkins user named `siva` with the password `8FmzjvFU6S`. Their full name should match
   `Siva`.

3. Utilize the `Project-based Matrix Authorization Strategy` to assign `overall read` permission
   to the `siva` user.

4. Remove all permissions for `Anonymous` users (if any) ensuring that the `admin` user retains
   overall `Administer` permissions.

5. For the existing job, grant `siva` user only `read` permissions, disregarding other permissions
   such as Agent, SCM etc.

> `Note:`
>
> 1. You may need to install plugins and restart Jenkins service. After plugins installation,
>    select `Restart Jenkins when installation is complete and no jobs are running` on plugin
>    installation/update page.
> 2. After restarting the Jenkins service, wait for the Jenkins login page to reappear before
>    proceeding. Avoid clicking `Finish` immediately after restarting the service.
> 3. Capture screenshots of your configuration for review purposes. Consider using screen
>    recording software like `loom.com` for documentation and sharing.

## Solution

All in the UI, logged in as `admin` / `Adm!n321`.

**1. Create the user.** Manage Jenkins, then **Users**, then **Create User**:

- Username `siva`
- Password `8FmzjvFU6S`
- Full name `Siva`

**2. Install the plugin.** Manage Jenkins, then **Security**, then **Authorization**: there is no
Matrix option yet. So Manage Jenkins, then **Plugins**, then **Available plugins**, search
`Matrix Authorization Strategy`, install, and tick **Restart Jenkins when installation is complete
and no jobs are running**. Wait for the login page, then log back in.

**3. Set the global grid.** Manage Jenkins, then **Security**, then **Authorization**, now pick
**Project-based Matrix Authorization Strategy**:

- Add user `siva`, tick **Overall / Read** only.
- `Anonymous`: untick everything on that row.
- Add `admin` and tick **Overall / Administer** before saving.
- Save.

**4. Grant job-level read.** Dashboard, open the existing job (`Hello World`), then **Configure**:

- Tick **Enable project-based security**.
- Add user `siva`, tick **Read** only. Leave Agent, SCM and the rest alone.
- Save.

**5. Verify.** Log out, log in as `siva` / `8FmzjvFU6S`. The job should be visible and read only,
with no **Configure** option.

## Why

| Step | Why |
|:--|:--|
| Install the plugin first | Stock Jenkins has no matrix strategy. The option simply is not in the Authorization list until the plugin is in and Jenkins has restarted. |
| **Project-based** matrix, not plain matrix | Step 5 needs per-job permissions. Only the project-based variant adds the per-job security section to a job's config. |
| `admin` gets Administer **before** you save | Saving replaces the whole authorization strategy at once. Save a grid with no administrator and nobody can get back in. |
| Job config as well as the global grid | Overall/Read only lets `siva` see Jenkins itself. The job's own grid is what makes that job visible to him. |

## Notes

- Anonymous usually starts with **Overall / Read** already ticked, which is the "(if any)" the task hedges about. Clearing the whole row is step 4.
- Locked yourself out? Set `<useSecurity>false</useSecurity>` in `/var/lib/jenkins/config.xml`, run `service jenkins restart`, then redo the grid.
- Logging in as `siva` is the real check. The grid can look right and still be wrong if the job-level box was missed.
- Overall/Read plus job Read means he can look and nothing else. No Build, no Configure.
