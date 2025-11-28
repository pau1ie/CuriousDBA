---
title: "Setting up VSCode for Ansible on Windows using Podman"
date: 2025-05-21T15:10:49+00:00
tags: ['PeopleSoft','Automation','Ansible','Windows','Visual Studio']
---

My work laptop runs Windows. I would prefer to run Linux, but some
programs I need only work on Windows. Another program I need, Ansible,
does not support Windows.
I get round this by not running it on Windows, but it would be nice
to be able to develop on my laptop to avoid frustration with network
slow downs etc.

VSCode supports dev containers. This is great, because development
happens in a container. But the officially supported way of doing this
is with Docker. Docker Desktop for Windows is not free. However, Podman
desktop is, so we can use that.

## Things that are needed

* [Podman](https://podman.io/)
* [VSCode](https://code.visualstudio.com/)
* The Ansible VSCode extension from RedHat
* The Ansible Dev Container extension from Microsoft

I installed Podman Desktop for Windows, then that downloaded the other
things that Podman needed.

## Set up Podman

First I set up Podman. It created it's Podman Machine which is where
it actually runs the containers. I accepted the defaults for this.

Have the desktop open when running the commands below to check that the
podman machine is running in Settings->Resources. Also check that the
following runs without error:

```
podman version
```

When I got an error, I tried restarting the podman machine, and that
fixed it.


## Set up VSCode Dev Container for Podman

In VSCode, open Settings by Edit -> Settings. Then navigate to 
Extensions -> Dev Containers. We need to change the following:

* *Docker Compose Path*: `podman-compose`
* *Docker path*: `podman`
* *Docker Socket path*: `/run/docker.sock`
I looked inside the podman machine and it looks like this was the correct
location.
* *Dev Containers -> Execute in WSL* This needs to be set to the default
which is *not selected*. 

## Set up the repository.

### Use the Ansible extension to create a container

This doesn't work.

On the left side of the VSCode window are some icons, including the
Ansible icon, which looks like an A in a circle. Clicking it beings up
the Ansible Development tools sidebar. Towards the end is the Add
section. One of the things we can add is a DevContainer. Click on it.

I get an error. It says 

```
ERROR: Could not create devcontainer. Please check that your destination
path exists and write permissions are configured for it.
```
If I try to alter the destination directory, I also get an error:
```
File system provider for C:%5CUsers%5Cmyid is not available.
```
Interestingly this message doesn't change even if we try change the path,
so something is wrong with this. So let's abandon that approach!

### Use the dev container to create a container

#### Documentation

Alex Dworjan's video: [Ansible Dev Server using VSCode Dev Containers](https://www.youtube.com/watch?v=kOGs6Ntt8JY)
is useful here. It refers to the RedHat Documentation on
[Installing Ansible development tools](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/developing_automation_content/installing-devtools)

I do not have a RedHat account, so I can't use the container from
RedHat, but instead I used the freely available 
[ansible-dev-tools](https://github.com/ansible/ansible-dev-tools)
container, which comes from the GitHub container registry.

So we need to follow the above documentation but changing the container
name. The repository contains a 
[devcontainer.json](https://github.com/ansible/ansible-dev-tools/blob/main/.devcontainer/podman/devcontainer.json)
file, which we will use.

#### Process

Create a directory called `.devcontainer` in the repository. Copy
[devcontainer.json](https://github.com/ansible/ansible-dev-tools/blob/main/.devcontainer/podman/devcontainer.json)
into that directory. Now when we click the *Open Remote Window* icon
on the bottom right we can see some options. Pick *Reopen in container*.
The window closes and reopens. If the container isn't running, it is
downloaded and started. Very nice.

## Working in the container

### Line Endings

It seems that the
folder is mounted in the linux container. The first thing I noticed
was that all of the files reported that they had changed, because of the
way Git handles line endings. The solution is in the 
[VSCode dev containers tips and tricks](https://code.visualstudio.com/docs/devcontainers/tips-and-tricks#_resolving-git-line-ending-issues-in-containers-resulting-in-many-modified-files)
and involves adding the following gitattributes file:
```
* text=auto eol=lf
*.{cmd,[cC][mM][dD]} text eol=crlf
*.{bat,[bB][aA][tT]} text eol=crlf
```
The document notes this needs a git version greater than 2.10, but that
was released in 2016, so that should be fine! Hving said that, some
linting tools such as shellcheck complain about the extra carriage
returns. It would be nice if Git didn't try to mess with line endings.

### Git Commands

I tend to use the command line when running git. But inside the container
the ssh keys aren't set up, so git doesn't work. However the folder is
mounted inside the container as a volume, which means that we can still
use git as normal from outside the container. This is probably the
easiest thing to do.

## Additional Extensions

I also like to have some other extensions enabled while developing.
In this case Shellcheck. We can add that in by altering the 
`devcontainer.json` file from:

```
  "customizations": {
    "vscode": {
      "extensions": ["redhat.ansible"]
    }
  }
```
to
```
  "customizations": {
    "vscode": {
      "extensions": ["redhat.ansible", "timonwong.shellcheck"]
    }
  }
```

This adds the shellcheck vscode extension to the container. We can add
other extensions as necessary.
