---
title: "Creating Infrastructure With Terraform"
date: 2021-04-16T11:45:06Z
tags: ["Docker","Terraform","Cloud","Automation"]
---

This is an enormous and very complicated area. The 
[hands on guide](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#deploy-a-custom-etherpad-image)
says to change
the memory limit for the application to make it run faster. So, how do we do this
in terraform?

In the project generated from cookiecutter, I can simply edit webapp.tf and change
the webapp module to add `memory_limit="1024M"`. Simple, but how do I know I can
do that?

I need to read up on the [Terraform language](https://www.terraform.io/docs/language/index.html)
to understand what is going on. It seems there are two types of module. A directory
of terraform code is called a 
[Root Module](https://www.terraform.io/docs/language/modules/index.html#the-root-module),
but within that code are module definitions.
These are called [Module blocks](https://www.terraform.io/docs/language/modules/syntax.html). 
So my webapp module defined with the module keyword is a module block.

```terraform
module "webapp" {
  source = "git::https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app.git"

  project          = local.project
  cloud_run_region = local.region
  memory_limit     = "1024M"

  # -- Snip --
}
```

Now,  the first thing I can see is that this is loading 
[gcp-cloud-run-app](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app), 
(which is also called a module) from a gitlab repository. Looking at the 
[main.tf](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app/-/blob/master/main.tf)
file inside this moule I can see `webapp` is a resource of
type 
[google_cloud_run_service](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloud_run_service). This is defined by the Google provider in Terraform I struggled to understand
what the documentation was telling me. The required part is: 
`template`->`Spec`->`Containers`->`Resources`->`limits`->`memory`. The page links
to the Kubernetes documentation rather than saying what can be changed,
but my browser opened at the wrong section. I think the values they
are referring to are [documented 
here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-requests-and-limits-of-pod-and-container)
and are:

>  * `spec.containers[].resources.limits.cpu`
>  * `spec.containers[].resources.limits.memory`
>  * `spec.containers[].resources.limits.hugepages-<size>`
>  * `spec.containers[].resources.requests.cpu`
>  * `spec.containers[].resources.requests.memory`
>  * `spec.containers[].resources.requests.hugepages-<size>`

So returning to the gcp-cloud-app file above, we can see that the memory limit is
taken from a variable. Variables are referenced as attributes on an object named var.
So, the variable must be declared somewhere. And that place is in 
[`variables.tf`](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app/-/blob/master/variables.tf#L57)
We can see the default value is assigned here: `512M`. So to use the variable (again this
doesn't seem to be documented very well) we have to assign a value to it. Since this
variable is used in my webapp module (Specifically in the gcp-cloud-run-app module)
I need to assign the value inside the module block. It seems then that all the other
definitions in this block are variables as well. 

Next we will look at connecting the application to the database.