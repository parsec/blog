---
author: "parsec"
date: 2023-02-19
linktitle: "Keel: What it is, and what it isn't"
type:
  - post
  - posts
title: "Keel: What it is, and what it isn't"
weight: 10
tags:
  - k3s
  - kubernetes
  - gitops
---

---

## GitOps for Kubernetes

There are a lot of options when it comes to managing the state of your Kubernetes cluster: ArgoCD, FluxCD, Helm, Terraform... the list goes on, and you'd be hard-pressed to pick one over the other. After all, they all do things in their own special way.

We're looking at a project I haven't heard of before, [Keel](https://keel.sh/).

## What's the deal with Keel?

My buddy fraq mentioned that they were now using Keel at work, so I thought I'd give it a shot. They have a nice website, reasonably good looking docs, and frankly I like the idea of not having to have a CLI tool to perform operations. But it's important to note that Keel doesn't really solve the "cluster described by GitOps" problem. It seems to be strictly for managing manifest versions, not entire cluster state.

## Let's set up Keel!

So, I started [following the setup guide](https://keel.sh/docs/#deploying-with-helm) to deploy with Helm. Something to note here: out of the box, Keel supports Helm v2. You can turn on Helm v3 support by setting the `helmProvider.version` value when you install:

```shell
$ helm install keel keel/keel --set helmProvider.version="v3"

Release "keel" does not exist. Installing it now.
NAME: keel
LAST DEPLOYED: Sun Feb 19 03:08:52 2023
NAMESPACE: keel
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. The keel is getting provisioned in your cluster. After a few minutes, you can run the following to verify.

To verify that keel has started, run:

  kubectl --namespace=keel get pods -l "app=keel"
```

So far so good. Let's see if we can see the pod after a few minutes.

```shell
$ k -n keel get po -l "app=keel"

NAME                   READY   STATUS    RESTARTS   AGE
keel-fd9fc9f66-xzvxx   1/1     Running   0          14m
```

Cool! We've got a Keel pod! Now let's do some more setup...

## The Iceberg

Let's check out what the pod is up to.

```shell
$  k -n keel logs keel-fd9fc9f66-xzvxx

E0219 08:25:23.237544       1 reflector.go:123] pkg/mod/k8s.io/client-go@v0.16.10/tools/cache/reflector.go:96: Failed to list *v1beta1.CronJob: the server could not find the requested resource
E0219 08:25:24.238663       1 reflector.go:123] pkg/mod/k8s.io/client-go@v0.16.10/tools/cache/reflector.go:96: Failed to list *v1beta1.CronJob: the server could not find the requested resource
E0219 08:25:25.239497       1 reflector.go:123] pkg/mod/k8s.io/client-go@v0.16.10/tools/cache/reflector.go:96: Failed to list *v1beta1.CronJob: the server could not find the requested resource
E0219 08:25:26.240414       1 reflector.go:123] pkg/mod/k8s.io/client-go@v0.16.10/tools/cache/reflector.go:96: Failed to list *v1beta1.CronJob: the server could not find the requested resource
```

Oh... well, that's not good...

After [some research](https://github.com/keel-hq/keel/issues/698), I discovered that Keel currently doesn't support the latest stable version of Kubernetes, or any version >=1.25.x

This... is really disappointing. So my options at this point to play with Keel are to downgrade my cluster, or hope the maintainers update it sometime soon...

## The Bottom of the Iceberg

Oh. It... would appear the maintainers have [not had time](https://github.com/keel-hq/keel/issues/677) to maintain the app, likely for some time. The last major update was back in 2020, and the app is no longer being actively maintained. The only update since 2020 on the GitHub repo was a chart version bump ~4mo ago.

Not only this, but the site that they recommended [in their docs](https://keel.sh/docs/#deploying-with-kubectl) for generating a Kubernetes manifest *also* [appears defunct](https://github.com/keel-hq/keel/issues/684):

```shell
$ k apply -f https://sunstone.dev/keel\?namespace\=keel\&username\=admin\&password\=admin\&tag\=latest

Unable to connect to the server: dial tcp 34.76.124.36:443: i/o timeout
```

## Conclusion

I could have spent time getting Keel working. Downgraded my version of Kubernetes, done the whole shebang, and written a more in-depth blog post about what Keel is, and what it isn't.

But the reality here is that until the maintainers (or some new maintainers) pick up the project, it's going to quickly become irrelevant as Kubernetes continues to be upgraded and maintained. This is really unfortunate, because I think Keel has a lot of potential. But it's currently too behind the curve to be useful for me. Particularly when I have FluxCD, which solves the "GitOps cluster management" problem for me much more elegantly. For now, I'll stick with my `f***ctl` commands (telling how long ago this was last updated... Flux v2 did away with `fluxctl` and introduced just `flux` as the CLI).

I sincerely hope someone else will carry the torch, or the author will have more time to dedicate in the future. Unfortunately my brain isn't wrinkly enough yet to be able to take on a challenge like this. But I can write some random words on my blog at the end of the universe, because why not?

The End…?
