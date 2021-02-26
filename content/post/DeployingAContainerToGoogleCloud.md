---
title: "Deploying a Container to Google Cloud"
date: 2021-02-26T10:09:23Z
tags: ["GitLab","Automation","Docker","Cloud"]
---

Having [created the container](../gitlabbuildingcontainerwithautodevops/), we now need to deploy it to the Google
cloud. We create a deploy project. This was created for me, I believe
from our [Boilerplate Google Cloud Deployment project](https://gitlab.developers.cam.ac.uk/uis/devops/gcp-deploy-boilerplate) 
This project can currently only be viewed from within the University. The
gitlab-ci.yml contained the following at the time of writing:

```yaml
# Use UIS deployment workflow, adding jobs based on the templates to deploy
# synchronisation images

include:
  - project: 'uis/devops/continuous-delivery/ci-templates'
    file: '/auto-devops/deploy.yml'
    ref: v1.2.0

  # Include template that lints local Terraform files
  - project: 'uis/devops/continuous-delivery/ci-templates'
    file: '/auto-devops/terraform-lint.yml'
    ref: v1.2.0

# Triggered by manually running the pipeline with DEPLOY_ENABLED="development"
# and WEBAPP_DOCKER_IMAGE set to the image to deploy
deploy_webapp_development:
  extends: .deploy_webapp_template
  environment:
    name: development/$DEPLOY_COMPONENT
    url: $WEBAPP_URL_DEVELOPMENT
  variables:
    DEPLOY_ENV: "DEVELOPMENT"
  rules:
    - if: $WEBAPP_DOCKER_IMAGE == null || $WEBAPP_DOCKER_IMAGE == ""
      when: never
    - if: $DEPLOY_ENABLED == "development"
      when: on_success
    - when: never
	
.deploy_webapp_template:
  extends: .cloud-run-deploy
  variables:
    # Informative name for image. This is used to name the image which we push
    # to GCP. It is *not* the name of the image we pull from the GitLab
    # container registry. The fully-qualified container name to *pull* should be
    # set via the WEBAPP_DOCKER_IMAGE variable.
    IMAGE_NAME: webapp

    # Prefix for service-specific variable names
    SERVICE_PREFIX: WEBAPP

    # Variables set by upstream deploy job
    RUN_SOURCE_IMAGE: "$WEBAPP_DOCKER_IMAGE"

    # The name of the deploy component - will be prefixed with the environment
    # name to create a gitlab deploy environment name
    DEPLOY_COMPONENT: webapp
```

So really 
What it does is to take the deploy template from the devops CI templates in
[the University GitLab](https://gitlab.developers.cam.ac.uk/uis/devops/continuous-delivery/ci-templates/-/tree/master/auto-devops) (This is publicly viewable as I write this) and use them to
deploy using some environment
variables to control what is being deployed. We can see the in the `deploy.yml`
the CI is instructed to do a `docker pull`
on the container that we built, tag it with the Google container name, then
push it to the Google cloud. 
There is another entry for staging, but first let's understand development.

As the comment notes, this is expected to be triggered manually having
the following variables set:

| Variable | Value |
|--|--|
|DEPLOY_ENABLED | "development" |
|WEBAPP_DOCKER_IMAGE | set to the image to deploy |


These are set in GitLab when the job is triggered. There are also some
variables set in GitLab for the development environment under 
`Settings->CI/CD->Variables` Notice they all
start with `WEBAPP` and all end with `DEVELOPMENT`. This is important
as noted in the comments in
[autodevops/deploy.yml](https://gitlab.developers.cam.ac.uk/uis/devops/continuous-delivery/ci-templates/-/blob/master/auto-devops/deploy.yml)
which was included in gitlab-ci.yml.

| Variable | Value |
|--|--|
|WEBAPP_DEPLOY_CREDENTIALS_DEVELOPMENT | JSON snippet containing deploy credentials. These were given to me when I ran set up. |
|WEBAPP_PROJECT_DEVELOPMENT | Project ID in the Google Cloud. This was given to me, the hands on guide explains how to to create it. |
|WEBAPP_RUN_SERVICE_NAME_DEVELOPMENT | Service Name inside the project. The hands on guide says "etherpad" |
|WEBAPP_RUN_SERVICE_REGION_DEVELOPMENT | Google Cloud Region. e.g. "europe-west1" |
|WEBAPP_URL_DEVELOPMENT | Development URL. This was set up for me. |

Going to the URL now shows Etherpad rather than the unicorn picture!

This seems a little backwards - we have deployed the container into some infrastructure.
How come the infrastructure was there? I know my colleague created it, but how? That is the subject of the next post.