---
title: "Tech17 - Auditing the Oracle Database"
date: 2017-12-07T14:45:16Z
tags: ["Tech17","UKOUG","Oracle","Security","Auditing"]
image: images/Engine.jpg
---

The presenter was [Pete Finnigan](http://www.petefinnigan.com/).

Pete talked through a solution (PFCATK) he had developed for auditing the oracle database. This appears to be 
currently in development and not available for general use. He seemed inclined to release
the core tool and sell a pretty front end or consultancy, but it seemed he hadn't yet
decided. The lack of anything to play with (Unless you ask for a copy of the
tool and agree to install it and give feedback) made this session less
interesting than it might otherwise be.

Anyway, the problem with the Oracle database is that by default
most auditing is switched off. If it is switched on with the defaults it
is OK for Oracle audit vault, but not really useful to detect
intrusions. To keep the auditing as performant as possible, the
events to be audited are things that shouldn't normally happen.
For example, SQL that doesn't parse could be generated by a
SQL injection attempt that is trying to figure out the syntactically
correct method to close the statement. Since these SQLs tend
to contain a comment to make the parser ignore the rest of the line,
 this could be searched for in the failing SQL that is logged.

Another attack is password guessing. Invalid login attempts
can be audited, and in many cases this won't cause a problem
because generally only one or two invalid login attempts are
generated by people forgetting passwords, and if an application
is incorrectly configured, it won't work anyway. An alert can
be generated if there are many login attempts in a short
amount of time as this can be caused by a script attempting
to brute force a password.

Pete's tool also seems to be able to get some information
about where the DBAs typically log in from, which fields (e.g.
credit card information) are typically accessed by which modules,
and alert when this is different than normal. If the search form
is attempting to access credit card information, this is likely
to be caused by SQL injection. Similarly it is unlikely that the
search form would call the crypto function to decrypt the credit
card information.

He mentioned GDPR, but auditing isn't required by GDPR, though
I suppose most would prefer to be notified of an intrusion
with their own monitoring, rather than seeing a message on
twitter that information has been dumped on pastebin.

The tool is still in early stages of development. Pete mentioned
reporting as one area where it needed work.
