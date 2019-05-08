---
title: "Automated Testing Users"
date: 2018-11-05T16:42:08Z
tags: ["Automation","Jenkins","Scripting","Security","Testing","Oracle"]
---

Imagine a situation where a production database is copied to test, and the data scrambled.
Our testers want to access the database because they need to find some data to use for testing.

* The testing is automated.
* The refresh is automated
* The scrambling is automated.

How do we get the testers the password in a secure fashion? Also, how do we ensure testers don't
just use the test user normally? They should log in with their own user name and passwords. Also,
the test user shouldn't have the same password all the time, it should change periodically. 
This will ensure that if a tester has found out the password because they need it to develop
the automated tests, they won't know it after they leave, or be tempted to use it when they
ought to be using their own user name.

Since everything is automated, we can take advantage of this to:

Create a source repository (Our testers use git/bitbucket. Other options are available and work just as well).

1. Create an encrypted file in it with the password.
* Use Gnu Privacy Guard (gpg) to encrypt and decrypt the password. 
* Push to bitbucket
* The automated test can pull the repo
* decrypt the password using gpg
* Use the password to connect to the database.

To make this even easier, my colleague [Tony Finch](https://dotat.at/) has written a wrapper
around gpg called [regpg](https://dotat.at/prog/regpg/). Everything is possible without regpg, it just makes
things a little easier to use with git.

What I did to set things up is:

1. Generate a gpg key as per the [regpg tutorial](https://dotat.at/prog/regpg/doc/tutorial.html), but make sure it has a blank password.  
* Repeat for all servers/VMs that will need access to the secrets.
* Create a project as per the tutorial using `regpg init`
* Copy all the public keys to each server and import them into each others keyring (`gpg --import`) so the servers/VMs know about each other
* Create a repository - regpg init
* Enrol the public keys from the other VMs as admins using `regpg add <key id>` so everyone can read and write.
* `git commit -m 'Enrol other servers'` 
* `git push`

Now everyone can get a copy of the empty repository. However, to be useful we need to put something in it. So to 
create a user in the database, we can do something like:

    cd /path/to/repo
    git pull
    user=testuser
    if [[ ! -d $ORACLE_SID ]]
    then
        mkdir $ORACLE_SID
    fi
    tr -cd '[:alnum:]' < /dev/urandom | head -c 29
    sqlplus / as sysdba <<-! 
    create user $user identified by "$pw"
    default tablespace users temporary tablespace temp;
    grant connect, admin to $user;
    !
    rm -f $ORACLE_SID/${user}.asc
    echo $pw | regpg encrypt -q - $ORACLE_SID/${user}.asc
    git add $ORACLE_SID/$user.asc
    git commit -m "$ORACLE_SID $user created"
    git push

This way we have a directory per database with files of one password per file. They are encrypted, so they can
be stored in a public repository without anyone who shouldn't be able to being able to see the password.

The testers need to add to their script 

    cd /path/to/repo
    git pull
    user=testuser
    passwd="$( gnpg -q -d $ORACLE_SID/$user )"

Then they can connect to the user at the remote database using the password in the variable `$passwd`.

