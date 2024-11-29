---
title: "Refreshing a Test PeopleSoft Campus Environment"
date: 2024-11-28T09:57:49+00:00
tags: ['PeopleSoft','Automation','SQL','Refresh','Clone','Search Framework','Elastic Search']
---

We have a number of PeopleSoft test environments. I have written about my
automated build process before, but I have not yet mentioned what we do
to the database when we refresh. 

My approach here is that I want as much as possible to build the
environment from scratch. This means that we have a consistent build.
There are also database fields that need to be changed. Recently
a colleague and I reviewed the tables that needed changing and came up
with the following.

## Oracle Supplied Scripts

Oracle supply some data mover scripts in `$PS_HOME/scripts` which seem
like a good start. We run these as part of the refresh process.


| Script | Function |
| ------ | -------- |
| `appmsgpurgeall.dms` | Purge the Integration Broker tables (It seems this was called Application Messaging in the past) |
| `perfmonpurgeall.dms` | Purge the Performance Monitor tables (See Doc ID [2816396.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2816396.1) and [660862.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=660862.1)) |
| `prcsclr.dms` | Process Scheduler tables. See Oracle Doc ID [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1) |
| `rptclr.dms` | Clear Reports. |
| `psrfclr.dms` | Clear Report Framework. See manual *PeopleTools 8.60: Process Scheduler* section *Process Scheduler Table Maintenance*. |
| `grant.sql` | Grants that need to be run after maintenance that may change tables by rename. We don't need to worry about this as part of the refresh, because the grants come across with the database as noted below. |

These do alter a lot of tables, but they don't do everything. We find
we still have to alter a lot more tables listed below.


## Tables we Alter

These are additional tables we find we have to alter. There are also some
noted in the Oracle Documentation that we don't find we have to alter,
which I also note below.

| Table | Action | Why  | Table purpose |
| ----- | ------ | ---- | -------------- |
| `PS.PSDBOWNER` | Update | Change the `OWNERID` (sysadm) and `DBNAME` Database name. See [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1) | Owner id and database name. |
| `PSACCESSPRFL` | N/A | `PSACCESSPROFILE` replaces `PSACCESSPRFL` beginning with PeopleTools 8.55 - see Doc ID [2107805.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2107805.1). This table now contains no rows. Needs to be granted to the connect id as per `grant.sql` | Acess Profile |
| `PSACCESSPROFILE` | Update | If there is to be a different access ID and/or password `PSACCESSPROFILE` must be updated. Also this need to be granted to the connect id as per `grant.sql`. This grant is expected to come across with the database, so isn’t done as part of the refresh. The Access ID (sysadm) password is updated using `CHANGE_ACCESS_PASSWORD` in datamover. | Access Profile |
| `PS_AERUNCONTROL` | Truncate | Clear down some run controls so jobs aren't run with production parameters in test. We have another process that schedules batch processes in test environments. | Application Engine Run Controls. |
| `PSAPMSGDOMSTAT` | Update | Contains production hostnames in `MACHINENAME`. Doc ID [1421339.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1421339.1). _>PeopleTools>Integration Broker>Service Operations Monitor>Monitoring>View Synchronous Details_. This is cleared by `appmsgpurgeall.dms`, then rows inserted that pertain to this environment. | Application Messaging (Integration Broker) Domain Status. |
| `PSAPMSGQUEUESET` | Truncate | Contains hostname in `MACHINENAME`. We truncate this table. | Application Messaging (integration Broker) Message Queue Set. |
| `PS_CDM_DIST_NODE` | Update | Update the distribution node i.e. the location to which the process scheduler posts reports. | Content Distribution (Report) Node definition. |
| `PSDOCSCMADFN` | Update | Contains URLs in `IB_TGTLOCATION`. These are updated to match `PSIBSVCSETUP` | Document Schema Managed Object |
| `PSGATEWAY` | Update | via Doc ID [1421339.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=1421339.1) Contains URL in `CONNURL`. | Integration Broker Gateway |
| `PSIBFAILOVER` | Truncate | Contains hostname in `MACHINENAME` field. _>Peopletools>Integration Broker>Service Operation Monitor>Administration>Domain Status_ | Integration broker failover configuration. |
| `PSIBHUBDATA` | Update | Fix the Integration hub page URLs. Contains URLs in: `IB_TGTLOCATION`, `IB_SECTGTLOCATION`, `IB_ENDPOINT`.  | Integration Broker Hub (Application Messaging) Node Data. |
| `PSIBLBURLS` | Update | Contains the integration broker URL in `CONNURL`. There is a unique constraint on this field, so we need to ensure only one row exists with the correct URL. | Integration Broker Load Balancer URLs |
| `PSIBOAUTH `| N/A | This has hostnames in the `MACHINENAME` column, but we don't think it's necessary to update them. It is used to clean up "Access Tokens" that have potentially expired if the functionality is enabled (PIA navigation is _People Tools > Security > oAuth2 Administration > Automated Expired Token_).  [2728992.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2728992.1). | OAUTH Tokens |
| `PSIBSVCSETUP` | Update | Contains Integration broker URLs in the following columns: `IB_SCHEMANAMESPACE`, `IB_TGTLOCATION`, `IB_SECTGTLOCATION`, `IB_RESTTGTLOC`, `IB_RESTSECTGTLOC` | Integration Broker Service Setup data. |
| `PSIBWSDLDFN` | N/A | Contains production URLs in `IB_WSDLURL`, but it appears unused, so we leave this alone. | Integration Broker Web Services Description Language Managed Object Definition |
| `PSMSGNODEDEFN` | N/A | As per [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1), but the URL fields aren't populated, so we don't alter this. | Holds definitions for Message Node objects. Message Nodes represent databases in Application Messaging (Integration Broker). |
| `PSNODEURITEXT` | Update | See [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1) Portal URI needs to be updated (Column `URL_TEXT`). | Integration Broker Node URLs. |
| `PSOPERATION` | Update | `IB_RESTBASE_URL` is reset for the some interfaces. In general we have different prod and non-prod interfaces set up, so most don't need to change. | Integration Broker Operations. |
| `PSOPRDEFN` | Update | Needs to be granted to connectid as per `grant.sql`. Passwords need to be changed in here. Locked out status of accounts is also changed. Email addresses are changed to prevent accidental release of test emails. | Defines Peoplesoft Users. |
| `PSOPTIONS` | Update | [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1): For PeopleTools 8.44 and higher, blank out the `GUID` in the `PSOPTIONS` table, This will get regenerated when the new Application Server Domain is started. `GUID` is used for performance monitor, so can be preserved to prevent lots of environments appearing in performance monitor with the same name. Enable change control locking and history in dev: `CHGCTL_USE_LOCKING='Y'`, `CHGCTL_USE_HISTORY='Y'` | This is a single row table containing PeopleTools system options chosen. |
| `PSOPTIONSADDL` | Insert | Change background colour for a refreshed environment | Presumably for options that didn’t fit in the above table! |
| `PSMCFRENURLID` | Truncate | Contains server names in a number of URL columns. | Real-time Event Notification Configuration. |
| `PSPRCSPRFL` | Updated | Allow users to update their own jobs. `update sysadm.PSPRCSPRFL set RQSTSTATUSUPD = 2 where classid in (select Distinct PRCSPRFLCLS from sysadm.PSOPRDEFN)` | Process profile |
| `PS_PRCSPURGELIST` | Update | Disable the Purge Process | Process scheduler purge list |
| `PS_PRCSSEQUENCE` | Update | Set process sequence back to 1: `update sysadm.PS_PRCSSEQUENCE set SEQUENCENO = 0, LSEQNO = 1` | Process scheduler sequence |
| `PSPRDMCNTPRV` | N/A | Mentioned in [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1), but doesn't seem to contain hostnames. | Portal Content Provider |
| `PSPRDMDEFN` | N/A | Mentioned in [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1), but doesn't seem to contain hostnames. | Portal Definition |
| `PSPRSMDEFN` | Update | Sanitise all links. Point some to non-prod environments. `PORTAL_URITEXT` contains links - look for ones that start with `http`. Column `PORTAL_URL_CHECKSUM` also needs to match the URL. I make the change in the application then copy the checksum manually. | Portal Structure Definition |
| `PSPTPNEVTCLT` | Truncate | Doc ID [2264459.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2264459.1), Doc ID [2633972.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2633972.1). Contains hostnames in the `HOST` column. Errors appear in the PIA servlet logs if this is not corrected. Also needs dummy rows. See document id [2992143.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2992143.1). This is supposed to fix high webserver CPU. | Push Notification Configuration |
| `PS_PTSF_ES_STATDTL` | Update | Contains hostname of the search instance in `PTSF_HOST_NAME` and `PTSF_KIB_HOST_NAME`. | Search framework |
| `PS_PTSF_OPTN_SAVED` | Update | Number of replicas updated to reflect the nodes in the current environment (Minus one) `update sysadm.PS_PTSF_OPTN_SAVED set PTSF_SEARCH_PVALUE = 0 where ptsf_srch_provider = 'ES' and PTSF_SEARCH_PNAME = 'PTSF_NOOF_REPLICAS'` | Search Framework |
| `PS_PTRTIHCOUNT` | Truncate | Contains process monitor server names in `HOSTNAME`. | Search Framework Real Time Indexing |
| `PS_PTSF_OPTN_LOAD` | Update | Default number of replicas updated to reflect the number of nodes in the current environment. `update sysadm.PS_PTSF_OPTN_LOAD set  PTSF_OPT_DEFVAL = 0 where ptsf_srch_provider = 'ES' and FIELDNAME = 'PTSF_NOOF_REPLICAS'` | Search framework default options |
| `PS_PTSF_RTI_PRCS` | Truncate | Field `HOSTNAME` contains production server. This appears to be something to do with index updates. | Search Framework |
| `PS_PTSF_SRCH_ENGN` | Update | Update hostnames and URLs for search instances in columns `PTSF_HOST_NAME`, `PTSF_ADMIN_SRV_URL`, `PTSF_QRY_SERV_URL`, `PTSF_SS_URL`, `PTSF_KIB_HOST_NAME` Also contains ports, usernames and passwords. | Search Framework |
| `PS_PTSF_SRCH_NODES` | Update | Search nodes are updated to reflect the current environment. Contains a host name in `PTSF_HOST_NAME`. | Search Framework Nodes |
| `PSQRYTRANS` | Truncate | Contains host information in `HOSTNAME` and `QRYMACHINENAME` fields. | Info on running queries |
| `PSREN` | Truncate | Contains server names. Only an issue if Realtime Event Notification is used. | REN Configuration |
| `PSRENCLUS_OWNER` | Truncate | Contains server names if REN is being used. | REN configuration |
| `PSRENCLUSTER` | Truncate | Contains server names if REN is being used. | REN configuration |
| `PSRTNGDFNCONPRP` | Update | Update Integration Broker routing properties for interfaces to point to development rather than production environments. `PROPVALUE` contains URLS (Where PROPNAME = ‘URL’) for (e.g.) UCAS | Integration Broker routing properties |
| `PS_SAD_UC_CONFIG` | Update | Change the UCAS token in `SAD_UC_ATTRT`, username in `SAD_UC_UCIDXML` and password in `SAD_UC_UCPWXML` to the development ones | UCAS Configuration |
| `PS_SCHDLDEFN` | Update | Pause scheduled jobs (Set `SCHEDULESTATUS = 2`), so only processes that run as part of the refresh run during the refresh, and search indexes aren’t updated until they are first created. | Schedule definition |
| `PSSTATUS` | N/A | Needs to be granted to connectid as per `grant.sql`. Not needed. The grant should be bought across with the refresh. | Peopletools System Control |
| `PSTRUSTNODES` | N/A | From [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1) This contains a node name, which we preserve in test environments. Not needed - Only contains node names. | Message Node |
| `PSURLDEFN` | Update | Reset performance monitor URL `update sysadm.PSURLDEFN set URL = 'http://perfmon/' where URL_ID ='PPM_MONITOR'` There may be other URLs in here. | URL Definitions |
| `PSUSEREMAIL` | Update | Change email addresses | User Emails |
| `PSWEBPROFNVP` | N/A | [643499.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=643499.1): Run the following SQL to check the webprofile Virtual Addressing settings: (Cookie domains). e.g. check `select * from sysadm.PSWEBPROFNVP where propertyname in ('AUTHTOKENDOMAIN', 'PSWEBSERVERNAME');` Not needed in our system, but may be in others. | Web Profile Configuration |
| `PSXP_FILEURL` | Truncate | Contains process instance information. Database name in the `DBNAME` column, and a URL in the `DOC_URL` column. Truncate on refresh. | Report file URLs. |


## Random Notes

### Mail Catcher

In addition to changing all the email addresses, we configure development
systems to send email to
[MailCatcher](https://mailcatcher.me/).
We really don't want to send mails from test systems to end up confusing
people, so we use the mail catcher, but also change email addresses just
in case emails somehow end up in the real mail system.

### Everyone is Different

This works for our system, but the interfaces we have, the technologies
we use etc will differ from others. For example we don't use the Real-time
Even Notification (REN) framework. We have the tables listed above
because we did experiment with it for a while. We are in the UK, so we
have an interface with the Universities and Colleges Admission Service
([UCAS](https://www.ucas.com/)). Presumably in other countries there are
other agencies which are we might not have set up. So this should not
be treated as a complete list. But I think it is a good start.

We also have a number of processes that run after a refresh which also
alter data. Our technical team takes care of those, so I don't know about
them. We have a number of custom tables, which I haven't included above,
as others won't have the same customisations as us. It is worth checking
those.

Lastly we use the Appsian Security Platform from Pathlock. I might
make a list of tables we alter for that product in a later post.
