---
title: "Peoplesoft Log Parsing with Regular Expressions"
date: 2024-09-25T09:00:49+00:00
tags: ['PeopleSoft','GCP',"Cloud","Observability","logs"]
---

As mentioned in my [previous post](../googlelogs/) on this topic, we need to configure BindPlane
to read our files. I chose to use the file source as there was no prepared
parser for PeopleSoft logs.

## Configuration again

### Application server and process Scheduler

On the application server there are three types of files. These are:


#### Application Logs

I set this up as a file, and added the following regex to split it into
fields. This is wrapped for readability - in reality it is all on one line.
The spaces are part of the regex, the newlines  are not.

```regex
^
(?P<ProcessType>\w+)\.
(?P<ProcessID>\d+) (\(
(?P<requestno>\d+)\) )?\[
(?P<timestamp>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}.\d{3})\s*
(?P<userdet>[^]]*)]
 (?P<SRID>[^\s]*)
 (?P<TopID>[^\s]*)
 (?P<UserID>[^\s]*) \(
(?P<LogLevel>[^)])\)
 (?P<message>.*$)
```

Sometimes there is no request number, e.g. for log entries in the application
server log pertaining to the perfmon agent. Also there are multiline
log entries on occasion in these log files, for example where
a SQL statement fails and the failing SQL is listed, or when an email is
sent and the body is listed. There doesn't seem to be
a way to capture these, so they will be lost.

This seems appropriate for the following files on the application server:

* `APPSRV_*.LOG`
* `MONITORSRV_*.LOG`
* `WATCHSRV_*.LOG`

And also for the following files on the process scheduler:

* `APPSRV_*.LOG`
* `DSTAGENT_*.LOG`
* `MONITORSRV_*.LOG`
* `MSTRSCHDLR_*.LOG`
* `WATCHSRV_*.LOG`


#### Tuxedo Logs

I used the following regex. Again it is wrapped for readability - in reality
it is all on one line.

```regex
^
(?P<Time>\d+)\.
(?P<hostname>\w+)\!
(?P<ProcessType>\??\w+)\.
(?P<processID>\d+)\.?
(?P<TransactionID>[-.\d]+)?(: )?
(?P<Message>.*)$
```

This is appropriate for the following on the application server:

* `TUXACCESSLOG.*`
* `TUXLOG.*`

And the following on the process scheduler:

* `TUXLOG.*`


### Web server

#### PIA Weblogic Logs

Once again this is wrapped for readability, in reality the expression is
all on one line. This is a kind of XML like format with the values all
in angle brackets. A question is whether the Message Text can include
a closing angle bracket, if so this would need to be altered.

```regex
^####<
(?P<Timestamp>[^>]+)> <
(?P<Severity>[^>]+)> <
(?P<Subsystem>[^>]+)> <
(?P<Machine>[^>]+)> <
(?P<Server>[^>]*)> <
(?P<ThreadID>[^>]*)> <
(?P<UserID>[^>]*)>?> <
(?P<TransactionID>[^>]*)> <
(?P<DiagnosticID>[^>]*)> <
(?P<RawTime>[^>]*)> <
(?P<SeverityValue>[^>]*)> <
(?P<MessageID>[^>]*)> <
(?P<MessageText>[^>]*)> *$
```

The log entries in these files are multi line, but in this case the log entries all start
with the string `####<`, thus we can specify this as the multiline start pattern.

This is appropriate for the following files in the web server log location:

* `PIA_weblogic.log`
* `peoplesoft.log`


#### PIA Servlet logfile

Wrapped for readability again. This is a tab separated file.

```regex
^
(?P<Timestamp>[-[:digit:]T.:]+)[ \t]+
(?P<SequenceNo>\d+)[ \t]+
(?P<ThreadID>\d+)[ \t]+
(?P<LoggingGroup>[^\t]+)[ \t]+
(?P<TRID>[^ \t]+)[ \t]+
(?P<TopInstanceID>[^ \t]+)[ \t]+
(?P<OperD>[^ \t]+)[ \t]+
(?P<LogLevel>[^ \t]+)[ \t]+
(?P<SourceClass>[^ \t]+)[ \t]+
(?P<SourceMethod>[^ \t]+)[ \t]+
(?P<LogMessage>.+)
```

This is a strange file. The first line is a different format than the rest,
so won't be picked up by the regex. Also there are some multiline entries
Unfortunately there is no new record marker, so I am not sure how to set
this up in bindplane. It seems  that mostly what is missed is java error
stack traces, the error message is still passed.

This is appropriate for the following log files:

* `PIA_servlets*.log.0`


#### PIA Stdout log file

This is similar to the Weblogic logs above, but with fewer fields and
without the start of record marker.

```regex
^<
(?P<Timestamp>[^>]+)> <
(?P<Severity>[^>]+)> <
(?P<Subsystem>[^>]+)> <
(?P<MessageID>[^>]*)> <
(?P<MessageText>[^>]*)> ?$
```

This was specified as having multiline records, with the ending pattern
being `>$`


#### PIA Access log file

Once again wrapped. This is a standard format, so parses nicely.

```regex
^
(?P<IPAddress>[.[:digit:]]+)
 (?P<Client>[^ ]+)
 (?P<Userid>[^ ]+) \[
(?P<timestamp>[^]]*)] "
(?P<MessageText>[^"]*)"
 (?P<StatusCode>\d+)
 (?P<Size>\d+) *$
```

#### Integration Broker Error log

This is a nasty file to parse, as it is in a non-standard HTML format.
The following seems to work:

```regex
^<BODY [^<]+<H4>(?P<Timestamp>[^<]+)<.*?Type[^<]+<[^>]+>
(?P<Type>[^<]+)</.*?ErrorLevel[^<]+<SPAN[^>]+>
(?P<ErrorLevel>[^<]+)<[^<]+<[^<]+Description[^<]+<[^<]+VALUE="
(?P<Description>[^"]+)".*?(.*?>Exception[^<]+<[^>]+VALUE="
(?P<Exception>[^"]+)")?(.*?MessageSet:[^>]+VALUE="
(?P<MessageSet>[^"]+)")?(.*?MessageID:[^>]+VALUE="
(?P<MessageID>[^"]+)")?(.*?MessageParms:[^>]+VALUE="
(?P<MessageParms>[^"]+)")?(.*?<BR>Stack Trace<BR>([^>]+>){2}
(?P<StackTrace>.+?)</TEXTAREA>)?(.*?<BR>Request<BR>([^>]+>){2}
(?P<Request>.*?)</TEXTAREA>)?(.*?<BR>Response<BR>([^>]+>){2}
(?P<Response>.*?)</TEXTAREA>)?
```

The `.*?` is a non greedy match of xero or more characters, i.e. it will
match the smallest amount of text possible. If this isn't included
the optional fields will never match. 

We use multiline parsing, specifying the line start as:
```regex
<BODY BACKGROUND="PSbackground.jpg">
```

### Other Logs

I also created configurations for the Oracle logs and the Elastic Search
logs using the delivered configuration.

## Handy Hint

Testing the above regular expressions is a bit of a pain. 

On Linux it is handy to be able to try them with `grep`. This can actually
be done by enabling the perl compatible regular expressions in grep. So to
find the lines in a file that would not be matched by a regular expression
listed above, one can run the following:

```bash
grep -Pv 'long regex string' filename
```

Then the mismatched lines are returned.

## Still to do

I need to share access with my colleagues. I need
to look at the terraform to see the correct way to do this and create a
merge request.

Also I notice the Oracle logs aren't being uploaded. I need to look
into why that is.

## Things to understand

* What happens if a log message is generated that doesn't match the regex?
It is silently ignored.

* How to drill down into the log entries that we want to see?
The log explorer has a rather confusing interface. It isn't easy to limit
log entries to a particular file. The easiest way seems to be to find an
entry from the file and to select *Show matching entries* from the context
menu. Alternatively it can be added to the query string using something like:
```
labels."log.file.name"="APPSRV_0716.LOG"
```

To use wildcards we can use a regex query:
```
labels."log.file.name"=~"APPSRV*LOG"
```
