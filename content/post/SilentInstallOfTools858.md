---
title: "Silent Installation of Tools 8.58"
date: 2020-06-04T10:00:00Z
tags: ['Anbsible', 'Automation', 'PeopleSoft','Upgrades','Windows']
---

## Silent install
A new feature of 8.58 is the silent installation. The documentation says you can run a command such as:

```bash
./psft_dpk_setup.sh \
  --silent \
  --response_file=response.txt \
  --customization_file=psft_customizations.yaml
```

The customizations file is optional, but I decided this time around to use it as much as possible
to create a system as close as possible to what I wanted to end up with. So I copied the file from 
`<psft_base_dir>/dpk/puppet/production/data/psft_customizations.yaml` and edited it to my satisfaction.
`<psft_base_dir>` above is whatever was specified when the installer asked interactively, or in the
response file. This means I had to do a test install before I could do a proper one, but that is fine.

The response file looks like this:
```ini
db_platform=ORACLE
db_name=
db_service_name=
db_host=
db_port=
db_protocol=TCP
connect_id=people
connect_pwd=
opr_id=
opr_pwd=
admin_pwd=
access_id=SYSADM
access_pwd=
weblogic_admin_pwd=
webprofile_user_id=
webprofile_user_pwd=
gw_user_id=administrator
gw_user_pwd=
psft_base_dir=
```

## Specify deploy type
The deploy type is still specified on the command line. I had assumed this could be 
done in `psft_customizations.yaml`, but it can't. So the command becomes something like
```bash
./psft_dpk_setup.sh \
  --silent \
  --env_type midtier \
  --domain_type appbatch \
  --response_file=response.txt \
  --customization_file=psft_customizations.yaml
```

## Passwords
I was confused for a while by the passwords. Appendix D of 
*PeopleSoft PeopleTools 8.58 Deployment Packages Installation* 
explains that the passwords are created using eyaml encrypt using public and private keys. 
But these keys are created by the install, so passwords can't be created before the install,
but also need to be created before the install to populate `psft_customizations.yaml`.
I realised this generally isn't required. Passwords can be provided in the clear in 
setup.txt, so there is no need to encrypt them!

## Weird error after setting db_host_name
I was also stumped by an error that seemed to indicate a coding issue:

```console
Error: Evaluation Error: Error while evaluating a Resource Statement, 
  Evaluation Error: Cannot reassign variable '$appsrvr_host_name_with_dot
```

It seems this is caused by setting the db_host_name variable in psft_customizations.yaml. 
It isn't clear what this does, but it isn't needed so can be removed. Then the install runs without issue.

## Operator Password
There does seem to be an error in the operator password validation. 
It is restricted to 8 characters without punctuation. This is not complex 
enough for my liking. So I supply a dummy password in the response file. 
Once the installation is complete I edit the
config files (psappsrv.cfg and psprcs.cfg) to add the password in clear text. 
Then I run a configure to encrypt them. I could also
use `expect` to drive psadmin, but that won't work in windows.

## PIA install fails
I had an annoying error when installing the PIA using a response file:

```console
# "/bin/bash" "/opt/ansible/PEOPLETOOLS-LNX-8.58.03/setup/psft-dpk-setup.sh" \
  "--silent" \
  "--env_type" "midtier" \
  "--domain_type" "pia" \
  "--response_file=/opt/ansible/response.txt"

Starting the PeopleSoft Environment Setup Process:

Validating User Arguments:
object of type 'NoneType' has no len()

Exiting the PeopleSoft environment setup process.
```

After some diving into the python installer code I found out that the installer was looking for two 
undocumented variables. If they are set as follows, the installer runs OK. They don't seem to be 
used at all, so setting them to xxx seems OK.

```ini
pia_psserver_list=xxx
prcs_server_name=xxx
```

## Cobol remote call is broken

The installer sets the value of `COBDIR` to blank in the `.bashrc` file. It is set correctly in 
`.bash_profile`, but since variables such as `LD_LIBRARY_PATH` are set in `.bashrc`, they end up being
wrong. Peoplesoft can't find the cobol installation, so can't run cobol programs. This can be tested by 
navigating to the peopletools debug utilities at PeopleTools->Utilities->Debug->PeopleTools Test Utilities.

Clicking the remote call test button puts up an error:

```
Cobol Program PTPNTEST aborted (-2,1)
FUNCLIB_UTILRC_TEST_PB. FieldChange PCPC:2151 Statement: 26
```

![Screenshot of the error](../../images/articles/RemoteCallFail.jpg)

The workaround is quite easy, simply edit `/home/psadm2/.bashrc` to change

```bash
export COBDIR=
```
to
```bash
export COBDIR=/opt/lib/cobol
```

## Windows service install

In windows, the `psadmin` menu allows a service to be created. This can be scripted as follows:

Edit `%PS_CFG_HOME%\appserv\pswinsrv.cfg` to add the process scheduler domain to the list, e.g.

```ini
Process Scheduler Domains=PRCSDOM
```

If there is more than one, they are comma separated as per the comment in the file.
Then run 

```cmd
%PS_HOME%\bin\server\WIN86\psntsrv.exe %PS_CFG_HOME% -install
```

The service then needs to be edited to be run as the correct user (that installed the software). I use win_service in ansible to do this:

```yaml
  - name: Edit service to run under ps_admin
    win_service:
      name: "{{ item }}"
      start_mode: "auto"
      username: "{{ ansible_host }}\\{{ ansible_user }}"
      password: "{{ ansible_password }}"
    with_items: 
      - "ORACLE ProcMGR V12.2.2.0.0_VS2017"
      - "TUXEDO 12.2.2.0.0_VS2017 Listener on Port 3050"
      - "PeopleSoft_C__Users_ps_admin_psft_pt_8.58"
```
It can also be done by hand.

The user needs to be able to log in as a service, by granting `SeServiceLogonRight` to the user.

## Conclusion

Having worked around the above issues, I am pleased with this method of installation. It seems to work well, and has fewer issues than
my previous approach of trying to customise a default installation.
