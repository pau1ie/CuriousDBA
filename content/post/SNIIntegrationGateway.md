---
title: "SNI and Integration Broker"
date: 2020-06-10T16:12:10+01:00
draft: true
tags: ['PeopleSoft', 'Fail', 'HTTP','Java','Network','Weblogic']
---

A colleague had created a new service operation in peoplesoft integration
broker, but it wasn't working. After a lot of investigation, and a call to 
Oracle, we found the cause was 
[SNI: Service Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication).
This is a facility which allows several different hostnames to be on the same IP
address, but still listen in HTTPS.

It seems the Integration Broker converts the hostname to an IP address before
trying  to contact the remote site. The problem is then that the remote site
doesn't know which website it wishes to talk to, and this causes a handshake
failure. The messages in the logs are not helpful to say the least.

The SSL checker services are useful here as they will tell you whether the
website uses SNI or not. It is possible to increase the logging on the TLS
handshake by adding  `-Djavax.net.debug=all` to the java command line.

## Testing the Service Operation

My colleague migrated the service to my playground environment (that I can
restart without stopping anyone working), and explained how to use the
service Operation Tester. Navigate to 
(PeopleTools -> Integration Broker -> Service Utilities -> Service Operation Tester)
and search for the service and service operation.

I was confused by seeing the following in the Weblogic log (after switching on debugging):

```
Ignoring unsupported cipher suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

This is the cipher suite that the SSL Checker website suggested that Java 8
would use
to connect to the remote site. However, it turned out  to be a red herring:
this cipher suite was in fact used in the end, even though the message still
appeared in the log.

So how would one tell what the problem was by looking in the logs? I found
Atlassian had created a java program called 
[SSLPoke](https://confluence.atlassian.com/download/attachments/117455/SSLPoke.java), 
which was useful as it
worked and could be used as a comparison. Once it was compiled, it could be run
as follows:

```bash
JAVA_HOME=/path/to/jdk
javac SSLPoke.java
java -cp . -Djavax.net.debug=all SSLPoke remoteserver 443
```

The output can be redirected to a file. Anyway, the difference in the logs is
that sslpoke says (Lines are truncated for brevity):

```
*** ClientHello, TLSv1.2
RandomCookie:  GMT: 1574953867 bytes = { 113, 192, 139, 17, 21, 15, 221, 19,
Session ID:  {}
Cipher Suites: [TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384, TLS_ECDHE_RSA_WITH_
Compression Methods:  { 0 }
Extension elliptic_curves, curve names: {secp256r1, secp384r1, secp521r1}
Extension ec_point_formats, formats: [uncompressed]
Extension signature_algorithms, signature_algorithms: SHA512withECDSA, SHA51
Extension extended_master_secret
Extension server_name, server_name: [type=host_name (0), value=remote.host]
***
[write] MD5 and SHA1 hashes:  len = 243
```

But the weblogic trace says (Again lines are truncated, and the weblogic log prefix is deleted):

```
*** ClientHello, TLSv1.2
RandomCookie:  GMT: 1574881618 bytes = { 84, 146, 178, 186, 141, 63, 212, 22 
Cipher Suites: [TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384, TLS_ECDHE_RSA_WITH_
Compression Methods:  { 0 }
Extension elliptic_curves, curve names: {secp256r1, secp384r1, secp521r1}
Extension ec_point_formats, formats: [uncompressed]
Extension signature_algorithms, signature_algorithms: SHA512withECDSA, SHA51 
Extension extended_master_secret
***
[write] MD5 and SHA1 hashes:  len = 185
```
Notice that this example is missing the `Extension server_name` line. This appears to be the clue!

Also, I don't show that the data written is supplied in hex dump format directly
afterwards. We can clearly see in the SSLPoke one that the hostname is being sent,
but in the Weblogic one it is not. The Weblogic one finishes as follows:

```
[Raw read]: length = 5
0000: 15 03 01 00 02                                     .....
[Raw read]: length = 2
0000: 02 28                                              .(
READ: TLSv1 Alert, length = 2
RECV TLSv1.2 ALERT:  fatal, handshake_failure
%% Invalidated:  [Session-5, SSL_NULL_WITH_NULL_NULL]
called closeSocket()
handling exception: javax.net.ssl.SSLHandshakeException: Received fatal alert: handshake_failure
```

This message is not very useful because it doesn't give any clue as to the reason
for the handshake failure.

# The Fix

The fix is to tell PeopleSoft to send the hostname on the services that connect to
websites that use SNI. This is done in the
integrationGateway.properties file as follows:

```ini
# This property should only be used when the hostname in 
#  3rd party URL should not be converted to IP address.
# By default hostname in URLs are converted to IP address 
# in IntegrationGateway. If you want to change that behaviour
# you can mention the externalnames of service operations for
# which IntegrationGateway should not convert hostname to IP address,
# as comma separated values.
#
ig.UseDomainName.ExternalOperationNames=MYSERVICE.v1
```

I wonder why this is isn't the default. It would seem that this should be 
enabled for all interfaces, as servers configuration could change without
warning depending on their hosting.

To find the externalnames as required above, search for the service in
PeopleTools->Integration Broker->Integration Setup->Services, and click
on the service. The correct name to use is the Service Alias followed by
the version number (including the v).

