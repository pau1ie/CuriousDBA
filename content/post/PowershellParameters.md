---
title: "Powershell Parameters"
date: 2018-04-03T16:17:46+01:00
image: images/shell.jpg
tags: ["Scripting","Automation","Windows","Powershell"]
---

I got an error from powershell:

{{<highlight console>}}
PS C:\> Start-Process -NoNewWindow -Verb RunAs
Start-Process : Parameter set cannot be resolved using the specified named
parameters.
At line:1 char:1
+ start-process -NoNewWindow -verb runas
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidArgument: (:) [Start-Process], ParameterB
   indingException
    + FullyQualifiedErrorId : AmbiguousParameterSet,Microsoft.PowerShell.Comma
   nds.StartProcessCommand
{{</highlight>}}

This is because the
[Start-Process](https://stackoverflow.com/questions/31750263/ambiguousparameterset-in-a-function-with-multiple-parametersetname)
command has two parameter sets. Looking at the page linked, the parameters are listed in two groups. These groups
can't be mixed and matched. You have to pick one or the other. So if you want to do runas, you can't
use NoNewWindow.

This was a bit of a revelation to me, but I suppose it is one of those things that is so obvious nobody feels the need to
point it out!

