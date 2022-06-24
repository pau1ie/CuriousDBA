---
title: "Encrypting PeopleSoft Internal Communication - Elastic Search and Kibana"
date: 2022-06-24T08:24:07+01:00
tags: ["TLS","Network","PeopleSoft","Security","Elastic Search","Kibana","openssl"]
---

Now we have the core PeopleSoft stack communicating encrypting traffic, we 
need to address the add-ons. Elastic search is becoming more important, and
it is difficult  to use PeopleSoft without it. Here is how to add
encryption to Elastic Search and Kibana.

There is two way communication between Elastic search and PeopleSoft, Both of
these directions need to be configured. Fortunately we have already done some
of this work.

This procedure is documented in PeopleBooks under *Products -> Development Tools ->
Search Technology -> Working with PeopleSoft Search Framework Security Features*.
Here are links for 
[Elasticsearch](https://docs.oracle.com/cd/F52213_01/pt859pbr3/eng/pt/tpst/ConfiguringSSLBetweenPeopleSoftAndElasticsearch.html)
and
[Kibana](https://docs.oracle.com/cd/F52213_01/pt859pbr3/eng/pt/tpst/ConfiguringSSLForKibana.html)


## Setting up TLS for Elastic Search

### Creating the Key Store

The elastic search system is written in Java, the same as WebLogic. So setting
up the identity is the same as we did for
[Weblogic](../encryptingpeoplesoftinternalcommunications/).
The location of the identity store (Which elastic search calls the keystore)
is configured in `<base>/pt/elasticsearch<version>/config/elasticsearch.yml` 
with the `orclssl.keystore` parameter. I put my key store in a `keystore` 
directory under this `config` directory.

Where: `<base>` is the value given to `install_base_dir` when running the DPK,
and `<version>` is the version number of elastic search being installed.

From here we proceed 
[exactly the same as for Weblogic](../encryptingpeoplesoftinternalcommunications/#setting-up-the-keystore),

* we create the key store
* generate the certificate signing request (CSR)
* transfer  it to the Certificate Authority (CA)
* get it signed
* copy the signed certificate back to the elastic search server
* import the signed certificate  to the key store

### Creating the Trust Store

The trust store is also be created in the same way as the 
[WebLogic one](../encryptingpeoplesoftinternalcommunications/#setting-up-the-trust-store),
or it can be created from scratch to only trust the certificates we want it to
as follows:


```bash
keytool -importcert \
  -alias 'selfroot' -file root.crt \
  -storepass <trustpass> -keystore trust.jks
```

Repeat this command for each certificate. This should create the trust store
if it does not already exist, but as with WebLogic it might be wise to start
with the existing Java key store and add any certificates required. Either way,
we need to add our CA's root certificate.


### Encrypting the Store Passwords

The key store and trust store are password protected. These passwords are
encrypted before placing them in the configuration file. This is done by use
of the `elasticsearchuser` utility located in the Elasticsearch bin directory:
`<base>/pt/elasticsearch<version>/bin`
as follows:

```bash
bash elasticsearchuser encrypt <password>
```

By default the file is not executable hence explicitly specifying it to be 
run in bash. 
The encrypted value is written to the screen, and is the value which needs
to go into the file below.


### Configuring Elastic Search

We need to add the following lines to the `elasticsearch.yml` file

```yaml
 orclssl.http.ssl: true
 orclssl.transport.ssl: true
 orclssl.truststore: <base>/pt/elasticsearch<ver>/config/keystore/trust.jks
 orclssl.truststore_password: vPxw...4wBliw==
 orclssl.keystore: <base>/pt/elasticsearch<ver>/config/keystore/keystore.jks
 orclssl.keystore_password: RxdF...8jDf3==
```

This converts the port that Elastic Search listens on from being unencrypted
to TLS. We will need to make changes in the PeopleSoft application to cope with
the fact the elastic search traffic is now encrypted. Before that, let's look
at Kibana.


## Setting up TLS for Kibana

Kibana also communicates with the application using HTTP, so we need to ensure
that is encrypted using TLS. The process
is different to Elastic Search. There are two configuration options:
`server.ssl.certificate`, and `server.ssl.key`
which are used to provide Kibanas identity.

The directory structure is similar to Elastic Search in that the installation
directory is called `Kibana` with a version number appended, and is located
in the base directory specified when installing with the DPK. I will call this
`<kibanadir>`.
I put the keys and certificates in `<kibanadir>/config/keystore`


### Creating the Key and Certificates

If Kibana is running on the same server as Elastic Search, it is possible to
use the same certificate as Elastic Search. Otherwise the certificate signing
request will have to be 
[generated](../encryptingpeoplesoftapplicationserver/#create-the-key-store-and-the-certificate-signing-request)
and 
[signed](../encryptingpeoplesoftapplicationserver/#signing-the-csr)
as we did on the Application server.

We need to create a store in pem format. `keytool` doesn't know about this
format, but it does know about the p12 format, so we can export it into
that format.

```bash
keytool -importkeystore -noprompt           \
  -srckeystore <elasticsearch>/keystore.jks \
  -destkeystore <kibana>/keystore.p12       \
  -srcstoretype jks                         \
  -deststoretype pkcs12                     \
  -srcstorepass pass:<password>             \
  -deststorepass pass:<password>
```
This is the format that the key store for the 
[application server](../encryptingpeoplesoftapplicationserver/#import-the-signed-certificate-and-root-certificate)
is created in. We can now use openssl to convert the keystore to pem format,
then to key format:

```bash
openssl pkcs12
  -in keystore.p12         \
  -passin pass:<password>  \
  -out keystore.pem        \
  -passout pass:<password>

openssl rsa
  -in keystore.pem         \
  -passin pass:<password>  \
  -out keystore.key        \
  -passout pass:<password>
```

This converts the private key to a *PEM RSA private key*. It feels like there
should be a way to convert this in one step, but this is what the Oracle
documentation suggested. Anyway, `ksystore.key` is the key Kibana uses as it's
identity.

We also need the signed certificate request `<hostname>.cer` and the root
certificate for our CA, `<root>.crt`.

### Configuring Kibana

Now we have managed to collect the key and certficates, we need to configure
Kibana to use them, as follows. We edit `<kibanadir>/config/kibana.yml` as
follows to
* Enable SSL
* Set up the identity certificate and key
* Trust the Elasticsearch server

```yaml
server.ssl.enabled: true
server.ssl.certificate: <kibanadir>/config/keystore/<hostname>.cer
server.ssl.key: <kibanadir>/config/keystore/keystore.key
elasticsearch.ssl.certificateAuthorities: <kibanadir>/config/keystore/root.crt
```

Note that these lines are scattered
through the config file, they don't all appear in one section as the way I
show above might suggest.


## Configuring the PeopleSoft Application

### Configure Trust

Next we have to configure PeopleSoft to trust our CA. The 
[Integration Broker](../digitalcertificates/#in-the-trust-store)
needs to trust it so it can fetch the search results. The process scheduler
needs to trust it, so it can post generated indexes. This is done 
[in the application](../digitalcertificates/#in-the-application-itself)
where we already imported our CA's root certificate. If this is not done, the
process scheduler doesn't produce an error message, it keeps trying
to upload the search index, using slightly more memory each time until
it is cancelled or runs out of memory and is killed by the operating system.
The log is full of messages like the following:
```
CCQChunkedProcessorObjRef::CompleteTransfer Max Error Count = 100
PTESMCurl::Run Start, Running[2]
PTESMCurl::curl_multi_info_read - CompletedMessage data.result[60], msgs_left[0]
PTESMCurl::Run Response:Not NULL,msg->data.result[60], Position[1], Response:Exists, msgs_left[0]
Elasticsearch Response Head:  Handle Position[1] - 
Transaction Statistics:  TransactionID[2], SegmentID[2], AttachmentSetID[0], AttachmentCount[0]
PTESMCurl::Run "es_rejected_execution_exception" resending data at postion [1] Rejection Count=1
PTESMCurl::CreateHandle Number of Transactions Created=3
PTESMCurl::Run Resetting data at position[1]
Current Directory /home/psadm2/psft/pt/8.59/appserv/prcs/PRCSDOM
```


### Update Search Instance Definition

In the application navigate to: 

*PeopleTools -> Search Framework -> Define Search Instances*

![Search Instance Properties](/images/SearchFrameworkES.png)

The *Search Instance Properties* page opens. The *Host Name* and *Port* are
already set up, but the *SSL Option* needs to be set to *Enable* in the Search
Instance Properties. If there are multiple elastic search instances, remember
to configure them all. This can be validated by use of the *Elasticsearch Interact*
button.

For Kibana, simply ensure that the *SSL Option* is set to *Enable*.

Presumably the *Call Back Properties* already has https set up if the application
is set to use https.


## Conclusion

We have encrypted communication at the 
[web](../encryptingpeoplesoftinternalcommunications/),
and [application](../encryptingpeoplesoftapplicationserver/)
tiers. We have now
also encrypted communication between PeopleSoft and Elastic Search. This
means that data in transit can't be sniffed. We only have  the database
left to complete, which will be done in a future post.