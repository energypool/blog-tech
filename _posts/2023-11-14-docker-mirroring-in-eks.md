---
title: Setting up Docker repository mirroring in EKS using Karpenter and Nexus
date: 2023-11-14 10:50:32 +0100
categories: [EKS, Containerd]
tags: [kubernetes, containerd, eks, karpenter, nexus]
author: t_falconnet
img_path: /assets/posts/2023-11-14-docker-mirroring-in-eks
---

Hi ! This post explains why and how we mirror public Docker repository in EKS. You will find detailed steps to set it up yourself.

## Context

We have been preparing for several months at Energy Pool for a migration to Kubernetes. Currenly, the majority of our workload is on AWS EC2, and we have recently set up an EKS cluster in our development environment.

During the installation and deployment of [Karpenter](https://karpenter.sh/), we quickly encountered the [Docker Hub rate limit](https://docs.docker.com/docker-hub/download-rate-limit/) due to excessive image pulls. We were repeatedly terminating our nodes for testing purposes.

To overcome this issue, there were different possibilities:
- Pay a Docker subscription: We were not interested.
- Synchronize public images to private ECR: Using a solution like [Skopeo](https://github.com/containers/skopeo) which I knew from a previous mission.
- Cache images using a mirror server: Theoretically, we knew it was possible, but we had no experience with that.

So, I started implementing the Skopeo solution. It has a few drawbacks:
- You have to continuously list all public images used.
- You have to update all components to use the corresponding synchronized image from your private repository.

It can be time-consuming at first to handle these drawbacks, but there was another need that Skopeo couldn't address.

Our Kubernetes cluster hosts [Github Action runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners). We have been using them for a few weeks with various workflows. We tried [Super Linter](https://github.com/marketplace/actions/super-linter) and we were quite amazed by the docker image size: 7.5GB (or 2GB in slim). We tought that it could be a great idea to store this image somewhere near our cluster to reduce pulling duration and save some energy 🌱. Unfortunately, there is no easy way to update the image used by a Github Action for a private one.

We decided to do some R&D around Docker repository mirroring, using Nexus as we already use it for other purpose at Energy Pool. You can find its Helm chart [here](https://github.com/sonatype/nxrm3-helm-repository/tree/main/nxrm-aws-resiliency). 

## Docker proxy

Let's go to Nexus administration view !

### Docker Hub proxy

In `Repository -> Repositories -> Create repository`:

1. Create a `docker(proxy)` repository
2. Name it such as `docker-hub`
3. Optionaly, check `Allow anonymous docker pull` (for unauthenticated pulling)
4. Add Remote Storage `https://registry-1.docker.io`
5. Set Docker Index to `Use Docker Hub`
6. Hit `Create repository`

![Nexus Docker Proxy Configuration](nexus-docker-proxy.png){: h="300" }

> Unfortunately, Docker mirrors can't be configured with an uri (like `NEXUS_URL/repository/docker-hub`).
> You can eventually enable `HTTP` or `HTTPS` in the previous repository configuration to dedicate a Nexus port to a repo. 
> I recommend to add a Nexus repository of type `docker group` (next step of this guide) to act as unique endpoint to multiple repositories (like `gcr.io`, `quay.io`, ...).  
{: .prompt-warning }

### Docker group

1. Create a `docker(group)` repository
2. Name it such as `docker`
3. Enable `HTTP` or `HTTPS`, and choose a port different from the one used by Nexus such as `8082`
3. Optionaly, check `Allow anonymous docker pull` (for unauthenticated pulling)
4. Add the previously created docker-proxy `docker-hub` in `Members repositories`
6. Hit `Create repository`

![Nexus Docker Group Configuration](nexus-docker-group.png){: h="400" }

## Authentication

Depending on your needs, set up Nexus either authenticated or unauthenticated. 
Unauthenticated will allow you to pull images directly from Nexus without login/password, but will also allow to read other repositories content by default.

### Unauthenticated

#### Docker Bearer Token Realm

In `Security -> Realms`, add `Docker Bearer Token Realm` to `Active` column and hit `Save`.

![Nexus Realms Configuration](nexus-realms.png){: h="300" }

#### Anonymous Access

In `Security -> Anonymous Access`, check `Allow anonymous users to access the server` and hit `Save`.

![Nexus Anonymous Access Configuration](nexus-anonymous.png){: h="300" }

### Authenticated

#### Privileges

In `Security -> Privileges -> Create privileges`:

1. Select `Repository View`
2. Name it such as `docker-pull`
3. Add Format `docker`
4. Add the previously created docker-group `docker` in Repository
5. Add Actions `read`

![Nexus Privileges Configuration](nexus-privileges.png){: h="300" }

#### Role

In `Security -> Roles -> Create roles`:

1. Select Type `Nexus role`
2. Add Role ID such as `docker-pull`
3. Add Role Name such as `Docker Pull`
4. Add the previously created privilege `docker-pull` in Privileges
5. Hit `Save`

![Nexus Role Configuration](nexus-role.png){: h="300" }

#### User

In `Security -> Users -> Create local user`:

1. Add ID such as `docker-pull`
2. Add First name such as `Docker`
3. Add Last name such as `Service Account`
4. Add Email and Password
5. Set Status to `Active`
6. Add the previously created role `Docker Pull` in Roles
7. Hit `Create local user` 
8. 
![Nexus User Configuration](nexus-user.png){: h="300" }

## Helm chart

If you are using a Helm chart for Nexus, you can setup a reverse-proxy on the port defined in the previously created docker-group `docker`:

1. Enable service & update `targetPort` with previously defined port [here](https://github.com/sonatype/nxrm3-helm-repository/blob/main/nxrm-aws-resiliency/values.yaml#L95-L100):
```yaml
service:
  docker:
    enabled: true
    targetPort: 8082
```
2. Enable ingress and setup it [here](https://github.com/sonatype/nxrm3-helm-repository/blob/main/nxrm-aws-resiliency/values.yaml#L60C48-L65):
```yaml
ingress:
  dockerIngress:
    enabled: true
    host: image-registry.my.domain
```
3. Redeploy Nexus

## Tests

You should be able to test Docker image pulling from your localhost, and see the pulled image appear in your Nexus repository.

#### Unauthenticated

```bash
docker pull image-registry.my.domain/nginx
```

or 

```bash
docker pull NEXUS_URL:8082/nginx
```

#### Authenticated

```bash
docker login image-registry.my.domain # Previously created user docker-pull for login
docker pull image-registry.my.domain/nginx
```

or 

```bash
docker login NEXUS_URL:8082 # Previously created user docker-pull for login
docker pull NEXUS_URL:8082/nginx
```

## EKS

Now that Nexus is correctly configured, we want our EKS nodes to use Nexus as a Docker repository mirror.

In our Kubernetes cluster in v1.28 (and recent Kubernetes version), Containerd is the default container runtime that handles Docker image pulling.

### Karpenter

To configure Containerd, we will use [Karpenter NodeTemplates userdata](https://karpenter.sh/v0.31/concepts/node-templates/#specuserdata) to inject configuration during node startup (we haven't upgraded Karpenter yet, but it is named `EC2NodeClass` since v0.32).

1. Add this configuration to your `NodeTemplate`:
```yaml
userData: |
  #!/bin/bash
  mkdir -p /etc/containerd/certs.d/docker.io

  sudo tee -a /etc/containerd/certs.d/docker.io/hosts.toml > /dev/null <<EOT

  server = "https://registry-1.docker.io"

  [host."https://image-registry.my.domain"]
    capabilities = ["pull", "resolve"]

    # For authenticated user
    [host."https://image-registry.my.domain".header]
    authorization = "Basic XXXXXXXX"                                             # `XXXXXXXX` is `user:password` with base64 encoding

  EOT
```
2. Redeploy Karpenter
3. Launch a rolling-update on your Karpenter `Machines`. 

Containerd should now be able to pull images from Nexus 🎉.

If your Nexus is unavailable, Containerd can still directly use Docker Hub.
{: .prompt-info }

## Documentations

If you need further digging, like using `HTTP` instead of `HTTPS`, there are some useful documentations:
- https://docs.docker.com/docker-hub/mirror/
- https://github.com/containerd/containerd/blob/main/docs/hosts.md#setup-a-local-mirror-for-docker

## Final words

If you want to set up additional Docker repository mirror for `gcr.io`, `quay.io`, you should be able to do it easily by:
- In Nexus, adding a new docker-proxy repository
- In Nexus, updating the docker-group repository with docker-proxies
- In Karpenter, creating a new config file, such as `/etc/containerd/certs.d/gcr.io/hosts.toml` and updating `server` url

I hope this short guide helped you.
