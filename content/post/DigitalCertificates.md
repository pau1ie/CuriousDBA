---
title: "TLS and PeopleSoft Integration Gateway"
date: 2022-04-28T16:11:07+01:00
tags: ["Weblogic","TLS",'HTTP',"Network","PeopleSoft"]
---

PeopleSoft in general leaves it to the administrator to ensure that digital
certificates are set up properly. Given digital
certificates don't tend to change very often, and the provider changes even
less frequently, it can be difficult to understand and remember how this works,
and prevent issues.


## What is a digital certificate?

### The two parts

There are two parts to a digital certificate. One part is the private key
which is used to encrypt data, and is installed, in our case, on the load
balancer. It is important to keep this safe so nobody can impersonate our
system. The public key is what we are mostly dealing with here. This
is available to anyone and can be used to decrypt the data, and check it 
came from someone we trust.


### The Public Certificate File
A digital certificate is a file which contains data in a key value format
including:

  * Who issued the certificate
  * What the certificate is for
  * Who the certificate was issued to
  * When the certificate is valid
  * Some numbers which are used to do clever maths for encryption

Looking at this website in a web browser, I can click on the padlock at the 
left side of the address bar. Depending on the web browser I can click a 
couple more times to see the actual certificate. At the moment I can see that
the certificate was issued to _*.netlify.app_ by _DigiCert TLS Hybrid..._
and that was issued by _DigiCert Global Root CA_. So for the website to be
displayed, the browser needs to trust the root certificate, then it will
trust all websites that have certificates issued by this root Certificate
Authority. They also need to be in date, and not revoked.

The web browser has a lot of certificate authorities that it trusts. These are
installed by the web browser provider. Oracle doesn't do this for PeopleSoft,
so the administrator needs to do it.

## Getting Hold of the Digital Certificate

### Using a Web Browser

A public certificate can be downloaded from a website. For example if 
using Chrome on Windows, the certificate can be viewed by clicking padlock at
the left of the URL, then clicking on *Connection is secure* in the menu
that pops up, then clicking *Certificate is valid* on the next menu. A
window appears. Clicking on the *Certification Path* tab shows a window
that looks like the following:

![view certificate](/images/DigitalCertificateExport1.png)

We want the root certificate, which is at the top of the list. Click on it,
then click the *View Certificate* button. Another window appears displaying
information on the root certificate.

![view certificate](/images/DigitalCertificateExport2.png)

In the *Details* tab, click the *Copy to File* button. A wizard opens.
Click *Next* to get past the welcome page, then the following appears:

![view certificate](/images/DigitalCertificateExport3Format.png)

Choose *Base-64 encoded X509 (.CER)* for the format. Click *Next*,
then save it somewhere convenient.

### Using keytool

The java keytool can also be used though it will download the entire
certificate chain.

```
keytool -printcert -rfc -sslserver curiousdba.netlify.app:443 > curiousdba.cer
```


## Where are the Digital Certificates?

There are several places we can find digital certificates in PeopleSoft.
A couple of the most important are:

### In the Application Itself

The menu path is:
    *PeopleTools -> Security -> Security Objects -> Manage Digital Certificates* 
	
The first is used for when the application wants to talk to the integration
broker. It actually uses a web request to
call the integration broker which then calls out to the remote system. In practice
this means that the certificate used by the application itself needs
to be trusted. In our case the load balancer is configured with our private
identity certificate. Other sites may have this configured in WebLogic itself.

This page is not maintained by Oracle meaning the certificate is likely
to be absent. When a new certificate is added to the load balancer, 
the administrator should check that the root certificate is trusted.

The certificate should be provided by the load balancer admins from the
certificate authority, or it can be downloaded as above. Click one of the +
buttons at the end of a line to open up a new line. Type in an alias for 
the certificate. It doesn't matter what it is, but it might be useful to use
the the *Common Name* (CN) part of the *Subject* field. Type something in the
*issuer alias* field, or you get an error. Then click the *Add Root* link
and paste in the cer file, including the *BEGIN CERTIFICATE* and *END CERTIFICATE* lines.
The application server will need to be restarted to pick up the new certificate.


#### What if it is Missing?

If the certificate is missing from the application, the integration gateway
won't work at all. This is confusing as it seems that we don't trust the
remote application certificate, but actually we don't trust our own!
Error messages tend to be quite poor indicating failure to contact the
remote system. Examples include:
  * Posting the elastic search index from the process scheduler to the elastic 
    search instance doesn't work because the elastic search certificate is 
	missing. The Elastic search TLS certificate needs to be trusted.
  * Calling an API from a remote host returns an error indicating that the 
    remote host isn't contactable. The PeopleSoft application certificate
	needs to be trusted, i.e. the certificate for the gateway URL defined
	in PeopleTools -> Integration Broker -> Configuration 
	-> Integration Gateways

Of course the above problems can also occur if the server isn't accessible
e.g. because of a firewall rule.

### In the trust store

This is configured in `integrationGateway.properties` as follows:

```ini
secureFileKeystorePath=.../piaconfig/keystore/pskey
secureFileKeystorePasswd={V2.1}/Ey2eR...G4LLrtkUVJUEQ==
```

The file is delivered and the default password is `password`. More
recently this password is set when PeopleTools is installed. This needs
to trust all websites that the integration broker is likely to visit.

This is more up to date than the application page above, but it might be
necessary to maintain it. Certificates for all services contacted by the
integration gateway need to be stored in this file.

To add a certificate to this file, we are supposed to use `pskeymanager.sh`
but it is just a Java key store, so I find it more convenient to use the Java
key manager as follows:

```bash
keytool -importcert -alias "myalias" -keystore piaconfig/keystore/pskey \
  -storepass password -file publickey.cer -trustcacerts -noprompt
```

Where `myalias` is the alias for the certificate, which can be anything, 
but more usefully would be the name of the issuer, and `password` is the
password to the key store. If keytool complains 
`Input not an X.509 certificate` the issue is normally extraneous spaces
e.g. at the end of a line.

#### What if it is Missing?

If the certificate is missing from the trust store any external
website that has its certificate supplied by this certificate authority
won't be trusted, so the integration broker will refuse to connect.


## Service Name Indication

Integration broker needs to be instructed to remember who it is talking to if the
remote site used SNI, which most do. See my 
[article on SNI](../sniintegrationgateway) for more information
about this.

## Conclusion

  * The key store in the application needs to trust the certificate assigned to PeopleSoft.
    Check this every time you change the application certificate provider.
  * Integration Broker's key store needs to trust all the sites integration broker will visit.
    Check when changing the application certificate provider as above, and also when
	adding and changing interfaces.
