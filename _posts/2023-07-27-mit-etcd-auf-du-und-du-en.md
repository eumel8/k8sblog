---
layout: post
tag: en
title: "hobnob etcd"
subtitle: "Recover Kubernetes data from an etcd backup. That sounds not so cosy, but was required recently, because someone deleted old CRDs"
date: 2023-07-27
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

After a successful update and migration, you might think that you no longer need old CRDs, such as `projectalertrules.management.cattle.io`.
They are quickly deleted in the Rancher cluster, but as the operation continues, Rancher complains that the resources are missing ... which are actually no longer needed. All right, let him have his CRDs back. Preferably from another cluster with the same configuration, or from a backup.


# RKE etcd snapshots
The [etcd](https://etcd.io/) in fact is *the* key-value store for the Kubernetes world. Hi is small, slim, and runs also in a container.
In principle, it's all about managing simple data and doing it as highly redundant as possible - i.e. in the etcd cluster with at least 3 nodes, i.e. containers.
[Rancher Kubernetes Engine RKE](https://www.rancher.com/products/rke) is a tool to simply create a Kubernetes Cluster. With RKE we can create snapshots for cluster backups. As result there will be all in one file, e.g. `SNAPSHOT-20230707_1409.db.zip`. If we unpack the file, we will have a RKE state file, a file with the status of the RKE cluster in the moment of creating the backup. And a `SNAPSHOT-20230707_1409.db` file. This is really big and THE etcd. The file is binary, how to read this?

# etcd dev environment
Normaly the restore of the etcd will performed on the system, where the etcd is running and the restore will directly pushed in the running instance. But we will restore only few data, the deleted CRD. 
On the Internet exists [this instruction](https://neilcameronwhite.medium.com/partial-etcd-recovery-on-openshift-kubernetes-7909da28867c). Let's beginning with the procedure.

We habve a backup file on a fresh Ubuntu 20.04 LCD Host copied and unpack them after  `apt install unzip`.

Now we need the etcd program itself.

```bash
apt install etcd-server
snap install yq etcd
```

There are different versions between the (outdated) Ubuntu Packages and the Snap packages. Because of the 2 versions, we install for the client and the server.

But now we start the restore:

```bash
/snap/etcd/current/bin/etcdctl snapshot restore /restore/backup/SNAPSHOT-20230707_1409.db --data-dir=/var/lib/default.etcd
```

And start the etcd:

```bash
/snap/etcd/current/bin/etcd --name default --listen-client-urls http://localhost:2379 --advertise-client-urls http://localhost:2379 --listen-peer-urls http://localhost:2380
```

that should be run and listen:

```
2023-07-27 18:08:00.288427 I | etcdserver/api: enabled capabilities for version 3.4
2023-07-27 18:08:00.290856 I | etcdserver: published {Name:default ClientURLs:[http://localhost:2379]} to cluster cdf818194e3a8c32
2023-07-27 18:08:00.291874 I | embed: listening for peers on 127.0.0.1:2380
2023-07-27 18:08:00.292100 I | embed: ready to serve client requests
2023-07-27 18:08:00.293225 N | embed: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
```

In another terminal we can ask the etcd with the client:


```bash
/snap/etcd/current/bin/etcdctl get / --prefix --keys-only|head -20
/registry/apiextensions.k8s.io/customresourcedefinitions/alertmanagerconfigs.monitoring.coreos.com

/registry/apiextensions.k8s.io/customresourcedefinitions/alertmanagers.monitoring.coreos.com

/registry/apiextensions.k8s.io/customresourcedefinitions/amazonec2configs.rke-machine-config.cattle.io

/registry/apiextensions.k8s.io/customresourcedefinitions/amazonec2machines.rke-machine.cattle.io

/registry/apiextensions.k8s.io/customresourcedefinitions/amazonec2machinetemplates.rke-machine.cattle.io

/registry/apiextensions.k8s.io/customresourcedefinitions/apiservices.management.cattle.io

/registry/apiextensions.k8s.io/customresourcedefinitions/apprevisions.project.cattle.io

/registry/apiextensions.k8s.io/customresourcedefinitions/apps.catalog.cattle.io

/registry/apiextensions.k8s.io/customresourcedefinitions/apps.project.cattle.io

/registry/apiextensions.k8s.io/customresourcedefinitions/assign.mutations.gatekeeper.sh
```

You should get an idea from the data structure. It goes simply at is with API groups and extenstions. The CRDs are located by  a how apiextenstions and we can fast get the missing things:


```bash
/snap/etcd/current/bin/etcdctl get /registry/apiextensions.k8s.io/customresourcedefinitions/projectalertrules.management.cattle.io --print-value-only |yq -P
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: projectalertrules.management.cattle.io
  uid: 89cd71f1-d2d8-404e-8071-0528bc216b2e
  generation: 2
  creationTimestamp: "2021-03-18T15:46:08Z"
spec:
  group: management.cattle.io
  version: v3
  names:
    plural: projectalertrules
    singular: projectalertrule
    kind: ProjectAlertRule
    listKind: ProjectAlertRuleList
  scope: Namespaced
  validation:
    openAPIV3Schema:
      type: object
      x-kubernetes-preserve-unknown-fields: true
  versions:
    - name: v3
      served: true
      storage: true
  conversion:
    strategy: None
  preserveUnknownFields: false
status:
  conditions:
    - type: NamesAccepted
      status: "True"
      lastTransitionTime: "2021-03-18T15:46:08Z"
      reason: NoConflicts
      message: no conflicts found
    - type: Established
      status: "True"
      lastTransitionTime: "2021-03-18T15:46:08Z"
      reason: InitialNamesAccepted
      message: the initial names have been accepted
  acceptedNames:
    plural: projectalertrules
    singular: projectalertrule
    kind: ProjectAlertRule
    listKind: ProjectAlertRuleList
  storedVersions:
    - v3
```

Can be easily apply to the cluster. Unfortunatelly the API version doesn't exists anymore. We must change the file:

```bash
% diff crd-old.yaml crd-new.yaml
3c3
< apiVersion: apiextensions.k8s.io/v1beta1
---
> apiVersion: apiextensions.k8s.io/v1
8d7
<   version: v3
15,18d13
<   validation:
<     openAPIV3Schema:
<       type: object
<       x-kubernetes-preserve-unknown-fields: true
20a16,19
>     schema:
>       openAPIV3Schema:
>         type: object
>         x-kubernetes-preserve-unknown-fields: true
```

The fields `validation` and `version` don't exists anymore or move to `schema` in the version. And the API version of course is different.

Another compare:

old:

```yaml
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: projectalertrules.management.cattle.io
spec:
  group: management.cattle.io
  version: v3
  names:
    plural: projectalertrules
    singular: projectalertrule
    kind: ProjectAlertRule
    listKind: ProjectAlertRuleList
  scope: Namespaced
  validation:
    openAPIV3Schema:
      type: object
      x-kubernetes-preserve-unknown-fields: true
  versions:
  - name: v3
    served: true
    storage: true
  conversion:
    strategy: None
  preserveUnknownFields: false
```

new:

```yaml
---
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1
metadata:
  name: projectalertrules.management.cattle.io
spec:
  group: management.cattle.io
  names:
    plural: projectalertrules
    singular: projectalertrule
    kind: ProjectAlertRule
    listKind: ProjectAlertRuleList
  scope: Namespaced
  versions:
  - name: v3
    schema:
      openAPIV3Schema:
        type: object
        x-kubernetes-preserve-unknown-fields: true
    served: true
    storage: true
  conversion:
    strategy: None
  preserveUnknownFields: false
```

This can be apply to the cluster and the life is go on:

```bash
% kubectl apply -f crd.yaml
customresourcedefinition.apiextensions.k8s.io/projectalertrules.management.cattle.io created
```
