---
author: "parsec"
date: 2022-01-24
linktitle: Let there be storage! - Pt. 02
type:
 - post
 - posts
title: Let there be storage! - Pt. 02
subtitle: Deploying nfs-subdir-external-provisioner
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

## Prerequisites

* Kubernetes cluster managed by FluxCD
* Ingress controller
* Load balancer

Personally, I love using Flux for my Kubernetes GitOps needs. I followed the [k8s@home template cluster](https://github.com/k8s-at-home/template-cluster-k3s), so if you want to see how I got my base cluster set up, check it out! It's a fantastic guide. I'll probably write a post on it at some point.

Before we get started, let's make sure we can hit all our nodes. For this, I use [go-task](https://taskfile.dev/#/) to trigger Ansible to probe my inventory with a ping. You can see more in [my taskfiles folder](https://github.com/parsec/home-cluster/tree/main/.taskfiles).

```shell
❯ task ansible:adhoc:ping
task: [ansible:adhoc:ping] ansible all -i /home/parsec/dev/parsec/home-cluster/provision/ansible/inventory/hosts.yml --one-line -m 'ping'
aegis | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"},"changed": false,"ping": "pong"}
omega | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"},"changed": false,"ping": "pong"}
alpha | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"},"changed": false,"ping": "pong"}
╭─    ~/dev/parsec/home-cluster  on   main ································································ ✔  ▼  at 21:45:45  
╰─
```

Awesome! My nodes are up, and now we're ready to start deploying things. 

## Deploying a Storage Provisioner

So, obviously, for a home media stack, we need some way to access storage.

I have a NAS, so the easiest way to get up and running is by using [kubernetes-sigs/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner). So let's do that!

### Consuming the `HelmRepository` w/ Flux's `SourceController`

With Flux, the way we deploy applications is via a `HelmRelease`, which is consumed by the `HelmController` to then deploy an app. First we need a source to get the chart from:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
	name: nfs-subdir-external-provisioner-charts
	namespace: flux-system
spec:
	interval: 15m
	url: https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
	timeout: 3m
```

The Flux `SourceController` will consume this file, and add this object as a source we can reference in our `HelmRelease`. Which, we'll write now!

### Writing a `HelmRelease` file

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nfs-subdir-external-provisioner
  namespace: kube-system
  labels:
    kustomize.toolkit.fluxcd.io/substitute: "disabled"
spec:
  interval: 5m
  chart:
    spec:
      # renovate: registryUrl=https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
      chart: nfs-subdir-external-provisioner
      version: 4.0.15
      sourceRef:
        kind: HelmRepository
        name: nfs-subdir-external-provisioner-charts
        namespace: flux-system
      interval: 5m
  install:
    createNamespace: true
  values:
    replicaCount: 3
    nfs:
      server: "10.1.0.3"
      path: "/mnt/Data/PVCs"
      mountOptions:
        - noatime
    storageClass:
      defaultClass: false
      pathPattern: "${.PVC.namespace}-${.PVC.name}"
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - nfs-subdir-external-provisioner
              topologyKey: kubernetes.io/hostname
```

Values for the chart can be found [here](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/charts/nfs-subdir-external-provisioner/values.yaml). And now that we've got that going, we can start deploying the media server!

## Profit!

Now that we have an NFS storage provisioner, we can request new NFS PVCs with a format similar to the following:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: app-config-v1
  namespace: media
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

We're using the default `storageClassName` here, `nfs-client`. This will request a PV from `nfs-subdir-external-provisioner` and give us some storage on our NFS server!
