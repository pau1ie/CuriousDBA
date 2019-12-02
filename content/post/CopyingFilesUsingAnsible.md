---
title: "Copying files using Ansible"
date: 2019-12-02T09:34:09Z
tags: ['Ansible','Automation','GitLab','Fail','Scripting']
---

## Introduction

Copying large numbers of files around - you would have thought this would be easy using
Ansible. Most of what we do with Ansible is copying files and editing them. 
It turns out to be rather difficult to get right.

## Copy module

The [copy module](https://docs.ansible.com/ansible/latest/modules/copy_module.html)
seems like the correct thing to use at first glance. It copies files from the build host to remote locations.
So I can clone the repository of files to deploy and send them off to where they need to go? Brilliant!
There is a [note](https://docs.ansible.com/ansible/latest/modules/copy_module.html#notes)
at the bottom

> The copy module recursively copy facility does not scale to lots (>hundreds) of files.

I counted my files. There were fewer than 100, so great! But after an hour waiting for them to
deploy I decided I needed another approach. The solution seems to be the synchronize module, which
is referenced in the [See Also](https://docs.ansible.com/ansible/latest/modules/copy_module.html#see-also)
section of the copy module's documentation.

## Synchronize

Again, at first glance the
[synchronize module](https://docs.ansible.com/ansible/latest/modules/synchronize_module.html)
looks perfect. It uses [rsync](https://rsync.samba.org), everyone's favourite tool to copy files
across the network. The problem is that this module is different to every other ansible module.
Ansible normally creates its own connection to the remote host and uses that to transfer python
modules which talk across that connection. But synchronise uses rsync to do the communication.
This is not an implementation detail, it means syncronize doesn't work in the way we have
come to expect an Ansible module to work. I found the following:

- Become doesn't work. You have to manually change the permissions afterwards.
- Because of this files are reported as changed when in fact they are not.
- Password connections don't work. I am not sure if this was a bug, but everything else in
  Ansible works when you specify a username and password, but not rsync. It hangs presumably waiting
  for a password prompt somewhere.

The [synopsis](https://docs.ansible.com/ansible/latest/modules/synchronize_module.html#synopsis) says:

> This module is not intended to provide access to the full power of rsync, but does make the 
> most common invocations easier to implement. You still may need to call rsync directly via
> command or shell depending on your use case.

So what to do? Here is the solution I used:

## Rsync

As suggested above, in some cases it is worth using [rsync](https://rsync.samba.org/). I have set up a process that:

- Set up an SSH key locally (If running from the gitlab runner, otherwise I assume the user has ssh keys
  already set up and use that).
- Set up the ssh keys in the authorized keys file for the relevant (remote) users to allow passwordless connection
- Run rsync using the appropriate key file.
- Remove the ssh keys from the authorized keys file.

Rsync is called as follows:

```yaml
  - name: Deploy files to websites
    command: # noqa 303 I want to use rsync!
      argv:
      - "rsync"
      - "-crlp"
      - "--rsh=ssh -i {{ sshkey }}"
      - "--out-format=<<CHANGED>>%n%L"
      - "{{ from }}"
      - "{{ user }}@{{ inventory_hostname }}:{{ to }}"
    delegate_to: localhost
    register: syncunix
    changed_when: "'<<CHANGED>>' in syncunix.stdout"
```

The noqa stops [ansible-lint](https://docs.ansible.com/ansible-lint/) suggesting you use the synchronize
module. I am using the
[ansible command module](https://docs.ansible.com/ansible/latest/modules/command_module.html#command-module)
to run rsync. It just takes the list under argv, and runs the command. The 
[rsync manual page](https://download.samba.org/pub/rsync/rsync.html) gives more
information about rsync, and there is more 
[documentation on the rsync website](https://rsync.samba.org/documentation.html). Here is a brief
description of the parameters I use:

- `c` Uses a checksum to compare files. The default uses the modification time, which is likely to be different
  because I am deploying files checked out of source control.
- `r` Recursive copy. Required to copy directories with files in.
- `l` Copy symbolic links.
- `rsh=ssh -i {{ sshkey }}` Rsync uses ssh to connect. We can pass parameters to ssh with this option. When I
  run the process from gitlab runner, I want to use a different location for the ssh key. This option sets
  that location to an Ansible variable.
- `--out-format=<<CHANGED>>%n%L` The main point of this is to ensure the string `<<CHANGED>>` appears if
  there is a change to any of the files.
- `{{ from }}` This is a variable containing the location of the files to copy.
- `"{{ user }}@{{ inventory_hostname }}:{{ to }}"` Builds up the remote location where the files will be copied to
from variables. Since rsync will connect as the user in the variable, it needs to have the ssh key set up.
- `delegate_to: localhost` The rsync needs to be run from the controller host, and copy files to the remote host.
- `register:` The output needs to be saved so that it can be referred to in changed_when.
- `changed_when` works with the out-format and register to look in the output to see if any lines have changed_when.

## Conclusion

It is a shame that copying files around is such a pain in Ansible. Presumably this is a difficult problem,
or the developers would already have solved it. It would be nice to have some rsync like behaviour which
uses Ansible modules so that the behaviour is consistent with the rest of Ansible. In the mean time, it seems
easier to use rsync natively, and with a little effort and know how it is possible to get it to fit into
the Ansible way of doing things.