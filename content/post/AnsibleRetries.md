---
title: "Ansible trick - Retry an intermittent error"
date: 2020-02-26T10:03:50Z
tags: ['Automation', 'Ansible','Fail','Scripting','Language']
---

We have recently been having issues with mounting windows shares. It occasionally doesn't work. We don't have access to fix it.
The playbook fails at this step. This is really annoying, because if it tried again it would probably work fine, and
the playbook could complete. It turns out that Ansible does have a way to retry. I got the idea from this [Stack Overflow
question](https://stackoverflow.com/questions/44134642/how-to-retry-ansible-task-that-may-fail)

The solution is the Ansible `retries` keyword. Here is a test of a command that intermittently fails:

```yaml
---
 - hosts: 127.0.0.1
   connection: local
   gather_facts: False
   tasks:
   - name: Test failure
     command: "bash -c 'exit $(( $RANDOM % 4 ))'"
     register: result
     retries: 5
     delay: 10
     until: result is not failed
     ignore_errors: True

   - debug: var=result
```

Here is the output. For intermittent failures like this retry is better than a rescue block because if the failure happens again the rescue block
can't be retried.

```console
[psh35@whisk ~]$ ansible-playbook test.yml
[WARNING]: provided hosts list is empty, only localhost is available.
            Note that the implicit localhost does not match 'all'


PLAY [127.0.0.1] ***********************************************

TASK [Test failure] ********************************************
FAILED - RETRYING: Test failure (5 retries left).
FAILED - RETRYING: Test failure (4 retries left).
FAILED - RETRYING: Test failure (3 retries left).
changed: [127.0.0.1]

TASK [debug] ***************************************************
ok: [127.0.0.1] => {
    "result": {
        "attempts": 4, 
        "changed": true, 
        "cmd": [
            "bash", 
            "-c", 
            "exit $(( $RANDOM % 4 ))"
        ], 
        "delta": "0:00:00.004382", 
        "end": "2020-02-26 11:23:17.888953", 
        "failed": false, 
        "rc": 0, 
        "start": "2020-02-26 11:23:17.884571", 
        "stderr": "", 
        "stderr_lines": [], 
        "stdout": "", 
        "stdout_lines": []
    }
}

PLAY RECAP *****************************************************
127.0.0.1   : ok=2  changed=1  unreachable=0  failed=0  skipped=0
              rescued=0  ignored=0   
```

## Switch off facts

Ansible takes a few seconds to gather facts. If they are not required, they can be switched off by
setting `gather_facts` to False.

## Quick digression. Booleans in Ansible

This [Stack Overflow answer](https://stackoverflow.com/questions/47877464/how-exactly-does-ansible-parse-boolean-variables) 
explains boolean variables in Ansible. I prefer to use `True` and `False` as it works
in Python, Jinja2 and YAML, and also on the command line. However, the
[Jinja2 documentation](https://jinja.palletsprojects.com/en/2.11.x/templates/#literals) says  to use
lower case for `true` and `false` (see the note at the end of the literals section). 
The problem is it often takes a little thought to understand which
language you are writing in with Ansible!

