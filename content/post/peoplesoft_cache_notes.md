---
title: "Notes on Caching"
date: 2024-06-14T08:00:49+00:00
tags: ['PeopleSoft','Performance']
---

Just a quick one this time. PeopleSoft has three different types of cache
at the application server level, and here are some notes on it.

## Unshared Cache

### What it is

So called because each application server process creates it's own cache
under `%PS_SERVDIR%` which is by default `$PS_CFG_HOME/domainname/CACHE/`
Under here are directories for each process, e.g. `PSAPPSRV_1` etc.

### How to use it

This is the default type of cache. It is used if you configure 

```
ServerCacheMode = 0
```

Or if you leave it unset, because `0` is the default. Or if it is set to
`1` but no shared cache has been delivered.

### Benefits and Drawbacks

This is easy to set up, so we often use it in test environments. The
drawback is that every time a cached object changes it needs to be
refreshed in every copy of the cache (one per application server).
Test environments tend to have fewer application servers configured
than production, so this doesn't matter as much in non-production.


## Shared Cache

### What it is

This is where all the application server processes in a domain share
cache. The cache is built by the process scheduler jobset `PLCACHE`, and
then needs to be deployed to the cache directory above but in the
`SHARE` directory.

If the cache wasn't deployed, it falls back to the unshared cache, which
is useful.

### How to use it

Set

```
ServerCacheMode = 1
```

Then the cache needs to be built by navigating to:

 PeopleTools -> Utilities -> Administration -> Load Application Server Cache
 
and running one of the prebuild cache jobs. We choose PLCACHE as it tends
to be quicker. It asks you where you want to put the prebuilt cache, but
it ignores the answer and instead puts it in

```
$PS_CFG_HOME/appserv/prcs/<DOMAIN>/CACHE/STAGE/stage
```
In general it does an incremental build, but for some reason after a tools
patch has been applied it rebuilds the entire cache which can take an
hour.

Then this prebuilt cache needs to be deployed to the application servers,
which need to be shut down while it is being deployed. This significantly
extends the downtime for tools patches.

### Benefits and Drawbacks

There is only one copy of the cache per VM. Any changes to cached objects
only need to be made once per VM, so every other application server on
that VM benefits. Less disc space is used.

The big drawback is of course the time taken to build and deploy the cache
both in terms of the elapsed time, and in terms of the operator time taken
to schedule the process, wait for it to complete, and to deploy the cache.


## Database Cache

### What it is

This needs to be built the same as the shared cache, but in this case
the application servers read the cache from the database. So if we have
application servers in different domains, these will share the cache.
This makes managing the cache easier, as it doesn't have to be deployed.

### How to use it

So to use unshared cache we set `ServerCacheMode = 0`, shared cache we set
`ServerCacheMode = 1`, so to use database cache you set...

```
EnableDBCache = Y
```

The documentation states that `ServerCacheMode` is ignored if this is set,
but our testing indicates that the database cache is only kept up to date
if `ServerCacheMode` is not set. If it is set to `1`, then the database
cache is only updated when the cache build job is run.

### Benefits and Drawbacks

This is even better than the shared cache in that there is only a single
copy, and any changes to cached objects only need to be updated once
(rather than once per VM in the shared cache and once per PSAPPSRV process
in the case of the unshared cache).

It still needs to be built, but it doesn't need to be deployed. It
seems the cache build process is able to do an incremental build after
a tools patch which is beneficial.

We do wonder whether storing the cache on the database server rather
than the application server has overhead, as it will need to be
transferred over the network. Overall the database cache seems to be the
best option, were it not for the...


### Issues

I had an issue with copies of my production database that I wasn't able
to reproduce in demo where using the database cache stopped the
application server and web server boot users from
logging in. They all have password failures. This means the application
won't work, so we can't use database cache. We know we have configured the
password correctly because if we remove the `EnableDBCache` parameter the
application boots fine in shared cache mode. We have a call open with
Oracle on this issue.