---
title: "Vagrant Multi VM provisioning"
date: 2018-08-10T15:02:45+01:00
tags: ["vm","automation","vagrant","scripting" ]
---

## Multiple VMs required ##

I am looking into Postgresql, replication and fail over. One way to set this up is to have a primary, a standby
and a witness server. Two run databases, and one - the witness to decide whether a fail over is needed. To test this
it would be nice to create three VMs on my desktop to help test ansible scripts. I have downloaded Virtualbox from Oracle, and my
desktop has loads of memory. It would be nice to be able to script the creation of the VMs and then be able to 
run the same ansible playbook that has been developed for the "real" VMs. Then I can learn postgres and help
develop the playbook and associated scripts.

An issue is that my desktop runs Windows, and while it is possible to persuade Ansible to run on
Windows, it isn't supported, and it is difficult to make it work. A solution to this is to
use the witness VM as the ansible controller.

## The Solution ##

The solution to getting three VMs fired up in a repeatable manner, and all talking to each other is
[Vagrant](https://www.vagrantup.com/).

Downloading and installing it was pretty straightforward. Configuring it was less so. Here is what I did:

The VMs created are controlled by the Vagrantfile. Here is the vagrant file I created:

### Vagrantfile ###

{{<highlight ruby>}}
Vagrant.configure("2") do |config|
  N = 2

  VAGRANT_VM_PROVIDER = "virtualbox"
  ANSIBLE_RAW_SSH_ARGS = []

  (1..N).each do |machine_id|
    config.ssh.insert_key = false
    config.vm.define "pgdb#{machine_id}" do |machine|
      machine.vm.box = "bento/ubuntu-16.04"
      machine.vm.hostname = "pgdb#{machine_id}"
      machine.vm.network "private_network", ip: "192.168.77.#{10+machine_id}"
    end
  end

  config.vm.define "controller" do |controller|
    config.ssh.insert_key = false
    controller.vm.box = "bento/ubuntu-16.04"
    controller.vm.network :private_network, ip: "192.168.77.10"
    controller.vm.hostname = "controller"
    # install ansible
    controller.vm.provision "shell", privileged: false, path: "install_ansible.sh"
    # run ansible
    controller.vm.provision "shell", privileged: false, inline: <<-EOF
      if [ ! -f /home/vagrant/.ssh/id_rsa ]; then
        wget --no-check-certificate https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant -O /home/vagrant/.ssh/id_rsa
        wget --no-check-certificate https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant.pub -O /home/vagrant/.ssh/id_rsa.pub
        chmod 600 /home/vagrant/.ssh/id_*
      fi
      rm -rf /tmp/provisioning
      cp -r /vagrant/provisioning /tmp/provisioning
      cd /tmp/provisioning
      export ANSIBLE_HOST_KEY_CHECKING=False
      ansible-playbook playbook.yml --inventory-file=inventory
    EOF
  end
end
{{</highlight>}}

This was adapted from the the example in the [github gist by tknerr](https://gist.github.com/tknerr/291b765df23845e56a29#from-a-separate-control-vm)

### What It Does ###

What this does is to set up two VMs pgdb1 and pgdb2 for the primary and standby database in a loop. These are not provisioned.
The insecure default keys are left alone
(config.ssh.insert_key = false), then the controller is set up and provisioned by calling some shell commmands. 
The two wget commands install the insecure private and public keys from the vagrant repository so that the two database servers
trust the controller. They are put on a private network on my desktop, so they can talk to each other. They are also given NAT access to
the internet so that the download works.

Then ansible is installed using a script, "install_ansible.sh". Lastly the playbook is is run. Ansible host checking is switched off because otherwise ssh would ask if
you really want to trust the remote host and hang waiting for an answer. 

The playbook runs the postgres install against all three VMs.

## Ansible setup ##

I haven't really looked at the ansible scripts yet, but the ansible inventory looks like this:

### Inventory ###

{{<highlight ini>}}
pgdb1      ansible_ssh_host=192.168.77.11
pgdb2      ansible_ssh_host=192.168.77.12
controller ansible_ssh_host=192.168.77.10

[postgres_db_servers]
pgdb[1:2]

[postgres_quorum_servers]
controller
{{</highlight>}}

Note that the IP addresses assigned above are used to identify the VMs.
