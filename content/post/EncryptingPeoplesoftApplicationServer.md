---
title: "Encrypting PeopleSoft Internal Communication - Application Server"
date: 2022-05-12T10:45:07+01:00
tags: ["WebLogic","TLS","Network","PeopleSoft","Security"]
---

In the last article we looked at encrypting communication between
[WebLogic and the load balancer](../encryptingpeoplesoftinternalcommunications/).
Now it is time to investigate the
traffic between WebLogic and the Application server. Without this
configuration the logs get filled with messages like this:
```
WARNING: LLE Configuration discovered!
   Note that LLE has been deprecated.
   You should upgrade to SSL to secure network links.
```

Let's upgrade to SSL then!


## Application Server

We already discussed TLS with regards to the 
[Integration Broker](../digitalcertificates/) and the
[Web Server](../encryptingpeoplesoftinternalcommunications/). 
The application server conceptually works in the same way,
but it (mostly) isn't written in Java, so in practice the procedure is 
slightly different.

### Create the Key Store and the Certificate Signing Request

The 
[documentation](https://docs.oracle.com/cd/F52213_01/pt859pbr3/eng/pt/tsvt/UnderstandingOracleWallet.html)
mentions we can use the Oracle Client  to create
a wallet or `openssl`. I will use the latter, as I am automating this
process and hope to be able to reuse the code elsewhere. PeopleSoft
much like WebLogic already has a demo key store which we shouldn't use
as the private keys are widely known. We will create a new one under the
application server domain security folder, then generate the Certificate
Signing Request (CSR). This needs to be done as the user that owns the 
domain (by default `psadm2`). I start in the domain directory, by default
`/home/psadm2/psft/pt/8.59/appserv/APPDOM` where `8.59` is the PeopleSoft
major version.

```bash
cd security
mkdir wallet.secure
# Create the server key
openssl genrsa -out wallet.tls/server.key 4096
# Create the CSR
openssl req -new -key wallet.tls/server.key -out wallet.secure/<hostname>.csr \
  -subj '/C=CN/CN=<hostname>'
```

Where `wallet.tls` is a name for the wallet, and `<hostname>` is the fully
qualified hostname. The CSR is now in `<hostname>.csr`. 


### Signing the CSR

In exactly the same way as
[we did with WebLogic](../encryptingpeoplesoftinternalcommunications/#signing-the-certificate-signing-request-csr), 
we need to:
  * transfer this signing request to the Certificate Authority (CA)
  * sign it
  * transfer it back to the application server together with the root certificate

The signing command is exactly the same for WebLogic:

```bash
openssl x509 -req -days 365 -in <hostname>.csr \
  -CA ca/root.crt -CAkey ca/root.key           \
  -out <hostname>.crt
```

Note that the oracle documentation referenced above says to add the 
`set_serial 01` parameter to the `openssl x509` command. This is not
recommended. Some blogs recommend using  the CASerial parameter. Again
this is not recommended. 
[RFC3280 states that](https://datatracker.ietf.org/doc/html/rfc3280#section-4.1.2.2) 

> The serial number MUST be a positive integer assigned by the CA to
> each certificate.  It MUST be unique for each certificate issued by a
>  given CA 

 They should not
be consecutive. By leaving out these parameters `openssl` acts in the
recommended way and creates a random serial number which is from a large
enough pool it is guaranteed to be unique. The serial number can be displayed
to verify this:
```bash
openssl x509 -in <hostname>.crt -noout -serial
```
A long hex number is displayed. It seems the `openssl` documentation
hasn't been updated with this useful feature which has caused some
confusion.


### Import the Signed Certificate and Root Certificate

We should now have three files back in our `wallet.tls` directory, 
`<hostname>.cer` and `root.crt`, together with the `server.key`
that we created above. We have all the pieces we need to create 
the wallet. We put them together to create the wallet as follows:

```bash
openssl pkcs12 -export -out wallet.tls/ewallet.p12
  -inkey wallet.tls/server.key
  -in wallet.tls/<hostname>.crt
  -chain -CAfile wallet.tls/root.crt
  -passout pass:<wallet_password>
```

Where `<wallet_password>` is a password for the wallet. We will
need this later.

We don't need the `server.key`, `root.crt` or `<hostname>.crt` files
to be in the wallet directory
any longer, they can be removed now if you want to keep things tidy


## Configuring the Application Server

Next we need  to configure the application server to use this wallet to
encrypt communication. We can do this by editing the configuration file, or by
using the `psadmin` utility interactively.


### Manually Editing psappsrv.cfg

My preferred approach is to edit the configuration files, as this is
easier to automate. 
Edit `psappsrv.cfg` in the domain directory. Find the `Oracle Wallet` section, 
and make it look like this:

{{< highlight ini "hl_lines=5-7" >}}
[Oracle Wallet]
;=========================================================================
; Settings for Oracle Wallet
;=========================================================================
SEC_PRINCIPAL_LOCATION=/home/psadm2/psft/pt/8.59/appserv/APPDOM/security
SEC_PRINCIPAL_NAME=tls
SEC_PRINCIPAL_PASSWORD=<wallet_password>
{{< / highlight >}}

Where `tls` is the name of the wallet directory without the leading `wallet.`, and
`<wallet_password>` is what we specified when creating the wallet.

Make a note of the `Port` and `SSL port` in the `Jolt Listener` section, and ensure that
`JSL Min Encryption` is set to `256`.

{{< highlight ini "hl_lines=6-8" >}}
[JOLT Listener]
;=========================================================================
; Settings for JOLT Listener
;=========================================================================
Address=//0.0.0.0
Port=9033
SSL Port=9010
JSL Min Encryption=256
JSL Max Encryption=256
{{< / highlight >}}

Run a configure:

```bash
psadmin -c configure -d APPDOM
```

If we check the wallet password in `psappsrv.cfg` then we will see the password
has been encrypted and now looks something like the following:

```ini
SEC_PRINCIPAL_PASSWORD={V2}gWUH+ZuNbSOmeRand0mL3tt3rsAndNumb3rsOr0=
```

### Using psadmin Interactively

The other option is to run `psadmin` interactively to configure the domain.
This is more difficult to automate, but if 
raising issues with Oracle it's worth doing the configuration in this way
to eliminate errors in editing that `psadmin` might be able to pick up on.
The procedure below is for tools 8.59. Options may change for different 
tools versions.

  * Run `psadmin`
  * Select option *1) Application Server*
  * Select option *1) Administer a domain*
  * Select the domain from the list (Probably option 1)
  * Select option *4) Configure this domain*
  * Agree to shut down the domain  (y)
  * Select option *15) Custom configuration*
  * Press Enter to leave the *Startup* section alone
  * Press Enter to leave the *Database Options* alone
  * Press Enter to leave the *Security* section alone
  * Press Enter to leave the *Inter-Domain Events* section alone
  * Press Y and enter to edit the *Oracle Wallet* section as below:

```
Do you want to change any values (y/n/q)? [n]:y
    SEC_PRINCIPAL_LOCATION []: /home/psadm2/psft/pt/8.59/appserv/APPDOM/security
    SEC_PRINCIPAL_NAME [psft]: tls
    SEC_PRINCIPAL_PASSWORD []: <wallet_password>

Values for config section - Oracle Wallet
    SEC_PRINCIPAL_LOCATION=/home/psadm2/psft/pt/8.59/appserv/APPDOM/security
    SEC_PRINCIPAL_NAME=tls
    SEC_PRINCIPAL_PASSWORD=*****
	Do you want to change any values (y/n/q)? [n]:
```
Where `tls` is the name of the wallet folder without the leading `wallet.`
and `<wallet_password>` is the password we gave to the wallet when we created 
it above.
  * Press Enter to not change any values.
  * Press Enter to leave the *Workstation Listener* section alone
  * Select Y to change the *JOLT Listener* section
  * Press Enter 3 times to leave the address, Port and SSL Port alone, but make a note of the Port and SSL Port
  * Select Y to change *JSL Min Encryption* to `256`

```
  JSL Min Encryption [0]: 256
```

  * Press enter on the other 8 prompts to keep existing values.
  * Select q to exit when asked *Do you want to change any values* again.
  * As shown above, press q to return to the domain administration menu
  * Select option *1) Boot this domain* if you wish to boot the domain
  * Select *q* three times to exit psadmin


## Configuring the Web Server
 
WebLogic needs to be 
[configured to trust our CAs root certificate](../encryptingpeoplesoftinternalcommunications/#importing-the-signed-certificate-to-the-identity-store).
If WebLogic hasn't been configured, just import `root.crt` into the trust store.
Then the `configuration.properties` file needs to be edited to communicate with 
the application servers TLS port. By default the weblogic domain is under 

`/home/psadm2/psft/pt/8.59/webserv/peoplesoft`

Where `8.59` is the tools version. The configuration.properties file is under here in:
`applications/peoplesoft/PORTAL.war/WEB-INF/psftdocs/ps/configuration.properties`
Where `ps` is the website which may be different in your installation.

Then simply change any ports in the `psserver` line to point to the encrypted ports.

```ini
psserver=<appserver_hostname>:9010
```

Where `<appserver_hostname>` is the fully qualified hostname of the application
server, and `9010` is the encrypted port noted above.

Restart WebLogic and test that everything works! 