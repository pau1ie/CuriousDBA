---
title: "Triggering an Action in Another Repository"
date: 2019-12-10T09:34:09Z
tags: ['Automation','GitLab','Scripting','Source']
---

## GitLab Organization

It seemed reasonable when specifying the repositories, to have three (at first):

* One to hold the Ansible roles to build the VMs
* One to hold the custom code that the above repository will deploy
* One to define what the environments look like, things like
  * Memory
  * Number of VMs at each tier
  * The names of the VMs
  * Passwords
  * And so on.

## Problem
 
The problem is that when  the developer commits code to his repository, they would like it to be deployed
to an environment so they can test it. It makes sense to do this automatically, we have all the information
about the environment in the environment repository. This is different to the repository where the code is
being checked in. It would be nice if we could call across to that and trigger the automated deploy.
 
## Calling a Different Repository
 
The way to do this seems to be to call a pipeline. At first glance it is difficult to know how this helps,
as it is only possible to define one pipeline per environment, in the `.gitlab-ci.yml` file.
However, the power is in variables. If a variable is defined in the calling repository, it can be read
in the one that is called. I decided to test this out.

### To be triggered

First I created a repository: psh35/citest to be triggered.

```yaml
stages:
  - deploy

deploy-dev:
  stage: deploy
  script:
  - echo "I have deployed to dev"
  - env
  rules:
  - if: '$DEPLOY_DEV == "true" && $REPO_TICKET == $TRIGGER_TICKET'
    when: on_success
  - when: manual
```

This defines a single stage, deploy. There is a single job within it, which creates a step
to echo a statement, and print the environment. This runs if a couple of conditions are met:
The variable `DEPLOY_DEV` must be equal to `true`. This is set in the calling process, and is the indication we want
to deploy to dev rather than another environment, which will be useful if I create extra jobs in this stage,
as I intend to. Next I check that `REPO_TICKED` is equal to `TRIGGER_TICKET`.
These are also variables which I create in the GitLab CI settings. I set it to the same random string
in both environments to ensure
that nobody can trigger the build accidentally by (e.g.) cloning the triggering repository.

### Triggering Pipeline

The pipeline defined above is triggered by this one:

```yaml
stages:
  - deploy
  
trigger-dev:
  stage: deploy
  trigger: psh35/citest
  variables:
    DEPLOY_DEV: "true"
    TRIGGER_TICKET: "$REPO_TICKET"
```

So again there is a single deploy stage, with a single job, and its only action is to trigger the
pipeline. It sets the DEPLOY_DEV variable, and also sets the `TRIGGER_TICKET` to `REPO_TICKET`, which is defined
in the CI settings for the repository. 

## What it looks like

![Gitlab Pipeline Screenshot](../../images/GitlabPipeline.png)

We can see in the image that my update of .gitlab-ci.yml has triggered a pipeline in the triggering repository.
The trigger-dev job has run and called the citest
pipeline. What is nice is that clicking on the citest button reveals the deploy stage from the citest
repository, and clicking on that will show the log.

## Another Issue

When triggering a pipeline, it is done as the user who did the commit, and uses their
permissions within GitLab. Therefore, everyone who can trigger a pipeline in the code
repository, also needs permission to run the pipelines they are triggering.

## Conclusion

So it is fairly easy to call one repository from another in GitLab... Once you know how! I have put a little
thought into security so the API can't be triggered accidentally. This also means I can unset the `REPO_TICKET`
variable should I wish to do maintenance on the repository without triggering the downstream pipelines. All
in all, I am quite pleased with what I have done!


