---
title: "Weird PeopleTools Install Issue On Windows"
date: 2018-05-23T15:34:25+01:00
tags: ["Oracle","PeopleSoft","Windows","Automation"]
image: images/clock.jpg
---

I am not sure how this situation arises, but I discovered that
sometimes the windows installer (Puppet based) on Windows gets
into a state where it isn't installed, but thinks it is, which
means that the install won't work, but it generates messages as
if it had worked.

Under c:\psft\pt\ps_home8.55.23 is only the jre directory.
Running the install again appears to work, but doesn't do
anything.

The solution is to run the uninstall, then the install will work.

To make ansible cope with this, I got the uninstaller to run if
the ps_home directory exists, rather than the psadmin.exe 
executable. This means the cleanup can be run from ansible
if the directory is in this state.

So now my uninstall step looks like:

{{< highlight yaml >}}
   - name: Run the uninstaller
     script: "psw/files/scheduleps1.ps1 -script \"{{ install_path }}\\psft-dpk-setup.ps1 -cleanup -silent\" -logfile {{ ansibletmp }}\\install.log "
     args:
       removes: "{{ ps_home }}"
     register: installfiles
{{< /highlight >}}

instead of

{{< highlight yaml >}}
   - name: Run the uninstaller
     script: "psw/files/scheduleps1.ps1 -script \"{{ install_path }}\\psft-dpk-setup.ps1 -cleanup -silent\" -logfile {{ ansibletmp }}\\install.log "
     args:
       removes: "{{ ps_home }}\appserv\psadmin.exe"
     register: installfiles
{{< /highlight >}}

Hopefully this will mean my automated build recovers from the issue,
even if it doesn't prevent it altogether.

I will give more detail later on what the above is doing.
