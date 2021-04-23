---
title: "Creating The Database"
date: 2021-04-23T09:19:07+01:00
tags: ["Docker","Terraform","Cloud","Database"]
---

The next step in the 
[Hands on Guide to Google Cloud](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#create-a-database-instance)
says we should connect the Etherpad instance to a database. Let's see how we create the database
using the automated processes we are developing.

## Creating the database using Terraform

### Cloud SQL module

A SQL instance was created by the terraform
code that was generated from the boilerplate. University members can see the 
[sql.tf template]( https://gitlab.developers.cam.ac.uk/uis/devops/gcp-deploy-boilerplate/-/blob/master/%7B%7B%20cookiecutter.product_slug%20%7D%7D-deploy/sql.tf). It contains the following:

```terraform
resource "random_id" "sql_instance_name" {
  byte_length = 4
  prefix      = "sql-"
}

# We make use of the opinionated Cloud SQL module provided by Google at
# https://registry.terraform.io/modules/GoogleCloudPlatform/sql-db/.
#
# The double-"/" is required. No, I don't know why.
module "sql_instance" {
  source  = "GoogleCloudPlatform/sql-db/google//modules/postgresql"
  version = "4.4.0"

  name = random_id.sql_instance_name.hex

  # ... Snip ...
}
```

The University has decided to standardise on Postgres for new database instances. The
comment helpfully links to the documentation for the 
[Google SQL module](https://registry.terraform.io/modules/GoogleCloudPlatform/sql-db/google/latest/submodules/postgresql).

This documentation has a handy box with provision instructions:

```terraform
module "sql-db_postgresql" {
  source  = "GoogleCloudPlatform/sql-db/google//modules/postgresql"
  version = "4.5.0"
  # insert the 5 required variables here
}
```

The required variables are:

 Variable | Notes
--|--
 database_version  | The description unhelpfully says _The database version to use_. Presumably somewhere under the covers, Terraform will call the Cloud SDK, so we can use that documentation to find out [which database versions we can use](https://cloud.google.com/sdk/gcloud/reference/sql/instances/create#--database-version). My colleague picked "POSTGRES_11".
 name | The [Hands On Guide](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#create-a-database-instance) says to use sql-{random string}. Looking at the terraform code above we can see it creates a `random_id` called `sql_instance_name` which is used to name the database. Because Terraform is really clever it doesn't give it a new random ID every time it is run!
 project_id | This is the name of the project which was created for me and described in the _Hands On Guide_ under [Create a Google Project](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#create-a-google-project)
 zone | The _Hands On Guide_ says _For Region choose europe-west2 (London) and keep Zone to Any._  According to the [Google Cloud documentation](https://cloud.google.com/compute/docs/regions-zones#identifying_a_region_or_zone) a Region is a collection of zones. Scrolling down it lists all the zones that can be picked for each type of resource. It seems we have to pick the whole zone in terraform rather than just the region.

Hang on - that's only 4! I am sure they said there were 5 required variables. 

## Finding the Project ID

The way the project is defined is interesting. Since all resources are grouped into a project,
this needs to be defined so all resources can use the same project name. 
A [local value in Terraform](https://www.terraform.io/docs/language/values/locals.html) is s special type of variable.
It is defined in a `locals` block, and is referenced in expressions as `local.<NAME>`.  
In my project there is a `locals.tf` (The 
[cookiecutter template](https://gitlab.developers.cam.ac.uk/uis/devops/gcp-deploy-boilerplate/-/blob/master/%7B%7B%20cookiecutter.product_slug%20%7D%7D-deploy/locals.tf) 
is accessible to University staff). It contains the following: 

```terraform
locals {
  # Project id for workspace-specific project.
  project = module.project.project_id
  #...
}
```

The project id is defined here, but takes it's value from the project module. This is defined in 
[main.tf](https://gitlab.developers.cam.ac.uk/uis/devops/gcp-deploy-boilerplate/-/blob/master/%7B%7B%20cookiecutter.product_slug%20%7D%7D-deploy/main.tf):

```terraform
resource "random_id" "project_name" {
  byte_length = 4
  prefix      = "${local.slug}-${local.short_workspace}-"
}

# The project for this specific deployment.
module "project" {
  source = "git::https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-project.git"

  project_name    = "${local.display_name} - ${terraform.workspace}"
  project_id      = random_id.project_name.hex
```

The [project module](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-project) is publicly available. We can see in 
[outputs.tf](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-project/-/blob/master/outputs.tf) 
that project_id is defined as an output, which is why it can be read as per the snippet of locals.tf 
from my project which is pasted above. We can see that
the project_id is created from a 
[random_id](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/id) 
called project_name, which is defined above the project. The random id
assigns a prefix from `local.slug` which is defined as a string in the project, and 
`local.short_workspace`. This is defined in locals.tf in my project as follows:

```terraform
locals {
    short_workspace = lookup(
    {
      development = "devel"
      staging     = "test"
      production  = "prod"
    },
    terraform.workspace,
    substr(terraform.workspace, 0, 4)
  )
}
```

### A note about Workspaces

A [terraform workspace](https://www.terraform.io/docs/language/state/workspaces.html) 
is an instance of the application. The boilerplate deals with
development, staging and production, but others can also be catered for. The current
workspace can be discovered using 

```console
terraform workspace list
  default
* development
```

The `development` workspace is the default workspace for 
[Logan](https://gitlab.developers.cam.ac.uk/uis/devops/tools/logan).
The terraform default workspace is
`default`.

So putting what we discovered above together, my project ID is `localslug-devel-randomid`

## More Parameters to the SQL module

There are a couple more parameters to the cloud SQL module which would be useful to set rather than allow to default:

Variable | Notes
--|--
db_name | A name for the database. This is required for the application to connect. The _Hands On Guide_ says to choose `etherpad`
user_name | This is the admin user for the database. This confused me for a while, because it isn't how the application user connects.


So that leaves me with something like this in my projects `sql.tf`.

```terraform
module "sql_instance" {
  source           = "GoogleCloudPlatform/sql-db/google//modules/postgresql"
  version          = "4.4.0"
  database_version = "POSTGRES_11"

  name = random_id.sql_instance_name.hex

  project_id        = local.project
  region            = local.region
  zone              = "${local.region}-b" # Can be any of a, b or c. Take your pick!

# Snip ...

  # Default database and user.
  db_name   = local.sql_instance.db_name
  user_name = local.sql_instance.user_name
# Snip ...
```

The local variables are defined in `locals.tf` in my project.

So all this is very nice, but how do I make the application connect to the database?

## Database Account

The application needs a user in the database. We don't use the admin user because this
will likely have more permissions than the application needs. This users name is assigned in 
`locals.tf` to a variable `webapp_sql_user`.
The user itself is created in `webapp.tf` using module 
[google_sql_user](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/sql_user).

```terraform
resource "random_password" "webapp_sql_user_password" {
  length  = 16
  special = false
}

resource "google_sql_user" "webapp_sql_user" {
  name     = local.webapp_sql_user
  instance = module.sql_instance.instance_name
  password = random_password.webapp_sql_user_password.result
}
```

First the password is created in a similar manner as the random string part of the project id. 
A [random_password resource](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password) 
is created and called `webapp_sql_user_password`.

Next the [google_sql_user resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/sql_user) 
is used to create a user in the database. The username is set from the local value mentioned above. It takes parameters
of the username, defined in locals (as mentioned above), the password that was just created, and the SQL instance which
was created above. The 
[Google sql-db module documents](https://registry.terraform.io/modules/GoogleCloudPlatform/sql-db/google/latest/submodules/postgresql?tab=outputs)
`instance_name` as one of the outputs from that module, so we can
use that as an input to specify the database to create the user in.

## Service Account

As the 
[Hands On Guide](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#create-a-web-application-service-account) 
notes, even though we have just created the database and a user, the application still needs permission to connect to it.

There are a couple of service accounts defined in the terraform code. One in main.tf is used in setting up the project and so has
more privileges. We actually want to grant access to the service account that gets created to run the project. It's
a bit like the user an application runs under in Unix. As we saw above the project is
created by our [gcp-cloud-run-app](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app)
terraform module. In 
[outputs.tf](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app/-/blob/master/outputs.tf)
we can see the `service_account` output is defined to return `google_service_account.webapp`. This is defined in
[main.tf](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app/-/blob/master/main.tf)
as
```terraform
resource "google_service_account" "webapp" {
  project      = var.project
  account_id   = coalesce(var.service_account_id, "${var.name}-run")
  display_name = coalesce(var.service_account_display_name, "Web application Cloud Run service account")
}
```
This is a [terraform resource](https://www.terraform.io/docs/language/resources/syntax.html)
of type `google_service_account` The 
[Google Service Account Documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_service_account)
notes that this exports some attributes including the email address of the service account. This email address is used to
identify the account when granting access in [main.tf](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app/-/blob/master/main.tf)
as follows:
```terraform
# The webapp service account has the ability to connect to the SQL instance.
# (Only if sql_instance_connection_name is non-empty.)
resource "google_project_iam_member" "webapp_sql_client" {
  count = (var.sql_instance_connection_name != "") ? 1 : 0

  project = local.sql_instance_project
  role    = "roles/cloudsql.client"
  member  = "serviceAccount:${google_service_account.webapp.email}"
}
```

This is actually quite clever. This makes use of the 
[count meta argument](https://www.terraform.io/docs/language/meta-arguments/count.html)
to effectively construct an if statement. It is saying create zero resources (i.e. loop 0 times) if the
sql connection is blank, but one (loop 1 time) if the connection name is set. If there is a database, 
obviously the application will need to connect to it!