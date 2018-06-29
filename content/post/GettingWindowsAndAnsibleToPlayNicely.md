---
title: "Getting Windows and Ansible to play nicely"
date: 2018-06-26T15:09:57+01:00
image: images/abstract-blue.jpg
tags: ["Scripting","Automation","Windows","Powershell"]
---

The challenge is to run an installer from a share, then run some extra configuration.
We do this in Unix (Well, GNU/Linux) all the time, and it is really, really easy.
We even have the software on a Windows share, so we just need to mount that and run it!

It turns out that Windows is different from Unix. A mounted share doesn't belong to
the system like in Linux, it belongs to a session. A windows session is what is
created when a user logs in to Windows, it contains the desktop, the windows,
and the various attributes of the connected user, their permissions for example.

When Ansible connects to Windows, it uses RDP. This uses
https, which is a stateless connection. It doesn't create a session, even for
the duration of the command being run, so you can't connect to shares.
There is nowhere to store the session variables that would be required.

Powershell has a command to create a session. This doesn't work in an
RDP connection. I don't know why you can't create a session in an RDP
connection, you just can't.

Windows administrators get around this is by running scripts that
require a session as a scheduler task. It is possible to
create a task from an Ansible connection, though a PowerShell script is
required to do this. Actually two. One to create the job and another to
be run by the job. 

To make Ansible work nicely I suggest copying the installer to the VM.
My storage admin doesn't agree with the internet that disc is cheap,
apparently if you add 10G to 100VMs, that is an extra 1T of SAN space
that is required, and with 1T here, and another 1T there, it soon 
starts to add up to a lot of space. Still, it does make life a lot easier.

Another issue with Windows, is that I/O doesn't work as consistently
as it does on Linux. The installer I am trying to run uses
the PowerShell Write-Host command to write out messages.
If the script is being run interactively, messages appear, but there is no
way to redirect them to a file. The trick of scheduling
a task to create a session works but, it creates a desktop with 
the windows in. It is not attached to a screen, so it isn't possible to see
error messages. As noted above they won't redirect either. This makes
debugging an exercise in frustration. Many programs aren't written
to be run non-interactively, so they happily pop up a window with an
error message and an OK button in the invisible desktop. They often
behave differently in the interactive desktop than in the invisible
one to make debugging extra hard.

Programs that were written to be interactive
which can easily be automated on Linux using `expect`, available
as a native command, a python library, or an Ansible module.
None of these are available for Windows. However I was lucky,
when reading the PowerShell installer script I noticed that it had a
parameter -silent, which made it work in a non-interactive mode.
It still needed a session to run in though.

GNU and Linux have a rich set of tools which allow you to infer
what a process is doing. Windows doesn't No strace, lsof, /proc filesystem etc.
Debugging is pretty much done blind.

Some things I found out:

If you have a command in a variable (e.g. the path to it) use invoke-expression.

If a command writes to stderr, PowerShell assumes this is an error (If called remotely). Redirect it using:

2>&1, same as bash.

Powershell comment is #, same as bash.

If installing an msi, you can find out the long string Ansible needs to tell if it is already installed using 

I used winrm, which uses port 5986 by default.

Most people know that c:\temp is a temporary directory. Programs leave
files there that can be safely deleted. It appears later versions
of Windows also have a temporary directory under c:\users\<name>\AppData\Local\Temp
Under here is a temporary folder which is numbered per session, and deleted on 
logout, but the installers like to leave files directly in TEMP which are
never cleared down.

The windows install seems to work most of the time, but occasionally fails
for no apparent reason. It just seems to stop. I don't know why. This is 
disappointing, because the whole point of Ansible is that the build
is supposed to be repeatable. 

The next post will go into more technical detail.

Using symlinks to a file won't work. You have to specify the whole file name (In ansible).


