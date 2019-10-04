---
title: "Ansible Tricks"
date: 2019-10-04:11:36+01:00
tags: ['Ansible','Automation','Performance','Scripting','Secrets','Security']
---

# Some Useful Ansible tricks

## Including a playbook

I am working on extending my automation to do some new things. We refresh test environments from production.
Previously we used to copy all the code from production back to development. Now we build the VMs from scratch,
so we don't need to do this any more, but it would be convenient to call this in the middle of the
refresh.

You can easily do this using import_playbook. This makes the playbook work in the same way as if it was run.

See the documentation for the difference between [include and import](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html).

## Starting at a particular task.

When developing a playbook, it is useful to be able to start off from where it failed. I typically have a
silly error in a play which I notice as soon as it fails. So long as all the tasks are named uniquely,
I can run the following:

```console
ansible-playbook -i inventory playbook.yml --start-at-task="Task Name"
```

The role can be included as well, in case two roles have the same task in:

```console
ansible-playbook -i inventory playbook.yml --start-at-task="role : Task Name"
```

This is the same format Ansible prints out the task name on the screen.

Sometimes the error is caused by an error in a script that was deployed by a previous step. In that case,
obviously once the script is corrected the playbook needs to be restarted from the step that deploys
the script.

## Timings and profiling.

Often I would like to see how long something took. If the playbook takes a long time to run, it is useful
to see what the slowest plays are. To discover this, simply put the following in `ansible.cfg`

```INI
callback_whitelist = profile_tasks
```

## Ansible Configuration File

I am not the sysadmin to my management server, and even though I know the root password
I don't want to step on anyone's toes by changing `ansible.cfg` for everyone.

It turns out that putting `ansible.cfg` into the same path as the playbook means it is 
read and overrides the default, so no need to mess with anyone elses settings, or
even to becom root.

## Viewing stdout of command/shell/script

When testing anisble playbooks, it is often useful to be able to see the stdout of
commands to see what is going wrong. To do this call ansible with a single -v

```console
ansible-playbook -vi inventory playbook.yml
```

## Human readable output

Once the above is done, the output is sent to the screen in a json format. It would be
nice to get something formatted as it came out of the script.

Here is how:

```console
ANSIBLE_STDOUT_CALLBACK=debug ansible-playbook -vi inventory playbook.yml
```

## Weird Error

I got the following error running a playbook:

```console
fatal: [hostname]: FAILED! => {"msg": "Failed to change ownership of the
temporary files Ansible needs to create despite connecting as a privileged
user. Unprivileged become user would be unable to read the file."}
```

This happened because the `become` user didn't exist, because I had missed out running
the part that created it. It is reasonable for it to fail, but the error isn't very clear.
Hopefully this will help someone in the future (Probably me!)
