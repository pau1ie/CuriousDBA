---
title: "Parsing XML with Ansible"
date: 2021-10-20T16:19:07+01:00
tags: ["Ansible","Automation","Weblogic","XML"]
---

I am trying to gather some information about an environment once it has been created and save it in a small Django app. This is about my adventures trying to discover the Weblogic version from the it's registry which is an XML file.

The XML registry is in the Oracle inventory, and starts like this:

```xml
<?xml version = '1.0' encoding = 'UTF-8' standalone = 'yes'?>
<registry home="/opt/oracle/psft/pt/bea" platform="226" sessions="7" xmlns:ns2="http://xmlns.oracle.com/cie/gdr/dei" xmlns:ns3="http://xmlns.oracle.com/cie/gdr/nfo" xmlns="http://xmlns.oracle.com/cie/gdr/rgy">
   <distributions>
      <distribution status="installed" name="WebLogic Server" version="12.2.1.4.0">
```

I'd like to extract the WebLogic Server version which is on line 4. 
As I don't know XML it's tempting to use `grep` and `sed` to find the information I want, but I notice 
there is an XML module in Ansible. It is community maintained, and
it's not stable, so it's behaviour might change. This is valid from versions between 2.4 and 4,
maybe later.

What I had to do was:

```yaml
  - name: Find weblogic version
    xml:
      path: "/weblogic/home/inventory/registry.xml"
      xpath: "/x:registry/x:distributions/x:distribution[@name='WebLogic Server']"
      content: attribute
      namespaces:
        x: "http://xmlns.oracle.com/cie/gdr/rgy"
    register: webinfo
    
  - debug: var=webinfo
```

## Namespaces

One thing I find confusing about XML is the namespaces. This document lists three namespaces, but we are only 
interested in one, the default. This is the one without an alias:

```xml
xmlns="http://xmlns.oracle.com/cie/gdr/rgy"
```

The annoying thing is that while this is the default namespace in the document, the tools used to parse the
document don't default to this namespace, they default to no namespace, which means we won't get any data returned.
So we have to tell Ansible about the namespace. This is a dictionary, so I just created x as an alias. If I needed more
than one namespace I could add more elements to the dictionary. I define it in Ansible as follows:

```yaml
      namespaces:
        x: "http://xmlns.oracle.com/cie/gdr/rgy"
```

Another thing that isn't obvious is that the attributes within the elements don't obey the default namespace. 
Attributes don't have a namespace unless it is explicitly stated.

## Xpath

Now I have set up the namespace, I need to add it to the elements in the xpath, but as I said above, not to the attributes. So
I am left with the following as an xpath:

```yaml
      xpath: "/x:registry/x:distributions/x:distribution[@name='WebLogic Server']"
```

I think of the XML document like a filesystem. I am selecting the element `/registry/distributions/distribution`.
There are a number elements that match this criterion. I only want the one where the `name` is set to `WebLogic Server`, which
is what the ```[@name='WebLogic Server']``` part does.

I'd like to be able to specify I only want the version attribute. I believe I should be able to do by 
adding ```@version``` on the end of the xpath, but that didn't seem to work. The documentation says 
to limit xpath selectors to simple expressions, maybe that is why. Or maybe I just don't understand XML.


## Using the output

Ansible gives me the following output from the debug statement:

```json
ok: [webserver] => {
    "webinfo": {
        "actions": {
            "namespaces": {
                "x": "http://xmlns.oracle.com/cie/gdr/rgy"
            },
            "state": "present",
            "xpath": "/x:registry/x:distributions/x:distribution[@name='WebLogic Server']"
        },
        "changed": false,
        "count": 1,
        "failed": false,
        "failed_when_result": false,
        "matches": [
            {
                "{http://xmlns.oracle.com/cie/gdr/rgy}distribution": {
                    "name": "WebLogic Server",
                    "status": "installed",
                    "version": "12.2.1.4.0"
                }
            }
        ],
        "msg": 1
    }
}
```

I want to access the version. It took me ages to work out how to do this because of the braces in the
key. The way to do it in Ansible is (I do this in the defaults, or vars):

```yaml
  weblogic_ver: "{{ webinfo.matches[0]['{http://xmlns.oracle.com/cie/gdr/rgy}distribution'].version }}"
```