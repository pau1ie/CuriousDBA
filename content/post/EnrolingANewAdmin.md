---
title: "Enroling A New Admin"
date: 2019-06-13T10:18:20+01:00
tags: ["Automation","Ansible","Secrets","VM","GitLab","Security"]
---

## The Idea

Sometimes you see a private shared lane which has a gate to stop people using it, but the people who are allowed
have padlocks in a chain. Anyone who has a padlock in the chain can open it with the key in their keyring.

![Padlocks on a gate](../../images/padlocks.jpg)

This is very much like how the secure keys work. If someone else wants to be able to use the gate, they have to
get one of the three key holders to go to the gate with them, open their padlock, and insert their padlock into
the chain.


## In Practice

You create a key pair. This has a private key, which is like the key for the padlock, you keep this secret. Also
the public key, this is more like the padlock, and is used to encrypt the files so anyone with permission can
decrypt them. These are kept on a keyring.

To create the key pair run:

```bash
gpg --gen-key
```

Answer the questions. Here are some pointers:

* Type of key: RSA and RSA
* Key size: 4096
* Key is valid for: 0
* User ID
  * Real name: Your name!
  * Comment: Note the desktop machine the key is being created on, or what the key is for.
  * Email address: Your email address!
* Passphrase: You need to remember this, but it provides some protection if the private key is compromised.

Then you can export the public key to a file:

```bash
gpg --list-secret-keys
gpg --output mykey.asc --armor --export email@add.ress

```

The first command lists the keys, and the second one exports by email address.

Send the key to someone who has acccess to the repository and they will add it to the keyring.


## Adding another key:

To perform actions on the repository, [regpg](https://dotat.at/prog/regpg/) is used as follows:

The key is enroled by
changing to the root directory of the project and running:

```bash
gpg --import mykey.asc
regpg add XXXXXXXX
regpg recrypt -r
```

The XXXXXXXX is the key name reported by ```gpg --import```. The first command imports the key in to the keyring on the local
host. Then it is added into the list of keys the repository knows about. Lastly it re-encrypts all the secrets so the
new key can decrypt them all.


