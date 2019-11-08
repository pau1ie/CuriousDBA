---
title: "Journey to the Cloud - HEUG EMEA 2019"
date: 2019-11-08T09:53:50Z
tags: ['HEUG2019', 'Project Management','Cloud']
---

## Is the cloud my friend?
I am concerned about the cloud. As a DBA I believe the idea is to replace 
most of what I do with a cloud provider. So if I want to keep working, should I 
work for a cloud provider, or change what I do? In the mean time I am also trying to ensure I can
deliver a better service than the cloud providers using similar technologies in a way that is more
responsive to the needs of my customers. So it is interesting to learn the experiences of 
[Jo Ellen Dinucci](https://twitter.com/boisestateavpfa?lang=en),
Associate VP, Finance and Administration - [Boise State University](https://www.boisestate.edu/) during their journey to the cloud.

There was a lot in this talk, so I have no doubt missed some points, and misunderstood others!
It made me think and I really enjoyed it.

## The Presentation
Boise State University decided to have a cloud first strategy when it came to IT. The University 
has an innovative culture, which helps justify doing risky things like being an early adopter of 
Oracle's cloud. It seems Oracle gave them a good deal, apparently it was a similar price to their existing license fees.
They are getting towards HCM implementation. They have an ERP HCM Cloud advisory board which she is on.
They have a monthly phone call for people working on cloud sharing tips and tricks.

The important benefit of cloud isn't cost but a continuous improvement mindset.
Previously their self hosted software solutions had a heavy cost up front cost. It was allowed to 
get out of date meaning this cost was replicated every few years, because they had to have a large
project to bring the system up to date. With cloud they consider the
heavy up front investment only needs to happen once. Then there is a steady low cost for ever.

Boise seems to have thought carefully about the impact on their technical staff, and planned several
years in advance of the project starting. Technical teams were run down and backfilled with
contractors/outsourcing. Effort was shifted from tech to functional improvement. This made the organisation more efficient.

They had limited bandwidth for this work, so had to work in one area at a time. Once each area was
stable they are able to move on to the next thing.

They are embedding change management and engaging with stakeholders.
There needs to be a process to manage this into the future. Making a change of this size could be done
by external or internal people. Outsourcing can be useful to get people to listen - we are paying these
people lots of money, so they must know best. Boise did change management in house.

## Critical Patches
Quarterly fixes are sent to the test system on the first Friday of the month. They go to production two 
weeks later. There is no way to delay this, so the business must be prepared. They test all the critical
business processes on first day. This leaves the rest of the two weeks to deal with the changes. For example, troubleshoot, develop
workarounds, update training materials, or change business process, e.g. to add or remove workarounds. 
Tests are automated with [selenium](https://www.seleniumhq.org/).

Institutions have limited capacity for change so need to cherry pick new functions to adopt. 
To adopt new functionality they have a project to switch it on.

Biose recommended that an experienced implementer be chosen - don't pay for someone else's research and development!
Training should be included in the up front cost and included in the budget - not just  the license fee. 
Just in time training seems to work best. If it is too early people forget. Patches should be applied during the upgrade, 
otherwise the project will fall behind the cloud solution and it will have to be restarted. 

Ensure role definitions are defined so you know exactly what the contractor is doing. Request variable and fixed fee quotes. 
The project is likely to take over 12 months. If the quote comes in shorter than that, something is likely to be missing in the scope.


## Tips and tricks
They created an office of continuous improvement. It handles change management and other business processes. 
It focusses on policy, process and people issues rather than technology. They have found this delivers great value across the campus.
With the self hosted solution they considered they were the technical experts. Now they leave the technical stuff to Oracle and 
assume they are the experts in that area.

Previously they did a lot of "fit gap" rather than the whole business process. Doing the latter could mean massive savings. 
Need not do what everyone says, but you need to hear them. Some processes should be consistent - this delivers savings, but not everything has to be.

She went on to describe the group formed to discover the business processes that needed to be supported - to see what people wanted from the solution.
93 role descriptions for the start up committee saying everything they do. Are they advisors or decision makers? This was all documented. 
33 campus liaisons were approved by leadership. Focus groups were created, and their expectations were set - they don't decide, they 
recommend and the VP make final decision. Intentional membership to capture perspectives across campus. Research, back office, student, etc. 
Additional people who were invited along were kicked out! 16 hours of meetings to agree workflow and approve it.

Think about what the model is going to be give people what they need, they started 
thinking about these things three years before signing up for cloud. They help the university 
adapt to the solution. They have an office of continuous improvement which takes care of much of the management.

For institutions thinking about starting a migration to the cloud she suggested some things to start doing now.
Allow and encourage staff to shift. Outsource gradually to allow staff to adapt.

## My Thoughts
I feel that this was an extremely well run project. The migration gave useful cover to improve 
the way of working. The change from the
focus on technology to that of business process, and continual improvement seems like the correct strategy. 
I think all the benefits could have been achieved in a self hosted environment.
I am not sure I agree with her assertion that the cloud has low
fees for ever. This assumes that the cloud provider will never change, or the existing provider will
always do major upgrades as part of their service. Also the business will never decide the application needs
to be re-implemented (Or data significantly changed) to support changing times or processes.
I expect Oracle will want to make profits
on its cloud service, as large or larger than it makes on its current offerings. The value for money Boise experiences
won't transfer to other customers. The technical effort has to be paid for somehow. 

At Cambridge we get patches in to production fairly quickly. Like Boise we use selenium for automated testing, so
again cloud isn't required for this. Hosting ourselves means we have the opportunity to be responsive
to the business needs for change freezes or zero downtime. Boise's experience may show the business
can survive better than they expected with short periods of downtime and change during critical periods.
So long as there is a process to deal with it!

The main thing I take away from this is that a well run project is more likely to succeed!