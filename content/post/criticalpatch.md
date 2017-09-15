---
title: "Criticalpatch"
date: 2017-07-21T11:02:09+01:00
tags: [ "Ansible", "Oracle", "Critical Patch" ]
image: images/patch.jpg
---

# Oracle Critical Patches

It was critical patch Tuesday this week. Because of the time differences
between here and the USA we actually only get to find out about the critical
patches on Wednesday, though there is a pre release announcement.

With software as
complex as Oracles you can pretty much guarantee that
there will be serious bugs somewhere in there which need patching every
quarter. 

Considering the types of organisations that run Oracle - the ones who can afford
to pay the license fees, i.e. rich and powerful ones, you can be sure
that someone is looking into what changed for each release, and how they
can benefit from stealing or changing information.

## Time is of the essence.
It's a race then. For us to be in with a chance of getting the patches
applied before the bad guys work out how to exploit them, we need to act fast.

But there is another side to this. What if the fix has a bug in it? Maybe
it fixes the security problem, but stops our system working properly? If
we apply that patch, we are carrying out a denial of service on ourselves.

So this is really hard, we need to apply patches really quickly, but we need
to test things carefully. To make things even worse, our test team is really
busy with project work, testing new functionality, and making sure the next
part of the business cycle will work with our software. How do we justify
this to managers who want shiny new stuff? 

## Automation to the rescue
### Installing

I am pretty pleased with the way we get these applied. First I download
the patch from Oracle and store it on my local server. I am doing Weblogic
at the moment because it has the highest CVSS score, and it is closest to
the internet. So I need to download 	
	
Patch 25869659: WLS PATCH SET UPDATE 12.1.3.0.170718
		
I check the Readme to make sure it is the same process to apply it as last
time. It is.

So now I just download the zip file to the share I have for Weblogic patches,
and update the Ansible job to install the patch. 

`  oracle_cpu: { patch: 25388793, patchfilename: p25388793_121300_Generic.zip, action: apply } `

changes to

`  oracle_cpu: { patch: 25869659, patchfilename: p25869659_121300_Generic.zip, action: apply } `

I do this on my Linux desktop where I do the development. I check what changes
I have outstanding, and if I am happy I commit the change and push to my master
repository.

~~~~
$ vi wls-patch/defaults/main.yml 
$ git status
$ git commit wls-patch/defaults/main.yml -m 'Updated weblogic CPU'
$ git push
~~~~

Now all I need to do on the management server is:

~~~~
$ git pull
$ ansible-playbook web-cpu.yml
~~~~

Then ansible will shut down the web server, apply the critical patch, and
restart it again.  Amazing. Even better, in production, we can limit this to
do half the web the web servers on one day, and the other half the next.
The load balancer can be configured to make sure that nobody is using the
web servers we are patching. Nobody gets kicked out or annoyed by the
patch being applied to the web server.


### Testing
So that is easy for me, what about the test team?

Their tests are automated too! All I have to do is to ask them to run tests on the test
server the day before and after I apply the patch. So long as the results don't change
between the two runs I can be confident I can apply the patch to production without any
problems.


## Can we do better?

It would be even better if I didn't have to ask the test team to run their tests. Or type in
the two lines to pull the code from git and run the install. Or configure the load balancer.

Maybe Jenkins is the answer to this. It is why I am installing it.
