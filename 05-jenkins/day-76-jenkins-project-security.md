# Day 76: Jenkins Project Security

## Question

The xFusionCorp Industries has recruited some new developers. There are already some existing jobs
on Jenkins and two of these new developers need permissions to access those jobs. The development
team has already shared those requirements with the DevOps team, so as per details mentioned below
grant required permissions to the developers.

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and password `Adm!n321`.

1. There is an existing Jenkins job named `Packages`, there are also two existing Jenkins users
   named `sam` with password `sam@pass12345` and `rohan` with password `rohan@pass12345`.
2. Grant permissions to these users to access `Packages` job as per details mentioned below:

   a.) Make sure to select `Inherit permissions from parent ACL` under `inheritance strategy` for
   granting permissions to these users.

   b.) Grant mentioned permissions to `sam` user : `build`, `configure` and `read`.

   c.) Grant mentioned permissions to `rohan` user : `build`, `cancel`, `configure`, `read`,
   `update` and `tag`.

> `Note:`
>
> 1. Please do not modify/alter any other existing job configuration.
> 2. You might need to install some plugins and restart Jenkins service. So, we recommend clicking
>    on `Restart Jenkins when installation is complete and no jobs are running` on plugin
>    installation/update page i.e `update centre`. Also Jenkins UI sometimes gets stuck when
>    Jenkins service restarts in the back end. In this case, please make sure to refresh the UI
>    page.
> 3. For these kind of scenarios requiring changes to be done in a web UI, please take screenshots
>    so that you can share it with us for review in case your task is marked incomplete. You may
>    also consider using a screen recording software such as loom.com to record and share your
>    work.

## Solution

**1. Plugin.** Open the `Packages` job, then **Configure**: there is no security section. Manage
Jenkins, then **Plugins**, then **Available plugins**, install **Matrix Authorization Strategy**,
then restart.

**2. Global strategy.** The job still shows no security section, because the per-job grid only
appears once the global strategy supports it. Manage Jenkins, then **Security**, then
**Authorization**, pick **Project-based Matrix Authorization Strategy**:

- `admin`: **Overall / Administer** (set this before saving).
- `sam` and `rohan`: **Overall / Read**, so they can log in and see Jenkins at all.
- Save.

**3. The job grid.** Back to the `Packages` job, then **Configure**:

- Tick **Enable project-based security**.
- Inheritance Strategy: **Inherit permissions from parent ACL**.
- Add `sam`: **Build**, **Configure**, **Read**.
- Add `rohan`: **Build**, **Cancel**, **Configure**, **Read**, **Update**, **Tag**.
- Save.

**4. Verify.** Log out, log in as `sam` / `sam@pass12345`. The `Packages` job should be visible
with Build and Configure available, and nothing else.

## Where those permissions live

The job grid groups its columns, and three of the six are not in the group people look in first:

| Permission | Column group |
|:--|:--|
| Build, Cancel, Configure, Read | **Job** |
| Update | **Run** |
| Tag | **SCM** |

## Why

| Step | Why |
|:--|:--|
| Global strategy first | The per-job security section does not exist until the global authorization is Project-based Matrix. That is why the job config looked empty on the first two visits. |
| `admin` keeps Administer | Saving replaces the whole authorization strategy at once. Save without an administrator and nobody can get back in, including you. |
| Overall / Read globally | Job permissions decide what someone can do with a job. Without Overall/Read they cannot reach Jenkins to use them. |
| **Inherit permissions from parent ACL** | Requirement 2a names it. It means the job's grid adds to what the global grid already grants, rather than replacing it. |

## Notes

- Note 1 says do not touch other jobs. Only the `Packages` job gets a project-based security block.
- The other inheritance option, **Do not inherit permission grants from other ACLs**, cuts the job off from the global grid entirely. Pick that by mistake and even admin can lose access to the job.
- Same lock-out escape as Day 70: set `<useSecurity>false</useSecurity>` in `/var/lib/jenkins/config.xml`, run `service jenkins restart`, then redo the grid.
- Logging in as `sam` is the real check. The grid can read correctly and still be wrong if the inheritance strategy was left on the default.
