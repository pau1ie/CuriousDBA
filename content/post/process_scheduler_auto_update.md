---
title: "Process Scheduler Auto Update"
date: 2024-04-26T16:16:00+00:00
tags: ['PeopleSoft','Automation','Ansible','Windows','Jinja']
---

I have been setting up process monitor auto update, and have managed to
get it to work - I can see the process status updates in the process
monitor screen. Here is what I did.

Oracle support document id 
[2772617.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2772617.1)
explains how to set this up manually. I wanted this to work as part of
the automated build, which means supplying the parameters as part of
`psft_customisations.yml`

## A Gotcha!

We have multiple application servers running on the same port
(but on different VMs).
This means we need different domain IDs for each process scheduler
because domain ID and port are used as a key. I would have thought
that the hostname should be included to make this a unique identifier,
but Oracle have chosen not to do that. Note that domain ID is
different from domain name. Oracle documentation suggests using
the database name in lower case. The DPK default is `APPDOM` (which
is also the default domain name). If either of these is used,
when you set up the inter domain event credentials on the process
scheduler and configure the domain (for example by running):

```bash
psadmin -p configure -d PRCSDOM
```

the following error messages will appear (wrapped for readability):

```console
CMDGW_CAT:1521: ERROR: Invalid parameter:
    Entry in DM_REMOTE (AS_APPDOM_9000) appears more than once
CMDGW_CAT:1522: ERROR: Invalid parameter:
    Entry in DOMAIN_ID (APPDOM_9000) appears more than once
CMDGW_CAT:1546: ERROR: dmloadcf: Above errors found during syntax checking
```

## Setting Domain IDs

### Application Server

So I have to think of a way to make the domain id unique. We name our
VMs with a number on the end, so I use the last character of the
hostname together with the first seven characters of the environment
name. In our Jinja template this ends up as:

```jinja
      Domain Settings/Domain ID:    "{{ environment_name[:7] | lower }}
        {{- inventory_hostname_short[-1] }}"
```

We set `environment_name` in our Ansible inventory, and 
`inventory_hostname_short` is set by ansible. Of course I need to
remember that this won't work if we have more than 10 application
server VMs.

As noted above, the domain name can be different to the domain ID,
so we keep a consistent name to ensure that the paths are
the same on each VM.

### Process Scheduler

We only have two process scheduler VMs, one on Unix and one on Windows,
so these have hardcoded domain IDs in the `psft_customizations.yml`

## Setting the Inter-Domain Event Credentials

### Credential String Format

It is these domain IDs which need to go into the credential string.
It appears to be the following format for the process schedulers:

```jinja
PRCS_{{ domain_id }}_{{ prcs_notification_port }}|
    {{- hostname }}:{{ prcs_notification_port }}
```

e.g.

```
PRCS_PSUNX_8988|testpsunx.example.com:8988
```

And the following for the application serves:

```jinja
{{ domain_id }}_{{ apps_listen_port }}|{{ hostname }}:{{ apps_notification_port }}
```

e.g.
```
testenv1_9000|testapps1.example.com:7988
```

Remember this credential string is what we are talking *to*,
so we need to specify
the application server hosts in the process scheduler configuration, and
vice versa.

### In the Process Scheduler Customisations YAML file

In the `psft_customizations.yaml` jinja template for ansible, we need to write
something like this:

```jinja
      Inter-Domain Events/Application Server Credentials: "
      {#- Whitespace control means this should all end up on one line -#}
      {%- for server in groups['app_servers'] -%}
         {{- environment_name[:7] }}
         {{- server.split('.')[0][-1] -}}
         _
         {{- apps_listen_port -}}
         |
         {{- server -}}
         :
         {{- apps_notification_port }}
         {%- if not loop.last -%}
           ,
         {%- endif -%}
      {%- endfor -%}
      "

```

So we are looping through the application servers, and filling in the
environment name, the ports and the server names. Each credential is
separated by a comma, but we don't want a comma at the end of the line.

it generates the string (wrapped for readability).

```yaml
      Inter-Domain Events/Application Server Credentials:
        "testenv1_9000|testapps1:7988,testenv2_9000|testapps2:7988"
```


### In the Application Server Customisations YAML file

Since in our setup we always have a Unix process scheduler, and
normally also a Windows one, I hardcode that information:

```jinja
      Inter-Domain Events/Process Scheduler Credentials: PRCS_PSUNX_8988|
        {{- groups['unix_process_servers'][0] -}}
        :
        {{- prcs_notification_port }}
        {%- if 'windows_process_servers' in groups -%}
          ,PRCS_PSNT_
          {{- prcs_notification_port -}}
          |
          {{- groups['windows_process_servers'][0] -}}
          :
          {{- prcs_notification_port -}}
        {% endif %}
```

This renders the line like this (Wrapped for readability):

```yaml
      Inter-Domain Events/Process Scheduler Credentials:
        PRCS_PSUNX_8988|:8988,PRCS_PSNT_8988|:8988
```


### Checking it Out

I have saved the best till last.
It is really useful to check snippets of jinja configuration like this.
 As I write this there
is a useful [jinja test parser](http://jinja.quantprogramming.com/).
It's 
[source is on GitHub](https://github.com/mzganec/jinja-live-parser?tab=readme-ov-file).
Note that it does appear to post the contents to the server, so don't
put anyting sensitive into it!
Paste the following into the *Values YAML* box (to simulate the ansible
inventory):

```yaml
groups:
  app_servers:
    - testapps1
    - testapps2
  unix_process_servers:
    - testpsunx
  windows_process_servers:
    - testpsnt

inventory_hostname_short: hostname1
hostname: hostname1.example.com
environment_name: testenv
apps_listen_port: 9000
apps_notification_port: 7988
prcs_notification_port: 8988
domain_id: domain
```

Then any of the snippets above can be pasted into the
template box, and the template will be rendered! Try
changing the names, the number of application servers
etc. to see the effect! How amazing is that!

## Troubleshooting

### Windows

I had issues with Windows - The application server could
not connect  to the process scheduler, despite them both being in the
same VLAN (i.e., there was no firewall between then to block traffic)

The following error (wrapped for readability) was written to the
application server TUXLOG file:

```
LIBGWT_CAT:1249: WARN: Connection to PRCS_PRCS_PSNT_8988
      address //testpsnt:8988 failed, Network error(115)
LIBGWT_CAT:1304: WARN: No more remote domain address for remote domain
      PRCS_PRCS_PSNT_8988
```

I remembered that Windows has it's own firewall (So does Linux in fact)
and it turned out that this is what was blocking access. Windows doesn't
have an equivalent to netcal -l, but 
[stack overflow](https://stackoverflow.com/questions/13129060/opening-up-a-port-with-powershell)
provides a powershell
script that can be used to test:

```powershell
$Listener = [System.Net.Sockets.TcpListener]8988;
$Listener.Start();
while($true) 
{
    $client = $Listener.AcceptTcpClient();
    Write-Host "Connected!";
    $client.Close();
}
$Listener.Stop();
```

I used Ansible to create a firewall rule to open port 8988:

```yaml
- name: Open the windows port so that auto update works
  community.windows.win_firewall_rule:
    name: PeopleSoft Process Scheduler Notification Port
    localport: 8988
    action: allow
    direction: in
    protocol: tcp
    state: present
    enabled: true
```

### Annoying errors

I also received some errors in the application server
TUXLOG. It turned out that I had
configured a process scheduler to talk to the application server,
but hadn't configured the application server to talk to the process
scheduler. Thus the application server kept rejecting the
connection attempts from the process scheduler.

Again the messages are wrapped for readability:

```
113631.testapps1!GWTDOMAIN.76250.2848200448.0: LIBGWT_CAT:1509:
    ERROR: Error occurred during security negotiation - closing connection
113730.testapps1!GWTDOMAIN.76250.2848200448.0: LIBGWT_CAT:1073:
    ERROR: Unable to obtain remote domain id (PRCS_PSUNX_8988)
        information from shared memory
```

### Everything is set up OK and it *Still* is not working!

Try clearing the caches! Including the web server cache.


## Conclusion

Despite what it would appear from Oracle's documentation, it is possible
to automate the setup of the process scheduler auto update. This will
save us a lot of time in setting up our environments.