---
title: Docker mirroring in EKS using Karpenter and Nexus
date: 2023-11-14 10:50:32 +0100
categories: [Kubernetes, Containerd]
tags: [kubernetes, containerd, eks, karpenter, nexus]
author: t_falconnet
img_path: /assets/posts/2023-11-14-docker-mirroring-in-eks
---

## Context

We have been preparing for several months at Energy Pool a migration to Kubernetes. We currently have most of our workload on AWS EC2, and we now have an EKS cluster in our development environment.

During [Karpenter](https://karpenter.sh/) installation and deployment, we quickly hit the [Docker Hub rate limit](https://docs.docker.com/docker-hub/download-rate-limit/) by pulling too much images: we were terminating our nodes in loop for testing purpose.

To overcome this issue, there are different possibilities:
- Pay a Docker subscription: we wasn't interested.
- Stop using public images: using a solution like [Skopeo](https://github.com/containers/skopeo) which I knew from a previous mission.
- Cache image using a mirror server: theoricaly, we knew it was possible, but we had no experience around that.

So I started to implement the Skopeo solution. It has a few drawbacks:
- you have to continuously list all public images used.
- you have to update all projects to use corresponding synchronized image from your private repository. 


[Containerd Mirror configuration]https://github.com/containerd/containerd/blob/main/docs/hosts.md#setup-a-local-mirror-for-docker
