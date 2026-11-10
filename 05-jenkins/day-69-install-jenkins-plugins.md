# Day 69: Install Jenkins Plugins

## Question

The Nautilus DevOps team has recently setup a Jenkins server, which they want to use for some
CI/CD jobs. Before that they want to install some plugins which will be used in most of the jobs.
Please find below more details about the task

1. Click on the Jenkins button on the top bar to access the Jenkins UI. Login using username
   `admin` and password `Adm!n321`.

2. Once logged in, install the `Git` and `GitLab` plugins. You may need to restart Jenkins to
   complete the plugin installation; if required, opt to **Restart Jenkins when installation is
   complete and no jobs are running** on the plugin installation/update page (Update Centre).

> `Note:`
>
> 1. After restarting Jenkins, wait for the login page to reappear before proceeding.
> 2. For tasks involving web UI changes, capture screenshots to share for review or consider using
>    screen recording software like loom.com for documentation and sharing.

## Solution

No terminal work. All of it is in the Jenkins UI:

1. Click the **Jenkins** button on the top bar.
2. Log in as `admin` / `Adm!n321`.
3. **Manage Jenkins** then **Plugins**, then the **Available plugins** tab.
4. Search `Git`, tick it. Search `GitLab`, tick it.
5. Install, then on the progress page tick **Restart Jenkins when installation is complete and no
   jobs are running**.
6. Wait for the login page to come back, then log in again.
7. Confirm both appear under **Manage Jenkins** then **Plugins** then **Installed plugins**.
8. Screenshot it, since there is no command output to show.

## Why

| Step | Why |
|:--|:--|
| **Available plugins** tab | **Installed plugins** only lists what is already there. New ones come from the Update Centre list. |
| The restart checkbox | Some plugins only load at startup. Skip it and they show as installed but are not active. |
| Logging in again | Jenkins drops your session while it restarts, which is why the task says to wait for the login page. |
| Checking **Installed plugins** | Proof the install actually finished rather than the progress bar having stalled. |

## Notes

- This task logs in as `admin`, not the `theadmin` user created on Day 68. Each lab starts fresh, so use whatever credentials the task gives you.
- A failed plugin download is usually transient. Restart Jenkins and try the install again.
- `Git` and `GitLab` are two separate plugins. Git is the SCM support; GitLab adds the integration on top, such as webhooks and connection settings.
