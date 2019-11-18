---
title: "Gitlab Inventories, Private Keys and Secrets"
date: 2019-11-18T10:03:50Z
tags: ['Secrets', 'Ansible', 'GitLab','Automation','Scripting','Security']
---

# Secrets and the Problem with Build Servers
I am not totally sure I understand this, so let's see if I do when I write it down.

I decided that gitlab needs to stop accessing hosts using keys from the gitlab runner, because this means:

1. Everything on gitlab runner can access every VM
2. If the gitlab runner moves, nothing can access any VM

So 1 is too open, and 2 just doesn't work. While I could configure a gitlab runner per environment, it also
strikes me that the secrets are in the ansible build directory and shouldn't be. We need a better 
approach.

## Planned Approach

Gitlab will have the public key in armoured format (i.e. in 7 bit ascii) in a file variable (i.e. it writes the
contents pasted into the box into a file, and sets a variable to the location of that file. 
It will have a blank password, as adding a password seems to add complication for no benefit.

The secrets will be moved to the inventory repository. Programs will be given access to  
the secrets, so they can be used to build VMs. Also added to these secrets will be the 
password (Access token) to any git repositories it needs, and the root password of any VMs
it needs to build.

Hopefully we can set up a way for the same process to be run from gitlab runner, or an
engineers PC, so long as they have access to the secrets (via regpg) and the git repositories (via e.g. host keys).

## In Practice

I discovered that you can tell gpg to use another keyring. This will be very useful.
So I do this by:

```bash
$ mkdir gpghome
$ chmod 700 gpghome
$ export GNUPGHOME=`pwd`/gpghome/
$ gpg --gen-key
gpg (GnuPG) 2.0.22; Copyright (C) 2013 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

gpg: keyring `./gpghome//secring.gpg' created
gpg: keyring `./gpghome//pubring.gpg' created
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: Gitlab Runner
Email address: dba@cam.ac.uk
Comment: To access secrets
You selected this USER-ID:
    "Gitlab Runner (To access secrets) <dba@cam.ac.uk>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
You need a Passphrase to protect your secret key.

gpg: ./gpghome//trustdb.gpg: trustdb created
gpg: key XXXXXXXX marked as ultimately trusted
public and secret key created and signed.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
pub   4096R/XXXXXXXX 2019-11-08
      Key fingerprint = XXXX XXXX XXXX XXXX XXXX  XXXX XXXX XXXX XXXX XXXX
uid                  Gitlab Runner (To access secrets) <dba@cam.ac.uk>
sub   4096R/XXXXXXXX 2019-11-08
```
Now the key is created, let's export it:
```bash
$ gpg --output runner.asc --armor --export dba@cam.ac.uk
$ gpg --output runnersec.asc --armor --export-secret-keys dba@cam.ac.uk
```

This creates runner.asc, which is the armoured public key file, and runnersec.asc which 
is the secret key file. This latter key is pasted into the gitlab variable, and labelled as a
file variable.

To give the key access to the secrets, in another window (Or having unset GNUPGHOME, so 
I can access my key that can give the other key access) I [enrol an admin](../enrolinganewadmin/):

```bash
$ gpg --import runner.asc
$ regpg add XXXXXXXX
$ regpg recrypt -r
```

So now I should be able to decrypt a key from the gitlab runner job. Let's 
 test:

```bash
$ export GNUPGHOME=`pwd`/gpghome/
$ gpg -qqd gpg-preload.asc && echo ""
True
```

`gpg-preload.asc` contains the value `True` as per the 
[regpg documentation](https://dotat.at/prog/regpg/doc/tutorial.html). 

# Setting up GitLab

This is using the key I created for GitLab runner because the `GPGHOME`
variable is set. I can do this for a GitLab CI job. As I say, the contents
of `runnersec.asc` needs to be pasted into a GitLab variable. In the project, navigate to
`Settings -> CI/CD -> Variables`. Expand the variables section, and type in a name
for the variable in the Key field. Paste the private key into the value, and click save.

## Assembling the Environment with a script
Now when the CI is run, a file is created with these contents. They key can't
be used as it is, we need to import it into a keyring. We discovered above
that it is possible to create a special `GPGHOME` for this job. So we create a script
that looks like this:

```bash
#!/bin/bash
# Set up the GPG home directory.
if [[ -f $PRIVATE_KEYFILE ]]
then
  export GNUPGHOME=`pwd`/gpghome
  rm -rf $GNUPGHOME

  mkdir $GNUPGHOME
  chmod 700 $GNUPGHOME

  # Import the private key
  gpg --import $PRIVATE_KEYFILE
  gpg --list-keys
fi

# Check we can see inside the encrypted file
gpg -qqd gpg-preload.asc && echo ""
echo "Above line should read True"

rm -rf $GNUPGHOME
```

As I did above manually, I tested using the `gpg-preload.asc` file that `regpg`
sets up. I proved the gpg decrypt works! I can detect if we are running from
GitLab runner by checking if my file variable exists. If not, I assume the
user has access to the secrets through their own GPG key.

## Getting the Other Repositories

As I mentioned, we also need to access other repositories, the one
containing the playbooks and the one with the code to deploy.

If we are running in GitLab runner, the job can access other repositories using
the variable `CI_JOB_TOKEN`. A user running the job from their PC would use the host keys
they have set up. We need to detect whether we are
running as a user or in gitlab runner. I use the following code to do this
in ansible:

```yaml
---
# Use templating to decide whether this is being run as part of a 
# GitLab job. If it is, use the CI_JOB_TOKEN. If not, then assume
# the user has set up ssh keys.
  citk: "{{ lookup('env','CI_JOB_TOKEN') }}"
  tokuri: "https://gitlab-ci-token:{{ citk }}@gitlab.developers.cam.ac.uk/"
  sshuri: "git@gitlab.developers.cam.ac.uk:"
  gitlab_uri: "{% if citk is defined and citk|length %}{{ tokuri }}{% else %}{{ sshuri }}{% endif %}"
```

So if the build is being run from gitlab runner, `gitlab_uri` becomes:<br/>`https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.developers.cam.ac.uk/`. Otherwise
it is set to: `git@gitlab.developers.cam.ac.uk:` Hopefully have host keys set up. The
repository name can then just be appended to this URI.

## Ansible Passwords

The reason for this was to ensure we don't have to log in using
host keys because any process running on a gitlab runner would be able
to access that host. So let's address that now.

I decided that if Ansible connected with a username and password rather
than with host keys, we would be able to control which hosts each CI job
connects to. So we need to configure two Ansible variables. This can be done
in the inventory.
```yaml
  vars:
    ansible_user: username
    ansible_password: password
```

But of course the password should be encrypted. Create a file with 
the encrypted password:

```bash
set +o history
echo -n "password" | regpg encrypt secrets/ansible_password.asc
set -o history
```

This is on my desktop, so I don't mind that the password is briefly in the process
listing, or in memory, or in the clear on the screen. I do like my passwords
not to end up in my bash history, so that is what the `set +o history` does.
Using bash without history is surprisingly annoying, so I switch it back on again.

The password doesn't contain a carriage return, so `echo -n` ensures it doesn't
write a newline at the end of the string.

Now the password is encrypted, so we can change the inventory:

```yaml
  vars:
    ansible_user: username
    ansible_password: "{{ 'secrets/ansible_password.asc' | gpg_d }}"
```

Beautiful. The windows username and password is different, so I have a similar setting
in the windows group which defines that, and it overrides this one nicely.

## Conclusion

The testing so far shows the approach is valid. No doubt there is some work
to get everything working, but hopefully that won't take too long!