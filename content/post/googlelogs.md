---
title: "Sending Logs to Google Observability Logging"
date: 2024-08-02T15:00:49+00:00
tags: ['PeopleSoft','GCP',"Cloud","Observability","logs"]
---

At present all our logs are in random files in random places in the
operating system. I would like to see whether we can improve this. As an
example of the problems caused by the current situation, if a user reports
an issue, they normally don't give a timestamp, so we have to assume the
issue occurs say up to 20 minutes before the call was raised. Then we have
to search the logs on the operating system for their user ID. As mentioned
these logs are in various places. It isn't easy to limit to a time span
using standard operating system tools. Also we have a redundant
architecture, meaning the users session could have been on any one of four
web servers, and four application servers. The error could have happened
on any of these 8 VMs.

It would be so nice if these logs could be aggregated into one place so I
could just search once for this information. Of course this problem has
been solved multiple times. One of the suppliers we use is Google Cloud
Platform, so it seems like a good place to try first.


## Installing the Software

First of all I want to test this in a non-production environment. To get
an understanding of what is involved, I like to set things up manually
first while creating documentation, then automate it. Here are my notes
on the manual setup process.


## Initial access

Colleagues in another team manage the Google cloud account, so they set
me up with an admin account. This is different to my normal Google
account. They 
[recommend](https://guidebook.devops.uis.cam.ac.uk/explanations/gcloudadmin-accounts.html)
only to use it when necessary, and not  to leave long lived browser
sessions logged into the account. This is presumably to prevent issues
where an admin is tricked into clicking a link with performs admin
actions - if we are not logged in, the link won't work.

OK, here goes! I log into the
[Google cloud platform console](https://console.cloud.google.com/). 
My colleagues had created a project for me, which I could see once I
selected my organization. I got an email telling me I had logged in. My
colleagues instructed me to create a service account with permissions for
logging.


## Installing BindPlane

Google states that we 
[need to download BindPlane](https://cloud.google.com/logging/docs/bindplane/on-premise-hybrid-logging)
So I follow their
[quick start guide](https://observiq.com/docs/getting-started/quickstart-guide/install-bindplane-op-server)
to get familiar with the product.

We need a license. Since we are using the Google Compute Platform, 
I requested a Google license, and that was emailed to me.

The webpage instructs us to run a one line script. This needs to be run
as root, or else it prompts for the sudo password. It then checks the
operating system, downloads the appropriate package and installs it.

Then I typed Y to continue with the setup process, as per the install
guide above. I selected a Google license, then pasted in the license
code I received in the email. It didn't work the first time, probably
because outlook helpfully added a space at the end. On the second attempt,
after deleting the space, the license key worked.

I set the server host to *0.0.0.0* to listen on all IP addresses (this VM
only has one public IP address anyway), accepted  the default port of
3001, and the default remote URL which is the full name of the server.

I chose *Single User* for the authentication method. This will definitely
have to change if we use this system in production, as we need to be
able to have users for each of the admins. The only other option is
*LDAP and Active Directory (Google Edition or Enterprise)*. I have no
idea how much work that would be to set up or maintain. If it is a
lot of work, that could be a problem for us.

For the backing event bus I chose local, once again this is because I am
just testing at the moment. Then I said Yes to automatically restart
BindPlane, and the installer ended.

There doesn't seem to be an opportunity to install a web certificate. I
will need to work out how to do that if we start using this in production,
as things stand the login password will be transferred over the network
in clear text, which is poor security practice.


## Configuring the first agent

When you click the button to install the first agent, you are presented
with a command to copy and run as root. Once this is complete, we see the
agent appear in the list of agents. That is pretty painless!

We did have an issue with connecting to the BindPlane server on port 3001
from one of the VLANs, we needed to open a port.


## Add a configuration

Next to make it actually do some work, we need to 
[add a configuration](https://observiq.com/docs/getting-started/quickstart-guide/build-your-first-configuration).
To do this, we go to *Configurations*, click *Create*, then enter a name,
platform and description. Since we are using OpenSearch as our search
framework, which was forked from Elastic search, we can pick this
predefined configuration. So I click Elastic Search. I added in all the
log locations. The metrics all seemed to be named elastic search, so they
might need to be changed for open search. I never got the metrics working,
so I might investigate that later, but the main point of this exercise is
to aggregate the logs, so I won't worry too much about that yet.

Next I need to add a destination. There are four options given:
* Chronicle Forwarder
* Google Chronicle
* Google Cloud
* Google Managed Prometheus

We are looking at log aggregation, so the last is not appropriate, as
it is used for Metrics. The first two seem aimed at Security information
and Event Management (SIEM), so again doesn't seem appropriate, so I
picked Google Cloud. The project ID was copied from the project that was
set up for me. The authentication method can't be auto as this is an
on-premises VM. I need to create a service account. 


## Creating a Service Account in Google Cloud

According to the
[documentation](https://cloud.google.com/iam/docs/service-accounts-create)
we can create a service account
[from the web](https://console.cloud.google.com/iam-admin/serviceaccounts?walkthrough_id=iam--create-service-account).

This is just a test, so we will do this manually using click ops. Ideally
we would want to be able to create this using terraform in the future.
I could not add the permissions, so got a colleague to do that for me
but I did create a JSON key file. 


## More configuration

I installed agents on the remainder of the tiers. The rest of the
software we use doesn't have prebuilt configurations
like Elastic Search did, so we need to create our own.
Actually the Oracle Database does, but it didn't seem to work - no
logs were uploaded, so I might need to do some investigation into that.
I plan to write another post about how I configured
the various PeopleSoft components.
