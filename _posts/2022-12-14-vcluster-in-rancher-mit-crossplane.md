---
layout: post
tag: de
title: "vcluster in Rancher mit Crossplane"
subtitle: "Im letzten Beitrag haben wir gelernt, vcluster in einem experimentellen Weg in Rancher zu implementieren. Jetzt automatisieren wir Dinge und erweitern unsere Dienste mit Crossplane"
date: 2022-12-14
background: '/images/vcluster-rancher.png'
twitter: '/images/vcluster.png'
author: eumel8
---

Ein sehr interessanter Anwendungsfall ist in [dieser Anleitung zum Deployen eines Monoliten](https://github.com/salaboy/from-monolith-to-k8s/tree/main/platform/crossplane-vcluster) beschrieben. Ich habe angefangen, über einen [vcluster operator](https://github.com/eumel8/vcluster-operator) zum Deployen von Vcluster für die Massen nachzudenken. Tatsächlich ist die Hauptbereitstellungsmethode von vcluster ein Helm Chart (welches grundsätzlich ein Statefulset deployt, in dem K3s läuft), und Crossplane hat einen [Helm Provider](https://github.com/crossplane-contrib/provider-helm).
Crossplane ist ein Werkzeug zum Erweitern der Kubernetes API mit eigenen Resourcen, wie `kind: vcluster`. Diese Resourcen können genauso verwaltet werden wie Kubernetes Resourcen:

```bash
$ kubectl get vclusters.caas.telekom.de
NAME     SYNCED   READY   COMPOSITION                AGE
kunde2   True     True    vcluster.caas.telekom.de   52m
kunde1   True     True    vcluster.caas.telekom.de   24m
```

Wie erreicht man das?

# Vorbedingungen

* <a href="https://k3s.io/">K3S</a> installiert auf Ubuntu 20.04 VM
* Existierender Rancher Upstream Cluster mit Rancher 2.6+
* Erstellter API Token mit cluster-weitem Zugriff
* Installierter [Crossplane Helm Provider und crossplane cli](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263#file-00_vcluster_crossplane-md)

# Composition

Mit einer [Crossplane Composition](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263#file-01_composition-yaml) beschreiben wir, was passieren soll, wenn eine unserer neuen Resourcen erstellt wird.
Es ist der Crossplane Helm provider, so definieren wir zwei Helm Charts und die erforderlichen Werte. Unter Umständen sollten diese Werte in den Resourcen definiert sein. Auf der anderen Seite haben wir einen zentralen Weg um die Chart Version und das Repo zu definieren.
Das erste Helm Chart wird den vcluster deployen. Es ist das normale offizielle Chart.
Das [zweite Chart](https://github.com/mcsps/helm-charts/tree/master/charts/rancher-cluster) ist ein einfacher
Batch Job zum Importieren des Vcluster in Rancher. Wir brauchen dazu die Rancher URL und den API Admin Token.

# CompositeResourceDefinition

Mit einer [Crossplane CompositeResourceDefinition](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263#file-02_compositeresourcedefinition-yaml) erweitern wir die Kubernetes API mit unserem eigenen Zeug. Wir geben dem einen
Namen, eine API Version und wir können Specs definieren. Es ist nichts anderes erforderlich. Crossplane wird das 
alles im Hintergrund erledigen wie CRDs erstellen usw.

Nachzulesen in der [Crossplane Dokumentation](https://github.com/crossplane/crossplane/blob/master/docs/concepts/composition.md)

# Vcluster

Am Ende ist es sehr einfach, [einen Vcluster zu deployen](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263#file-03_vcluster-yaml)

Und wir haben den Vcluster in Rancher:

<img src="/images/2022-12-14.png" width="950" height="330" />

```bash
$ vcluster list

 NAME              NAMESPACE   STATUS    CONNECTED   CREATED                         AGE        CONTEXT
 kunde2-vcluster   kunde2      Running               2022-12-14 16:56:33 +0100 CET   1h14m26s   local
 kunde1-vcluster   kunde1      Running               2022-12-14 17:25:07 +0100 CET   45m52s     local
```

Das ist alles. Hier ist das [Gist](https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263) with required resources:

<script src="https://gist.github.com/eumel8/c08a17fd259c98f6de832bdcdf87a263.js"></script>

# Zusammenfassung

Vom Sicherheitsstandpunkt sind VCluster 1 und Vcluster 2 in zwei Namespaces separiert. Wenn wir wollen,
können wir noch 2 unterschiedliche Rancher Projekte verwenden. In diesen Projekten kkann man Rollen definieren wie
`vcluster operator` für Leute, die Vcluster betreiben sollen. Als Project Owner sollte das funktionieren.
Für den Kunden des Vcluster 1 kann man seinen Account als Cluster Owner über Rancher hinzufügen, manuell oder automatisiert.
Vielleicht in einem dritten Teil dieser Reihe beschrieben.
