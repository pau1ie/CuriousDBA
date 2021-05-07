---
title: "Troubleshooting Etherpad In Google Cloud"
date: 2021-05-07T14:28:30Z
tags: ["Automation","Cloud","Database","Terraform"]
---

There is quite a lot that could go wrong with the work in the 
[previous post](../connectingthedatabase/), and I feel I experienced most of the issues!

We need to remember there are two projects in play here. One to create the container inside
the infrastructure, and the other to create the infrastructure the conatiner lives in.
A problem could be caused by either of these, and so the correct action needs to be taken
to correct the issue. Most of the issues I encountered were caused by the infrastructure, so
after I corrected each problem I ran:

```bash
logan terraform plan
logan terraform apply
```
to correct the infrastructure. As part of this process the container is restarted.

## Lint Terraform

When I check in my terraform code, by default it is linted to check it complies with
Terraform best practices. I didn't have equal signs lined up, so it failed. This was
quite easy to fix, and useful as it encourages best practice when writing code.

It is probably a good idea to get into  the habit of  doing  this by running:
```bash
logan terraform fmt -diff -recursive -check 
```
Terraform will give a diff style output telling you what needs  to be changed to make it happy.

## Looking at the logs

When a container fails to deploy, a link to the logs is helpfully displayed. 
The logs can be viewed from the [Google Cloud platform console](https://console.cloud.google.com/).
There are a couple of options to get to the logs. One is to
go to the _Cloud Run_ entry in the menu of Google Cloud Platform, select the name
of the webapp, then select the logs tab.

The other is to use the Log Explorer which is under Operations->Logging->Logs Explorer
in the Google Cloud Platfor menu, and is also accessible from above. The logs can be downloaded
from Actions here, which I found useful. 

## Secret decryption

### EXTRA_SETTINGS_URLS not needed

At first I left the `EXTRA_SETTINGS_URLS` variable from the boilerplate alone, so it 
would still be created in the container. My thinking was that it
couldn't do any harm even though it wasn't needed. This turned out not to be true as
Berglas identified as a secret it needed to decrypt and was confused about the format
as it contains two entries. The error in the log from Berglas said (Wrapped and edited):

```console
failed to access secret
 sm://project/webapp-secret-settings#1,gs://project/webapp-settings:
 failed to access secret:
 rpc error:
 code = InvalidArgument
 desc = Resource ID
 [projects/id/secrets/name/versions/1,gs://project/webapp-settings]
 is not in a valid format.
```

I didn't need this variable so I removed it from my Terraform code.

### Granting Permissions

Next I hadn't 
[given proper permission to the password secret](../connectingthedatabase/#granting-access-to-the-secret)
so Berglas couldn't access it:

```console
failed to access secret
 sm://project/webapp-secret-dbpass#1:
 failed to access secret:
 rpc error:
 code = PermissionDenied
 desc = Permission 'secretmanager.versions.access' denied for resource
 'projects/project/secrets/webapp-secret-dbpass/versions/1'
 (or it may not exist).
```

I corrected this and Berglas stopped complaining.


## Connecting to the Database - Connection Name

I fired up the container, but once again it failed. This time it got
as far as Etherpad, but the error message was more cryptic (Slightly 
edited and lines wrapped):

```console

               ^
    if (result.rows.length == 0) {
/opt/etherpad-lite/src/node_modules/ueberdb2/postgres_common.js:80
Container called exit(1).
[2021-03-12 16:22:57.829] [ERROR] console - (node:9) 
    [DEP0018] DeprecationWarning:  Unhandled promise rejections
    are deprecated. In the future, promise rejections that are
    not handled will terminate the Node.js process with a non-zero
    exit code.
[2021-03-12 16:22:57.829] [ERROR] console - (node:9)
        UnhandledPromiseRejectionWarning:
    Unhandled promise rejection. This error originated either by
    throwing inside of an async function without a catch block,
    or by rejecting a promise which was not handled with .catch().
    (rejection id: 1)
    at GetAddrInfoReqWrap.onlookup [as oncomplete] (dns.js:56:26)
[2021-03-12 16:22:57.829] [ERROR] console - (node:9)
        UnhandledPromiseRejectionWarning:
    Error: getaddrinfo ENOTFOUND project:europe-west2:sql-nnnnnnnn 
	project:europe-west2:sql-nnnnnnnn
process exited non-zero: exit status 1
"TypeError: Cannot read property 'rows' of undefined
    at Query.callback 
        (/opt/etherpad-lite/src/node_modules/ueberdb2/postgres_common.js:80:16)
    at Query.handleError
        (/opt/etherpad-lite/src/node_modules/pg/lib/query.js:145:17)
    at process.nextTick 
        (/opt/etherpad-lite/src/node_modules/pg/lib/client.js:66:13)
    at process._tickCallback (internal/process/next_tick.js:61:11)"
``` 

We can see it can't connect to the database, and it can't use it
after that. It seems to be referring to the database I passed.

The 
[hands on guide mentions](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#connect-the-database-to-the-web-application)
that the database name needs to have
`/cloudsql/` added to the start of the name, so I changed my Terraform 
to do that, and that worked.

## Wrong Username

The next error was like this:

```console
    if (result.rows.length == 0) {
/opt/etherpad-lite/src/node_modules/ueberdb2/postgres_common.js:80
Container called exit(1).
[2021-03-12 17:43:19.904] [ERROR] console - (node:9) [DEP0018] DeprecationWarning:
    Unhandled promise rejections are deprecated. In the future, promise
    rejections that are not handled will terminate the Node.js process
    with a non-zero exit code.
[2021-03-12 17:43:19.904] [ERROR] console - (node:9) 
    UnhandledPromiseRejectionWarning: Unhandled promise rejection. This 
    error originated either by throwing inside of an async function without
    a catch block, or by rejecting a promise which was not handled with
    .catch(). (rejection id: 1)
    at Pipe.onStreamRead [as onread] (internal/stream_base_commons.js:94:17)
    at Socket.Readable.push (_stream_readable.js:224:10)
    at readableAddChunk (_stream_readable.js:269:11)
    at addChunk (_stream_readable.js:288:12)
    at Socket.EventEmitter.emit (domain.js:448:20)
    at Socket.emit (events.js:198:13)
    at Socket.<anonymous>
        (/opt/etherpad-lite/src/node_modules/pg/lib/connection.js:129:22)
    at Connection.parseMessage 
        (/opt/etherpad-lite/src/node_modules/pg/lib/connection.js:413:19)
    at Connection.parseE 
        (/opt/etherpad-lite/src/node_modules/pg/lib/connection.js:614:13)
[2021-03-12 17:43:19.904] [ERROR] console - (node:9)
         UnhandledPromiseRejectionWarning:
    error: password authentication failed for user "admin"
process exited non-zero: exit status 1
"TypeError: Cannot read property 'rows' of undefined
    at Query.callback 
        (/opt/etherpad-lite/src/node_modules/ueberdb2/postgres_common.js:80:16)
    at Query.handleError 
        (/opt/etherpad-lite/src/node_modules/pg/lib/query.js:145:17)
    at process.nextTick
        (/opt/etherpad-lite/src/node_modules/pg/lib/client.js:66:13)
    at process._tickCallback (internal/process/next_tick.js:61:11)"
```

When connecting to a database
there are three pieces of information that need to be correct. The database needs
to be the correct one, the user id needs to be correct and so does the password. 
I had corrected the database, so assumed the password was wrong.
In fact it was the user ID - I had copied the admin user in terraform
rather than the webapp one. Once I corrected that, it worked. However it is
useful to be able to check that the username and password is correct.

### Testing the username and password

In the Google Cloud Platform website, go to Databases->SQL in the menu.
Select the database for your project, and click on it. Under the CPU graph
is a box saying _Connect to this instance_ If you expand _Connect using Cloud Shell_
then a cloud shell will open at the bottom of the page with a pre-populated command:
```bash
gcloud sql connect sql-nnnnnnnn --user=postgres --quiet
```
Change  the username to the one the application will use. An _Authorise Cloud Shell_ 
popup will appear asking for permission to make a GCP API call. Click on Authorise, 
then it will take a few seconds to
allowlist the incoming IP address. Finally  it will ask for the password.

This is really useful because it lets us test whether an error is caused by an
invalid username and password or something else. A valid login looks like this:
```console
me@cloudshell:~ (project)$ gcloud sql connect sql-nnnnnnnn --user=myuser --quiet
Allowlisting your IP for incoming connection for 5 minutes...done.
Connecting to database with SQL user [myuser].Password:
psql (13.2 (Debian 13.2-1.pgdg100+1), server 11.10)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
webapp=>
webapp=> \q
```
And an invalid login looks like this:
```console
me@cloudshell:~ (project)$ gcloud sql connect sql-nnnnnnn --user=wronguser --quiet
Allowlisting your IP for incoming connection for 5 minutes...done.
Connecting to database with SQL user [wronguser].Password:
psql: error: FATAL:  password authentication failed for user "wronguser"
FATAL:  password authentication failed for user "wronguser"
me@cloudshell:~ (project)$
```

## Conclusion

While it was annoying  to encounter so many errors in setting up the application,
I found it was useful to start to understand how to debug issues, to look in logs
and manually test the database connection. This also concludes
a mini series on the Google Cloud.