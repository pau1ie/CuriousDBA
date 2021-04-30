---
title: "Connecting To The Database"
date: 2021-04-30T09:19:07+01:00
tags: ["Docker","Terraform","Cloud","Database"]
---

Now that we have a database it is time to connect Etherpad to it.

We see in the 
[Hands on Guide to Google Cloud](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#connect-the-database-to-the-web-application)
that the way to achieve this is by setting
some environment variables to be read by Etherpad when it starts. Lets have a look how to do that.

## Adding Environment Variables
Once again this is set up by the 
[cloud run app module](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app).
which as I noted before is  the basis of my webapp. 

The boilerplate webapp module contains the following environment variable setting

```terraform
module "webapp" {
  source = "git::https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app.git"

  ...

  environment_variables = {
    EXTRA_SETTINGS_URLS = join(",", [
      module.webapp_secret_settings.url,
      "gs://${google_storage_bucket.configuration.name}/${google_storage_bucket_object.webapp_settings.output_name}",
    ])
}
```

This sets the `EXTRA_SETTINGS_URLS` variable which is used by Django webapps written by
the University such as the 
[Lecture Capture Preferences webapp](https://gitlab.developers.cam.ac.uk/uis/devops/lecture-capture/preferences-webapp/-/blob/master/README.md#loading-secrets-at-runtime).
While Etherpad won't understand this, it is instructive to
see how this works so we can set up the variables it does understand.

As we saw in [an earlier post](../creatinginfrastructurewithterraform/) 
the cloud run app module uses the [Google Cloud Run Service](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloud_run_service)
It notes that the environment variables are set in the env block, `name` being  the name
of the variable, and `value` is it's value. These are set by the [cloud run app module](https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app/-/blob/master/main.tf#L111)

So I can simply add the variables in as per the [hands on guide](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#connect-the-database-to-the-web-application):


```terraform
module "webapp" {
  source = "git::https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app.git"

  ...

  environment_variables = {
    TRUST_PROXY = "true"
    DB_TYPE     = "postgres"
    DB_USER     = "etherpad"
    DB_NAME     = "etherpad"
    DB_HOST     = "/cloudsql/etherpad-abfiopps:europe-west2:sql-ldpmpiea1"
    DB_PASS     = "Password for etherpad user created earlier"
	
  }
}
```

Great! But... These are what the hands on guide says to use. The SQL user we set up in terraform is webapp, so maybe we should change it to that. But there is already a variable in locals - we should obviously use that. Actually all the above are wrong, apart from the db_type.

## Asking Terraform for Values

Can we get them out of terraform?

```terraform
module "webapp" {
  source = "git::https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app.git"

  ...

  environment_variables = {
    TRUST_PROXY = "true"
    DB_TYPE     = "postgres"
    DB_USER     = local.webapp_sql_user
    DB_NAME     = local.sql_instance.db_name
    DB_HOST     = "/cloudsql/${module.sql_instance.instance_connection_name}"
    DB_PASS     = google_sql_user.webapp_sql_user.password
  }
}
```

That's better, but now we are passing the password in plain text. 

## Creating a Secret to hold the Password

The hands on guide explains how to use a secrets manager
to create a secret. In fact the extra settings URL parameter is managed as a secret, but it contains a JSON document, which
isn't what we want. Lets add another secret just containing the password.

```terraform
# A Secret Manager secret which holds the DB password
module "webapp_secret_dbpass" {
  source = "git::https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-secret-manager.git"
    
  project   = local.project
  region    = local.region
  secret_id = "webapp-secret-dbpass"   
    
  secret_data = google_sql_user.webapp_sql_user.password
}
```

## Granting Access to the Secret

I added access as well, again taking a cue from the 
webapp_secret_settings:

```terraform
# The web application's service account needs to be able to read the password
# secret.
resource "google_secret_manager_secret_iam_member" "webapp_secret_dbpass" {
  project   = module.webapp_secret_dbpass.project
  secret_id = module.webapp_secret_dbpass.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${module.webapp.service_account.email}"

  provider = google-beta
}
```

## Passing the secret to the application

Once we have changed the Etherpad container to use Berglas, we can pass the
URL of the secret, rather than the secret itself. So the environment variables
section is changed as follows:


```terraform
module "webapp" {
  source = "git::https://gitlab.developers.cam.ac.uk/uis/devops/infra/terraform/gcp-cloud-run-app.git"

  ...

  environment_variables = {
    TRUST_PROXY = "true"
    DB_TYPE     = "postgres"
    DB_USER     = local.webapp_sql_user
    DB_NAME     = local.sql_instance.db_name
    DB_HOST     = "/cloudsql/${module.sql_instance.instance_connection_name}"
    DB_PASS     = module.webapp_secret_dbpass.url
  }
}
```
That is to say we pass the URL of the secret in the Google Secrets Manager that contains 
the password rather than the password itself. Then Berglas can fetch the secret.

## Running Terraform

Running `terraform plan` says

```console
Error: Module not installed

  on webapp.tf line 26:
  26: module "webapp_secret_dbpass" {

This module is not yet installed. Run "terraform init" to install all modules
required by this configuration.
```

Running `terraform init` followed by `terraform plan` seemed to fix this problem.

What if it doesn't work? That is the focus of the next post.