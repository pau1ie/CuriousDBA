---
title: "SAML SSO for Django"
date: 2021-11-15T08:10:07+01:00
tags: ["Django","SAML","Security"]
---

The University has a Single Sign On (SSO) system. There are a number of ways that it can be used. In this case I
am investigating the use of Security Assertion Markup Language (SAML). There is also Shibboleth which is related,
and our SSO can also use, but I will leave that till another time.

I am creating a test application running in django on my desktop. Django by default only listens on the loopback
interface which means it can provide friendly information to developers safe in the knowledge that anyone who can
view it is logged on to my desktop. Sites are identified using their URL, so I need to add a unique hostname. I edited /etc/hosts and added myhost.local as a hostname to the end of the line that starts 127.0.0.1. Now
I can visit http://myhost.local:8000 in my web browser and get to my test website.

## Getting Started
I need a test website of course. I am using the [python3-saml](https://github.com/onelogin/python3-saml) library
which has a test Django website in it, which is useful.

On my linux desktop I cloned the repositories:

```bash
git clone https://github.com/onelogin/python3-saml.git
```

I installed the dependencies for [xmlsec](https://pypi.org/project/xmlsec/) which python3-saml depends on. 
This is RedHat, there are notes for other environments in the [xmlsec](https://pypi.org/project/xmlsec/) page.

```bash
dnf install libxml2-devel xmlsec1-devel xmlsec1-openssl-devel libtool-ltdl-devel
```

If this isn't done, the pip install errors.

Now we are ready to create a python virtual environment for the demo app. I am never sure where to create them, 
I normally put them in the project, but don't upload them to git. Once the environment is created it is activated
so all the dependencies get installed here. I find I always need to upgrade pip as the first step as it tends
to be so far out of date it doens't always install newer programs proplerly.

```bash
cd python3-saml/demo-django
python3 -m venv demoenv
. demoenv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

Now if I run the application I can point the web browser at it, and see the home page. If I try to log in it will try
to use the OneLogin SSO, and it won't work. I want to use the University SSO. I need to do some more configuration.


## Configuration
### Creating the certificate
Looking at the [readme for python3-saml](https://github.com/onelogin/python3-saml/blob/master/README.md) I can see
I need to create a certificate. As the readme says I can do the following

```bash
openssl req -new -x509 -days 3652 -nodes -out sp.crt -keyout sp.key
Generating a RSA private key
.............................................................................+++++
...................+++++
writing new private key to 'sp.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:GB
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:Cambridge
Organization Name (eg, company) [Default Company Ltd]:University of Cambridge
Organizational Unit Name (eg, section) []:UIS
Common Name (eg, your name or your server's hostname) []:myname.local
Email Address []:myname@cam.ac.uk
```

This creates an X.509 certificate valid for 10 years. The nodes argument stands for *no DES* and means it won't be
password encrypted.

I have seen in other places suggestions to use additional parameters as follows:

- -sha1 This is the digest format, and appears to be the default.
- -new A new request. Presumably this is the default
- -newkey rsa:2048 which is the algorithm followed by the size. Again this appears to be the default.

The certificate can be viewed by:

```bash
openssl x509 -text -in sp.crt 
```

Which gives the same information as was input when the certificate was being created.

### Configuring Django

The python3-saml library provides a settings.json file which needs to be filled in. As I say I am using
the University SSO rather than OneLogin, so I need to change all the information in there. SAML has it's
own set of abbreviations whihc need to be learned. They include:

- SP - Service Provider, i.e. my website
- IDP - Identity Provider, i.e. the SSO servide.

So I need to configure settings.json something like the following. I don't think my identity provider
has single log out, so I just deleted those lines.

```json
{
    "strict": false,
    "debug": true,
    "sp": {
        "entityId": "http://myname.local:8000/metadata",
        "assertionConsumerService": {
            "url": "http://myname.local:8000/?acs",
            "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
        },
        "NameIDFormat": "urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified",
        "x509cert": "",
        "privateKey": ""
    },
    "idp": {
        "entityId": "https://shib.raven.cam.ac.uk/shibboleth",
        "singleSignOnService": {
            "url": "https://shib.raven.cam.ac.uk/idp/profile/SAML2/Redirect/SSO",
            "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
        },
        "x509cert": "The X509 cert from my identity provider"
    }
}
```


### Registering with the Identity Provider

SAML requires the identity provider to be told about the application. Fortunately ours has a
handy web form, so I don't need to waste anyones time. It has three boxes requesting a name for the
registration, a description and the SAML Metadata describing the site. The last one seems a bit tricky.
python3-saml fortunately has a page /metadata which generates the required metadata.

This is an XML snippet, not a full XML document, so it doesn't have the XML header. This means
the namespaces may or may not be defined. In my case they are not, so I had to add them in manually.
If you have the same problem, take the xml document from metadata and add in the XML
namepsaces as per the highlighted lines below:

{{< highlight xml "linenos=table,hl_lines=2 11" >}}
<md:EntityDescriptor
    xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
    validUntil="2021-11-11T14:00:03Z"
    cacheDuration="PT604800S"
    entityID="http://myname.local:8000/metadata">
  <md:SPSSODescriptor
      AuthnRequestsSigned="false"
      WantAssertionsSigned="false"
      protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <md:KeyDescriptor use="signing">
      <ds:KeyInfo xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
        <ds:X509Data>
          <ds:X509Certificate>
// Certificate from sp.cer //
          </ds:X509Certificate>
        </ds:X509Data>
      </ds:KeyInfo>
    </md:KeyDescriptor>
  <md:NameIDFormat>
    urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified
  </md:NameIDFormat>
  <md:AssertionConsumerService
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
      Location="http://psh35.local:8000/?acs"
      index="1"/>
  </md:SPSSODescriptor>
  <md:Organization>
    <md:OrganizationName xml:lang="en-US">
      organization: name: from advanced_settings.json
    </md:OrganizationName>
    <md:OrganizationDisplayName xml:lang="en-US">
      organization: Displayname from advanced_settings.json
    </md:OrganizationDisplayName>
    <md:OrganizationURL xml:lang="en-US">
      organization: url: from advanced_settings.json
    </md:OrganizationURL>
  </md:Organization>
  <md:ContactPerson contactType="technical">
// Contact details from advanced_settings.json //
  </md:ContactPerson>
</md:EntityDescriptor>
{{< / highlight >}}

I added this to our SSO server, and clicked save. Now when I click login on the demo site, I get a list of
attributes back. Great!

## Incorporating SSO into my application

This is all well and good, but I have a Django website which I want to incorporate into SSO. I think this is for the next
post. 


