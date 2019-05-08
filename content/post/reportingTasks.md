---
title: "ReportingTasks"
date: 2019-04-13T11:25:23+01:00
draft: true
tags: ["automation","Project management","Critical Patch","Language","TaskJuggler"]
---

# Are we there yet?

Over the last two posts I have been creating a project in [TaskJuggler](http://taskjuggler.org) to help
me schedule the quarterly security patches to my systems. Today I am going to talk about reporting.

What we have so far is a valid project, and [taskjuggler](http://taskjuggler.org) can schedule it.
But it won't tell you anything
about what it has done, which is quite frustrating. We need to set up a report, or several.

The report is defined in two parts, the report definition and it's output format. 

First, lets look at a couple of reports we could do. I am most interested in a calendar of who needs to
do what and when. I need to raise change requests for production. The two most appropriate for me seem
to be the taskreport, which includes the 
