---
title: "GitLab Runner"
date: 2019-05-08T11:52:56+01:00
Tags: ["automation","VM","Jenkins", "GitLab","Scripting","Project Management","Version Control"]
---


## The Problem

Our release process means that we have to refresh test environments on a particular schedule around the date of the
release. These dates were put into a confluence page, but we discovered that the problem with this is that if a release
is moved we have to change all the refresh dates. Also sometimes projects mean particular environments aren't refreshed
or the schedule is altered slightly.

So it ends up that we have to:

* Check the confluence page every day
* Ask the environment owner if they really want a refresh if one is scheduled
* If they do, we send an email the day before.
* Lastly do the work on the target date.

The problem is that this is a lot of manual boring repetitive work that is prone to error. In short, it is the type
of thing that a computer can do really easily and well. I have been playing with [GitLab](../../tags/gitlab) - Maybe it can help here as well?


## The Script

I wrote a script to read a GitLab page. Confluence requires a login to view the page, so the script can't easily
read the dates directly from there. The page has a list of environments and their refresh dates. The script will
send an email:

* Two days before the refresh to the environment owners. This will allow them to stop the refresh if they change their minds.
* The day before to an email list we use for this type of communication.
* One the day to us DBAs who will do the work.

It takes account of the fact that people don't work over
the weekend, though at present it doesn't take account of bank holidays. Eventually I will get the script to call
the refresh process, but one step at a time.

It is a self contained project. The readme contains instructions, there is a script which sends the email,
the rest is configuration. Email templates, lists of email addresses to send the emails to etc, and of course the
list of dates.


## Configuring GitLab

### Installing GitLab Runner

I wanted to send the emails once a day, which means finding a way to run a script. I have a VM which is running
[Jenkins](../../tags/jenkins) already, so I asked the sysadmin to install gitlab runner on that. Then I had to
[register](https://docs.gitlab.com/runnders/register) gitlab runner with my project. 


### Registering GitLab Runner

I had to get the registration token by navigating to the project, then click on Settings->CI/CD then following
the instructions under _Setting up a specific runner manually_. I tagged it with ```shellci```, it is just a tag, I didn't have 
to, but that means the runner can run jobs with this tag. If I had a job which needed to run on Windows I could use a different
tag so a different runner would take care of it. Then as root I ran:

{{< highlight bash >}}
gitlab-runner register
{{</highlight>}}

and followed the prompts.


### Setting the Schedule

By default jobs are triggered on commit, so I set up a schedule under CI/CI->Schedules (Not the one under settings).
I clicked _Every day (at 4:00am)_.


### Configuring What is Run

Now I need to tell the job what to do, and not to run on commit, only on the schedule. I created a file .gitlab-ci.yml
which contained the following:

{{< highlight yaml >}}
send-email:
  tags:
    - shellci
  script:
    - scripts/emailrefresh.sh
  only:
    variables:
      - $CI_PIPELINE_SOURCE == "schedule"
{{</highlight>}}

First I created a job called send-email. I have a tag so the runner knows it can run it (If I hadn't tagged the runner, I wouldn't have had to tag the
job). Then I have to tell the job to run a script, which is part of the source tree. Lastly I tell it to only run on the schedule,
or more specifically, only run if the variable CI_PIPELINE_SOURCE is set to "schedule". At first the job wouldn't
run, and I discovered that I had made a mistake in this section which meant the condition could never be true,
so the job would never be run. In this case it was never passed to the runner, gitlab never ran the job. When I corrected it,
the job ran OK and I got the emails.

## Conclusion

Using GitLab runner and a script we have a nice method to automate a process which which took a lot of manual effort before.
We can give the environment owners access to the GitLab repository, then they can update dates themselves. The whole process is self managing!

I have learned a little about GitLab, which I hope will come in useful later on.

Also, there is room for improvement! I will eventually get the script to call my refresh script and do the actual refresh.
