---
title: Periodically Clean Merged Git Branches
date: 2021-03-31 17:00:00 +0100
categories: [continuous integration]
tags: [git, automation, powershell]
toc: false
---

![Cleaning picture]({{site.url}}assets/clean.jpg)

# Introduction

In our team we had some trouble with old Git branches not being cleaned up although they were merged in the past.
Jenkins (on Windows) would then also try to scan those which resulted in more resource usage.
Sometimes somebody would manually delete the old branches but this is not a maintainable solution.

# Solution: The Cleaning Crew

Just like any office building a regular cleaning crew comes in and cleans the office.
We need something similar for our situation as well.
Preferrably every night it should cleanup old branches.

## Step 1: The Cleaning Script
    {% highlight powershell %}
    cd <path_to_my_repo>

    # Get newest branches
    git.exe fetch --all

    # Get all merged branches except main ones like master and HEAD, then delete them on the remote (origin)
    git.exe branch --all --merged remotes/origin/master |`
        Select-String -NotMatch "master" | `
        Select-String -NotMatch "HEAD"   | `
        Select-String "remotes/origin/"  | `
        Foreach-Object { `
            $_.ToString().Replace("remotes/origin/", "").Trim() `
        } | Foreach-Object { `
                git.exe push origin --delete $_ `
            }
    {% endhighlight %}

Save this script to a path somewhere. In my situation this is `C:\scripts\Git-CleanMergedRemoteBranches.ps1`.

## Step 2: Run Script Nightly

To execute something periodically (every night at 11 PM in this example), the Task Scheduler in Windows is most suitable for this.
Use the following script to add it as a new scheduled task:

    {% highlight powershell %}
    # Create a new task action
    $taskAction = New-ScheduledTaskAction `
        -Execute 'powershell.exe' `
        -Argument '-File C:\scripts\Git-CleanMergedRemoteBranches.ps1'
    $taskAction

    # Add it to the scheduled task list
    $taskTrigger = New-ScheduledTaskTrigger -Daily -At 11PM
    $tasktrigger

    # Register the new PowerShell scheduled task

    # The name of your scheduled task.
    $taskName = "GitCleanMergedRemoteBranches"

    # Describe the scheduled task.
    $description = "Deletes all merged branches on Git remote."

    # Register the scheduled task
    Register-ScheduledTask `
        -TaskName $taskName `
        -Action $taskAction `
        -Trigger $taskTrigger `
        -Description $description
    {% endhighlight %}

You can view the newly added task via the Windows key -> type in "Task Scheduler" -> Hit ENTER.
This brings up the task scheduler user interface and there you should see the task `GitCleanergedRemoteBranches`.
Right-click it and choose "Run" to do a test run. It should show a "... operation completed successfully (0x0)" result.
