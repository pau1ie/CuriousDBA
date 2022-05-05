---
title: "Encrypting PeopleSoft Internal Communication - WebLogic Server"
date: 2022-05-05T16:50:07+01:00
tags: ["WebLogic","TLS",'HTTP',"Network","PeopleSoft","Security"]
---

In the past we used to assume communication on our network was protected
by physical security. Now
that seems not to be a reasonable assumption. So we should probably encrypt
communication within PeopleSoft. Here is how I do that. 

We control both ends of communication between tiers of PeopleSoft. So
we can create our own certificate authority, and instruct our software
to trust it. 


## Creating the Certificate Authority (CA)

We need a root certificate for our Certificate Authority (CA).
I created it as follows:

```bash
mkdir ca
openssl req -new -x509 \
    -days 5653 \
    -extensions v3_ca \
    -keyout ca/root.key -out ca/root.crt
cat ca/root.key | regpg encrypt ca/root.key.asc
rm ca/root.key
```

The `openssl` command prompts for the following information:
| Key | Notes |
|---|---|
| Country Name | I believe this is defined by  ISO 3166, so the code for the United Kingdom is GB. |
| State or Province Name |  I left this blank. |
| Locality Name |   |
| Organization Name |  |
| Organizational Unit Name | The department I work for. |
| Common Name | A description of what the certificate will be used for. |
| Email Address |  A group email address. |


It also prompts for a password. This needs to be kept safe as the CA can't
be used without it.

I only need to run this step once. I won't need to create the CA again until
this one expires, or if it gets lost or compromised, or the password gets lost.
I do need to decrypt the CA key to be able to sign Certificate Signing Requests 
(CSRs), but I automate that.


### How it Works

We have a certificate authority which has a public certificate 
and a private key.
The public certificate needs to be trusted for our certificate authority
to work, so it needs to be installed in trust stores.

The private key is used to sign certificate signing requests (CSRs),
to establishes a chain of trust. Anyone who trusts the root
certificate will trust anything signed by the private key. This is
convenient because we only need to set up the trust once.


### Keeping Secrets

The private key needs to be kept private.
It is normally kept encrypted and password
protected on a filesystem with permissions set on a server with
access control. CSRs are transferred to the CA, not the CA to the CSR.
We don't want to end up with lots of copies of the private key
all over the place, this would make it harder to keep it secret!


## Load Balancer to WebLogic

Having created the CA, we can now use it to encrypt communications.
By default, WebLogic is set up with Development trust and identity. This uses
the same root certificate for everyone, and means that anyone who can obtain
Oracle Software can sign a certificate WebLogic will trust. So this needs to be 
changed.

WebLogic is written in Java and uses a Java key store for it's trust and identity
stores. The trust store contains public keys from other websites that we trust.
the identity store contains our private key. That needs to be kept safe.


### Setting up the Keystore

The key store is created using the Java `keytool` utility as follows. First we
need to create a key store. On the WebLogic server run:

```bash
keytool -genkey -alias <hostname> -keystore <keystorefile>      \
  -keyalg RSA -keysize "2048" -validity "180"                   \
  -dname "cn=<hostname>, ou=<department>, o=<org>, c=<country>" \
  -storepass <password> 
```

Next we need to generate a certificate signing request

```bash
keytool -certreq -alias <hostname> -ext "san=dns:<hostname>"    \
  -keystore <keystorefile> -keyalg RSA -file <hostname>.csr     \
  -storepass <password>
```

Where:

| Key | Value |
|---|---|
| hostname | Fully qualified hostname |
| keystorefile | Full path of the key store |
| department/org/country | Identify the organisation. |
| password | is a password! |

Now we have a key store which contains the private key. This
stays where it is, so it doesn't get compromised. The
`<hostname>.csr` file contains the certificate signing
request that is transferred to our certificate authority
for signing.


### Signing the Certificate Signing Request (CSR)

Java isn't installed on the location I use for the CA
but I can use `openssl` to sign the request as follows.

On the CA host run the following:

```bash
regpg decrypt ca/root.key.asc > ca/root.key
openssl x509 -req -days 730 -in <hostname>.csr \
  -CA ca/root.crt -CAkey ca/root.key           \
  -out <hostname>.crt
rm ca/root.key
```

Hostname is the fully qualified hostname of the 
WebLogic server. Once again I need to supply 
the password for the CA root key.

We now have the signed certificate in `<hostname>.crt`
This needs to be transferred back to the web server. 
Also, transfer the CA
public certificate `root.crt` as that needs to be trusted as well


### Importing the Signed Certificate to the Identity Store

Back on the webserver we need to import the signed certificate
into the identity store:

```bash
keytool -importcert -keystore <keystorefile> \
  -file <hostname>.crt -alias <hostname>     \
  -storepass <password>
```

The root public certificate needs to be imported to the 
trust store. For the trust store it is probably worth starting
with the Java or PeopleSoft supplied store and adding
certificates as necessary. If using the PeopleSoft
supplied one, the trust for the demo key should be removed.

```bash
keytool -importcert -noprompt -keystore <truststorefile> \
  -file root.crt -alias "Self signed root CA"          \
  -storepass <password>
```


### Configuring WebLogic to use the Custom Stores

Having created the custom identity and custom trust stores, we now
need to tell WebLogic about them. 
The [
Oracle Documentation](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/12.2.1.4/secmg/identity_trust.html#GUID-72723A30-24AB-4227-B96A-9C886008BA66)
on configuring identity and trust in WebLogic tells us how to do this.
See also the 
[hardening guide](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/12.2.1.4/lockd/secure.html).

To do this, log in to the WebLogic
console and navigate to *Environment -> Servers*. Click *PIA*, then select
the *Configuration* tab, and the *Keystores* tab underneath.
Click *Lock & Edit*, click on the *Change* button next to *Keystores*
and select *Custom Identity and Custom Trust*.

![WebLogic Keystore Form](../../images/weblogicKeystore.png)

 Click *Save*, then 
fill in the form for the locations for the identity and trust stores.

![WebLogic Keystore Locations Form](../../images/weblogicKeystoreLocations.png)


On the *SSL* tab, fill in the details for the key in the identity
store to use for identity, its location and passphrase.
![WebLogic SSL Locations Form](../../images/weblogicSSLKeys.png)


Alternatively we can use the *WebLogic Scripting Tool* (WLST)
to do this for us, which is handy when using an automated build.
For the script below to work, WebLogic needs to be shut down.


```python
readDomain('/path/to/domain')
cd ('Servers/PIA')

cmo.setKeyStores('CustomIdentityAndCustomTrust')
cmo.setCustomIdentityKeyStoreFileName('<Identity store>')
encpassi=encrypt('<Identity Store Password>')
encpasst=encrypt('<Trust Store Password>')
cmo.setCustomIdentityKeyStorePassPhraseEncrypted(encpassi)
cmo.setCustomIdentityKeyStoreType('JKS')
cmo.setCustomTrustKeyStoreFileName('<Trust Store File>')
cmo.setCustomTrustKeyStorePassPhraseEncrypted(encpasst)
cmo.setCustomTrustKeyStoreType('JKS')

cd('SSL/PIA')
cmo.setServerPrivateKeyPassPhraseEncrypted(encpassi)
cmo.setServerPrivateKeyAlias('<hostname>')
 
# Disable anonymous remote IIOP and T3
cd("/SecurityConfiguration/peoplesoft")
cmo.setRemoteAnonymousRMIIIOPEnabled(false)
cmo.setRemoteAnonymousRMIT3Enabled(false)

updateDomain()
closeDomain()
```


### Load Balancer

The load balancer also needs to trust this certificate, or
it might be possible to configure it not to check the certificate.
It depends on the level of security required.


## Summary

We have looked at how to create a Certificate Authority and 
encrypt data in transit between WebLogic
and the load balancer. There are other communication channels
within PeopleSoft which also need to be secured. Next I hope
to look at encrypting communication between the web and
application tiers.