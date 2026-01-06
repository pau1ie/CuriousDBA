---
title: "Secrets, Ansible and Regpg"
date: 2019-06-07T10:13:36+01:00
tags: ["Automation","Ansible","Secrets","VM","GitLab","Security",'GPG']
---

I like Ansible, but I find one omission in the way it works is the lack of a way to manage secrets,
i.e. things like private keys, passwords, and access tokens.

I stored passwords in the inventory file. This
means the inventory file is large, and can't be checked into version
control, which makes it difficult to manage.

My first test is to create another git repository to check out onto the VM. This contains some application
code which needs to be installed. To check this out  to any one of 100 or so VMs I am using
[gitlabs deploy token](https://docs.gitlab.com/ee/user/project/deploy_tokens/)
functionality, which creates a URL like this:

```
https://user:pass@gitlab.developers.cam.ac.uk/path/to/repo.git
```

The challenge is to store this username and password encrypted in version control.


## Regpg to the Rescue

My colleague [Tony Finch](https://dotat.at) has written a perl script
called [regpg](https://dotat.at/prog/regpg) to deal with this. The idea is that the secrets are
stored in the repository encrypted, and can be decrypted as they are needed.

**Edit:** It turns out this was wrong - I was trying to run the source not the compiled version.
The file to run is repgp - not repgp.pl

<s>
I am using Red Hat Enterprise Linux 7. The latest version of regpg requires a later
version of perl than is supplied with this version of Red Hat. I am using
[version 106 of regpg](https://gitlab.developers.cam.ac.uk/fanf2/regpg/blob/fd7274faa42c37883962b10bcd065cf60b33878f/regpg.pl)
from September 2018.
</s>


## Set up the Project

To set it up I more or less followed the [tutorial](https://dotat.at/prog/regpg/doc/tutorial.html).

I generated a key on my workstation. gpg-agent was already installed and running, so that was all good.
I already had regpg installed using the _quick and dirty way!_

I already had a project, so I changed to the project directory and ran:

```bash
regpg init
```

Then on the management server I wanted to be able to add a key without a password.
This is problematic as if anyone gets hold of the private key we will need to change all the passwords.
The benefit is it can be run noninteractively, which will allow VMs to be recreated overnight as I want.


## Create a Key on the Management Server

On the management server I tried to create the key as the jenkins user. This didn't work. When it
prompted for the password, it said:

```
You need a Passphrase to protect your secret key.

gpg: cancelled by user
gpg: Key generation canceled.
```

This is because you can't log in as the Jenkins user, so I log in as root and run:

```bash
su -s /bin/bash jenkins
```

to create a shell as jenkins. gpg tries to  take control of the tty, but can't because it is
owned by root.

I created the key as root. Even that was problematic
because gpg couldn't connect to the gpg_agent:

```
You need a Passphrase to protect your secret key.

gpg: can't connect to the agent: IPC connect call failed
gpg: problem with the agent: No agent running
gpg: can't connect to the agent: IPC connect call failed
gpg: problem with the agent: No agent running
gpg: Key generation canceled.
```

I ended up restarting gpg-agent (though presumably there should be a way to find out how to set
the environment variable without such drastic action). When it is started, gpg_agent supplies
environment variable assignments which enable the connection to work:

```console
# pkill -9 gpg-agent
# eval $(gpg-agent --daemon )
```

The ```GPG_AGENT_INFO``` variable is now set, so creating the key worked. I had already checked
that the keyring was empty, so I used the following:

```console
# gpg2 --homedir /var/lib/jenkins/.gnupg --gen-key 
gpg: WARNING: unsafe ownership on homedir `/var/lib/jenkins'

... Some stuff snipped ...

You don't want a passphrase - this is probably a *bad* idea!
I will do it anyway.  You can change your passphrase at any time,
using this program with the option "--edit-key".

We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: /var/lib/jenkins/trustdb.gpg: trustdb created
gpg: key XXXXXXXX marked as ultimately trusted
public and secret key created and signed.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
pub   4096R/XXXXXXXX 2019-06-05
      Key fingerprint = XXXX XXXX XXXX XXXX XXXX  XXXX XXXX XXXX XXXX XXXX
uid                  jenkins (ansible) <jenkins@host>
sub   4096R/XXXXXXXX 2019-06-05
```

I got told off for not setting a passphrase. If I ever lose the private key I will have to change
all the passwords, as they will be compromised.

The files are owned by root. I changed the owner:

```console
# cd /var/lib/jenkins/.gnupg
# chown jenkins:jenkins secring.gpg pubring.gpg~ \
     pubring.gpg trustdb.gpg random_seed
```

## Enrolling the Key into the Repository.

Now I need to add the public key to the keyring in the git repository.
This will let the management server decrypt the secrets.
[Tony's tutorial](https://dotat.at/prog/regpg/doc/tutorial.html)
explains how to do this. Here is how I did it.

On the management server I exported the public key:

```console
$ gpg --list-secret-keys
/var/lib/jenkins/.gnupg/secring.gpg
-----------------------------------
sec   4096R/XXXXXXXX 2019-06-05
uid                  jenkins (ansible) <jenkins@host>
ssb   4096R/XXXXXXXX 2019-06-05

$ gpg --output jenkins.asc --armor --export jenkins@host
$ cat jenkins.asc 
-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: GnuPG v2.0.22 (GNU/Linux)

... Loads of random alphanumeric characters ...

-----END PGP PUBLIC KEY BLOCK-----
```

I copied and pasted the above key into a file on my workstation, and enrolled it into the
repository keyring:

```console
$ gpg --import jenkins.asc
gpg: key XXXXXXXX: public key "jenkins on host (ansible) <jenkins@host>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
$ cd path/to/git/repo
$ regpg add XXXXXXXX
gpg: key XXXXXXXXXXXXXXXX: public key "jenkins on host (ansible) <jenkins@host>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
```

## Creating Some Secrets.

I created a directory called secrets in my project to store them in,
and created a file for the username and another for the password:

```console
$ mkdir secrets
$ echo -n "Username" | regpg encrypt secrets/unixhomedeploytokenuser.asc
$ echo -n "MyPassword" | regpg encrypt secrets/unixhomedeploytokenuser.asc
$ git commit -m "Set up regpg"
[Development 5c42e2d] Set up regpg
 11 files changed, 324 insertions(+), 2 deletions(-)
 create mode 100644 ansible.cfg
 create mode 100644 gpg-preload.asc
 create mode 100644 gpg-preload.yml
 create mode 100644 library/gpg_d.py
 create mode 100644 plugins/action/gpg_d.py
 create mode 100644 plugins/filter/gpg_d.py
 create mode 100644 pubring.gpg
 create mode 100644 secrets/unixhomedeploytokenpass.asc
 create mode 100644 secrets/unixhomedeploytokenuser.asc
$ git push
```

The ```echo -n``` is necessary because otherwise the secret will have a carriage return on the end which
doesn't work!

## Making Ansible use the Secrets

I realise I need a play to actually do the work. Here it is:

```yaml
---

  - name: Clone homes from git repository
    git:
      repo: "https://{{ 'secrets/unixhomedeploytokenuser.asc' | gpg_d }}:{{ 'secrets/unixhomedeploytokenpass.asc' | gpg_d }}@gitlab.developers.cam.ac.uk/uis/ea-dba/camsis/unix-homes.git"
      depth: 1
      force: yes
      dest: /psoft/apphomes
```

## Conclusion

I found regpg a really nice way to manage secrets. I really like the way it hooks into ansible (and git).
It encourages you not to store the unencrypted secrets. It is really easy to manage with one secret per
file. There is always going to be a certain amount of overhead managing keys and key rings, but I think
it is worth it to be able to store passwords in version control.
