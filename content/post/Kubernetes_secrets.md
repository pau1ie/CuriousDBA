---
title: "Kubernetes, Terraform and Secrets"
date: 2021-07-23T17:28:30+01:00
tags: ["Automation","Cloud","Secrets","Terraform","Kubernetes","Docker"]
---

## Getting started with Kubernetes and Terraform

I've been looking into how to learn terraform. I have also discovered for my project I need to use
Kubernetes. 
It turns out that it is really easy to create a kubernetes cluster on the local desktop to have
a play with. Here goes:


I got started using the [following tutorial](https://learn.hashicorp.com/tutorials/terraform/kubernetes-provider?in=terraform/use-case)
I used Kubernetes in Docker (kind) to test. This turned out to be really easy to install.
I already had Docker installed, so I didn't need to worry about  that. I downloaded kind from it's website - there is a compiled executable which is easiest.
[The instructions say](https://kind.sigs.k8s.io/docs/user/quick-start) on Linux:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

It also notes that kubectl will be required to control the cluster:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(<kubectl.sha256) kubectl" | sha256sum --check
mv kubectl /some-dir-in-your-PATH/kubectl
```

Then I followed the tutorial to create a cluster:

```bash
 kind create cluster --name terraform-learn --config kind-config.yaml
 kind get-clusters
 ```
 
 ## Secrets - pulling containers
 
 Once I had finished the tutorial, I wanted to be able to start working on some containers I had made in our local docker repo.
 This requires me to use a secret. To do this, I needed to add the secret to kubernetes:
 
 ```bash
 kubectl create secret docker-registry mysecret \
  --docker-username=myusername \
  --docker-password=mypassword \
  --docker-email=me@example.com
 ```
 
 Kubernetes replies:

```
secret/mysecret created
```

This is a secret which allows me access to the
 docker registry. So I can now pull containers from the registry if I set it up properly in terraform:
 
 ```terraform
 resource "kubernetes_deployment" "mydeployment" {
 
  spec {
    template {
      spec {
        image_pull_secrets {
          name  = "mysecret"
        }
        container {
          image = "registry.example.com/path/to/my/container/image/master:latest"
...
 ```

The secret sends the username and password I placed in the secret I created above to the registry to log in to pull the container.

I can look at the secret using the following:

```bash
$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-hzz2g   kubernetes.io/service-account-token   3      13d
mysecret              kubernetes.io/dockerconfigjson        1      22m
```

This lists the secrets, but that doesn't show the values. I can get more information by specifying an output format:

```json
kubectl get secret mysecret -o json
```
This replies:

```json
{
    "apiVersion": "v1",
    "data": {
        ".dockerconfigjson": "eyJhdXRocyI6eyJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOnsidXNlcm5hbWUiOiJteXVzZXJuYW1lIiwicGFzc3dvcmQiOiJteXBhc3N3b3JkIiwiZW1haWwiOiJtZUBleGFtcGxlLmNvbSIsImF1dGgiOiJiWGwxYzJWeWJtRnRaVHB0ZVhCaGMzTjNiM0prIn19fQ=="
    },
    "kind": "Secret",
    "metadata": {
        "creationTimestamp": "2021-07-23T10:33:22Z",
        "name": "mysecret",
        "namespace": "default",
        "resourceVersion": "2000896",
        "uid": "7b480748-61e0-4141-9518-6822b861a553"
    },
    "type": "kubernetes.io/dockerconfigjson"
}
```

The real secret is base64 encoded in `.dockerconfigjson`. I can decode it using `base64d` like this:

```bash
kubectl get secret mysecret -o json \
  | jq -r '."data".".dockerconfigjson" | @base64d' \
  | jq
```

Which gives:

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "username": "myusername",
      "password": "mypassword",
      "email": "me@example.com",
      "auth": "bXl1c2VybmFtZTpteXBhc3N3b3Jk"
    }
  }
}
```

I wondered what `auth` is, apparently it is just the base64 encoded version of `username:password`

```bash
kubectl get secret mysecret -o json \
  | jq -r '."data".".dockerconfigjson" | @base64d' \
  | jq -r '.auths."https://index.docker.io/v1/".auth | @base64d'
```

Returns

```
myusername:mypassword
```

## Secrets - Passing through the Environment

So, now I want to create a database password I can use in the container. These are generic secrets. I can create  one like this:

```bash
kubectl create secret generic dbpass \
    --from-literal=mypass1=p@assw0rd \
    --from-literal=mypass2=s0pers3kr3t
secret/dbpass created
```

I can look at it as before:

```bash
kubectl get secret dbpass -o json 
```

which replies with

```json
{
    "apiVersion": "v1",
    "data": {
        "mypass1": "cEBhc3N3MHJk",
        "mypass2": "czBwZXJzM2tyM3Q="
    },
    "kind": "Secret",
    "metadata": {
        "creationTimestamp": "2021-07-23T13:47:49Z",
        "name": "dbpass",
        "namespace": "default",
        "resourceVersion": "2020579",
        "uid": "754a42a2-724a-4daa-9992-0824255ab887"
    },
    "type": "Opaque"
}
```

For some reason while you specify _generic_ when the password is created, 
kubernetes says the type is _opaque_ when you read it back. The inconsistent 
terminology can be a little confusing.

Once again these are base64 encoded:

```bash
kubectl get secret dbpass -o json | jq -r '.data.mypass1 | @base64d'
p@assw0rd
```

I can pass this into my container as an environment variable as follows:

```terraform
resource "kubernetes_deployment" "mydeployment" {
  spec {
    template {
      spec {
        container {
          image = "registry.example.com/path/to/my/container/image/master:latest"
          name  = "mycontainer"
          env_from {
              secret_ref {
                name = "dbpass"
              }
            }
          }
```

Now if I run a container which has bash, I can connect to it and see the environment variables.

```bash
kubectl get pods
kubectl exec --stdin --tty mycontainer-8588bb5c4f-xr9lm --/bin/bash
env | grep pass
mypass2=s0pers3kr3t
mypass1=p@assw0rd
```

So the secrets are exposed in the environment. They aren't base64 encoded which is handy!


## Secrets - Mounting

We can modify the terraform above to pass the secrets to a file as follows:

```terraform
resource "kubernetes_deployment" "mydeployment" {
  spec {
    template {
      spec {
        volume {
          name   = "passwords"
          secret {
            secret_name = "dbpass"
          }
        }
        container {
          image = "registry.example.com/path/to/my/container/image/master:latest"
          name  = "mycontainer"

          volume_mount {
            mount_path = "/mnt"
            name = "passwords"
            read_only = true
          }
```

Again, we can see these secrets by connecting  to the running container:

```console
$ kubectl get pods
$ kubectl exec --stdin --tty mycontainer-8588bb5c4f-xr9lm --/bin/bash
# cd /mnt
# ls
mypass1  mypass2
# cat mypass1
p@assw0rd
# cat mypass2
s0pers3kr3t
```

## A note about config maps

Kubernets has config maps. These work in the
same way, as secrets, but aren't secret. So instead of passing a 
dozen environment variables, we can pass a config
map that contains them all.

```bash
kubectl create configmap myconfig \
  --from-literal=PORT=1521 \
  --from-literal=HOST=mydesktop \
  --from-literal=DBNAME=DEVELOPMENT
```

To share it as an environment variable, we do exactly the same as with the secrets, but replace `secret_ref`
with `config_map_ref`. Similarly to mount a config map we replace the secret configuration:

```terraform
        volume {
          name = "passwords"
          secret {
            secret_name = "dbpass"
          }
        }
```

with the config map configuration

```terraform
        volume {
          name = "passwords"
          config_map {
            name = "dbpass"
          }
        }
```
Obviously we wouldn't be storing passwords in the config map, so the names would change as well, but this demonstrates how similar 
config maps and secrets are. The volume once defined is mounted in exactly the same way as with the secrets.


## Conclusion

So now I have learned how to use Secrets in kubernetes. I can mount them as a filesystem, or I can
pass them as environment variables. The same applies to config maps. This is nice as we can have a
config map and secret pair for each of for development, staging and production.