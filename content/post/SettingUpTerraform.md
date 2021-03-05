---
title: "SettingUpTerraform"
date: 2021-03-05T10:38:34Z
tags: ['Logan','Docker','Terraform','Cloud']
---

Before starting  to change the configuration with Terraform, there is some
set up work that needs to be done.

While the getting started guides are fine, in practice this leaves a problem
of how to work with colleagues, and how to manage secrets.

My colleagues have created a tool called 
[Logan](https://gitlab.developers.cam.ac.uk/uis/devops/tools/logan)
which they use to run terraform.
It is installed using pip, but it is a docker container, so will require a working
docker to run properly. I have RedHat 7 installed on my desktop, so I had to
`yum install python3`. I found I had to upgrade pip. Since there are other requirements
to install I created a virtual environment to install and run it in:

```console
python3 -m venv logan
logan/bin/activate
pip3 install --upgrade pip
wget https://gitlab.developers.cam.ac.uk/uis/devops/tools/logan/-/raw/master/requirements.txt
pip3 install -r requirements.txt
pip3 install git+https://gitlab.developers.cam.ac.uk/uis/devops/tools/logan.git
```

I already had docker set up, and also the ssh key for Gitlab. I 
[installed the Google cloud SDK](https://cloud.google.com/sdk/docs/install) and 
[authenticated as a service account](https://cloud.google.com/docs/authentication/production)
as per the notes in the [Logan readme](https://gitlab.developers.cam.ac.uk/uis/devops/tools/logan/-/blob/master/README.md). 
Then I was ready to install and run Logan (still in
the python virtual environment).

Now in my project I was able to run 

```console
logan terraform init
```

Terraform sees that the project requires Google Cloud modules, and downloads them.
We can see that Logan takes care of mounting the current folder under /workdir inside
the container, so all the terraform commands will work properly. It also creates a volume
with the name of the current folder and -terraform-data appended which contains the
terraform state. This is a nice way to keep the state without polluting the project.