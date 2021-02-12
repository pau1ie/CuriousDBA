---
title: "Gitlab Runner for Auto Devops"
date: 2021-02-12T12:28:00Z
tags: ["GitLab","Automation"]
---

The Auto DevOps pipeline in GitLab is 
supposed to be magic! It just reads your 
mind and does whatever you wanted! At least that 
seems to be what the sales blurb says!

Over the next few posts I will investigate how we use it. 
It is pretty clever, but we do need to set a couple of things first.

# Requirements
This assumes docker is installed and working on the machine that will
be used for the runner. I used my Linux desktop for this exercise which has Docker already set up.

# Set up
First  set up the GitLab runner. It uses the Docker executor. I believe this
requires access to run Docker so as root I 
gave it permission:

```bash
usermod -aG docker gitlab-runner
```

Now set up the runner. In GitLab there is a page where you set up a runner for
a project or a group. For a project it is under `Settings->CI/CD->Runners`
That page gives the GitLab instance URL and the registration
token to use in the setup.

```console {hl_lines=[1,6,8,10,16,18]}
# gitlab-runner register
Runtime platform      arch=amd64 os=linux pid=25467 revision=943fc252 version=13.7.0
Running in system-mode.                            
                                                   
Enter the GitLab instance URL (for example, https://gitlab.com/):
https://gitlab.developers.cam.ac.uk/
Enter the registration token:
XXXXXXXXXXXXXXXXXXXX
Enter a description for the runner:
[mydesktop.cam.ac.uk]: My docker exeutor
Enter tags for the runner (comma-separated):

Registering runner... succeeded                     runner=XXXXXXXX
Enter an executor: kubernetes, docker+machine, custom, docker, docker-ssh,
 parallels, shell, ssh, virtualbox, docker-ssh+machine:
docker
Enter the default Docker image (for example, ruby:2.6):
docker:stable
Runner registered successfully. Feel free to start it, but if it's running already 
the config should be automatically reloaded! 
```

The important things here are the docker executor, and the image. 

Once this is done it is important  to give the runner privileged access.
If this is not done, it will error when trying to connect the docker repository.
Edit `/etc/gitlab-runner/config.toml` and find the runner that was
just set up. Change the line

```ini
    privileged = false
```

to read

```ini
    privileged = true
```

Then restart the runner using:

```bash
gitlab-runner restart
```

If this isn't done, the following error will appear in the job output.

```console
error during connect: Post http://docker:2375/v1.40/auth:
    dial tcp: lookup docker on 10.0.64.1:53: no such host
```

Now the runner is set up. In the next post I will investigate how
we use it to build a container.