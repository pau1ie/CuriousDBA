---
title: "Jenkinsconfig"
date: 2017-09-12T16:52:04+01:00
draft: true
---

Configuring Jenkins

As I mentioned in a previous post, we need to restrict Jenkins from listening on anything other than the loopback interface
to ensure that ll user authentication etc is done through the reverse proxy which we set up, apache and ucamwebauth, or Raven
as we call it.

First of all, create the reverse proxy. This can be tested to make sure it connects properly. This was done in the previous
article, [Apache UCam Webauth](apacheucamwebauth).


Next, Jenkins needs to be configured to accept logins using a reverse proxy. This means logging on to the 
user interface directly (You can do it through the reverse proxy as well, you need to log in to that then log in to Jenkins)

A plugin needs to be installed to allow jenkins to defer to the remote proxy. Install
[Reverse Proxy Auth Plugin](https://wiki.jenkins.io/display/JENKINS/Reverse+Proxy+Auth+Plugin).
From the main screen, click Manage Jenkins in the menu, then Manage Plugins from the list in the main screen, 
Click the Available tab in the middle of the screen, then use the filter to search. Tick the install tick box, then
click on Install without restart.

Now we can set up the security. Back on the main menu (Click on the Jenkins icon or text at the top of the screen) click
on Manage Jenkins again, but this tie we need Configure Global Security. Under Access Control we can click on 
HTTP Header by reverse proxy. The defaults should be:

Header User Name: X-Forwarded-User
Header Groups Name: X-Forwarded-Groups

These are correct. Click Save. If youare not logged in through the proxy, you will now get an error in Jenkins as it forgets who you are!
Log in through the proxy to fix this.

Now that the proxy login is tested and working, we can reconfigure Jenkins to only listen on the loopback interface, so it can
only be accessed through the reverse proxy. This is pretty easy to do, just alter the file /etc/sysconfig/jenkins to 
set JENKINS_LISTEN_ADDRESS to 127.0.0.1. Here is the play to do that:


- name: Configure Jenkins Listen Address
  become: yes
  when: jenkins_listen_address is defined
  lineinfile: dest=/etc/sysconfig/jenkins regexp=^JENKINS_LISTEN_ADDRESS= line=JENKINS_LISTEN_ADDRESS={{jenkins_listen_address}}

Now when I go to port 8080, I can't connect. I have to go through the proxy to access Jenkins. So you can't connect to Jenkins
unless you are logged in. Hopefully this makes it safer from people trying to compromise it. It can only be accessed from
inside our network anyway.

So, now we can log in to jenkins, but it still doesn't do anything. To be able to do much useful, jenkins needs to be able to access
remote servers. The easiest way to do this is with ssh keys. To start with, I will create a key for the jenkins user on 
the underlying operating system. Log in to the server, and switch to the jenkins user. Since this user has no shell, it has to
be specified:

su - jenkins -s /bin/bash

Now create a public/private key pair:

ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa

This creates a key file without a password. This can now be used in Jenkins when setting up credentials. 
I then added this public key to git to allow Jenkins to retrieve my ansible files, I also installed the
ansible plugin to allow me to run ansible. I added ssh access to all my servers (I probably shouldn't
be logging in as root, but that is something to fix another day):





