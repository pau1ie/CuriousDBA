---
title: "Port and TLS scanning with nmap"
date: 2025-01-14T10:02:47+00:00
tags: ['HTTP','PeopleSoft','Security','Systemd','Unix','Network']
---

I have had a couple of instances where I have needed to look at which
ports are open. On one occasion a firewall change meant I needed to check
in a hurry whether ports I needed were open. On another instance, another
team raised concerns with some of the TLS ciphers listening on some of the
ports in our system.

I do not recommend port scanning across the internet. All these scans were
completed within my employers infrastructure as part of my job. If you
would like to try these commands and don't have a similar job to me, I would
suggest using devices on your home network such as a raspberry pi, or
scanning a VM running on your laptop. The nmap security scanning book
has
[a chapter on legal issues](https://nmap.org/book/legal-issues.html).

With that said, here are some useful commands. I am scanning a vanilla
PeopleSoftoft installation, and have redacted the hostnames and IP
addresses. The port numbers are default ones chosen by Oracle.
The options used come from the
[port scanning tutorial](https://nmap.org/book/port-scanning-tutorial.html)
in the nmap security scanning book.

## Scanning Common Ports

To scan commonly used ports on a host:

```
$ nmap host

Starting Nmap 7.92 ( https://nmap.org ) at 2025-01-06 10:52 GMT
Nmap scan report for host (10.0.0.3)
Host is up (0.00084s latency).
Not shown: 989 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
1521/tcp open  oracle
2000/tcp open  cisco-sccp
2049/tcp open  nfs
5060/tcp open  sip
7000/tcp open  afs3-fileserver
8000/tcp open  http-alt
8443/tcp open  https-alt

Nmap done: 1 IP address (1 host up) scanned in 0.09 seconds
```

## Scanning all the Ports

But I know there are many more ports listening. To scan all ports on a host:

```
$ nmap -p0- host
Starting Nmap 7.92 ( https://nmap.org ) at 2025-01-06 10:57 GMT
Nmap scan report for host (10.0.0.3)
Host is up (0.00045s latency).
Not shown: 65504 closed tcp ports (conn-refused)
PORT      STATE SERVICE
22/tcp    open  ssh
111/tcp   open  rpcbind
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
1521/tcp  open  oracle
2000/tcp  open  cisco-sccp
2049/tcp  open  nfs
5060/tcp  open  sip
7000/tcp  open  afs3-fileserver
7003/tcp  open  afs3-vlserver
8000/tcp  open  http-alt
8443/tcp  open  https-alt
9033/tcp  open  unknown
9034/tcp  open  unknown
9035/tcp  open  unknown
9036/tcp  open  unknown
9037/tcp  open  unknown
9038/tcp  open  unknown
10100/tcp open  itap-ddtp
10101/tcp open  ezmeeting-2
10200/tcp open  trisoap
10201/tcp open  rsms
14325/tcp open  unknown
15885/tcp open  unknown
16957/tcp open  unknown
17472/tcp open  unknown
17991/tcp open  unknown
20048/tcp open  mountd
26853/tcp open  unknown
28925/tcp open  unknown
33799/tcp open  unknown
36419/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 2.08 seconds
```

We can run the script both inside and outside the firewall to see which
ports are being blocked.

This is interesting. There are a lot of ports I didn't know about. I
think nmap is making some mistakes about what is listening on some of the
ports though. Port 7000 is the work station listener for example, not
afs3, so it looks like nmap is using a list of what typically runs on the
ports.


## More Aggressive Detection

We can make it run some scripts
to do some more aggressive detection of what is running on ports. This
output is pretty long, so I have trimmed it.

```
$ nmap -A -p0- host
Starting Nmap 7.92 ( https://nmap.org ) at 2025-01-06 11:14 GMT
Nmap scan report for host (10.0.65.20)
Host is up (0.0017s latency).
Not shown: 65504 closed tcp ports (conn-refused)
PORT      STATE SERVICE          VERSION
22/tcp    open  ssh              OpenSSH 8.0 (protocol 2.0)
111/tcp   open  rpcbind          2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      20048/tcp   mountd
|   100005  1,2,3      20048/tcp6  mountd
|   100005  1,2,3      20048/udp   mountd
|   100005  1,2,3      20048/udp6  mountd
|   100021  1,3,4      22699/tcp6  nlockmgr
|   100021  1,3,4      36175/tcp   nlockmgr
|   100021  1,3,4      52356/udp   nlockmgr
|   100021  1,3,4      58112/udp6  nlockmgr
|   100024  1          17697/tcp6  status
|   100024  1          57476/udp   status
|   100024  1          59060/udp6  status
|   100024  1          61403/tcp   status
|   100227  3           2049/tcp   nfs_acl
|_  100227  3           2049/tcp6  nfs_acl
139/tcp   open  netbios-ssn      Samba smbd 4.6.2
445/tcp   open  netbios-ssn      Samba smbd 4.6.2
1521/tcp  open  oracle-tns       Oracle TNS Listener (unauthorized)
|_oracle-tns-version: ERROR: Script execution failed (use -d to debug)
2000/tcp  open  tcpwrapped
2049/tcp  open  nfs_acl          3 (RPC #100227)
5060/tcp  open  tcpwrapped
7000/tcp  open  afs3-fileserver?
|_irc-info: Unable to open connection
7003/tcp  open  tcpwrapped
8000/tcp  open  ldap
|_http-title: Index Page
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     Connection: close
|     Date: Mon, 06 Jan 2025 11:15:15 GMT
|     Content-Length: 1164
|     Content-Type: text/html; charset=UTF-8
|     X-ORACLE-DMS-ECID: ef478bf5-65d4-4340-a0f2-3cbc31bc0415-0000d58b
|     X-ORACLE-DMS-RID: 0
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: DENY
|     Origin-Agent-Cluster: ?0
--- HTML trimmed ---
8443/tcp  open  ssl/ldap
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=host/organizationName=MyOrganization/
    stateOrProvinceName=MyState/countryName=US
| Subject Alternative Name: DNS:host.internal
| Not valid before: 2024-07-17T11:53:36
|_Not valid after:  2039-07-18T11:53:36
|_http-title: Index Page
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     Connection: close
|     Date: Mon, 06 Jan 2025 11:15:11 GMT
|     Content-Length: 1164
|     Content-Type: text/html; charset=UTF-8
|     X-ORACLE-DMS-ECID: ef478bf5-65d4-4340-a0f2-3cbc31bc0415-0000d58a
|     X-ORACLE-DMS-RID: 0
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: DENY
|     Origin-Agent-Cluster: ?0
--- html trimmed ---
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Connection: close
|     Date: Mon, 06 Jan 2025 11:15:11 GMT
|     Content-Length: 0
|     X-ORACLE-DMS-ECID: ef478bf5-65d4-4340-a0f2-3cbc31bc0415-0000d589
|     X-ORACLE-DMS-RID: 0
|     Allow: GET, HEAD, OPTIONS, POST
|_    Origin-Agent-Cluster: ?0
9033/tcp  open  tcpwrapped
9034/tcp  open  unknown
9035/tcp  open  unknown
9036/tcp  open  unknown
9037/tcp  open  unknown
9038/tcp  open  unknown
10100/tcp open  java-rmi         Java RMI
| rmi-dumpregistry: 
|   APPDOM/PSAPPSRV_2/ServerRuntime/DefaultConnector
|     javax.management.remote.rmi.RMIServerImpl_Stub
|     @10.0.65.20:17009
|     extends
|       java.rmi.server.RemoteStub
|       extends
|         java.rmi.server.RemoteObject
--- Trimmed 3 more objects that look similar to above ---
10101/tcp open  java-rmi         Java RMI
10200/tcp open  java-rmi         Java RMI
| rmi-dumpregistry: 
|   PRCSDOM/DomainRuntime/DefaultConnector
|     javax.management.remote.rmi.RMIServerImpl_Stub
|     @host:10201
|     extends
|       java.rmi.server.RemoteStub
|       extends
|         java.rmi.server.RemoteObject
|   PRCSDOM/PSAESRV_1/ServerRuntime/DefaultConnector
|     javax.management.remote.rmi.RMIServerImpl_Stub
|     @10.0.65.20:15031
|     extends
|       java.rmi.server.RemoteStub
|       extends
|         java.rmi.server.RemoteObject
--- Trimmed 5 more objects that look similar to above ---
10201/tcp open  java-rmi         Java RMI
13633/tcp open  oracle           Oracle Database
15031/tcp open  java-rmi         Java RMI
17009/tcp open  java-rmi         Java RMI
17472/tcp open  tcpwrapped
20048/tcp open  mountd           1-3 (RPC #100005)
20905/tcp open  java-rmi         Java RMI
31347/tcp open  java-rmi         Java RMI
33605/tcp open  java-rmi         Java RMI
36175/tcp open  nlockmgr         1-4 (RPC #100021)
61403/tcp open  status           1 (RPC #100024)
2 services unrecognized despite returning data. 
--- Fingerprints trimmed ---

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-01-06T11:16:24
|_  start_date: N/A

Service detection performed. Please report any incorrect results
Nmap done: 1 IP address (1 host up) scanned in 207.85 seconds
```

This took much longer - well over 3 minutes. The detection of the
services is better, but still not perfect - it still gets port 7000
wrong, but I expect not many people use the technology behind the
workstation listener, so that's not surprising.


## Checking TLS Security

I want to see what ciphers
are running on each port to see whether any are out of date. Since the
output is quite verbose, I will only list the differences from above.

```
$  nmap -p0- -sV --script ssl-enum-ciphers host
Starting Nmap 7.92 ( https://nmap.org ) at 2025-01-08 09:59 GMT
--- Snip repeated output ---
8443/tcp  open  ssl/ldap
| ssl-enum-ciphers: 
|   TLSv1.2: 
|     ciphers: 
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_DHE_RSA_WITH_AES_256_GCM_SHA384 (dh 2048) - A
|       TLS_DHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (dh 2048) - A
|       TLS_DHE_RSA_WITH_AES_128_GCM_SHA256 (dh 2048) - A
|     compressors: 
|       NULL
|     cipher preference: server
|   TLSv1.3: 
|     ciphers: 
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 (secp256r1) - A
|     cipher preference: server
|_  least strength: A
--- Snip repeasted output ----
Nmap done: 1 IP address (1 host up) scanned in 238.50 seconds
```

This looks great - all the ciphers are A class.


## What our Proprietary Scanner Found

What nmap didn't pick up on in our system which is still
on tools 8.60 is that the Jolt Service Listener and it's
handlers is by default listening using TLSv1.0, which is too
old. We need to stop it listening on the old versions of
TLS by setting the following bash variable, which we do in
the bash_profile of the user that runs the application
server processes:

```bash
export TM_TLS_FORCE_VER="TLSv1.2"
```


## Checking the firewall

One thing that would have been useful when we had a firewall
issue some weeks ago would have been to work out whether we
had access to ports we needed. Let's say I want to check I
have access to the ssh port and the web server ports of
a number of web server VMs. I can use something like
the following:

```
nmap -p 22,80,8000,8443 host1 host2 host3
```

I could compare the output inside and outside the firewall
and the output of the two could be compared to see which 
ports are being blocked by the firewall.

## Conclusion

Nmap is a complex program, but it is widely installed on
our VMs. The manual is very useful, and written so you can
quickly get started with some useful options. This makes
it really useful to check what our
VMs are doing, and how the network is behaving.