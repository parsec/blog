---
author: "Seth Beckman"
date: 2021-05-28
linktitle: Deploying the ELK ECK Operator with fluxCD
type:
  - post
  - posts
title: Deploying the ELK ECK Operator with fluxCD
weight: 10
series:
  - Kubernetes, fluxCD, and ELK, Oh My!
tags:
  - elk
  - k3s
  - kubernetes
  - flux
  - gitops
---

---

## Prerequisites

Before we can begin, we need to have a Kubernetes cluster setup with the FluxCD GitOps Toolkit. The easiest way to do this is with the `flux` cli tool using `flux bootstrap`. Install this with your favorite package manager for your distro or OS-- for me that's Homebrew on macOS. I'm also assuming here you already have a Kubernetes cluster, and that you have your kubeconfig file set up to access the cluster remotely. I used `k3s`(https://k3s.io/) and `k3sup` (https://github.com/alexellis/k3sup) for this.

First a prerequisite, `gcc`:


```bash
brew install gcc
```

Next we'll install the Flux CLI tool:

```bash
brew install fluxcd/tap/flux
```

Now we'll check our cluster to confirm it satisfies the flux prerequisites:

```bash
$ flux check --pre
► checking prerequisites
✔ kubectl 1.18.3 >=1.18.0
✔ kubernetes 1.18.2 >=1.16.0
✔ prerequisites checks passed
```

Now we can bootstrap our cluster. Here's an example of a default bootstrap command:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

For a much, much more complete guide, check out the [flux2 getting started guide](https://toolkit.fluxcd.io/get-started/).

## Creating a `GitRepository` Resource

For this example I created a basic `GitRepository` source, which the [Source Controller](https://toolkit.fluxcd.io/components/source/controller/) uses to create an artifact that can be referenced later.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: eck-operator
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/elastic/cloud-on-k8s
  ref:
    branch: master
  ignore: |
    # exclude all
    /*
    
	 # include eck-operator helm chart directory
    !/deploy/eck-operator
```

**Important things to note:**

* `name` defines the name of the SourceRef artifact.
* `namespace` defines the namespace it will operate in. The flux docs use the `default` or `flux-system` namespace, but I decided to use the `monitoring` namespace for the ECK operator and the rest of the Elastic cluster. 
* `spec.interval` defines how often the Source Controller checks for repository updates...
* and `spec.url` defines the repository. For a `GitRepository` resource, this is the repo URL for whichever Helm chart you want to deploy. **This must be the root of the repository or it will not work.** Read more below about how I figured this out...
* `spec.ignore` works kind of like a `.gitignore` file. In this case, we can't directly pick the subdirectory of the repo that contains the Helm charts and values, so we exclude everything except that directory.

## Creating a `HelmRelease` Resource

For this example, I created a fairly basic `HelmRelease` resource, which references the artifact created by the SourceController to apply the Helm chart.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: eck-operator
  namespace: flux-system
spec:
  interval: 5m
  chart:
    spec:
      chart: ./deploy/eck-operator
      sourceRef:
        kind: GitRepository
        name: eck-operator
        namespace: flux-system
      interval: 1m
```

**Important things to note:**

* Similar to before, `name` defines the name of the HelmRelease, `namespace` defines the namespace it resides in.
* Here, `spec.interval` actually references the interval at which we reconcile the HelmRelease.
* `chart.spec.chart` in this case refers to the relative path of the Helm chart in the Git repo (in this case, it's simply `Chart.yaml` but the chart could be stored in a number of ways or in a different folder on the repo).
* `chart.spec.sourceRef` defines what SourceRef the HelmRelease should pull from. In this case we're referencing our `GitRepository` `eck-operator` SourceRef in the `monitoring` namespace.
* `chart.spec.interval` defines how often we check the Source (our `GitRepository` Source) for updates. Default is the previously defined `HelmReleaseSpec.Interval`.

Great! So now we can can run a quick command to watch our HelmRelease deploy and...

```bash
$ flux get helmreleases --all-namespaces
NAMESPACE   NAME   READY   MESSAGE
flux-system  eck-operator False   HelmChart 'flux-system/flux-system-eck-operator' is not ready
```

**Oh.**

## Problems Afoot...

So I came across a problem that was [discussed in the FAQ](https://toolkit.fluxcd.io/faq/#how-to-debug-not-ready-errors) in the flux docs, and had to start troubleshooting that.

The error that I'd gotten meant that my HelmChart artifact wasn't even getting pulled. What do you know, the docs had me covered, because when I checked my sources I figured out that it wasn't able to clone the repository...

```bash
$ flux get sources git --all-namespaces 
NAMESPACE        NAME      READY     MESSAGE
flux-system    eck-operator False   failed to clone repo: repository not found
```

Turned out that, initially, I was calling the repo incorrectly (I tried to clone a subdirectory and not the main repo) which *doesn't work*. Doh. I fixed that in the example above so that this didn't become an insanely long post with me re-pasting corrections and the like... 

So, I corrected that and then my source was pulling correctly. Hurray! But the problems don't end there... Now the HelmRelease failed to install. I found [this GitHub issue](https://github.com/fluxcd/helm-controller/issues/174) that referenced the same issue I had, however this was not the solution, as my pods weren't showing that everything was okay... Instead, when I ran `kubectl get pods --all-namespaces` I discovered that my `elastic-operator` pod had a status of `ImagePullBackOff`, which seems to indicate an error with the image.

## On to Troubleshooting

So, to delve into that `ImagePullBackOff` error, we need to do some troubleshooting. The quickest way to figure out what's going on is probably with `kubectl describe`, so we're going to use that to figure out what the hell happened.

First, we'll get all our pods so we know which one we want:

```bash
❯ kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS             RESTARTS   AGE
kube-system   helm-install-traefik-mg7qt                0/1     Completed          0          19d
kube-system   metrics-server-86cbb8457f-6z7ks           1/1     Running            1          19d
kube-system   svclb-traefik-cch6f                       2/2     Running            2          19d
kube-system   local-path-provisioner-5ff76fc89d-n5964   1/1     Running            1          19d
kube-system   coredns-854c77959c-m5k8h                  1/1     Running            1          19d
flux-system   kustomize-controller-5dd4d4fd4f-hfn9t     1/1     Running            1          19d
default       podinfo-8574699f75-zwvmq                  1/1     Running            1          19d
flux-system   helm-controller-6d4885f6d8-xg5s6          1/1     Running            1          19d
default       podinfo-8574699f75-tx75b                  1/1     Running            1          19d
flux-system   notification-controller-f9d655df7-mzlpg   1/1     Running            1          19d
kube-system   traefik-6f9cbd9bd4-8m7nv                  1/1     Running            1          19d
flux-system   source-controller-6b4d8df7f7-m8kvp        1/1     Running            1          19d
flux-system   elastic-operator-0                        0/1     ImagePullBackOff   0          10d
```

Alright, so our namespace is `flux-system` and our pod is `elastic-operator-0`. Let's describe that and see what happens.

```yaml
❯ kubectl describe -n flux-system pod/elastic-operator-0
Name:         elastic-operator-0
Namespace:    flux-system
Priority:     0
Node:         aegis/10.0.0.226
Start Time:   Sat, 03 Apr 2021 01:01:52 -0400
Labels:       app.kubernetes.io/instance=eck-operator
              app.kubernetes.io/name=elastic-operator
              controller-revision-hash=elastic-operator-545f64d76c
              statefulset.kubernetes.io/pod-name=elastic-operator-0
Annotations:  checksum/config: 6d88f163db95affe7f7652089d2d428f9c82812a983086702e9fb551f4bf7a26
              co.elastic.logs/raw:
                [{"type":"container","json.keys_under_root":true,"paths":["/var/log/containers/*${data.kubernetes.container.id}.log"],"processors":[{"conv...
Status:       Pending
IP:           10.42.0.25
IPs:
  IP:           10.42.0.25
Controlled By:  StatefulSet/elastic-operator
Containers:
  manager:
    Container ID:
    Image:         docker.elastic.co/eck/eck-operator:1.6.0-SNAPSHOT
    Image ID:
    Port:          9443/TCP
    Host Port:     0/TCP
    Args:
      manager
      --config=/conf/eck.yaml
      --distribution-channel=helm
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Limits:
      cpu:     1
      memory:  512Mi
    Requests:
      cpu:     100m
      memory:  150Mi
    Environment:
      OPERATOR_NAMESPACE:  flux-system (v1:metadata.namespace)
      POD_IP:               (v1:status.podIP)
      WEBHOOK_SECRET:      elastic-operator-webhook-cert
    Mounts:
      /conf from conf (ro)
      /tmp/k8s-webhook-server/serving-certs from cert (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from elastic-operator-token-5pnbc (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  conf:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      elastic-operator
    Optional:  false
  cert:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  elastic-operator-webhook-cert
    Optional:    false
  elastic-operator-token-5pnbc:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  elastic-operator-token-5pnbc
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason   Age                    From     Message
  ----     ------   ----                   ----     -------
  Warning  Failed   35m (x63814 over 10d)  kubelet  Error: ImagePullBackOff
  Normal   BackOff  52s (x63970 over 10d)  kubelet  Back-off pulling image "docker.elastic.co/eck/eck-operator:1.6.0-SNAPSHOT"
```

Oh, *interesting*. We appear to be pulling an extremely recent snapshot image from Elastic, which, if the `ImagePullBackOff` error is any indicator, might just *not exist*. Well, let's just try using a stable build then. Normally with a `HelmRepository` we could pick which version of the chart we want to use (or exclude alpha/snapshot/etc releases) but in this case since we're using a `GitRepository` source... we can change the branch to `'1.5'`.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: eck-operator
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/elastic/cloud-on-k8s
  ref:
    branch: '1.5'
  ignore: |
    # exclude all
    /*
    
	 # include eck-operator helm chart directory
    !/deploy/eck-operator
```

And now flux should reconcile the...

```
❯ kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS             RESTARTS   AGE
...
flux-system   elastic-operator-0                        0/1     ImagePullBackOff   0          10d
```

What the hell? Okay, fine. Spit in my face. Let's try the brute-force option to make it redeploy...

`kubectl remove -n flux-system pod/elastic-operator-0`

Alright, now what?

```bash
❯ kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS             RESTARTS   AGE
...
flux-system   elastic-operator-0                        0/1     ContainerCreating   0          5s
```

Oh shit that's promising! Let's give it a few minutes and...

```bash
❯ kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
...
flux-system   elastic-operator-0                        1/1     Running     1          50s
```

Success!

## Now What?

Well, now we've successfully deployed the ECK Operator, on k3s, via a Helm chart, on the Elastic git repo, with flux. Pretty cool, right?! Well, what can we do now? Now, we can start deploying Elastic applications via the ECK Operator! Those will probably be yet more blog posts though...
