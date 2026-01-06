---
title: "Auto Restart Oracle Databases with Systemd"
date: 2025-04-02T14:19:07+01:00
tags: ["Automation","Database","DBA","Linux","Oracle","Scripting","systemd"]
---

## The Problem

Our databases didn't stop and start automatically with the OS. The oracle 
supplied scripts don't cater for standby databases. Lastly we have one 
listener per database. Whether this is necessary or not is open to debate, but
I would like to make our databases stop and start with the operating system.
I'd also like the following features:

  * Stop all the databases whether or not they are in the oratab on shutdown.
  * Start all database that were stopped on startup.
  * Deal with standby and logical standby databases.
  * Deal with our unusual listener setup which has one listener per database instance.

Oracle provides scripts called `dbstart` and `dbshut` in `$ORACLE_HOME/bin` 
which are documented in the administrator guide, but they only start and
stop databases in the oratab.

## Scripts

{{% notice warning %}}

The scripts pasted into this article are not our tested production scripts.
They illustrate our approach. They have been edited and simplified for
the purposes of this post. There is no guarantee they will work. In 
particular setting the environment has been glossed over. These edited
scripts haven't been tested.
{{% /notice %}}


I already have a script which when run on Linux detects which `ORACLE_HOME` 
the database is running out of and sets the `ORACLE_HOME` variable
correctly to shut it 
down. I also have a script which detects running databases and calls a 
different shutdown script, so I just need to incorporate these two. Testing is
of course difficult as the system will need to be restarted.

 
## Other Services

Our scripts stop and start all Oracle things including Oracle Enterprise 
Manager (OEM), and the Gateway. I was trying
to think of clever ways to handle this, but in the end I just created a
pair of scripts, `start_other.sh` and `stop_other.sh` which starts and
stops other things on the server such as the Oracle Gateway. These aren't
shown below, to keep things simple.

## Identify databases

I like the idea of detecting running databases and shutting them down. So 
something like this:

```bash
# Create a file to hold the status
stat_dir=/var/opt/oracle

if [[ ! -d $stat_dir ]]
then
  mkdir -p $stat_dir
  chown oracle:oinstall $stat_dir
fi

if [[ -f $stat_dir/stopped_databases.txt ]]
then
  mv $stat_dir/stopped_databases.txt $stat_dir/stopped_databases_prev.txt
fi

pgrep ora_smon_ | while read pid
do
  # Find the owner of the process.
  owner=$(stat -c '%U' /proc/$pid )
  # Get the ORACLE_SID from the process environment
  ORACLE_SID=$( ( cat /proc/${pid}/environ; echo) | \
      tr "\000" "\n" | grep ORACLE_SID | cut -f2 -d= )
  standby=$( su "$owner" -c "$script_dir/check_db_role.sh \"$ORACLE_SID\"" )
  su - "$owner" -c /scripts/generic_stop_db.sh $ORACLE_SID
  echo "${owner}:${ORACLE_SID}:$standby" >> $stat_dir/stopped_databases.txt
done
```

## Checking the database role

The check_db_role.sh script returns whether the database is a standby or
or primary. I include a simplified version of the environment setting code
which might work. In reality I use the Oracle supplied `oraenv`
script together with the oratab, then I run another script to update the
environment.

```bash
#!/bin/bash
ORACLE_SID=$1
pid=$( pgrep -alf ora_smon_${ORACLE_SID}$ )
export ORACLE_HOME=$( ( cat /proc/${pid}/environ; echo) | \
    tr "\000" "\n" | grep ORACLE_HOME | cut -f2 -d= )
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
dbrole=$(
  sqlplus -s / as sysdba <<-!
  set hea off
  select DATABASE_ROLE from v\$database;
  exit;
!
)
if grep -q "PHYSICAL STANDBY" <<< "$dbrole"
then
  echo "standby"
elif grep -q "PRIMARY" <<< "$dbrole"
then
  echo "primary"
fi
```

Anyway, once the environment is set, we check whether the database is
a primary or a standby by querying `v$database`. The result is returned
to the stop script (see above) so it can make a note of the role.


## Shut down the database and listener

This is accomplished using the `/scripts/generic_stop_db.sh` script referenced above.

I feel that rather than setting the environment based on the `oratab`, we should work out
the environment based on the running process like we did above. We know that
the script is running as the correct user, as we switched above. The parameter
is the database to shut down. So we need to do the following:

```bash
export ORACLE_SID=$1
pid=$( pgrep -alf ora_smon_${ORACLE_SID}$ )
if [[ -n "$pid" ]] && [[ -d /proc/$pid ]]
then
  # DB is running - find out the details and stop it!
  export ORACLE_HOME=$( ( cat /proc/${pid}/environ; echo) | \
      tr "\000" "\n" | grep ORACLE_HOME | cut -f2 -d= )
  export PATH=$ORACLE_HOME/bin:$PATH
  export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
  sqlplus / as sysdba <<-!
    shutdown immediate
    exit
!
  lsnrctl stop $ORACLE_SID
fi
```

This approach is useful because it works even if the database 
configuration was deleted before remembering to shut it down.
Or if the database was upgraded in the `oratab` while the database
was still running. With the default scripts it is difficult to
shut down a database in this state. In the past I have ended up with the
database running twice because it didn't shut down in one home
before I started it in the other!

In reality our scripts are a little more complex, we check whether
the database and listener are running separately. As I said, this 
just gives an idea of our approach.


## Start the database and listener

To start the database and listener, we need to set  the environment
as in the `oratab` - there isn't a process running we can get it from.

First of all we should find the databases to start. We could use
`oratab` for this:

```bash
grep -v '^#' /etc/oratab | while read line
do
  ORACLE_SID=$( echo $line | cut -f1 -d: )
  DBTYPE=$( echo $line | cut -f3 -d )
  case $DBTYPE in
    Y ) echo starting database.
	    /scripts/generic_start_db.sh "$ORACLE_SID"
	;;
	N ) echo Not starting $ORACLE_SID
	;;
	S ) echo Starting standby
	    /script/generic_start_standby.sh "$ORACLE_SID"
	;;
	default )
	  echo Unknown database option.
	;;
  esac
done
```

### Start only previously stopped databases

In the stop script we made a list of databases which were stopped,
so instead of the oratab, we could use that list to start the databases.
I still use the `ORACLE_HOME` from the oratab, but I list through the
list we created of shut down databases. Of course this doesn't work if 
the server has crashed. Instead the list of databases at the previous
shutdown is used.

```bash
script_dir=$( dirname "$( realpath "${BASH_SOURCE[0]}" )" )

stat_file=/var/opt/oracle/stopped_databases
if [[ ! -f "$stat_file" ]]
then
  mv "${stat_file}_prev" "$stat_file"
fi
date
echo "Starting previously stopped databases on $(hostname)"
IFS=:
while read -r owner sid type
do
  echo "Switching to $owner to start $sid as role $type"
  if [[ a$type = astandby ]]
  then
    nohup su "$owner" -c "$script_dir/generic_start_standby_db.sh \"$sid\"" >
	      "$log_dir/${sid}_start.log" 2>&1 &
  elif [[ a$type = aprimary ]]
  then
    nohup su "$owner" -c "$script_dir/generic_start_db.sh \"$sid\"" >
          "$log_dir/${sid}_start.log" 2>&1 &
  else
    echo "Can't start database $sid as $owner - $type is not recognised."
  fi
done < "$stat_file"
```

I haven't included the start scripts as they are quite similar to the stop script.
The environment is set using `oraenv`, then the database is started using the
`sqlplus` `startup` command.

For the standby, the following startup is used, again pretty simple, we don't
use dataguard manager at present. We should probably look into that when we get
a chance!

```
startup nomount
alter database mount standby database;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT;
```

## Systemd integration

We can integrate the above with systemd as follows:

Create a unit file called (e.g.) `/etc/systemd/system/our_databases.service`
```ini
[Unit]
Description=Our Oracle Databases
After=network.target

[Service]
Type=oneshot
User=root
RemainAfterExit=yes
WorkingDirectory=/var/opt/oracle
ExecStart=/scripts/system_start.sh
ExecStop=/scripts/system_stop.sh
TimeoutSec=900

[Install]
WantedBy=multi-user.target
```

This is pretty simple. `oneshot` means that the command runs once, then exits.
`RemainAfterExit` means that once the command exits systemd will consider the
service to still be running, so that it gets shut down.

To enable the service we need to do:

```bash
systemctl daemon-reload
systemctl enable --now our_databases
```

`daemon-reload` makes systemd read the new unit file.

`enable` enables the service (i.e. makes it stop and start on reboot),
and `--now` starts the service. Though the script is run, no databases
will be started as none were shut down. This was handy as the databases were
already running at this point.


## Another approach

Another approach would be to use systemd to control the database and listener
as follows. This approach was tested on the oracle free VM downloaded from
Oracle hence the path names (Though I have shortened them a little).


```ini
[Unit]
Description=Our Oracle Databases
Requires=network.target local-fs.target 

[Service]
Type=forking
User=oracle
WorkingDirectory=/home/oracle
Environment="ORACLE_HOME=/opt/oracle/23ai/dbhomeFree" "ORACLE_SID=FREE"
ExecStart=/oracle/23ai/dbhomeFree/bin/sqlplus /nolog @/etc/systemd/start.sql
ExecStop=/oracle/23ai/dbhomeFree/bin/sqlplus /nolog @/etc/systemd/stop.sql
RestartSec=30
Restart=always

[Install]
WantedBy=multi-user.target
```

This has the advantage that we can check on the status of the
processes (Output is edited to reduce the width).

```
# systemctl status oracle_free
● oracle_free.service
   Loaded: loaded (oracle_free.service; static; vendor preset: disabled)
   Active: active (running) since Wed 2025-01-22 15:37:02 UTC; 4min 3s ago
  Process: 12339 ExecStart=sqlplus /nolog @/start.sql (code=exited, status=0/SUCCESS)
 Main PID: 12292 (code=exited, status=0/SUCCESS)
    Tasks: 73 (limit: 24588)
   Memory: 2.0G
   CGroup: /system.slice/oracle_free.service
           ├─12345 db_pmon_FREE
           ├─12349 db_clmn_FREE
           ├─12353 db_psp0_FREE
...
```
But we also need the listener to be started. At first I thought I could create a socket file
but it seems that the listener doesn't support that. It would need to
have been coded to enable systemd to pass the open socket to the
listener, and that hasn't been done. Since we have one listener per
database, I just changed the database scripts to start and stop
the listener.

So in the end I abandoned this idea, and stuck with my original plan.


## It didn't work! Systemd killed my processes!

So with great excitement, we rebooted the server, and all the databases
were killed less than a second after the shutdown script was invoked.
The processes were read into the loop, but by the time it came to
trying to extract information, they were dead. The alert logs noted
that the processes were being killed.

### Don't kill my processes

It seems systemd is working as designed, killing user processes.
We need to instruct it not to do so, by editing
`/etc/systemd/logind.conf`
and adding:
```
KillExcludeUsers=oracle
```

### What does linger do?


But wait!

We can create services dynamically

systemd-run --uid=oracle --unit=FREE --service-type=forking  /opt/oracle/product/23ai/dbhomeFree/bin/dbstart

We can move processes into cgroups by writing their pids to a file:

/sys/fs/cgroup/systemd{name}/cgroup.procs

The cgroup name for a process is in: 

/proc/<pid>/cgroup
1:name=systemd:/system.slice/FREE.service

So we need another script to run in a cgroup which then adds the database processes to that cgroup. This is done by 