---
title: "Managing Windows with Ansible"
date: 2018-06-26T18:05:23+01:00
tags: ["Automation","Ansible","Windows","PowerShell"]
image: images/engine-rooms-with-gears.jpg
---

After a lot of trial and error, I finally found a way to make it work. Here is what I did.

As [noted before](../gettingwindowsandansibletoplaynicely/), rather than trying to run an installer from a share,
I simply copied it on to the local disc of the VM.

Before running Ansible on windows, the operating system has to be configured using (for example) the 
[ConfigureRemotingForAnsible.ps1](https://github.com/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1)
script. I took the defaults.  I didn't bother setting up any of the advanced options like CredSSP, as in practice there didn't seem to be
any benefit.

I assume you are pretty familiar with Ansible. The structure is what you will recognise.

{{< highlight console >}}
|____inventory
| |____inventory
|____win
| |____tasks
| | |____main.yml
| |____files
| | |____schedrun.ps1
| |____templates
| | |____copyinstaller.bat
|____win.yml
{{< /highlight >}}


Inventory File
---

{{< highlight ini >}}
[win]
winhost

[win:vars]
ansible_user="ansible_user"
ansible_password="Password1"
ansible_connection="winrm"
ansible_winrm_server_cert_validation=ignore
{{< /highlight >}}

I use password authentication. It's all very low tech.

Now, the playbook. This copies the file to the local location.

win.yml
---

{{< highlight YAML >}}
---
  - name: Windows
    hosts: winpsu
    roles:
      - psw
{{< /highlight >}}

This just calls the role.

main.yml
---

{{< highlight YAML >}}
- name: Create directories for use by the install
    win_file:
      path: "{{ item }}"
      state: directory
    with_items:
      - "{{ install_base }}"
      - "{{ ansibletmp }}"
      - "{{ scriptdir }}"

  - name: Deploy copy script
    win_template:
      src: templates/copyinstaller.bat.j2
      dest: "{{ ansibletmp }}\\copyinstaller.bat"

  - name: Copy install files to local VM
    script: 'files/schedbat.ps1 -script {{ ansibletmp }}\\copyinstaller.bat'
    register: copy

  - debug: var=copy

  - name: Install puppet from the MSI file
    win_package:
      path: "{{ puppet_installer }}"
      state: present

  - name: Stop the puppet agent service from running to stop all the error messages.
    win_service:
      name: "Puppet Agent"
      state: Stopped
      start_mode: disabled

  - name: Run the installer
    script: "files/scheduleps1.ps1 -script \"{{ install_path }}\\psft-dpk-setup.ps1 -env_type midtier -deploy_only -silent\" -logfile {{ ansibletmp }}\\install.log"
    args:
      creates: "{{ ps_home }}\\appserv\\psadmin.exe"
    register: installfiles

  - debug: var=installfiles

{{< /highlight >}}

The first step creates some directories. The next copies across a batch file, which is used to copy
the installer from the share where it is downloaded. 

copyinstaller.bat
---

{{<highlight batchfile>}}
#jinja2:newline_sequence:'\r\n'
{{ '>' }} "{{ ansibletmp }}\copy.txt" {{ '2>&1' }} (
net use Z: \\internal\general\Peoplesoft_Applications /USER:{{ win_user }} {{ win_pass }}
mkdir "{{ install_base }}"
robocopy "z:\{{ install_dir }}" "{{ install_base }}\{{ install_dir }}" /MIR
)
{{< /highlight >}}

 This is a template because it was easier to pass the directories this way. Different
versions of the installer are in different directories. The bash script is simply a 
few commands wrapped in brackets, and the output is all redirected to a file.

This script is copied with the variables replaced.

Next we run a powershell script.

{{<highlight PowerShell>}}
param(
  [string]$script
)
function Invoke-Script {
  param(
    [string]$script
  )
  $invokeScript = {
    param(
      [string]$script
    )
    $f=[system.io.fileinfo]$script ;
    $baseFile = join-path $f.DirectoryName $f.BaseName
    if (Test-Path("$baseFile.DONE")) {
      Move-Item "$baseFile.DONE" "$baseFile.$((Get-Date -Format O) -replace ':','').BAK" ;
    }
    Set-ExecutionPolicy Bypass -Force | Out-File "$baseFile.PROCESSING" -Append;
    Get-ExecutionPolicy -list | Out-File "$baseFile.PROCESSING" -Append;
    $Out=Start-Process -FilePath "$env:comspec" -ArgumentList "/c $script" -Verb runAs -Wait;
    Write-Output "$?" | Out-File "$baseFile.PROCESSING" -Append ;
    Write-Output $Out | Out-File "$baseFile.PROCESSING" -Append ;
    Move-Item "$baseFile.PROCESSING" "$baseFile.DONE" ;
  } ;
  # remove the job (if exists), create a new one.
  # NOTE: this will clobber any previous job
  $jobName = "Invoke-Script" ;
  $result = Get-ScheduledJob | Where { $_.Name -eq "$jobName" } ;
  if ($result) {
    Unregister-ScheduledJob -Name "$jobName" ;
  }
  $O = New-ScheduledJobOption -RunElevated
  Register-ScheduledJob -Name "$jobName" -RunNow -ScriptBlock $invokeScript  -ArgumentList $script -ScheduledJobOption $O ;
}
# when passed in from ansible, be sure to wrap windows path in ''
$script = $script -replace "'","" ;
$f=[system.io.fileinfo]$script ;
$baseFile = join-path $f.DirectoryName $f.BaseName
Write-Host "Running: $script" ;
Invoke-Script -script $script ;

# monitor for DONE file and report
$doneFile =  "$baseFile.DONE" ;
Start-Sleep -s 5 ;
while(!(Test-Path $doneFile)) {
  Start-Sleep -s 5 ;
}
write-output "Base file $baseFile" ;
Get-Content $doneFile ;
{{< /highlight >}}

This is based on a script by Jason Huggins in an
[ansible bug report](https://github.com/ansible/ansible-modules-extras/issues/287)
Thanks Jason!

This is because a Windows share is mounted as part of a session. The RDP connection is not
allowed a session. But a scheduled task is. The RDP connection is allowed to create
a task. So that is what we do.
The job is scheduled to call the contents of the $invokeScript variable. This sets the
execution policy to allow the file to be run, then runs the batch file. Lastly it
renames a file to .DONE. The rest of the script waits for this rename, so we know
once the script completes, the scheduled task has completed. Once it is complete, it
writes the two files to standard output. Ansible can be told to register the output and
write it out using a debug statement.

The next step installs puppet, which is what Oracle have chosen to manage PeopleSoft.
This creates a job which keeps trying to talk to the puppet master (Puppet has an agent
installed on the node which talks to a controller, the puppet master, to get instructions.
Since we don't have a master, this continually errors, so it is best to switch it off.

Lastly the installer is run from from the folders that were copied across using the same script
as before to create it a session. I am not sure if this is necessary, it was left over
from when I was trying to run it from the share, which proved rather painful. If you
do want to try running from a share, you have to change Internet Explorer security to allow it.

There is more in my playbook, but this is the difficult part. The rest is just doing
the same I would in unix, but replacing modules where necessary with the
[windows equivalents](https://docs.ansible.com/ansible/latest/modules/list_of_windows_modules.html). 


