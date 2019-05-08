---
title: "Ansible Strings And Numbers"
date: 2018-07-10T11:06:14+01:00
tags: ["Automation","Ansible"]
---

I had a playbook fail because of the error ```dict object has no element```

It took me ages to debug it, but the problem is that I had defined the index as a string, but was trying to
match it  to a number. To fix it, I only had to quote the version in the inventory, and everything worked.

I was only caught by this because I was using an old inventory file to test with. I think it used to
automatically cast numbers to string, but that stopped a while ago.

var.yml 
---

{{< highlight yaml>}}
--- 
 - hosts: webservers
   tasks:

    - debug: var=toolsversion

    - debug: var=installed_homes

    - debug: var=installed_homes["8.55"]

    - debug: var=installed_homes[toolsversion]

    - debug: var=item
      with_items:  "{{ installed_homes[toolsversion] }}"
{{< /highlight >}}
 

group_vars/webservers.yml
---

{{< highlight yaml>}}
  installed_homes:
    "8.55":
      - "C:\\psft\\pt\\bea\\tuxedo"
      - "C:\\psft\\pt\\oracle-client\\12.1.0.2"
    "8.56":
      - "C:\\psft\\pt\\bea\\tuxedo"
      - "C:\\psft\\pt\\oracle-client\\12.1.0.2"

  toolsversion: 8.55
{{< /highlight >}}


inventory 
----

{{< highlight ini >}}
[webservers]
whisk
{{< /highlight >}}

Run Output
---

{{< highlight console >}}
$ ansible-playbook -i inventory var.yml 

PLAY [webservers] ***************************************************

TASK [Gathering Facts] **********************************************
ok: [whisk]

TASK [debug] ********************************************************
ok: [whisk] => {
    "toolsversion": 8.55
}

TASK [debug] ********************************************************
ok: [whisk] => {
    "installed_homes": {
        "8.55": [
            "C:\\psft\\pt\\bea\\tuxedo", 
            "C:\\psft\\pt\\oracle-client\\12.1.0.2"
        ], 
        "8.56": [
            "C:\\psft\\pt\\bea\\tuxedo", 
            "C:\\psft\\pt\\oracle-client\\12.1.0.2"
        ]
    }
}

TASK [debug] ********************************************************
ok: [whisk] => {
    "installed_homes[\"8.55\"]": [
        "C:\\psft\\pt\\bea\\tuxedo", 
        "C:\\psft\\pt\\oracle-client\\12.1.0.2"
    ]
}

TASK [debug] ********************************************************
ok: [whisk] => {
    "installed_homes[toolsversion]": "VARIABLE IS NOT DEFINED!"
}

TASK [debug] ********************************************************
fatal: [whisk]: FAILED! => {"msg": "dict object has no element 8.55"}
        to retry, use: --limit @/home/me/ansible-test/var.retry

PLAY RECAP **********************************************************
whisk               : ok=5    changed=0    unreachable=0    failed=1   
{{< /highlight >}}


The Fix
---

So the fix is, change the variables to quote the version number, because it needs to be a string.

{{< highlight yaml >}}
  toolsversion: "8.55"
{{< /highlight >}}

Simple. Nobody would spend an entire morning working this out...
