---
title: "Gnu PG Cache Time To Live"
date: 2020-02-27T10:03:50Z
tags: ['Automation', 'Ansible','Fail','Scripting','Language']
---

As [discussed previously](../../tags/secrets), we use [regpg](https://dotat.at/prog/regpg/)
to manage Ansible secrets. This has been really
useful. One annoyance though is that some tasks can take up to 6 hours to run, but
the gpg agent only caches the gpg passphrase for 10 minutes or so. I end up having to
type the passphrase in several times during a run. I occasionally kick off a  run before
I leave for the day. It would be a shame if it was stalled overnight due to waiting for
a passphrase.

It is possible to change the amount of time the passphrase is cached for. Lets assume I start work
first thing in the morning. I want the passphrase to be cached all day, and if I kick off a process
before I leave for the day, I want it to complete. So the passphrase should be cached for
say 8 hours with a maximum ttl of 14 hours.

This can be done by changing the configuration in `gpg-agent.conf`. The entries in this
file is the same as the long form command line options but without the leading double dash.

The options to set are 
[default-cache-ttl](https://www.gnupg.org/documentation/manuals/gnupg/Agent-Options.html#index-default_002dcache_002dttl)
and 
[max-cache-ttl](https://www.gnupg.org/documentation/manuals/gnupg/Agent-Options.html#index-max_002dcache_002dttl)
The times are in seconds as per the manual, and there are 3600 seconds in an hour, so we need
to set the following:

```
default-cache-ttl 28800
max-cache-ttl 50400
```

Then the gpg agent needs to be instructed to load the file. This can be done by:

```
gpg-connect-agent reloadagent /bye
```
or
```
gpgconf --reload gpg-agent
```
or by killing  the agent process, it will be restarted automatically the next time a
it is asked to provide a passphrase.

The [Arch Linux wiki](https://wiki.archlinux.org/index.php/GnuPG#Configuration_2)
has a lot of useful information, but not all of it works on my Red Hat system, either
because it has older version of programs, or because I haven't installed everything I needed.

Hopefully this will mean I will only be prompted once a day for my gpg key passphrase.

