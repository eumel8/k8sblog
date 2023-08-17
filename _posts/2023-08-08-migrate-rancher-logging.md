---
layout: post
tag: de
title: Migration Rancher Logging
subtitle: Wie migriert man von Rancher Logging zu Kube Logging, und warum
date: 2023-08-08
background: '/images/k8s-cosmos.png'
twitter: 'images/cosignwebhook.png'
author: eumel8
---

# Einstieg

Rancher stellt eine Applikation names "Logging" in seinem Appstore zur Verfügung. 

"Collects and filter logs using highly configurable CRDs. Powered by Banzai Cloud."

In Wahrheit ist es der [Banzai Cloud Logging Operator](https://github.com/kube-logging/logging-operator), adaptiert und angereicher von Rancher mit speziellen Logging-Mechanismen wie RKE Logging oder Logging für K3S.
In diesem Blog Post werden wir erklären, wie man von Rancher Logging zum neuen Logging Operator, bereitgestellt unter der [kube-logging Organisation](https://github.com/kube-logging), direkt migriert.

Der Logging Operator ist ursprünglich ein Projekt erstellt von Banzai Cloud, ein Kubernetes Certified Service Provider beheimatet in Ungard. Das Startup Unternehmen wurde von Cisco übernommen, aber später entschieden sich die ursprünglichen Erfinder eine neue Firma namens Axoflow zu gründen. Es ist weiterdieselben Betreuer und ist ist weiterhin ein Open Source Projekt. Das ist der historische Hintergrund, warum die Custom Resources Definitions weiterhin den Namen haben, `logging.banzaicloud.io`.

# Warum migrieren?

In der Vergangenheit entwickelte Rancher seine eigenen Dienste für Logging und Monitoring, Version 1, kontrolliert vom Upstream Cluster mit dessen eigenen Resourcen und Software. Es konnte nicht von anderen Lieferanten adaptiert werden, weil es nur mit Rancher funktionierte.

In Version 2 forkte Rancher bekannte Open Source Projekte wie Kube Prometheus Stack für Monitoring und Banzai Cloud Logging Operator für Logging. Die Software wurde adaptiert für spezielle Rancher Installations wie RKE1, RKE2, K3S, getestet und versioniert. Am Ende hat man eine stabile Version integriert und lauffähig in Rancher. Nur der Support ist limitert. Wenn man Fragen hat, wird das Problem nur zur Upstream Community weitergeleitet. Diese Integrationen und Tests dauern auch ihre Zeit, während Rancher nicht so viel Energie in diese Nebenprojekte spendet. Nur alle 3-6 Monate wird eine neue Version veröffentlich.

Die Upstream Projekte arbeiten viel schneller, neue Funktionen werden in Tagen oder Wochen bereitgestellt. Und man hat direkt Support von den Verwaltern. Wenn man an neuen Funktionen interessiert ist und nicht so sehr auf die Rancher-spezifischen Sachen achtet, ist man ein Kandidat für die Migration.

# Rancher Logging

Rancher Logging wird mit zwei Helm Charts geliefert,rancher-logging und rancher-logging-crds. In Abhängigkeit werden beide parallel installiert.

Man sucht im Appstore nach Logging:

<img src="/images/rancher-logging-1.png" width="850" height="475" />

Nach der Auswahl bekommt man einige Informationen präsentiert:

<img src="/images/rancher-logging-2.png" width="850" height="475" />

Im nächsten Schritt werden cluster-spezifische Konfigurationen erforscht. Hier K3S:

<img src="/images/rancher-logging-3.png" width="850" height="475" />

Die Ausgabe des Installationsprozesses kann man verfolgen:

<img src="/images/rancher-logging-4.png" width="850" height="475" />

Nach der Erstellung der CRDs wird ein neuer Menüpunkt im Rancher erstellt, Logging:

<img src="/images/rancher-logging-5.png" width="850" height="475" />

Jetzt können Ressourcen verwendet werden wie Logging, Flows, Outputs, um eine Fluentd Instanz einzurichten und alles zu definieren, was die Logs zu ihrem endgültigen Ziel schickt.

# Kube Logging

Um Kube Logging in Rancher zu installieren, brauch man das Helm Chart Verzeichnis mit dem Logging Operator Helm Chart.
Man navigiert zum Apps Menü in Rancher:

<img src="/images/rancher-logging-6.png" width="850" height="475" />

Unter "Repositories" kann man ein neues erstellen. Man wählt einen eigenen Namen und HTTP Ziel mit Adresse des Kube Logging Helm Chart Verzeichnis:

<img src="/images/rancher-logging-7.png" width="850" height="475" />

Wenn der Status gesynct und aktiv ist, sind einige neue Apps sichtbar:

<img src="/images/rancher-logging-8.png" width="850" height="475" />

# CRDs

Mit Helm 3 gibt es weiterhin keine Optionen, um CRDs zu aktualisierens. Die ganze Geschichte ist in [hip-0011]({https://github.com/helm/community/blob/main/hips/hip-0011.md) beschrieben. Für die Migration gibt es 2 Möglichkeiten:

## Löschen/Neuerstellung

Eine Deinstallation der rancher-logging und rancher-logging-crd Chart wird automatisch die CRDs löschen und damit alle abhängigen Custom Resources. Also Obbacht, zum Beispiel in rancher-monitoring-crd sind Jobs implemengtiert mit `annotations."helm.sh/hook": pre-delete`, welcher alle CRDs löscht, wenn das Chart gelöscht wird. Beste Idee ist, ein Backup aller Custom Resources zu machen bevor man anfängt:

```bash
% for i in `kubectl api-resources --api-group='logging.banzaicloud.io' -o name`; do kubectl get $i -A -o yaml > $i.yaml;done
% for i in `kubectl api-resources --api-group='logging-extensions.banzaicloud.io' -o name`; do kubectl get $i -A -o yaml > $i.yaml;done
```

Danach können sicher alle CRDs vom Cluster gelöscht werden:

```bash
% for i in `kubectl api-resources --api-group='logging.banzaicloud.io' -o name`; do kubectl delete crd $i;done
% for i in `kubectl api-resources --api-group='logging-extensions.banzaicloud.io' -o name`; do kubectl delete crd $i;done
```

Die Installation des kube-logging Chart wird die neuen CRDs installieren. Danach können die Resourcen vom Backup wieder hergestellt werden.

## Aktualisieren

Aktualisieren von CRDs sind oft manuelle Prozesse. Solange die Version vom CRD `alpha` oder `beta` ist, kann derselbe CRD sehr oft mit derselben Version geändert werden. Ein Cluster Administrator kann dies auf folgendem Weg tun:

```bash
% git clone https://github.com/kube-logging/logging-operator.git
% cd logging-operator
% git checkout 4.2.3 # or the target Helm chart version what you want to install
% cd charts/logging-operator
% kubectl apply -f crds/ 
...
Warning: resource customresourcedefinitions/loggings.logging.banzaicloud.io is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.

The CustomResourceDefinition "loggings.logging.banzaicloud.io" is invalid: metadata.annotations: Too long: must have at most 262144 bytes
```

Das schlägt oft fehl wegen der limitierten Grösse von client-side metadata.annotations, in dem der komplette Inhalt von CRDs gehalten wird.


Ein besserer Ansatz ist die Benutzung des server-side Flag:

```bash
% kubectl apply -f crds/ --server-side --force-conflicts
```

# Installation Kube-Logging

Hier fährt man fort mit der Auswahl der logging-operator App im Appstore:

<img src="/images/rancher-logging-9.png" width="850" height="475" />

Üblicherweise kann man den Ziel Namespace auswählen und der App einen Namen geben.

Man aktiviert vor dem Start `Customize Helm options before start`:

<img src="/images/rancher-logging-10.png" width="850" height="475" />

Die geladene values.yaml zeigt alle Konfigurationsptionen, man klickt `Next`, wenn die Standardeinstellungen okay sind:

<img src="/images/rancher-logging-11.png" width="850" height="475" />

Sicherstellen, dass `Apply custom resource definition` aktiviert ist:

<img src="/images/rancher-logging-12.png" width="850" height="475" />

Nach der Installation sieht man dasselbe `Logging` Menü in Rancher und kann starten mit der Einrichtung von Logging, Flows und Outputs.
Getestet mit Rancher 2.7.5.

Fröhliches Logging!
