---
title: "PeopleSoft Change Assistant"
date: 2026-01-06T11:59:47+00:00
tags: ['Ansible','Automation','Change Assistant','PeopleSoft','Automation']
---

I am looking into running Change Assistant automatically, but also with
the option of manually running it if necessary. The problem
with this is permissions. We are supposed to log in as our own users
and not as admins if we can help it.


## Inspiration

The clever people at
[psadmin.io](https://psadmin.io)
came up with 
[a method](https://psadmin.io/2017/03/09/running-change-assistant-without-as-administrator/)
which is pretty good start. There are a couple of refinements to this
I felt needed to be made. These mean that the solution is a bit more
complicated than the above.

First, it grants control to all users on the folder. This is not
great from a security perspective. While we would generally hope that
only proper admins would use that VM, in practice the VM is part of a
domain, so we are granting access to anyone who can log in from that
domain, which is likely too wide.

Secondly, when Change Assistant runs, it creates and uses a number of
files in the OS. We should ensure that permissions on these are set
correctly to ensure any admins can use them.


## Improvements

First we need to find a group which contains the admins. I am going to
call it `campusadmins` for the sake of this document. Then we need to
look into the files and folders touched by change asistant.


### Fork - Change Assistant Set Up

I mentioned in a previous article how I
[set up Change Assistant](../changeassistantbasicconfiguration/).
The folders and files set up there are important below.


### File and Folder Locations

| File | Notes |
| ---- | ---- |
| `C:\Download` | Download directory as per the ini file |
| `C:\Output` | Output directory as per the ini file |
| `C:\Program Files\PeopleSoft\Change Assistant` | Where change assistant was installed |
| `C:\PS` | This seems to be a default folder created by peoplesoft |
| `C:\psreports` | Default reports folder |
| `C:\psft\PS_CFG_HOME` | Config home |
| `C:\Staging` | Staging directory as per the ini file |
| `C:\temp\datamove.log` | Datamover log created by change assistant |
| `C:\temp\ununstlog` | Another log file created by change assistant |

So ideally we need all of these to be readable and writable by any
user of change assistant. 


### Formulating a Solution

To do this we first enable inheritance on
the folders. I use Ansible to do this, I am sure it is possible with
powershell.

```yaml
- name: Enable inheritance on each target path
  ansible.windows.win_acl_inheritance:
    path: "{{ item }}"
    state: present
  loop: "{{ acl_existing_paths }}"
```

Then we clear any read only flags:

```shell
attrib -R {{ item }} /S /D
```

Lastly we allow the group to modify the files under each of the locations
above.

```yaml
- name: Grant acl rights to identities on target paths
  ansible.windows.win_acl:
    path: "{{ item }}"
    user: campusadmins
    rights: "Modify"
    type: allow
```


## Implications

* Anyone in the group can use change assistant.
* Anyone not in the group can not alter the way Change Assistant runs
* One admin should be able to take over from another.
