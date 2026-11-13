# Day 72: Jenkins Parameterized Builds

## Question

A new DevOps Engineer has joined the team and he will be assigned some Jenkins related tasks.
Before that, the team wanted to test a simple parameterized job to understand basic functionality
of parameterized builds. He is given a simple parameterized job to build in Jenkins. Please find
more details below:

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username
`admin` and password `Adm!n321`.

1. Create a `parameterized` job which should be named as `parameterized-job`

2. Add a `string` parameter named `Stage`; its default value should be `Build`.

3. Add a `choice` parameter named `env`; its choices should be `Development`, `Staging` and
   `Production`.

4. Configure job to execute a shell command, which should echo both parameter values (you are
   passing in the job).

5. Build the Jenkins job at least once with choice parameter value `Development` to make sure it
   passes.

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

Logged in as `admin` / `Adm!n321`:

1. **New Item**, name `parameterized-job`, **Freestyle project**.
2. Tick **This project is parameterised**:
   - **String Parameter**, name `Stage`, default value `Build`.
   - **Choice Parameter**, name `env`, choices one per line:

     ```
     Development
     Staging
     Production
     ```
3. Build Steps, **Execute shell**:

   ```bash
   echo "This is Stage parameter value: $Stage"
   echo "This is env parameter value: $env"
   ```
4. Save, then **Build with Parameters**, pick `Development`, build.
5. Open the build's **Console Output** and confirm both values printed.

## Why

| Piece | Why |
|:--|:--|
| **This project is parameterised** | Adds the Parameters section and turns the sidebar **Build** link into **Build with Parameters**. |
| Default `Build` on `Stage` | What the job uses when nobody types anything, including when it is triggered by another job. |
| Choices one per line | The textarea splits on newlines, and the first line is the default. `Development` on top makes it the preselected option. |
| `$Stage` and `$env` | Parameters arrive in the shell step as environment variables under exactly those names. |

## Notes

- Case matters. `$env` is not `$ENV`, `$Stage` is not `$STAGE`. Get it wrong and the line prints an empty value while the build still goes green, so it looks like it worked.
- Nothing here needs a plugin. Parameters and Execute shell are core Jenkins, and note 1 is boilerplate that shows up on every Jenkins task.
- Quote the strings in `echo`. An empty or space-containing value then shows up as nothing rather than mangling the line.
- Console Output is the only real check. A green tick means the shell exited `0`, not that the values were right.
