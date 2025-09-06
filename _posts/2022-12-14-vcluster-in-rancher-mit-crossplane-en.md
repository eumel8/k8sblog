---
layout: post
tag: en
title: "vcluster in Rancher With Crossplane"
subtitle: "In last post we learnt vcluster in an experimental way to implement in Rancher. Now we automate things and extend our services with Crossplane"
date: 2022-12-14
background: '/images/vcluster-rancher.png'
twitter: '/images/vcluster.png'
author: eumel8
---

A very interesting use case is described in [this tutorial to deploy a monolith](https://github.com/salaboy/from-monolith-to-k8s/tree/main/platform/crossplane-vcluster). I started to think about a [vcluster operator](https://github.com/eumel8/vcluster-operator) to deploy vcluster for the masses. In fact the main deploy method of vcluster is a Helm chart (which basically deployed a statefulset with k3s running), and Crossplane has a [Helm Provider](https://github.com/crossplane-contrib/provider-helm).
Crossplane is a tool to extend your Kubernetes API with your own resources, like `kind: vcluster`. You can manage this
resources in the same way like main Kubernetes resources:

```bash
$ kubectl get vclusters.caas.telekom.de
NAME     SYNCED   READY   COMPOSITION                AGE
kunde2   True     True    vcluster.caas.telekom.de   52m
kunde1   True     True    vcluster.caas.telekom.de   24m
```

How to archive this? 

# Preconditions

* <a href="https://k3s.io/">K3S</a> installed on Ubuntu 20.04 VM
* Existing Rancher upstream cluster running Rancher 2.6+
* Generated API token with cluster-wide access
* Installed [Crossplane Helm provider and crossplane cli](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263#file-00_vcluster_crossplane-md)

# Composition

With a [Crossplane Composition](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263#file-01_composition-yaml) we describe, what should happen if one of our new own resources are created.
It's the Crossplane Helm provider, so we define two Helm charts and apply required values. Under circumstances this
values should defined in the resource defintion. On the other hand we have a central way to configure chart versions
and repo.
The first Helm chart will deploy the vcluster. It's the normal official thing.
The [second chart](https://github.com/mcsps/helm-charts/tree/master/charts/rancher-cluster) is basically a batch job
to import the vcluster into Rancher. For this we need the Rancher URL and the API admin token.

# CompositeResourceDefinition

With a [Crossplane CompositeResourceDefinition](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263#file-02_compositeresourcedefinition-yaml) we extend the Kubernetes API with our own stuff. We give them
a name, an api version and can define specs. There is no other things required. Crossplane will manage all
this CRD deployment in the background.

Please refer [Crossplane Documentation](https://github.com/crossplane/crossplane/blob/master/docs/concepts/composition.md)

# Vcluster

At the end it's very simple to [deploy the vcluster](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263#file-03_vcluster-yaml)

And we have vcluster in Rancher:

<img src="/images/2022-12-14.png" width="950" height="330" />

```bash
$ vcluster list

 NAME              NAMESPACE   STATUS    CONNECTED   CREATED                         AGE        CONTEXT
 kunde2-vcluster   kunde2      Running               2022-12-14 16:56:33 +0100 CET   1h14m26s   local
 kunde1-vcluster   kunde1      Running               2022-12-14 17:25:07 +0100 CET   45m52s     local
```

That's all. Here is the [Gist](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263) with required resources:

<script src="https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263.js"></script>

# Conclusion

From security perspective vcluster 1 and vcluster 2 are separated in two namespaces, and if you want
in two different Rancher projects. In this project you can define roles like `vcluster operator` for people
to manage this vcluster stuff. As a project owner this could work out.
For the customer of vcluster 1 you can assign accounts as cluster-owner via Rancher, manually or in a automatic way.
Maybe part 3 of this journey.
