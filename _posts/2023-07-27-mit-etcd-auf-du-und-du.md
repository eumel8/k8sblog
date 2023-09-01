---
layout: post
tag: de
title: "Mit etcd auf Du und Du"
subtitle: "Kubernetes Daten aus einem etcd Backup wiederherstellen. Klingt nicht so erstrebenswert, war aber kürzlich notwendig, weil jemand alte CRDs gelöscht hat *hüstel*"
date: 2023-07-27
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

Nach erfolgreichem Update und Migration könnte man der Meinung sein, man brauch alte CRDs nicht mehr, wie etwa `projectalertrules.management.cattle.io`.
Im Rancher Cluster sind sie schnell gelöscht, aber Rancher beschwert sich im weiteren Betrieb, dass die Resource fehlen ... die man eigentlich nicht mehr brauch. Also gut, soll er seine CRDs wieder haben. Am besten von einem anderen Cluster mit gleicher Konfiguration, oder aus dem Backup.


# RKE etcd snapshots
Die [etcd](https://etcd.io/) ist quasi *der* key-value store für die Kubernetes-Welt. Er ist klein, schlank und läuft auch im Container.
Im Prinzip gehts auch nur drum, einfache Daten zu verwalten und das möglichst hochredundant - also im etcd Cluster mit mindestens 3 Nodes, also Container.
[Rancher Kubernetes Engine RKE](https://www.rancher.com/products/rke) ist wiederum ein Werkzeug, um einfach Kubernetes Cluster zu erstellen. Mit RKE kann man auch etcd snapshots erstellen, die als Backup für einen Cluster dienen. Als Ergebnis hat man alles in einer Datei, etwa `SNAPSHOT-20230707_1409.db.zip`. Wenn man diese auspackt, erhält man zum einen das RKE-Statefile, eine Datei,die den Status des RKE Clusters zum Zeitpunkt des Backups beschreibt. Und eine `SNAPSHOT-20230707_1409.db` Datei. Die ist riesengross und ist quasi DIE etcd. Die Daten sind binär, also wie da rankommen?

# etcd dev Umgebung
Normalerweise macht man den Restore der etcd auf dem System, wo es eigentlich läuft und dort dann direkt in die laufende Instanz. Wir wollen aber
nur ein paar Daten rausholen, nämlich unser gelöschtes CRD. Im Internet gabs [diese Anleitung](https://neilcameronwhite.medium.com/partial-etcd-recovery-on-openshift-kubernetes-7909da28867c). Fangen wir also mal an.

Wir haben die Backupdatei auf einen frischen Ubuntu 20.04 LXD Host kopiert und nach `apt install unzip` ausgepackt.

Jetzt brauchen wir noch die etcd selber.

```bash
apt install etcd-server
snap install yq etcd
```

Es gibt da Unterschiede zwischen den (veralteten) Ubuntu Packages und den Packages von Snap. Deswegen die 2 Varianten, um Client und Server zu installieren.

Aber jetzt können wir schon das Restore anstossen:

```bash
/snap/etcd/current/bin/etcdctl snapshot restore /restore/backup/SNAPSHOT-20230707_1409.db --data-dir=/var/lib/default.etcd
```

Und die etcd starten:

```bash
/snap/etcd/current/bin/etcd --name default --listen-client-urls http://localhost:2379 --advertise-client-urls http://localhost:2379 --listen-peer-urls http://localhost:2380
```

da sollte dann was laufen und lauschen:

```
2023-07-27 18:08:00.288427 I | etcdserver/api: enabled capabilities for version 3.4
2023-07-27 18:08:00.290856 I | etcdserver: published {Name:default ClientURLs:[http://localhost:2379]} to cluster cdf818194e3a8c32
2023-07-27 18:08:00.291874 I | embed: listening for peers on 127.0.0.1:2380
2023-07-27 18:08:00.292100 I | embed: ready to serve client requests
2023-07-27 18:08:00.293225 N | embed: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
```

In einem anderen Fenster kann man dann mit dem Client die etcd abfragen:


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

So bekommt man schon mal einen Eindruck vom Aufbau der Datenbank. Es geht einfach nach API-Gruppen und Erweiterungen. Die CRDs stehen bei a wie apiextenstions und so können wir schnell unsere fehlende raussuchen:


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

Könnte man jetzt gleich in den Cluster wieder eindeployen. Leider gibts die API-Version gar nicht mehr. Also müssen wir das File etwas anpassen:


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

Die Felder `validation` und `version` gibts nicht mehr bzw. wird daraus das `schema` unter der Version. Und die API Version ist natürlich anders.

Vielleicht doch noch mal der Vergleich:

alt:

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

neu:

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

Die lässt sich dann auf den Cluster deployen und das Leben geht weiter:

```bash
% kubectl apply -f crd.yaml
customresourcedefinition.apiextensions.k8s.io/projectalertrules.management.cattle.io created
```
