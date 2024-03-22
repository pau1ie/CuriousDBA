---
title: "Installing Change Assistant"
date: 2024-03-22T16:57:49+00:00
tags: ['PeopleSoft','Automation','Ansible','Windows','Change Assistant']
---

I am not sure what best practice would be as to a location to install and
run change assistant. To get the GUI it has to be on Windows. Since we
use a VPN we can't really connect from a laptop as if the VPN drops out
we will interrupt the process which can run for several hours. So at
present we use a Windows server which has a full PeopleSoft installation
to run change assistant.

To install Change Assistant I use Ansible to create a task to run the installer.
This assumes
that PeopleTools has already been installed. The task runs a command prompt with
the following parameters (Wrapped for readability)

```cmd
/c C:\psoft\pt\ps_home8.60.99\setup\PsCA\silentInstall.bat
"C:\Program Files\PeopleSoft\Change Assistant"
UPGRADE
BACKUP
```

I experimented with different options, and this works, so I stuck with it.

We have tried to get Change Assistant to
work if we log in with an unprivileged account. The idea is that the
install is done as an administrator, as Oracle requires this, but we
can run the upgrade as a normal user.

There is an outstanding 
[enhancement request](https://community.oracle.com/mosc/discussion/4390885/run-change-assistant-without-as-administrator),
but it has not got much traction. However, the the
[psadmin.io]( https://psadmin.io/2017/03/09/running-change-assistant-without-as-administrator/)
people have discovered that if you allow change assistant  to write to it's installation folder, it works OK.

This is the ansible command I use to achieve that:
```yaml
- name: Allow users to update Change Assistant folder
  ansible.windows.win_acl:
    path: "C:\\Program Files\\PeopleSoft\\Change Assistant"
    user: "Authenticated Users"
    rights: FullControl
    state: present
    type: allow
```

Next, I set some default options. It seems to create a database I need to
give change assistant my login details, which I don't really want to do.
Colleagues will be using this desktop as well as myself. This is something
Change Assistant doesn't seem to cater for.
So as a start I will just create the basic defaults. 
First I create a response file. Since we have had issues with long filenames
in the past, we try to keep the paths as short as possible. Also I am not
particularly creative with naming directories!

```ini
[GENERAL]
MODE=UM
ACTION=OPTIONS
OUT=c:\Output\options.log
EXONERR=Y

[OPTIONS]
REPLACE=Y
SWP=False
PSH=C:\psoft\pt\ps_home8.60.99
PAH=C:\psoft\APPS
PCH=C:\psoft\CUST
STG=C:\Staging
OD=C:\Output
DL=C:\Download
SQH=C:\psoft\pt\oracle-client\19.3.0.0\bin\sqlplus.exe
EMYN=N
SRCYN=N
```

The Oracle documentation: 
_PeopleTools 8.60: Change Assistant and Update Manager_ is a little confusing
here. I raised an SR to hopefully get it corrected. This is the explanation
of the options I am setting:

| Option | Setting | Notes |
| ------ | ------- | ----- |
| Mode | UM | Update Manager |
| ACTION | OPTIONS | Set the default options for Update Manager |
| OUT | _file_ | Write a log file. |
| EXONERROR | Y | Exit on error. If set to N, the change assistant window will stay open. This is useful for testing, but since I am running this from ansible, I won't have access to the window, so won't be able to access the window. So it should be closed. |
| REPLACE | Y | Replace any currently existing configuration |
| SWP | False | Don't Show the Welcome Page (Why is this "False" and not "N" like everything else?!) |
| PSH | _folder_ | PSHOME directory. This was incorrect in the documentation. |
| PAH | _folder_ | PS APP Home directory. Also incorrect in the documentation. I suspect this can't be set with this command, as it isn't a general option. |
| PCH | _folder_ | PS CUST Home directory.  Again this isn't a general option, so it probably can't be set here. |
| STG | _folder_ | Staging directory for patches. |
| OD | _folder_ | Output directory |
| SQH | _file_ | Path to sqplplus, which is the SQL Query tool for Oracle. |
| EMYN | N | Don't configure EM Hub - We don't use it for tools patches |
| SRCYN | N | Don't configure the PUM source home. Again this isn't needed for tools patches. |

Then I apply them as follows (From a command prompt:

```cmd
cd "C:\Program Files\PeopleSoft\Change Assistant"
changeassistant.bat -INI C:\ca_options.ini
```

Where `ca_options.ini` is the response file created above.

We will have to have a think about accounts for change assistant. Currently
we run the process manually, so we use our own accounts. If we are moving to
an automated process, maybe we should create a service account for automated
application of tools updates?

