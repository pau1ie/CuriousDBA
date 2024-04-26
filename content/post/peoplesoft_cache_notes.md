---
title: "Notes on Caching"
date: 2024-04-24T14:57:49+00:00
tags: ['PeopleSoft']
---

## Overview of Cache

Just a quick one this time. PeopleSoft has three different types of cache at the application server level:

### Unshared Cache

So called because each application server process creates it's own cache
under `%PS_SERVDIR%` which is by default `$PS_CFG_HOME/domainname/CACHE/`
Under here are directories for each process, e.g. `PSAPPSRV_1` etc.

### Shared Cache

This is where all the application server processes in a domain share
cache. The cache is built by the process scheduler jobset `PLCACHE`, and
then needs to be deployed to the cache directory above but in the
`SHARE` directory.

If the cache wasn't deployed, it falls back to the unshared cache, which
is useful.


### Database Cache

This needs to be built the same as the shared cache, but in this case
the application servers read the cache from the database. So if we have
application servers in different domains, these will share the cache.
This makes managing the cache easier, as it doesn't have to be deployed.

## Issues

I did run into
