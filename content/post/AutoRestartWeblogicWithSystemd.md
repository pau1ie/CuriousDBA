---
title: "Auto Restart Weblogic with Systemd"
date: 2022-03-03T12:19:07+01:00
tags: ["Weblogic","Automation","Fail","Linux","PeopleSoft","systemd"]
---

## The Problem

Our WebLogic server recently crashed. There were a number of issues identified in the post mortem,
which were addressed to prevent the same thing happening again. It also occurred to us that WebLogic should
restart itself automatically if it fell over. This is easily achieved using systemd, but for
whatever reason Oracle chose not to configure it to do this.


## Oracles Default Setup

This is a PeopleSoft system which might be configured slightly differently to other 
WebLogic installations. PeopleSoft
by default provides a systemd unit file which is mostly copyright warnings, so I won't
include it here. Oracle have created a
legacy `init.d` file which is called by systemd as a `oneshot`. This in turn calls the `startPIA.sh` and `stopPIA.sh` scripts
as appropriate. So if the web server crashes, it stays stopped.


## Systemd to the Rescue

I could of course write a script to continually loop and restart the web server whenever it falls over.
The problem is of course I'd have to provide a way to not auto restart when I really want to stop WebLogic.
Then I'd have to document and support this procedure.

Systemd does all this for me. It seems like a good idea to use that!


## Systemd Unit File

The main changes I made are in the service section of `/usr/lib/systemd/system/psft-pia-peoplesoft.service`
It now looks like this (I shortened the paths for clarity):

```systemd
[Service]
Type=simple
User=psadm2
WorkingDirectory=/webserv/peoplesoft
ExecStartPre=-/usr/sbin/logrotate -f --state /myetc/rotate.state /myetc/rotate.conf
ExecStart=/webserv/peoplesoft/bin/startPIA.sh -foreground -capture
ExecStop=-/webserv/peoplesoft/bin/stopPIA.sh
RestartSec=30
Restart=always
```

The systemd documentation is pretty good, and worth a read. I referred most to the 
[unit file](https://www.freedesktop.org/software/systemd/man/systemd.unit.html) and
[service section](https://www.freedesktop.org/software/systemd/man/systemd.service.html#)
pages.
I found it best not to call the init script as this just confuses things. Here is what I
worked out I needed:

| Line | Description |
|---|---|
| Type | `simple` allows systemd to manage the service. Before it was `oneshot` which runs the script and doesn't care what happens afterwards. |
| User | This is the user the service runs as. The default in PeopleSoft is `psadm2`. Systemd takes care of is switching users. |
| WorkingDirectory | The directory that the process is started in. |
| ExecStartPre | This is run before running the start command. I take the opportunity to rename log files so they don't get overwritten when the server is restarted. The dash is to ignore return codes - so the server starts even if the file rotate failed. |
| ExecStart | This is the command that actually runs WebLogic. It is supplied by Oracle, but I found running it in the foreground but still logging the output as normal worked best. |
| ExecStop | The command to stop WebLogic. This also has the dash prepended to ignore errors. |
| RestartSec | This is the magic! If systemd notices the process died, it will wait this long then restart it. I am not sure what a good value is here, but I certainly couldn't restart it in 30 seconds, so that seemed good. The default is 100ms, so potentially this could be set lower. |
| Restart | The conditions under which the process is restarted. If the return code of  the start script reflected whether the webserver had crashed, this could be used to only restart after a crash. However it always exits with a success return code, so `always` seems to be the best option. |

On testing this by killing WebLogic (see below) I found that systemd will run the command defined in
`ExecStop` after a crash has been
detected. This seems reasonable - there might be other processes, memory segments or files to tidy up. However,
the Oracle supplied script exits with an error return code if webLogic isn't running, then systemd doesn't run the startup,
which isn't want I wanted to happen. So we need to instruct systemd to ignore the return code from the stop
script. This is what the leading dash does.

## Telling Systemd To Reread the Configuration

After changing a systemd unit file, we need to tell it to 
[reload it's configuration](https://www.freedesktop.org/software/systemd/man/systemctl.html#daemon-reload).
```bash
systemctl daemon-reload
```


## Does It Work?

Once the server boots, systemd has started the service (I have edited the output to help it fit better)

```
# systemctl status psft-pia-peoplesoft.service
🟢 psft-pia-peoplesoft.service - Weblogic Server PIA
   Loaded: loaded (psft-pia-peoplesoft.service; enabled; vendor preset: disabled)
   Active: active (running) since 15:33:51 GMT; 1 weeks 4 days ago
  Process: 491 ExecStartPre=logrotate -f --state logrotate.state logrotate.conf 
      (code=exited, status=0/SUCCESS)
 Main PID: 493 (startPIA.sh)
    Tasks: 77
   Memory: 1.2G
   CGroup: /system.slice/psft-pia-peoplesoft.service
           ├─493 /bin/sh /webserv/peoplesoft/bin/startPIA.sh -foreground -capture
           └─567 java -server -Xms1024m -Xmx1024m -Dtoplink.xml.platform=oracle.t...

15:33:51 web1 systemd[1]: Starting Weblogic Server PIA...
15:33:51 web1 systemd[1]: Started Weblogic Server PIA.
15:33:51 web1 startPIA.sh[493]: Attempting to start WebLogic Server PIA
15:33:51 web1 startPIA.sh[493]: No activity will be logged to this window.
15:33:51 web1 startPIA.sh[493]: Server activity will be logged to /webserv/peopl...*
```

We can see that the logrotate ran successfully, and so did the startPIA script. If I kill WebLogic
I can see systemd noticing and running the stopPIA script:

```bash
kill -9 567
```

```
● psft-pia-peoplesoft.service - Weblogic Server PIA
   Loaded: loaded (psft-pia-peoplesoft.service; enabled; vendor preset: disabled)
   Active: deactivating (stop) since 17:35:25 GMT; 15s ago
  Process: 493 ExecStart=/webserv/peoplesoft/bin/startPIA.sh -foreground -capture
      (code=exited, status=0/SUCCESS)
  Process: 491 ExecStartPre=logrotate -f --state logrotate.state logrotate.conf
      (code=exited, status=0/SUCCESS)
 Main PID: 493 (code=exited, status=0/SUCCESS);         : 820 (stopPIA.sh)
    Tasks: 12
   Memory: 824.4M
   CGroup: /system.slice/psft-pia-peoplesoft.service
           └─control
             ├─820 /bin/sh /webserv/peoplesoft/bin/stopPIA.sh
             └─891 java -classpath /psft/pt/jdk...

17:33:51 web1 startPIA.sh[493]: Attempting to start WebLogic Server PIA
17:33:51 web1 startPIA.sh[493]: No activity will be logged to this window.
17:33:51 web1 startPIA.sh[493]: Server activity will be logged to /webserv/peopl...*
17:35:25 web1 startPIA.sh[493]: startPIA.sh: line 180: 567 Killed      ...derr.log
17:35:25 web1 startPIA.sh[493]: WebLogic is no longer running.
17:35:25 web1 startPIA.sh[493]: PID for WebLogic Server PIA is:
17:35:26 web1 stopPIA.sh[820]: Submitting shutdown command for WebLogic Server P...0
17:35:26 web1 stopPIA.sh[820]: No activity will be logged to this window.
17:35:26 web1 stopPIA.sh[820]: Server activity will be logged to /webserv...hutdown*
17:35:26 web1 stopPIA.sh[820]: Stopping Weblogic Server...
```

Then it starts up again:

```
🟢 psft-pia-peoplesoft.service - Weblogic Server PIA
   Loaded: loaded (psft-pia-peoplesoft.service; enabled; vendor preset: disabled)
   Active: active (running) since 17:36:29 GMT; 7s ago
  Process: 820 ExecStop=stopPIA.sh (code=exited, status=0/SUCCESS)
  Process: 968 ExecStartPre=logrotate -f --state logrotate.state logrotate.conf
     (code=exited, status=0/SUCCESS)
 Main PID: 969 (startPIA.sh)
    Tasks: 19
   Memory: 404.5M
   CGroup: /system.slice/psft-pia-peoplesoft.service
           ├─969 startPIA.sh -foreground -capture
           └─040 java -server -Xms1024m -Xmx1024m -Dtoplink.xml.platform....

17:36:29 web1 systemd[1]: psft-pia-peoplesoft.service holdoff time over,
    scheduling restart.
17:36:29 web1 systemd[1]: Stopped Weblogic Server PIA.
17:36:29 web1 systemd[1]: Starting Weblogic Server PIA...
17:36:29 web1 systemd[1]: Started Weblogic Server PIA.
17:36:29 web1 startPIA.sh[19969]: Attempting to start WebLogic Server PIA
17:36:29 web1 startPIA.sh[19969]: No activity will be logged to this window.
17:36:29 web1 startPIA.sh[19969]: Server activity will be logged to /web...*
```

## Procedure Change

Now we have to change our procedures. We must always stop and start WebLogic using systemd.
If we forget and use the `stopPIA.sh` script directly, systemd will restart WebLogic again.
If we start weblogic using `startPIA.sh`, we won't get the benefit of the systemd configuration.


## More Possible Improvements

I worked around a couple of issues above. They are caused by the Oracle supplied
scripts not setting return codes in the way systemd expects. `startPIA.sh` would probably
work in the way systemd expects if it were changed to propagate the return code from the `java` process. Then
we could use `stopPIA.sh` to stop WebLogic, and systemd wouldn't restart it again.

`stopPIA.sh` would need to be changed to return an error only if it failed to stop WebLogic.
This is a little more complicated than above, but could be done. 

I prefer not to change Oracle supplied scripts due to the overhead in maintaining them, so I am
happy with the current situation.