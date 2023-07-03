---
title: "More Django and SAML"
date: 2023-07-03T15:19:07+01:00
tags: ["Django","SAML","Security"]
---

Some time ago I set up a website using 
[Django](https://www.djangoproject.com/)
which I protected using SAML and
[Python Social Auth](https://python-social-auth.readthedocs.io/en/latest/index.html).
I drafted this post, as a follow up to my 
[original one](../djangosaml/) 
but never published it until now.


The way  to integrate SAML into Django, indeed to integrate most Single Sign On/Identity provider solutions into
most python based websites (Sorry, Service Providers) is to use 
[Python Social Auth](https://python-social-auth.readthedocs.io/).

## Installation

We need to ensure the following are installed in the environment

- python-social-auth
- python3-saml

I did read that the saml extra is required for python-social-auth which is installed using 

```bash
# This isn't needed!
pip install python-social-auth[saml]
```

but that didn't do anything extra when I tried it.

And python3-saml requires the following packages to be installed:

- libxml2-devel
- xmlsec1-devel
- xmlsec1-openssl-devel
- libtool-ltdl-devel

## Configuration

All the configuration this time goes into `settings.py`. We need to follow the general documentation for
[configuring python-social-auth](https://python-social-auth.readthedocs.io/en/latest/configuration/django.html) 
as well as the specific  
[documentation for saml](https://python-social-auth.readthedocs.io/en/latest/backends/saml.html).

The list of authentication backends in `settings.py` needs to include `SAMLAuth` as follows:

{{< highlight python "linenos=table,hl_lines=2" >}}
AUTHENTICATION_BACKENDS = [
    'social_core.backends.saml.SAMLAuth',
    'django.contrib.auth.backends.ModelBackend',
]
{{< / highlight >}}

 Looking at the
details returned by Raven (These differ by identity provider), when I log in I see the following, which is 
explained in the 
[SAML Attributes documentation for Raven](https://docs.raven.cam.ac.uk/en/latest/saml2-attributes/#all-service-providers):


| Key | Example value | Notes |
|-----|---------------|-------|
| 'urn:oid:0.9.2342.19200300.100.1.3' | ['myname@cam.ac.uk'] | Email address |
| 'urn:oid:1.3.6.1.4.1.5923.1.1.1.10' | | Anonymous identfier. This is a dictionary of data. |
| 'urn:oid:1.3.6.1.4.1.5923.1.1.1.6' | ['myname@cam.ac.uk'] | Principal Name. This looks like an email address, but shouldn't be used as one.  |
|'urn:oid:1.3.6.1.4.1.5923.1.1.1.6-eppn-nameid' | ['myname@cam.ac.uk'] | I am not sure how different this is from above. |
| 'urn:oid:1.3.6.1.4.1.5923.1.1.1.7' | - | Entitlement. |
| 'urn:oid:1.3.6.1.4.1.5923.1.1.1.9' | ['member@cam.ac.uk'] | Scoped affiliation. So I am in the University directory. |

In particular it doesn't include my name. This is probably because I specified a .local address for my website
rather than .cam.ac.uk. We should note all of these are returned in python lists, even if there is only one value.

So to create a userid I need to specify the key name above - `'urn:oid:0.9.2342.19200300.100.1.3'` for
email address to be used to create my local account.

The entries which were in the json file in the previous post need to go into `settings.py` as follows. I am
using the same certificate and key as before as this is still running on my desktop.

{{< highlight python "linenos=true" >}}
SOCIAL_AUTH_SAML_SP_ENTITY_ID = 'http://myname.local'
SOCIAL_AUTH_SAML_SP_PUBLIC_CERT = """certificate from sp.cer"""
SOCIAL_AUTH_SAML_SP_PRIVATE_KEY = """key from sp.key"""
SOCIAL_AUTH_SAML_ORG_INFO = {
    "en": {
        "name": "My test app",
        "displayname": "My test application",
        "url": "http://myname.local",
    }
}

SOCIAL_AUTH_SAML_TECHNICAL_CONTACT =  {
    "givenName": "Me",
    "emailAddress": "myname@cam.ac.uk"
}

SOCIAL_AUTH_SAML_SUPPORT_CONTACT = {
    "givenName": "Me",
    "emailAddress": "myname@cam.ac.uk",
}

SOCIAL_AUTH_SAML_ENABLED_IDPS = {
    "raven": {
        "entity_id": "https://shib.raven.cam.ac.uk/shibboleth",
        "url": "https://shib.raven.cam.ac.uk/idp/profile/SAML2/Redirect/SSO",
        "x509cert": """x509 cert from id provider""",
        "attr_user_permanent_id": "urn:oid:0.9.2342.19200300.100.1.3",
        "attr_username": "urn:oid:0.9.2342.19200300.100.1.3",
    }
}
{{< / highlight >}}

An issue I discovered here is that the language is `en-US` whereas I had specified `en_US`
which caused an error. I notice `en-GB` works as well, and so does just `en` by itself - maybe I should use that.

Once all this has been filled in we are ready to create the view to generate the metadata
to register ourselves at the identity provider. I found the missing imports a little
confusing, but I got there in the end. I added the following to `views.py`

{{< highlight python "linenos=true" >}}
from django.urls import reverse
from social_django.utils import load_strategy, load_backend

def saml_metadata_view(request):
    complete_url = reverse('social:complete', args=("saml", ))
    saml_backend = load_backend(
        load_strategy(request),
        "saml",
        redirect_uri=complete_url,
    )
    metadata, errors = saml_backend.generate_metadata_xml()
    if not errors:
        return HttpResponse(content=metadata, content_type='text/xml')
{{< / highlight >}}

This generates a very similar XML document fragment to before. Since I am using the same
certificate I thought the previous one should work again, but it didn't. So I had to edit it again
as before to add in the XML namespace definitions for `md` in the `EntityDescriptor` tag 
and for `ds` in the `KeyInfo` tag. See [my previous post](../djangosaml/) if this doesn't make sense.

## Configuring the Application

There is some more work to set up Django. Another page on the 
[python-social-auth website](https://python-social-auth.readthedocs.io/en/latest/configuration/settings.html)
instructs us.
I have to make the system automatically select the Identity Provider to log in when
attempting to access a protected page. To do this, I need to set the following in `settings.py`:

```python
LOGIN_URL = '/accounts/login/saml/?idp=raven'
```

Where `raven` is what I called the identity provider in the `SOCIAL_AUTH_SAML_ENABLED_IDPS` dictionary

To automatically create a user in Django once authenticated by Raven, I do the following in `settings.py`. 
Actually I don't need to do this it is the default as mentioned in the 
[social-auth documentation](https://python-social-auth.readthedocs.io/en/latest/pipeline.html)

```python
# This isn't needed after all!
SOCIAL_AUTH_PIPELINE = (
    'social_core.pipeline.social_auth.social_details',
    'social_core.pipeline.social_auth.social_uid',
    'social_core.pipeline.social_auth.auth_allowed',
    'social_core.pipeline.social_auth.social_user',
    'social_core.pipeline.user.get_username',
    'social_core.pipeline.user.create_user',
    'social_core.pipeline.social_auth.associate_user',
    'social_core.pipeline.social_auth.load_extra_data',
    'social_core.pipeline.user.user_details',
)
```

Another thing that isn't required, but might be later on is to ensure only users from 
Cambridge can authenticate. We do this by whitelisting  the Cambridge domain as follows:

```python
SOCIAL_AUTH_WHITELIST_DOMAIN=['cam.ac.uk']
```

The nice thing about this is that any views that require protecting
just need to have the `@login_required` decorator to be applied. When  the user tries to access
a page containing such a view when they are not logged in, they are redirected to SSO and logged in
automatically.

## Troubleshooting

The ID provider said my website wasn't configured. When I looked at the application metadata
I realised it was because the entityID was set to the hostname of the VM running the application
rather than the hostname the application would be given through the load balancer. I fixed this
in settings.py by changeing the following:

SOCIAL_AUTH_SAML_SP_ENTITY_ID=https:/
SOCIAL_AUTH_SAML_ORG_INFO['en']['url']