---
title: "Ansible On Windows"
date: 2018-01-30T15:12:11Z
draft: true
---

Historically I have been a Unix user, so I don't understand windows. I am
pleased with my ansible build of the peoplesoft web servers, application
servers and unix process schedulers. However, it appears that our
Windows process schedulers aren't going away any time soon, so I ought
to convert that to be an automated process.

The ansible documentaton [mentions Windows](docs.ansible.com/ansible/latest/intro_windows.html)
breifly, but it isn't clear what it means. I connect using a local user, but
then use an Active Directory (AD) user to mount a share, and run the installer.

The ansible documentation suggests that basic authentication is the easiest. All
you need to do is

{{< highlight console >}}
pip install pywinrm
{{< /highlight >}}

Smashing.

But it doesn't work...

Here is my setup. 

{{< highlight console >}}
|____inventory
| |____inventory
|____win
| |____tasks
| | |____main.yml
| |____files
| | |____maprun.ps1
|____win.yml
{{< /highlight >}}

Here is my inventory:


{{< highlight ini >}}
[win]
winhost

[winpsu:vars]
ansible_user="ansible_user"
ansible_password="Password1"
ansible_connection="winrm"
ansible_winrm_server_cert_validation=ignore
win_user="our.ad.domain.cam.ac.uk\ad_user"
win_pass="Password2"
{{< /highlight >}}

See how secure our passwords are!

And the playbook:

{{< highlight YAML >}}
win.yml 

---
  - name: Windows PSU
    hosts: winpsu
    roles:
      - psw
{{< /highlight >}}

map.ps1
{{< highlight powershell >}}
param(
  $map_user,
  $map_password,
  $script
)
# net use Z \\internal\general\Peoplesoft_Applications $map_password /user:$map_user
echo a
#$PWord = ConvertTo-SecureString $map_password -AsPlainText -Force
$PWord="$map_password"|ConvertTo-SecureString -AsPlainText -Force
echo b
#$myCreds = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $map_user,$PWord
$myCreds = New-Object System.Management.Automation.PsCredential($map_user,$PWord)
echo c
echo "$map_user"
echo "$map_password"
New-PSDrive -Name "Z" -PSProvider "FileSystem" -Root "\\internal\general" -Credential $myCreds
echo Invoke-Command -ScriptBlock $script
{{< /highlight >}}

And main.yml
{{< highlight YAML >}}
---
  - name: Ping windows server
    win_ping:

  - name: Remove services
    win_service:
      name: "{{ item }}"
      state: absent
    with_items:
      - service1
      - service2

  - name: Remove the peoplesoft installation
    script: 'files/maprun.ps1 -map_user "{{ win_user }}" -map_password "{{ win_pass }}" -script "dir Z:"'
    register: cleanup

  - debug: var=cleanup
{{< /highlight >}}

Give it a go, and see what happens:
{{< highlight console >}}
$ ansible-playbook -i inventory/csopswin psw.yml 

PLAY [Windows PSU] ***********************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************
/usr/lib/python2.7/site-packages/urllib3/connectionpool.py:769: InsecureRequestWarning: Unverified HTTPS request is being made. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.org/en/latest/security.html
  InsecureRequestWarning)
ok: [winhost]

TASK [psw : Ping windows server] *********************************************************************************************************************************
/usr/lib/python2.7/site-packages/urllib3/connectionpool.py:769: InsecureRequestWarning: Unverified HTTPS request is being made. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.org/en/latest/security.html
  InsecureRequestWarning)
ok: [winhost]

TASK [psw : Remove services] *************************************************************************************************************************************
/usr/lib/python2.7/site-packages/urllib3/connectionpool.py:769: InsecureRequestWarning: Unverified HTTPS request is being made. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.org/en/latest/security.html
  InsecureRequestWarning)
ok: [winhost] => (item=service1)
ok: [winhost] => (item=service2)

TASK [psw : Remove the peoplesoft installation] ******************************************************************************************************************
/usr/lib/python2.7/site-packages/urllib3/connectionpool.py:769: InsecureRequestWarning: Unverified HTTPS request is being made. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.org/en/latest/security.html
  InsecureRequestWarning)
fatal: [winhost]: FAILED! => {"changed": true, "failed": true, "rc": 1, "stderr": "New-PSDrive : A specified logon session does not exist. It may already have \r\nbeen terminated\r\nAt C:\\Users\\ansible_user\\AppData\\Local\\Temp\\ansible-tmp-1517324720.24-2169618948479\r\n68\\maprun.ps1:16 char:1\r\n+ New-PSDrive -Name \"Z\" -PSProvider \"FileSystem\" -Root \"\\\\internal\\general\" \r\n-Crede ...\r\n+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~\r\n~~~\r\n    + CategoryInfo          : InvalidOperation: (Z:PSDriveInfo) [New-PSDrive], \r\n    Win32Exception\r\n    + FullyQualifiedErrorId : CouldNotMapNetworkDrive,Microsoft.PowerShell.Com \r\n   mands.NewPSDriveCommand\r\n \r\n\r\n", "stdout": "a\r\nb\r\nc\r\nour.ad.domain.cam.ac.uk\\ad_user\r\nPassword2\r\nInvoke-Command\r\n-ScriptBlock\r\ndir Z:\r\n", "stdout_lines": ["a", "b", "c", "our.ad.domain.cam.ac.uk\\ad_user", "Password2", "Invoke-Command", "-ScriptBlock", "dir Z:"]}
        to retry, use: --limit @/home/psh35/scripts/camsis-ansible/psw.retry

PLAY RECAP *******************************************************************************************************************************************************
winhost                   : ok=3    changed=0    unreachable=0    failed=1   
{{< /highlight >}}

So it works apart from I can't get it to map the share. I think the reason for
this is because I need to use credential delegation. To do this, according to
the documentation I need to add ansible_winrm_transport=credssp. So now my
inventory looks like this:


{{< highlight ini >}}
[win]
winhost

[winpsu:vars]
ansible_user="ansible_user"
ansible_password="Password1"
ansible_connection="winrm"
ansible_winrm_transport=credssp
ansible_winrm_server_cert_validation=ignore
win_user="our.ad.domain.cam.ac.uk\ad_user"
win_pass="Password2"
{{< /highlight >}}

Also, I need to install credssp for python, or I get the following error:

{{< highlight console >}}
$ ansible-playbook -i inventory/csopswin psw.yml 

PLAY [Windows PSU] ***********************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************
fatal: [csopspsw]: UNREACHABLE! => {"changed": false, "msg": "credssp: requests auth method is credssp, but requests-credssp is not installed", "unreachable": true}
        to retry, use: --limit @/home/psh35/scripts/camsis-ansible/psw.retry

PLAY RECAP *******************************************************************************************************************************************************
csopspsw                   : ok=0    changed=0    unreachable=1    failed=0   
{{< /highlight >}}

Install it with pip

{{< highlight console >}}
# pip install pywinrm[credssp]
Requirement already satisfied: pywinrm[credssp] in /usr/lib/python2.7/site-packages
Requirement already satisfied: xmltodict in /usr/lib/python2.7/site-packages (from pywinrm[credssp])
Collecting requests>=2.9.1 (from pywinrm[credssp])
  Downloading requests-2.18.4-py2.py3-none-any.whl (88kB)
    100% |████████████████████████████████| 92kB 2.7MB/s 
Requirement already satisfied: requests_ntlm>=0.3.0 in /usr/lib/python2.7/site-packages (from pywinrm[credssp])
Requirement already satisfied: six in /usr/lib/python2.7/site-packages (from pywinrm[credssp])
Collecting requests-credssp>=0.0.1 (from pywinrm[credssp])
  Downloading requests_credssp-0.1.0-py2.py3-none-any.whl
Collecting certifi>=2017.4.17 (from requests>=2.9.1->pywinrm[credssp])
  Downloading certifi-2018.1.18-py2.py3-none-any.whl (151kB)
    100% |████████████████████████████████| 153kB 2.6MB/s 
Collecting chardet<3.1.0,>=3.0.2 (from requests>=2.9.1->pywinrm[credssp])
  Downloading chardet-3.0.4-py2.py3-none-any.whl (133kB)
    100% |████████████████████████████████| 143kB 3.3MB/s 
Collecting idna<2.7,>=2.5 (from requests>=2.9.1->pywinrm[credssp])
  Downloading idna-2.6-py2.py3-none-any.whl (56kB)
    100% |████████████████████████████████| 61kB 5.1MB/s 
Collecting urllib3<1.23,>=1.21.1 (from requests>=2.9.1->pywinrm[credssp])
  Downloading urllib3-1.22-py2.py3-none-any.whl (132kB)
    100% |████████████████████████████████| 133kB 3.0MB/s 
Requirement already satisfied: python-ntlm3 in /usr/lib/python2.7/site-packages (from requests_ntlm>=0.3.0->pywinrm[credssp])
Collecting pyOpenSSL>=16.0.0 (from requests-credssp>=0.0.1->pywinrm[credssp])
  Downloading pyOpenSSL-17.5.0-py2.py3-none-any.whl (53kB)
    100% |████████████████████████████████| 61kB 5.8MB/s 
Collecting ntlm-auth (from requests-credssp>=0.0.1->pywinrm[credssp])
  Downloading ntlm_auth-1.0.6-py2.py3-none-any.whl
Collecting cryptography>=2.1.4 (from pyOpenSSL>=16.0.0->requests-credssp>=0.0.1->pywinrm[credssp])
  Downloading cryptography-2.1.4-cp27-cp27mu-manylinux1_x86_64.whl (2.2MB)
    100% |████████████████████████████████| 2.2MB 458kB/s 
Collecting cffi>=1.7; platform_python_implementation != "PyPy" (from cryptography>=2.1.4->pyOpenSSL>=16.0.0->requests-credssp>=0.0.1->pywinrm[credssp])
  Downloading cffi-1.11.4-cp27-cp27mu-manylinux1_x86_64.whl (406kB)
    100% |████████████████████████████████| 409kB 1.5MB/s 
Requirement already satisfied: enum34; python_version < "3" in /usr/lib/python2.7/site-packages (from cryptography>=2.1.4->pyOpenSSL>=16.0.0->requests-credssp>=0.0.1->pywinrm[credssp])
Collecting asn1crypto>=0.21.0 (from cryptography>=2.1.4->pyOpenSSL>=16.0.0->requests-credssp>=0.0.1->pywinrm[credssp])
  Downloading asn1crypto-0.24.0-py2.py3-none-any.whl (101kB)
    100% |████████████████████████████████| 102kB 7.2MB/s 
Requirement already satisfied: ipaddress; python_version < "3" in /usr/lib/python2.7/site-packages (from cryptography>=2.1.4->pyOpenSSL>=16.0.0->requests-credssp>=0.0.1->pywinrm[credssp])
Requirement already satisfied: pycparser in /usr/lib/python2.7/site-packages (from cffi>=1.7; platform_python_implementation != "PyPy"->cryptography>=2.1.4->pyOpenSSL>=16.0.0->requests-credssp>=0.0.1->pywinrm[credssp])
Installing collected packages: certifi, chardet, idna, urllib3, requests, cffi, asn1crypto, cryptography, pyOpenSSL, ntlm-auth, requests-credssp
  Found existing installation: chardet 2.2.1
    Uninstalling chardet-2.2.1:
      Successfully uninstalled chardet-2.2.1
  Found existing installation: idna 2.4
    Uninstalling idna-2.4:
      Successfully uninstalled idna-2.4
  Found existing installation: urllib3 1.10.2
    Uninstalling urllib3-1.10.2:
      Successfully uninstalled urllib3-1.10.2
  Found existing installation: requests 2.6.0
    DEPRECATION: Uninstalling a distutils installed project (requests) has been deprecated and will be removed in a future version. This is due to the fact that uninstalling a distutils project will only partially uninstall the project.
    Uninstalling requests-2.6.0:
      Successfully uninstalled requests-2.6.0
  Found existing installation: cffi 1.6.0
    Uninstalling cffi-1.6.0:
      Successfully uninstalled cffi-1.6.0
  Found existing installation: cryptography 1.7.2
    Uninstalling cryptography-1.7.2:
      Successfully uninstalled cryptography-1.7.2
  Found existing installation: pyOpenSSL 0.13.1
    DEPRECATION: Uninstalling a distutils installed project (pyOpenSSL) has been deprecated and will be removed in a future version. This is due to the fact that uninstalling a distutils project will only partially uninstall the project.
    Uninstalling pyOpenSSL-0.13.1:
      Successfully uninstalled pyOpenSSL-0.13.1
Successfully installed asn1crypto-0.24.0 certifi-2018.1.18 cffi-1.11.4 chardet-3.0.4 cryptography-2.1.4 idna-2.6 ntlm-auth-1.0.6 pyOpenSSL-17.5.0 requests-2.18.4 requests-credssp-0.1.0 urllib3-1.22
{{< /highlight >}}

Try again...

{{< highlight console >}}
[psh35@whisk camsis-ansible]$ ansible-playbook -i inventory/csopswin psw.yml 

PLAY [Windows PSU] ***********************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************
fatal: [csopspsw]: UNREACHABLE! => {"changed": false, "msg": "credssp: 'module' object has no attribute 'TLSv1_2_METHOD'", "unreachable": true}
        to retry, use: --limit @/home/psh35/scripts/camsis-ansible/psw.retry

PLAY RECAP *******************************************************************************************************************************************************
csopspsw                   : ok=0    changed=0    unreachable=1    failed=0   
{{< /highlight >}}

This is more difficult. Ansible needs pyOpenSSL version 17 or something, which
is installed by pip. However RHEL installs version 0.13, which is first in the
python path. 


{{< highlight console >}}
$ python
Python 2.7.5 (default, May  3 2017, 07:55:04) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-14)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> print (sys.path)
['', '/usr/lib/python2.7/site-packages/oraclesql_pygments_lexer-0.1-py2.7.egg', '/usr/lib64/python27.zip', '/usr/lib64/python2.7', '/usr/lib64/python2.7/plat-linux2', '/usr/lib64/python2.7/lib-tk', '/usr/lib64/python2.7/lib-old', '/usr/lib64/python2.7/lib-dynload', '/usr/lib64/python2.7/site-packages', '/usr/lib64/python2.7/site-packages/gtk-2.0', '/usr/lib/python2.7/site-packages']
{{< /highlight >}}

The RPM version is in /usr/lib64/python2.7/site-packages/OpenSSL while the pip
version is in /usr/lib/python2.7/site-packages/OpenSSL/
Since the RPM version is first in the path it gets called.

