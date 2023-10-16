---
layout: post
tag: en
title: Kubernetes Command Line Tools
subtitle: How to work with Kubernetes on a daily basis
date: 2023-08-18
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Kubectl
kubectl is the ultimate tool for working with Kubernetes. It can be installed as a package for different distributions, or as a single file. Details on the [Kubernetes website](https://kubernetes.io/de/docs/tasks/tools/install-kubectl/) . To connect to a Kubernetes cluster, you need credentials, which you can get from your provider as a KUBECONFIG file. This includes the endpoint of the Kubernetes cluster and the credentials, either a certificate, password or token. The operation is intuitive for exploring the Kubernetes resources or creating them.

# Kubectl-Plugins
If the functionality of kubectl is not enough for you, you can have it expanded with plugins. [Krew](https://krew.sigs.k8s.io/plugins/) is a plugin manager. You can also download and use plugins manually. Here is a selection:

## ketall

[https://github.com/corneliusweig/ketall](https://github.com/corneliusweig/ketall)

kubectl get-all (ketall) list really all resources in the cluster or a namespace, which you don't see with kubectl.

## kubectl-images

[https://github.com/chenjiandongx/kubectl-images](https://github.com/chenjiandongx/kubectl-images)

kubectl images list all container images of a Pod in the cluster or a namespace. Good visible, if you want to know, which registry or repo is in use.

```
$ ./kubectl-images -n demoapp
[Summary]: 1 namespaces, 1 pods, 2 containers and 2 different images
+-------------------------+-----------------------------+--------------------------------------------------------+
|           Pod           |          Container          |                         Image                          |
+-------------------------+-----------------------------+--------------------------------------------------------+
| demoapp-57bf45f76-bgkwb | demoapp                     | mtr.devops.telekom.de/cosigndemo/nginx-non-root:latest |
+                         +-----------------------------+--------------------------------------------------------+
|                         | (init) checkvulnerabilities | mtr.devops.telekom.de/caas/caas-tools:latest           |
+-------------------------+-----------------------------+--------------------------------------------------------+
```

## kuota-calc

[https://github.com/postfinance/kuota-calc](https://github.com/postfinance/kuota-calc)

kuota-calc compute the required resources of a deployment on behalf of the upgrade strategy:

```
$ cat cosignwebhook/manifests/demoapp.yaml | ./kuota-calc  --detailed
Version    Kind          Name       Replicas    Strategy         MaxReplicas    CPU     Memory    
apps/v1    Deployment    demoapp    1           RollingUpdate    2              400m    512Mi     
```


## kubectl-curl

[https://github.com/segmentio/kubectl-curl](https://github.com/segmentio/kubectl-curl)

A plugin to get http endpoints of a Pod with curl.

## kubepug

[https://github.com/rikatz/kubepug](https://github.com/rikatz/kubepug)

find deprecated API endpoints

```
$ ./kubepug 

Deleted APIs:
     APIs REMOVED FROM THE CURRENT VERSION AND SHOULD BE MIGRATED IMMEDIATELY!!
PodSecurityPolicy found in policy/v1beta1
     ├─ Deleted at: 1.25
     ├─ PodSecurityPolicy governs the ability to make requests that affect the Security Contextthat will be applied to a pod and container.Deprecated in 1.21.
```

## kubectl-ai

[https://github.com/sozercan/kubectl-ai](https://github.com/sozercan/kubectl-ai)

ChatGPT Plugin. Ask ChatGPT which project do you want to realize and get the right Kubernetes manifests:

<iframe width="560" height="315" src="https://www.youtube.com/embed/j6lO-zvWVdc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## kubectl-oomd

[https://github.com/jdockerty/kubectl-oomd](https://github.com/jdockerty/kubectl-oomd)

When is a Pod oomkilled:

```
./oomd 
POD                                                   CONTAINER           REQUEST     LIMIT     TERMINATION TIME
cae-feeder-preview-0                                  cae-feeder          2Gi         2Gi       2023-05-02 05:46:12 +0200 CEST
elastic-worker-5d6df7fbb4-lvvd5                       elastic-worker      768Mi       768Mi     2023-07-17 12:26:20 +0200 CEST
replication-live-server-delivery-environment-01-0     content-server      1Gi         1Gi       2023-08-12 13:54:49 +0200 CEST
user-changes-0                                        user-changes        768Mi       768Mi     2023-05-02 20:01:46 +0200 CEST
workflow-server-0                                     workflow-server     768Mi       768Mi     2023-06-22 09:10:44 +0200 CEST
```

A lot, is this right:

```
$ kubectl describe pod workflow-server-0
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Wed, 26 Apr 2023 11:05:19 +0200
      Finished:     Thu, 22 Jun 2023 09:10:44 +0200
```

seems so

With the Release page of the project you can mostly download the program, or install it with Kreq.

Have a lot of fun!
