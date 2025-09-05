---
layout: post
tag: de
title: Cluster API mit Vcluster
subtitle: Kubernetes Cluster erstellen mit Kubernetes Cluster
date: 2024-06-09
background: '/images/cluster-api-vcluster.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

Heute beschäftigen wir uns mal mit der Härtesten Kategorie in Kubernetes, der Cluster-Erstellung. Da gibt es von der Community schon den Standard kubeadm, aber für die Verwaltung mehrerer Cluster auch den neuen Dienst [Cluster API](https://cluster-api.sigs.k8s.io) 

# Vorbereitungen

Cluster API erstell und verwaltet Kubernetes Cluster. Dazu brauchts es selber erstmal einen Cluster, der die Informationen der Downstream Cluster vorhält und die daraus notwendigen Arbeiten in einem Operator durchführt. Die Downstream Cluster können mittels Provider in verschiedenen Umgebungen installiert werden, etwa mit `clusterctl init --infrastructure gcp` in der Google Cloud Plattform. Eine vollständige Liste gibt es [hier](https://cluster-api.sigs.k8s.io/reference/providers).
Wenn man keine Public Cloud oder VMware benutzt, erscheint mir der [K3S Provider](https://github.com/k3s-io/cluster-api-k3s/) am Vielversprechensten. Als klassischer [Quickstart](https://cluster-api.sigs.k8s.io/user/quick-start.html) wird `kind` empfohlen (Kubernetes in Docker), wir wollen uns mit [Vcluster Quickstart](https://github.com/loft-sh/cluster-api-provider-vcluster/blob/main/docs/quick-start.md) versuchen.

Wir brauchen einen Rechner mit Ubuntu 22.04 und einige Kommandozeilenwerkzeuge:

```bash
apt update
apt install docker-ce
curl -Lo /usr/local/bin/kind https://github.com/kubernetes-sigs/kind/releases/download/v0.23.0/kind-linux-amd64
chmod +x /usr/local/bin/kind
curl -Lo /usr/local/bin/vcluster https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64
chmod +x /usr/local/bin/vcluster
curl -Lo /usr/local/bin/clusterctl https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.7.2/clusterctl-linux-amd64
chmod +x /usr/local/bin/clusterctl
curl -Lo /usr/local/bin/kubectl "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x /usr/local/bin/kubectl 
```

# Cluster Erstellen

Zuerst erstellen wir den Upstream-Cluster, oder auch Bootstrap-Cluster/Management-Cluster genannt. Dieser soll als Docker-Container mit `kind` erstellt werden:

```bash
kind create cluster
```

Als nächstes initialisieren wir den Management-Cluster mit dem Vcluster Provider:

```bash
clusterctl init --infrastructure vcluster
```

Wenn es dabei keine Fehlermeldung gab, können wir überprüfen, ob der Cluster API Operator mit allen Komponenten läuft:

```bash
# kubectl get pod -A| grep capi
capi-kubeadm-bootstrap-system          capi-kubeadm-bootstrap-controller-manager-85cd65d6c5-fwjjn        1/1     Running   0          2m39s
capi-kubeadm-control-plane-system      capi-kubeadm-control-plane-controller-manager-f8d8578c8-9cqck     1/1     Running   0          2m38s
capi-system                            capi-controller-manager-7f49f575d4-xxcp9                          1/1     Running   0          2m39s
```

Alsdann können wir den ersten Downstream-Cluster erstellen:

```bash
export CLUSTER_NAME=kind
export CLUSTER_NAMESPACE=vcluster
export KUBERNETES_VERSION=1.23.4 
export HELM_VALUES="service:\n  type: NodePort"

kubectl create namespace ${CLUSTER_NAMESPACE}
clusterctl generate cluster ${CLUSTER_NAME} \
    --infrastructure vcluster \
    --kubernetes-version ${KUBERNETES_VERSION} \
    --target-namespace ${CLUSTER_NAMESPACE} | kubectl apply -f -
```

Kontrolle:

```bash
# vcluster list

    NAME |  CLUSTER  | NAMESPACE | STATUS  | VERSION | CONNECTED |            CREATED            | AGE | DISTRO
  -------+-----------+-----------+---------+---------+-----------+-------------------------------+-----+---------
    kind | kind-kind | vcluster  | Running | 0.11.1  |           | 2024-06-09 20:31:21 +0000 UTC | 19s | OSS
```

Das `clusterctl generate` generiert tatsächlich nur die Manifeste, die dann mit `kubectl apply -f` zum Management-Cluster geschickt werden. Schauen wir mal in so ein Manifest rein.

vcluster2.yaml:

```yaml
kind: Cluster
metadata:
  name: kind
  namespace: vcluster2
spec:
  controlPlaneRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
    kind: VCluster
    name: kind
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
    kind: VCluster
    name: kind
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: VCluster
metadata:
  name: kind
  namespace: vcluster2
spec:
  controlPlaneEndpoint:
    host: ""
    port: 0
  helmRelease:
    chart:
      name: null
      repo: null
      version: null
    values: ""
  kubernetesVersion: 1.29.0
```

Die Sache mit dem NodePort Service haben wir in dieser Instanz mal weggelassen:


```bash
kubectl create ns vcluster2
kubectl apply -f vcluster2.yaml
```

Es wird also jeweils ein Cluster Objekt und ein Vcluster Objekt erstellt:

```bash
# kubectl get cluster -A
NAMESPACE   NAME   CLUSTERCLASS   PHASE          AGE   VERSION
vcluster    kind                  Provisioned    15m
vcluster2   kind                  Provisioning   6s
# kubectl get vcluster -A
NAMESPACE   NAME   AGE
vcluster    kind   15m
vcluster2   kind   16s
# vcluster list

    NAME |  CLUSTER  | NAMESPACE | STATUS  | VERSION | CONNECTED |            CREATED            |  AGE   | DISTRO
  -------+-----------+-----------+---------+---------+-----------+-------------------------------+--------+---------
    kind | kind-kind | vcluster  | Running | 0.11.1  |           | 2024-06-09 20:31:21 +0000 UTC | 16m54s | OSS
    kind | kind-kind | vcluster2 | Running | 0.11.1  |           | 2024-06-09 20:46:32 +0000 UTC | 1m43s  | OSS
```

Zugriff bekommt man entweder über `vcluster connect` oder 

```bash
clusterctl -n vcluster get kubeconfig kind
```

Da wird man dann freilich merken, dass der API-Endpunkt der interne Service Endpunkt im Namespace ist. Im Original Quickstart von Cluster API ist das ein NodePort, damit er von aussen erreichbar ist. Ansonsten bräuchte man einen LoadBalancer Service, was unser Minimalsetup sprengen würde.

# Fazit

Mit Cluster API wurde jetzt auch das Erstellen von Clustern "kubernetisiert", ist also mittels CRD als Resourcen beschreibbar. Das ganze soll meta-mässig standardisiert sein, also egal welche Infrastruktur darunter liegt. Soweit sogut. Was im ersten Test noch fehlt:

- API Zugriff Downstream Cluster (siehe oben)
- Multicluster Verwaltung wie bei Rancher (ist ansatzweise gegeben)
- Multiuser Verwaltung wie bei Rancher (würde auch über OIDC direkt im [kube-api](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) server gehen.
- graphische Benutzeroberfläche wie Rancher (klassisches [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) oder [Headlamp](https://github.com/headlamp-k8s/headlamp), hat allerdings noch Bugs)

Aber das soll auch erst der Einstieg in Cluster API sein und keine Rancher Migration. Die kommt erst später :-)

