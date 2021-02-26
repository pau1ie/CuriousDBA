---
title: "Building A Container with AutoDevops in GitLab"
date: 2021-02-19T13:46:22Z
tags: ["GitLab","Automation","Docker","Cloud"]
---

In the [last post](../gitlabrunnerforautodevops/) I set up a runner. Now lets see if we can use
it to create a container.

I am looking into how my
[colleagues](https://guidebook.devops.uis.cam.ac.uk/en/latest/) at
[Cambridge](https://www.cam.ac.uk/) do things. They have a [handy
guide to the Google cloud](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/)
using what colleagues call _click ops_. When I ran through it I was given a project which already had an
ID, so had to make sure I used the project ID I was given rather than the one in the document. This
also applied to the project registry.

This gives a quick overview of what is available on the cloud - how to create an application and connect
it to a database, create users and allocate memory and CPU. It isn't how you would do things
in production though - you would create a project in git and use automation to deploy it.

In the University we [have our own instance](https://gitlab.developers.cam.ac.uk/explore) of [GitLab](https://gitlab.com/explore/).
This is actually quite a complex piece of software. In addition to hosting
git projects, it supports automation of the build and deploy phases, which
is quite useful. This is what I will use to orchestrate the
build and deploy of an application.

## Making a project

My colleague set me up with a project which deploys the default container with a picture of a unicorn.
So I decided to try to deploy [etherpad](https://etherpad.org/) as in the
[tutorial](https://techdesign.uis.cam.ac.uk/en/latest/guidance/hands-on-google-cloud/#deploy-a-custom-etherpad-image). To do this, I needed to create a project to
create the container. I created a project called etherpad-test in gitlab, and in the project I created
a dockerfile as suggested in the tutorial:

```docker
FROM etherpad/etherpad:1.8.4
```

Then we need to tell Gitlab to build the file. This was largely copied from another project.

```yaml
# This file pulls in the GitLab AutoDevOps configuration via an include
# directive and then overrides bits. The rationale for this is we'd like this
# file to eventually have zero local overrides so that we can use the AutoDevOps
# pipeline as-is.

include:
  # Bring in the AutoDevOps template from GitLab.
  # It can be viewed at:
  # https://gitlab.com/gitlab-org/gitlab-ee/blob/master/lib/gitlab/ci/templates/Auto-DevOps.gitlab-ci.yml
  - template: Auto-DevOps.gitlab-ci.yml

  # Overrides to AutoDevOps for testing
  - project: 'uis/devops/continuous-delivery/ci-templates'
    file: '/auto-devops/tox-tests.yml'

  # Overrides to AutoDevOps for building with extra tags
  - project: 'uis/devops/continuous-delivery/ci-templates'
    file: '/auto-devops/extra-tags.yml'

variables:
  DOCUMENTATION_DISABLED: "true"
  TEST_DISABLED: "true"
```

The two includes are public, and can be viewed at 
[the University GitLab](https://gitlab.developers.cam.ac.uk/uis/devops/continuous-delivery/ci-templates/-/tree/master/auto-devops)

So I now have a project with two files, the Dockerfile which describes how
to build the container, and .gitlab-ci.yml which describes what to do
with it.

This will run the Auto DevOps pipeline, pretty much with the defaults. 
I have switched off the documentation and tests because there aren't any. Let's have a look to see what happens.

![Auto DevOps Pipeline running](../../images/GitLab/AutoDevOpsPipeline.png)

The pipeline detects the Dockerfile, generates the image and runs some
tests and analysis on it. Nice!

Looking at the log for the build step, we can see that the Docker image
is pushed to a repository:

```console
The push refers to repository
 [registry.gitlab.developers.cam.ac.uk/me/myproject/master]
```

The next problem is how we deploy this. I will [look into that](../deployingacontainertogooglecloud/) next.


