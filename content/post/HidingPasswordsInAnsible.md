---
title: "Hiding Passwords In Ansible"
date: 2019-09-27T15:00:36+01:00
tags: ["Automation",'Scripting',"Ansible","Secrets",'Security']
---

## Setup

Start with the following playbook for demonstration purposes:

```yaml
---
 -  hosts: 127.0.0.1
    connection: local
    tasks:
    - name: This works
      command: "echo This is my secret password"

    - name: This fails
      shell: "echo This is my secret password && false"
```

And run it:

{{< highlight console "hl_lines=17 21 22" >}}
$ ansible-playbook  play.yml 

 [WARNING]: provided hosts list is empty, only localhost is
available. Note that the implicit localhost does not match 'all'


PLAY [127.0.0.1] ***********************************************

TASK [Gathering Facts] *****************************************
ok: [127.0.0.1]

TASK [This works] **********************************************
changed: [127.0.0.1]

TASK [This fails] **********************************************
fatal: [127.0.0.1]: FAILED! => {"changed": true,
"cmd": "echo This is my secret password && false",
"delta": "0:00:00.003950", "end": "2019-09-27 14:59:52.194098",
"msg": "non-zero return code", "rc": 1,
"start": "2019-09-27 14:59:52.190148", "stderr": "",
"stderr_lines": [], "stdout": "This is my secret password",
"stdout_lines": ["This is my secret password"]}

PLAY RECAP *****************************************************
127.0.0.1 : ok=2    changed=1    unreachable=0    failed=1
   skipped=0    rescued=0    ignored=0   
{{< / highlight >}}

Oops, my secret password has been logged to the standard output. If this is picked up by logging software it could be stored where people who shouldn't know the password can see it.

## Hiding the password

I will just change the two tasks to set no_log:

{{< highlight yaml "hl_lines=7 11" >}}
---
 -  hosts: 127.0.0.1
    connection: local
    tasks:
    - name: This works
      command: "echo This is my secret password"
      no_log: True

    - name: This fails
      shell: "echo This is my secret password && false"
      no_log: True
{{< / highlight >}}

And run it:

{{< highlight console "hl_lines=14 15" >}}
$ ansible-playbook  play.yml 
 [WARNING]: provided hosts list is empty, only localhost is
available. Note that the implicit localhost does not match 'all'

PLAY [127.0.0.1] ************************************************

TASK [Gathering Facts] ******************************************
ok: [127.0.0.1]

TASK [This works] ***********************************************
changed: [127.0.0.1]

TASK [This fails] ***********************************************
fatal: [127.0.0.1]: FAILED! => {"censored": "the output has been
hidden due to the fact that 'no_log: true' was specified for this
result", "changed": true}

PLAY RECAP ******************************************************
127.0.0.1 : ok=2    changed=1    unreachable=0    failed=1
   skipped=0    rescued=0    ignored=0   
{{< / highlight >}}

Brilliant" My password isn't logged!. But, I have an error in my script that I want to debug.
One of those really annoying ones where it works from the command prompt but not from Ansible.
If only I could see the output, I would be able to debug it.

## Hide the password when I want to

{{< highlight yaml "hl_lines=7 11" >}}
---
 -  hosts: 127.0.0.1
    connection: local
    tasks:
    - name: This works
      command: "echo This is my secret password"
      no_log: "{{ showpass is not defined }}"

    - name: This fails
      shell: "echo This is my secret password && false"
      no_log: "{{ showpass is not defined }}"
{{< / highlight >}}

Here I have introduced a variable. If it isn't defined it is false and produces the censored output
as above. But if I define the variable and run it, as I do below, I get the error message 
(including the password) as at the top:

```console
ansible-playbook --extra-vars "showpass=True" play.yml
```
