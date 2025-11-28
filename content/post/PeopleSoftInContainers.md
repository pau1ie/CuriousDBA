---
title: "PeopleSoft in Containers"
date: 2025-11-28T09:13:40Z
tags: ["PeopleSoft","Containers","Podman","DPK","Automation"]
---

{{% notice warning %}}
TL;DR: If you need Cobol, there is no supported way to get this working.
{{% /notice %}}


Oracle mentioned that PeopleSoft is supported in Containers. So, with
great excitement I decided to try to get it working.

The general approach is as follows:


## Install the build environment

This uses the DPK. We need to ensure Podman is installed, and the 
associated build tools. Then we run the DPK install to create the
build environment:

```bash
dnf install container-tools
mkdir /scratch
cd /scratch || exit
cp /stage/PEOPLETOOLS-LNX-8.61.09_*of4.zip .
unzip PEOPLETOOLS-LNX-8.61.09_1of4.zip

cat <<-! > response.txt
env_type=midtier
deploy_only=True
deploy_type=tools_home
user_config_home_dir="/scratch/psftuser/pscfg_pt861_silent"
db_platform=ORACLE
psft_base_dir="/scratch/psft"
db_is_unicode=True
!

./setup/psft-dpk-setup.sh --silent --response_file=response.txt
```


This effectively does a tools only-installation. The important parts are
under the container home.

## Build the Container Image

### Copy the Required Files

We use the build environment created above. It's easiest to set an
environment variable.

```bash
CONTAINER_HOME=/scratch/psft/pt/ps_home8.61.13/containers/OraclePeopleTools
```
Next we need to copy the DPK files to the build location:

```bash
cd ${CONTAINER_HOME}/containerfiles/pt
cp /stage/PEOPLETOOLS-LNX-8.61.09_*of4.zip .
```

If we are using infra DPKs, these can also be copied into the same
location. Infra DPKs tend to be made available a few weeks after the
critical patches they rely upon.

### Run the script to create the image

We use Podman to create the container image. Note that I have used a
non-standard naming convention for the container.
```bash
podman build --no-cache -f ./Containerfile -t \
  http://registry.example.com/peoplesoftvanilla:8.61.09 .
```

### Push the image to the container repository

Log in to the container registry
```bash
podman login http://registry.example.com/ -u me --password
```
Push the image to the container repository

```bash
podman push http://registry.example.com/peoplesoftvanilla:8.61.09
```

## But What Happened?

### Installation

We ran a Podman build command, but what did that actually do?
The `$CONTAINER_HOME` directory contains a file called `Containerfile`.
What it does is takes a base image, Oracle Linux 8 in this case, and
uses it as a base. It copies the shell scripts from `$CONTAINER_HOME`
into the container. It:

* updates Linux,
* installs some additional packages that are required,
* does some checks, e.g. that there is enough space,
* Runs `install-dpk-lnx.sh` which is one of the files that was copied
  earlier.

This script runs

```bash
psft-dpk-setup.sh --silent --response_file=/in/pt/install.rsp \
  --customization_file=/in/pt/psft_customizations.yaml --oke_build
```

The install response file contains the following:

```ini
env_type=midtier
deploy_only=True
db_platform=ORACLE
psft_base_dir="/opt/oracle"
user_home_dir=/home
```

This tells the DPK what to install. We are installing all the software,
but no domains (`deploy_only`) or database (`env_type=midtier`)

The customizations file contains the following:

```yaml
---
setup_share: false
setup_sysctl: false
```

This just switches off some features that don't work in a container.


### Metadata

The `podman build` command adds some metadata to the container. This includes:

* Exposed ports,
* volumes,
* health check script (Which is one of the scripts copied above),
* command, i.e. what gets run when the image is started.


### In Summary

The container image has the PeopleSoft software installed, but no domains
have been created. This makes sense because the domains are what
contains the configuration which is specific to each deployment.

The image has metadata attached which includes the command which is run
when the container is started.

## Deploy/Start

### Concepts

For those of us unfamiliar with Containers, the terminology can be a
little confusing. So far we have created a container image, which is like
the filesystem and some metadata. When this is deployed, in effect the
processes are started. At this point we need to inject the information
specific to the deployment. If this is an application server for example,
it needs to know specific things such as:

* Database host
* Database port
* Database Service name
* Connect ID
* Connect Password
* Operator ID
* Operator Password

The software installed to the image is generic, but when a container is
deployed it needs to know things specific to that deployment.

Let's take a look at what that is.


### Preparing to Run the Container

We need the same DPK install as was done for the container build above.
In reality we only seem to need a few files under `$CONTAINER_HOME`. 
Here is how Oracle say we should deploy the container. This is confusing
as it is very similar to the build. The difference is that:

* Files relating to the build are in `$CONTAINER_HOME/containerfiles/pt`
* Files relating to the deploy are in `$CONTAINER_HOME/in/pt`

We need to:
* copy the DPK zip files into `$CONTAINER_HOME/in/pt` (even though they
  were already copied into the build location)
* Create an `install.rsp`
* Create `psft_customizations.yaml`

### An Example

Let's pretend I am creating a process scheduler. Here is what I would do.
First we copy in the DPK files.

```bash
CONTAINER_HOME=/scratch/psft/pt/ps_home8.61.13/containers/OraclePeopleTools
cd ${CONTAINER_HOME}/in/pt || exit
cp /stage/PEOPLETOOLS-LNX-8.61.09_*of4.zip .
```
If we have infra DPKs they get copied in as well.

Next I create the response file for the install
(`${CONTAINER_HOME}/in/pt/install.rsp`).
```ini
env_type=midtier
domain_type=prcs
db_platform=ORACLE
db_name=CSDEMO
db_service_name=CSDEMO
db_host=csdemodb
db_port=1521
db_protocol=TCP
connect_id=people
connect_pwd=peop1e
domain_connect_pwd=
opr_id=me
opr_pwd=mypassword
psft_base_dir=/opt/oracle
user_home_dir=/home
```
Lastly I create the customizations file,
`${CONTAINER_HOME}/in/pt/psft_customizations.yaml`:

```yaml
---
setup_share: false
setup_sysctl: false
```
The documentation suggests these are the only supported options, but
when I raised a service request with Oracle, I was told that I could use
all my normal customisations.

At last, we are ready to deploy the container.

### Actually run the deploy

We run the following two commands. The first creates a bridged network,
and the second starts the container using `run.sh` as configured in the
metadata.

```bash
podman network create --driver bridge ps-net
podman run -d --name prcsdemo.example.com \
  --hostname prcsdemo.example.com \
  -e DB_WAIT_TIMEOUT=1 \
  -p 10100:10100 -p 10101:10101 \
  -p 7000-7003:7000-7003 -p 7010:7010 \
  -p 8000:8000 -p 8001:8001 \
  -v ${CONTAINER_HOME}/in/pt:/in/pt \
  http://registry.example.com/peoplesoftvanilla:8.61.09
```

### What Actually Happens?

The `run.sh` script is being run, because that is waht the metadata
instructed. This calls the DPK install script again:

```bash
psft-dpk-setup.sh --silent --response_file=/in/pt/install.rsp \
  --customization_file=/in/pt/psft_customizations.yaml --oke_build
```

The response file and the customisations file were supplied above. So
this is doing a DPK install all over again, but this time it is creating
the domains as specified in `psft_customizations.yaml` and `install.rsp`.
This is why we have to copy the DPK zip files again and create the 
response files.


## What would I do differently

I think there are a number of opportunities to improve this process.

* Install patches using `OPatch` and `unzip` for the java patches.
  This will enable us to install patches promptly without having to wait
  for Oracle to supply the infra DPK.
* Install customisations such as Pathlock, Cobol and custom branding
* Investigate whether there are options for a quicker start up so we don't
  need to copy the DPK files, and run a full DPK install.

This is possible. The original image build creates a full install of all
the required software without domains. This can be patched by using the
original image as a base image, and then saved as a new image.

Taking this idea further, we could have an image at each level, which 
means if we were investigating a bug, we could just start the image
without branding, without customisations or without patches to see if
the issue was introduced by one of these levels.

We could speed start up by not running the full DPK install again, but
we could create domains using `psadmin` as we did in the days before
DPKs were introduced.

## The Insurmountable Issue

Cobol. Campus Solutions needs Cobol for admissions and for fees
calculation. These are not things we can just do without. There is 
no mention of Cobol in the install documentation, so it should 
*just work* right?

```bash
./setup_cobol_devhub_redhat -silent -IacceptEULA --installlocation...

        Running in a container.
        Unable to continue with install.
      -=-===================================-=-
```

This is the runtime version. I would be happy to compile on a virtual
machine and then copy the compiled code. It seems the supplier of Cobol
wants to control where you run their code. This is why we get periodic
issues when the license manager crashes and we suddenly can't run any
Cobol.

Oracle support
[Doc ID 2802464.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2802464.1)
states:

> PeopleSoft customers cannot use Oracle delivered Visual COBOL license
> with a Docker environment. The Visual Cobol Docker products and license
> are not available from Oracle and not supported by Oracle.

This is a little confusing, because I am not using Docker, I am using
Podman, but it appears the same applies.

I raised an SR to clarify what the supported approach was given that 
[Doc ID 2991346.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=2991346.1)
and the installation
[documentation it refers to](https://docs.oracle.com/en/applications/peoplesoft/peopletools/)
state that containers _are_ supported. It turns out they are supported
except for Cobol, without which Campus Solutions is useless.

It is disappointing that Oracle didn't point this out in their
documentation. I could have used the time I spent investigating this on
something more useful.

## Conclusion

This is an exciting technology, but it isn't any use for most (all) people
running PeopleSoft due to Cobol licensing and support issues.
