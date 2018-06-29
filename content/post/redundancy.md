---
title: "Redundancy"
date: 2018-03-28T17:25:35+01:00
tags: ["Disaster Recovery","DR"]
image: images/boats.jpg
---

I have been thinking about our DR process, and how to improve it. More on that
in a later post. However we recently had a couple of planned server room
outages, where my previous planning has been beneficial.

Production is redundant and resilient across our server rooms. However,
the other environments are not because redundancy is expensive and only
really needed for production systems.

So what we do is to have half of the non-production systems at one site.
and the other half at the other. This means that we can shut a machine
room down, and half the development environments will continue to run.

This was very useful the other week when maintenance work on the power
supply for one of the machine rooms meant it had to be powered down.

Production can continue as it is resilient, and development can continue
in a restricted fashion
