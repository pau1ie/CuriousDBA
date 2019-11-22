---
title: "PeopleCode Debugger - Setup and Diagnosis"
date: 2019-11-22T10:06:09Z
tags: ['PeopleSoft','Network','Windows','Unix']
---

The debugger requires application
designer to be run in three tier mode, i.e. it should connect to the application server
workstation listener rather than to the database itself.

We found that to achieve this we had to do the following:

On  the application server, run `psadmin`, and administer the domain
where the debugger is to be switched on. As of tools 8.57 this is done
by choosing the following options from the menu. Note that the domain will be
shut down once the configure option is chosen.
```
1) Application Server
1) Administer Domain
1) APPDOM
4) Configure this domain
y to continue and shut down the domain
```
At this point the configuration menu is shown. Ensure the following
options are configured. It seems WSL is short for Workstation Listener, and PC
Debugger is short for PeopleCode Debugger.
```
6) WSL         : Yes
7) PC Debugger : Yes
```

Once this is complete, the domain can be configured and booted as follows:

```
14) Load config as shown
1) Boot this domain
1) Boot
```

We have a firewall between the application server and the 
desktop PCs, and found that the following ports had to be opened:

* 7000: For the Workstation Listener
* 7001-700n: For the Workstation Handlers. This depends on how many have been configured.
* 9500: For the PeopleCode Debugger.

The number of workstation handlers is configured in the psappsrv.cfg file in the _Workstation Listener_
section. To start with we have the following:
```ini
[Workstation Listener]
...
Min Handlers=1
Max Handlers=3
```
So we need ports 7000-7003 open, as we have a maximum of 3 handlers. Looking at the netstat
output below it seems that the highest port is used first.

# Debugging  the Problems

We seemed to get a lot of problems. Here are the issues we encountered and their solutions

## Application server won't start

The first time I started the workstation listener in our development environment, it
didn't start:

```console
exec PSSUBDSP -o ./LOGS/stdout -e ./LOGS/stderr -s PSSUBDSP_dflt:Dispatch -- -D APPDOM -S PSSUBDSP_dflt :
        process id=13298 ... Started.
exec PSMONITORSRV -o ./LOGS/stdout -e ./LOGS/stderr -A -- -ID 126982 -D APPDOM -S PSMONITORSRV :
        process id=13304 ... Started.
exec WSL -o ./LOGS/stdout -e ./LOGS/stderr -A -- -n //appsrvr:7000 -z 0 -Z 256 -I 5 -T 60 -m 1 -M 3 -x 40 -c 5000 -p 7001 -P 7003 :
        Failed.

tmboot: CMDTUX_CAT:827: ERROR: Fatal error encountered; initiating user error handler

exec tmshutdown -qy
```

The log files were extremely unhelpful. The only relevant messages were in the Tuxedo log file:

```console
163604.appsrvr!WSL.13363.4069581312.0: 11-15-2019: Tuxedo Version 12.2.2.0.0, 64-bit
163604.appsrvr!WSL.13363.4069581312.0: LIBTUX_CAT:262: INFO: Standard main starting
163604.appsrvr!WSL.13363.4069581312.0: WSNAT_CAT:1008: ERROR: Could not establish listening address on network //appsrvr:7000
163604.appsrvr!WSL.13363.4069581312.0: LIBTUX_CAT:250: ERROR: tpsvrinit() failed
```

The error message would be more helpful if it mentioned the IP address it was listening on.

There is a brief note at the end of Oracle support Doc ID 607604.1 which mentions that the hosts
file may be wrong. Sure enough the IP address in the hosts file is not the same as the IP address of the VM.
The WSL process is looking at this IP address and trying to listen on it, but it can't as the IP address in question
doesn't belong to this VM. Once this was corrected, the WSL booted.

## Checking the ports are open

Application designer still wouldn't connect.

The Unix host has an IP address,
but to route network communication to the correct program, it has a number of ports. It is possible
to tell who is listening on each port using netstat. In the output below we can see under _Local Address_
the ports being listened on, and the last column shows the program that is listening on this port.
```bash
# netstat -ltnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address    Foreign Address  State   PID/Program name    
tcp        0      0 10.0.99.99:7003  0.0.0.0:*        LISTEN  41449/WSH           
tcp        0      0 0.0.0.0:9033     0.0.0.0:*        LISTEN  41450/JSL           
tcp        0      0 0.0.0.0:9034     0.0.0.0:*        LISTEN  41451/JSH           
tcp        0      0 0.0.0.0:9035     0.0.0.0:*        LISTEN  41452/JSH           
tcp        0      0 0.0.0.0:9036     0.0.0.0:*        LISTEN  41453/JSH           
tcp        0      0 0.0.0.0:9037     0.0.0.0:*        LISTEN  41454/JSH           
tcp        0      0 0.0.0.0:9038     0.0.0.0:*        LISTEN  41455/JSH           
tcp        0      0 127.0.0.1:3310   0.0.0.0:*        LISTEN  1595/clamd          
tcp        0      0 0.0.0.0:111      0.0.0.0:*        LISTEN  1/systemd           
tcp        0      0 0.0.0.0:10902    0.0.0.0:*        LISTEN  -                   
tcp        0      0 0.0.0.0:22       0.0.0.0:*        LISTEN  1175/sshd           
tcp        0      0 10.0.99.99:7000  0.0.0.0:*        LISTEN  41448/WSL           
tcp        0      0 127.0.0.1:25     0.0.0.0:*        LISTEN  1467/master         
tcp6       0      0 :::9500          :::*             LISTEN  41336/PSDBGSRV      
tcp6       0      0 :::13404         :::*             LISTEN  41238/PSAPPSRV      
tcp6       0      0 :::32927         :::*             LISTEN  41281/PSQRYSRV      
tcp6       0      0 :::12773         :::*             LISTEN  41195/PSAPPSRV      
tcp6       0      0 :::24718         :::*             LISTEN  -                   
tcp6       0      0 :::111           :::*             LISTEN  1/systemd           
tcp6       0      0 :::10100         :::*             LISTEN  55539/rmiregistry   
tcp6       0      0 :::10101         :::*             LISTEN  29933/PSMONITORSRV  
tcp6       0      0 :::22            :::*             LISTEN  1175/sshd           
tcp6       0      0 :::10200         :::*             LISTEN  35270/rmiregistry   
tcp6       0      0 ::1:25           :::*             LISTEN  1467/master         
```
This proves that the workstation listener is listening on port 7003 (First line) and the work station handler
is listening on port 7000. It is also interesting to note that while the Jolt Service Handlers are listening on all interfaces (ip address 0.0.0.0), the workstation listener and handler are listening on one interface with the IP address 10.0.99.99. This is why it looks in the hosts file to find the IP address as mentioned above.

## Firewall
So having proven that the application server is at least listening on the correct ports, why can't
application designer connect? 

![Could Not Connect to Application server. Possible causes are: the server name/IP address and port for the application server alias are incorrect, the application server is not booted, or the network is unreachable. Contact your system administrator or check the Tuxedo log for more information.](/images/PeopleCodeDebugger/CouldNotConnectToApplicationServer.png)

Ticking off the possible causes listed:

* server name/IP address and port for the application server alias are incorrect? No, they are correct.
* The application server is not booted? No I booted it.
* The network is unreachable.

I have  a terrible feeling I am the system administrator, so I checked the Tuxedo log, but there was 
nothing useful in it. So we should check the network. In the old days we would try to connect using
telnet, but this is often not installed any more. So on Linux I use nc as follows:

```bash
$ nc appsrvr 7000
Ncat: Connection timed out.
```

And on Windows I use 

```powershell
PS C:\windows\system32> Test-NetConnection -ComputerName  appsrvr -Port 7000
WARNING: TCP connect to (10.0.99.99 : 7000) failed


ComputerName           : appsrvr
RemoteAddress          : 10.0.99.99
RemotePort             : 7000
InterfaceAlias         : Ethernet
SourceAddress          : 10.1.99.98
PingSucceeded          : True
PingReplyDetails (RTT) : 1 ms
TcpTestSucceeded       : False
```

We can't connect to the application server on that port. It looks like a 
firewall issue. Once I asked the administrator to open ports 7000-7003 this test worked, and
we could connect. However, our developers were getting the following error:

![Debugging disabled. Communication failure during connect(): 0. (143,9) Explanation: A communication failure has caused debugging to be disabled. Typical causes can include a process crashing or a network failure. Exit the Application Designer and try again.](/images/PeopleCodeDebugger/DebuggingDisabled.png)

This was caused by port 9500 not being open for the developers. Once I asked the firewall admin to open
the port, the developers were finally able to connect!

## Setting Up Application Designer

It looks like it should work if you just type the name into the login screen, but I added the environments in configuration manager. Go to the profile tab, then edit the default profile. Add a connection of:

![Configuration Manager - Default Profile](/images/PeopleCodeDebugger/ConfigManager.png)

* Type: Application Server. 
* Application server name is just a name (e.g. Development)
* Machine name: appsrvr (i.e. the hostname of your application server)
* Port: 7000 (This is the default)
* Leave the rest blank.
* Click Set to move it to the list.

A number of application servers can be added in this way for the user to pick the one they require.