---
title: "Testing IO with SLOB"
date: 2020-05-19T14:40:50Z
tags: ['Oracle', 'DBA', 'Performance']
---

We recently purchased a new database server. We ran a performance test which our brilliant testing
team have put together and found out the system with the new server was slightly slower than the 
system with the old one. The testing team was happy that the test was a pass - the users wouldn't
notice the difference. However, new servers are supposed to be quicker than old ones, so I was
disappointed. I investigated a little further, and found that the AWR report showed there were
more waits on IO on the new hardware than on the old.

Since I had identified IO to be the problem, I thought it would be worth having a repeatable 
test to run on old and new hardware to have some figures to report back to our sysadmins. It is really
nice to have hard numbers to report back.
A lot of people seem to use [SLOB](https://kevinclosson.net/2012/02/06/introducing-slob-the-silly-little-oracle-benchmark/).


## Setting SLOB up

First of all SLOB has to be [downloaded](https://kevinclosson.net/slob/).
Unzip the tar file somewhere convenient - on the database server if possible.
It took me a while to work out how to get started. There is a lot that can be 
configured, but very little that needs to be. Here is what seems to me to be
the bare minimum to get up and running.

### Edit slob.conf. 

I set the following:

```bash
DBA_PRIV_USER="SYS"
SYSDBA_PASSWD="SYS AS SYSDBA"
DATABASE_STATISTICS_TYPE=awr
```

Because I am connecting as sysdba on the database server, I can `connect / as sysdba`.
If I left user blank it defaulted to system. Also I found I couldn't have a leading
space in the password field. So `connect sys/sys as sysdba` works just as well.
I used AWR as we have a tuning and diagnostics pack license.
If you want to use statspack, you have to install it.

### Create the Tablespace
I also edited the `misc/ts.sql` script as follows:
```sql
set echo on
set timing on
drop tablespace IOPS including contents and datafiles;
create smallfile tablespace IOPS datafile '/path/to/users/iops.dbf' size 1G 
NOLOGGING ONLINE PERMANENT EXTENT MANAGEMENT LOCAL 
AUTOALLOCATE SEGMENT SPACE MANAGEMENT AUTO ;
alter database datafile '/path/to/users/iops.dbf' 
autoextend on next 200m maxsize unlimited;
exit;
```
and ran it. 

### Create the Schemas
Now run setup with 8 schemas. I chose 8 because that is what is in the documentation, so presumably works OK.

```bash
./setup.sh IOPS 8
```
### Compile the wait kit

This is required for the tests to work.
```bash
cd wait_kit
make
cd ..
```

## Running it

Now run the benchmark with a parameter of 8 for the 8 schemas created earlier.
```bash
./runit.sh 8
```

## Test cases

The diagnostic files are left in the current directory. So after each run, I found it
convenient to create a directory and copy the reports into it:
```bash
mkdir testcase1
mv *stat.out awr_* awr.* testcase1
```


## Troubleshooting

I did run into an issue where SLOB wouldn't create an AWR report on a newly cloned database. This 
is because the code to identify the snapshot assumes all snapshots are for the current database,
so just takes the one with the highest ID, which was for the old database. I fixed this by
changing line 587 of runit.sh from
```sql
SELECT MAX(SNAP_ID) FROM dba_hist_snapshot;
```
to
```sql
SELECT MAX(s.SNAP_ID) FROM dba_hist_snapshot s, v\$database d where d.dbid = s.dbid;
```
