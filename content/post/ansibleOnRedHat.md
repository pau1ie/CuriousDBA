---
title: "The Problem with Ansible on RedHat"
date: 2024-12-11T09:59:47+00:00
tags: ['Ansible','Automation']
---

Normally newer versions of Operating systems have newer packages. But not RedHat
when it comes to Ansible. On my workstation:

```
$ cat /etc/redhat-release 
Red Hat Enterprise Linux release 8.10 (Ootpa)
$ ansible --version
ansible [core 2.16.3]
  config file = /etc/ansible/ansible.cfg
  ...
  python version = 3.12.6
  jinja version = 3.1.2
  libyaml = True
```

But on the management server:

```
$ cat /etc/redhat-release 
Red Hat Enterprise Linux release 9.4 (Plow)
$ ansible --version
ansible [core 2.14.17]
  config file = /etc/ansible/ansible.cfg
  ...
  python version = 3.9.18
  jinja version = 3.1.2
  libyaml = True
```

Wait, what? The older operating system has a newer version of Ansible, and Python?

This is what RedHat is doing these days. They decided to keep stable Ansible
and Python versions to match the rest of the operating system. 

But this causes real problems.

```
ansible-galaxy collection install --upgrade ansible.windows

ansible-playbook <do something that needs ansible.windows>

[WARNING]: Collection ansible.windows does not support Ansible version 2.14.14
```

RedHat [say](https://www.redhat.com/en/blog/updates-using-ansible-core-in-rhel):

> RHEL 9.3 and later are not planned to receive new Ansible Core
> releases, and will continue utilizing Ansible Core 2.14 which is
> planned to be supported for the remainder of the RHEL 9 lifecycle.

RedHat also say:

Ansible 2.14 is 
[unmaintained and end of life](https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html).

This clearly a different definition of *supported* than I am used to.

Why does it install collections it can't use?

They don't even mention in the change log which version of Ansible is
supported by the collections, which would have been useful. They also
don't maintain separate documentation for the supported versions like
they used to.


## How to fix it

I install collections as a user, which by default end up in 
`~/.ansible/collections/ansible_collections`. I discovered that the compatible
version is mentioned in the collection in `meta/runtime.yml`. So I can check
the following:

```
grep requires_ansible ~/.ansible/collections/ansible_collections/*/*/meta/runtime.yml
```

Then back off the versions until I find one that supports the version
of Ansible RedHat installs. It seems that Ansible 2.14 was de-supported
around November 2023, but I just went back until I found the first version
that supported Ansible 2.14. For the collections I checked these were:

| Collection  | Version |
| --- | --- |
| ansible.windows | 2.3.0 |
| community.general | 9.5.2 |
| community.windows | 2.2.0 |

It looks like community.general version 9.5 is being maintained by RedHat
for Ansible 2.14, as there have been recent releases. But I couldn't find
this documented anywhere.