---
author: "parsec"
date: 2022-01-24
linktitle: Building a Kubernetes Cluster with k8s@Home - Pt. 00
type:
  - post
  - posts
title: Building a Kubernetes Cluster with k8s@Home - Pt. 00
subtitle: Making my Home Cluster Repo Public!
weight: 10
series:
  - "The Great Kubernetes Migration!"
tags:
  - k3s
  - kubernetes
  - flux
  - gitops
---

---

## What's happening?!

It's time! Time to migrate my cluster to something *even more exciting*! I've got a couple new nodes, I've got some motivation, and some time on my hands!

Following the [k8s@home template cluster](https://github.com/k8s-at-home/template-cluster-k3s), I'm going to be migrating my cluster to a public format so that it's viewable online, and also secure!

We'll be migrating my app deployments from my personal cluster, which I had running on a Hades Canyon NUC, to 3 nodes:

- Dell PowerEdge R720 (Worker Node)
- Cisco UCS C220 M3 (Worker Node)
- Hades Canyon NUC (Control Node)

I'll be adding content from the k8s@Home readme below for their template cluster, so you can see what we'll be doing! I'll be following this initially to a T, so this is accurate!

## Prerequisites

### :computer:&nbsp; Systems

- One or more nodes with a fresh install of [Ubuntu Server 20.04](https://ubuntu.com/download/server). These nodes can be bare metal or VMs.
- A [Cloudflare](https://www.cloudflare.com/) account with a domain, this will be managed by Terraform.
- Some experience in debugging problems and a positive attitude ;)

### :wrench:&nbsp; Tools

:round_pushpin: You should install the below CLI tools on your workstation. Make sure you pull in the latest versions.

#### Required

| Tool                                               | Purpose                                                                                                                                 |
|----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| [ansible](https://www.ansible.com)                 | Preparing Ubuntu for Kubernetes and installing k3s                                                                                      |
| [direnv](https://github.com/direnv/direnv)         | Exports env vars based on present working directory                                                                                     |
| [flux](https://toolkit.fluxcd.io/)                 | Operator that manages your k8s cluster based on your Git repository                                                                     |
| [age](https://github.com/FiloSottile/age)          | A simple, modern and secure encryption tool (and Go library) with small explicit keys, no config options, and UNIX-style composability. |
| [go-task](https://github.com/go-task/task)         | A task runner / simpler Make alternative written in Go                                                                                  |
| [ipcalc](http://jodies.de/ipcalc)                  | Used to verify settings in the configure script                                                                                         |
| [jq](https://stedolan.github.io/jq/)               | Used to verify settings in the configure script                                                                                         |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Allows you to run commands against Kubernetes clusters                                                                                  |
| [sops](https://github.com/mozilla/sops)            | Encrypts k8s secrets with Age                                                                                                           |
| [terraform](https://www.terraform.io)              | Prepare a Cloudflare domain to be used with the cluster                                                                                 |

#### Optional

| Tool                                                   | Purpose                                                  |
|--------------------------------------------------------|----------------------------------------------------------|
| [helm](https://helm.sh/)                               | Manage Kubernetes applications                           |
| [kustomize](https://kustomize.io/)                     | Template-free way to customize application configuration |
| [pre-commit](https://github.com/pre-commit/pre-commit) | Runs checks pre `git commit`                             |
| [gitleaks](https://github.com/zricethezav/gitleaks)    | Scan git repos (or files) for secrets                    |
| [prettier](https://github.com/prettier/prettier)       | Prettier is an opinionated code formatter.               |

### :warning:&nbsp; pre-commit

It is advisable to install [pre-commit](https://pre-commit.com/) and the pre-commit hooks that come with this repository.
[sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit) and [gitleaks](https://github.com/zricethezav/gitleaks) will check to make sure you are not by accident committing your secrets un-encrypted.

After pre-commit is installed on your machine run:

```sh
task pre-commit:init
```
**Remember to run this on each new clone of the repository for it to have effect.**

Commands are of interest, for learning purposes:

This command makes it so pre-commit runs on `git commit`, and also installs environments per the config file.
```
pre-commit install --install-hooks
```
This command checks for new versions of hooks, though it will occasionally make mistakes, so verify its results.
```
pre-commit autoupdate
```

## Let's go!

For the rest of the initial setup, I'm following the README.md on [k8s@home's Template Cluster](https://github.com/k8s-at-home/template-cluster-k3s/blob/main/README.md). Feel free to follow along!

The next post will be about my infrastructure-specific setup, as well as getting everything up and running!
