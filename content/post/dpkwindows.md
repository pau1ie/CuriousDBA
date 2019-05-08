---
title: "Peoplesoft DPK on Windows"
date: 2019-01-23T12:06:52Z
draft: true
---

Once again I have the pleasure of trying to automate the installation of
peoplesoft on windows. Oracle have chosen in their wisdom to use puppet,
a nice open source automated configuration management tool, and wrap it
in an interactive script with no silent options, forcing us to type
things in.

This is easy to get round in Unix because it allows redirection of input,
so I can use the interactive installer and a program like expect. 
Windows is much less friendly than Linux as I have found previously.

I previously did this with 8.55, but the installer has changed from poweshell
to a batch script which calls python. Looking at the code I see there is a
silent option.

psft-dpk-setup.bat --env_type midtier --deploy_only --silent --response c:\temp\response.txt

It needs a response file with the following parameters:

psft_base_dir=C:\psft
db_name=CS92U011
db_service_name=CS92U011
db_host=dbserver
connect_id=people
connect_pwd=peop1e
admin_pwd=Abcdefgh_123
opr_pwd=x
access_pwd=x

Presumably the rest are defaulted. It works anyway. The problem with this
approach is that it is not supported. Now I bet Oracle have this
option because they use it to provision their "cloud" servers, so
I am sure it would work, but as professionals obviosuly we try not
to run unsupported processes. Of course, this is a possible approach
and if we got an error we could run the install manually for a 
supported configuration and reproduce it. This is something we
do in other areas.

The install I need to do has the following options:

psft-dpk-setup.bat --env_type midtier --deploy_only

And asks the following questions;

Enter the PeopleSoft Base Folder: C:\psft
Are you happy with your answer? [Y|n|q]:
Enter the PeopleSoft database platform [Oracle]:
Is the PeopleSoft database unicode? [Y|n]:
Are you happy with your answers? [y|n]: y
Do you want to continue with the default initialization process? [y|n]: y

So, can I create a file like the following and redirect it into the process?

C:\psft



y
y

type cat.txt | psft-dpk-setup.bat --env_type midtier --deploy_only

This works. I notice oracle deliver python, which has the possibility
of installing pexpect. However it isn't delivered with it. It could
be installed with pip (Pip doesn't work directly, but can be invoked
from python using 

python -m pip install

But since the above works, I didn't bother.

