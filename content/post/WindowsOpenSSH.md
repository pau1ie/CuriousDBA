---
title: "Installing Open SSH on Windows (Automatically)"
date: 2019-06-04T10:44:34+01:00
tags: ["Ansible","Automation","Chocolatey","OpenSSH","PeopleSoft","PowerShell","Scripting","VM","Windows"]
---

Our PeopleSoft system has a couple of maintenance tasks which are kicked off from the database server.
I am converting it to use Ansible and a management server, but in the meantime I need this to work.

We had been using Bitvise SSH server on Windows, but experienced problems with it locking up
occasionally. Also we needed to create some new Windows VMs and wondered if there was a way
to do the work without paying for more licenses. Also as we are upgrading to Windows server 2016, I am
seeing if there is a way to automate this as part of my Ansible build.


## The Options

The title is a bit of a spoiler, but the options I considered were:

* RDP from linux e.g. using [FreeRDP](https://github.com/FreeRDP/FreeRDP). This is remote
  desktop protocol and may be heavier weight
  than we need. It also isn't clear whether it is possible to exit the session after the script
  completes. Probably only one connection is allowed at a time which is likely to cause problems
  later on.
* WinRM - This is what ansible uses. [PYWinRM](https://pypi.org/project/pywinrm/0.2.2/)
  is an option but needs to be installed.
* Install an OpenSSH server. The
  [Microsoft Powershell team maintain a version](https://github.com/PowerShell/Win32-OpenSSH/releases)
  which is integerated into the next version of Windows. This is probably the way to go.
* [WinEXE](https://github.com/skalkoto/winexe). This has not been maintained for 6 years
  so is not a good option.

We are using Windows Server 2016. Server 2019 has OpenSSH integrated, so hopefully using
it now will ease the transition.


## Installing OpenSSH

Ansible uses [Chocolatey](https://chocolatey.org/) as a package manager for Windows, which is
kind of like the package managers that come with the Linux operating systems. So all I have to
do to make sure I have the latest version of OpenSSH is the following in ansible:

{{< highlight yaml >}}
- name: Install openssh
  win_chocolatey:
    name: openssh
    package_params: /SSHServerFeature
    state: present
  tags: openssh
{{< /highlight >}}

## Configuring OpenSSH

Once the above play is run, OpenSSH is installed and running with sensible defaults. The default
shell is the command prompt, which is what we wanted, and it is running and we can log in using
a username and password. This is almost too easy! All I need now is to work out how to configure
paswsordless logins, using keys.

In Unix all you have to do is create an authorized_keys file in the .ssh directory under the
users home, and make sure the permissions are correct. It looks like this should work in Windows
(The home directory is ```/users/username```) but it didn't seem to work. I think in part
it is because I wanted to connect as an administrator. I don't understand the Microsoft permission
model very well, it seems quite complicated.

The OpenSSH configuration options are in ```C:\ProgramData\ssh```. In this folder is a file called
[sshd_config](https://github.com/PowerShell/openssh-portable/blob/v7.9.0.0/contrib/win32/openssh/sshd_config),
which contains the configuration options. At the end of this file is a section
which reads:

```
Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

So I created the file ```administrators_authorized_keys``` and added the public key for the server
I was connecting from (Which was in ```/home/user/.ssh/id_rsa.pub```), and changed the permissions
as per the screenshot below. Note in partticular the owner ```hostname\Administrators```, and 
permissions, ```SYSTEM``` and ```Administrators``` have full control, but no others. In particular
I had to remove the read permission for all users. To do this, I had to disable inheritance.

![Windows authorized_keys file permissions screenshot](../../WindowsOpenSSH/permissions.png)

## Automating with Ansible

So, all we have to do is add a line to a file, and then make sure the permissions are correct.
How hard can that be?

{{< highlight YAML >}}
  - name: Install openssh
    win_chocolatey:
      name: openssh
      package_params: /SSHServerFeature
      state: present
    tags: openssh

  - name: Set up authorised key file
    win_lineinfile:
      path: C:\ProgramData\ssh\administrators_authorized_keys
      line: "{{ lookup('file', 'sshkey/files/{{ dbserver }}.pub') }}"
      create: Yes
    tags: openssh

  - name: Disable inheritence
    win_acl_inheritance:
      path: C:\ProgramData\ssh\administrators_authorized_keys
      state: absent
    tags: openssh

  - name: Set the owner on the authorised keys file.
    win_owner:
      path: C:\ProgramData\ssh\administrators_authorized_keys
      user: "{{ ansible_hostname }}\\Administrators"
    tags: openssh

  - name: Don't allow users to view the file
    win_acl:
      path: C:\ProgramData\ssh\administrators_authorized_keys
      user: "Authenticated Users"
      rights: Read,Write,Modify,FullControl,Delete
      type: allow
      state: absent
    tags: openssh

  - name: Do allow admins to access the file
    win_acl:
      path: C:\ProgramData\ssh\administrators_authorized_keys
      user: "{{ item }}"
      rights: FullControl
      type: allow
      state: present
    with_items:
      - SYSTEM
      - "{{ ansible_hostname }}\\Administrators"
    tags: openssh
{{< /highlight >}}

Note that in the first step, I am using a lookup for the public key I am storing in another role. You will probably
want to store that key more sensibly, but it contains the contents of ```~/.ssh/id_rsa.pub``` from the database
server user I want to connect from. Also I should probably store the filename in a variable.


## Debugging

### Chocolatey failing

There was a change in the return codes that chocolatey delivered when no software was installed
which isn't reflected in the version of Ansible I have. To get round this I needed to revert the
previous behaviour of chocolatey:

{{< highlight PowerShell >}}
choco feature disable --name="'useEnhancedExitCodes'"
{{< /highlight >}}


### Checking Authorised Key File Permissions

To check the permissions are set up properly, a PowerShell script is delivered. Once I had done
the above, it worked fine:

{{< highlight PowerShell >}}
cd 'C:\Program Files\OpenSSH-Win64\'
.\FixHostFilePermissions.ps1
{{< /highlight >}}

This produces output like the following:

![FixHostFilePermissions output](../../WindowsOpenSSH/checkperms.png)


### Running OpenSSH in Debug Mode
To test the connection it is possible to stop the SSH service then run it in debug mode.
Bear in mind that you have to connect as the user who is running the SSHd.

{{< highlight PowerShell >}}
cd 'C:\Program Files\OpenSSH-Win64\'
Stop-Service sshd
.\sshd.exe -d
{{< /highlight >}}

Test connecting, and if there are problems, hopefully the debug messages will be helpful. When I disconnected,
the SSHd exited. Then I started the service again:

{{< highlight PowerShell >}}
Start-Service sshd
{{< /highlight >}}

It is also possible to run the client in verbose mode. Logging is increased for each v added up to three:

{{< highlight bash >}}
ssh -vvv user@hostname
{{< /highlight >}}

