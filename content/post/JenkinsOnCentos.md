---
title: "Jenkins On Centos"
date: 2017-07-14T16:40:55+01:00
image: images/Jenkins.svg
tags: ["jenkins"]
---

# Install Jenkins. Automatically?
The documentation for Jenkins is pretty sparse, but fortunately a colleague had set up Jenkins already, and was able
to give me some pointers.

The idea is to use the Apache web server as a reverse proxy to access Jenkins. This can also take care of user
authentication. 

## Prerequisites
So first of all [download Java from Oracle](http://www.oracle.com/technetwork/java/javase/downloads/jre8-downloads-2133155.html). I download it manually and keep it in case the link changes. I use the RPM and install it with yum. Note
that there isn't an easy way to automate this. There is no repository for Java which means you have to check every
quarter, when Oracle release their [Critical Patches](https://www.oracle.com/technetwork/topics/security/alerts-086861.html) then download and update it.

## Jenkins
Next, Jenkins can be downloaded and installed. There is a [repository for Jenkins](https://pkg.jenkins.io/redhat) which can
be added to yum as per the details on that page. While installing packages it is also worth installing git and curl.

## Fire it up!
When Jenkins is started it creates an admin user. It creates this entry in the log file at /var/log/jenkins/jenkins.log.

    *************************************************************
    *************************************************************
    *************************************************************

    Jenkins initial setup is required. An admin user has been created and a password generated.
    Please use the following password to proceed to installation:

    f3d063fb49924b558efa808bd31d1aa1

    This may also be found at: /var/lib/jenkins/secrets/initialAdminPassword

    *************************************************************
    *************************************************************
    *************************************************************

The user interface tells you to enter the password, which proves you have access to the server and 
provides some security before you start setting up Jenkins. I didn't create any users here, I just
continued with the admin user as I will use the reverse proxy to take care of that.

I installed the recomended plugins, but for the next part I also found I needed to install the
[Reverse Proxy Auth Plugin](https://wiki.jenkins.io/display/JENKINS/Reverse+Proxy+Auth+Plugin).
