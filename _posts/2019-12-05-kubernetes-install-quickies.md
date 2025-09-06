---
layout: post
tag: de
title: "Kubernetes Install Quickies"
subtitle: Kubernetes erweckt den Eindruck, hochkompliziert zu sein. Und meistens ist es das auch. Aber es gibt einige sehr leichgewichtige Tools bzw. Methoden, um Kubernetes schnell zu installieren. Es sind nur ein Kommandozeilen und hier sind einige Methoden
date: 2019-12-05
background: '/images/quickies.png'
---

Vorbedingungen

Virtuelle Maschine Ubuntu 18.04

Minikube


<a href="https://kubernetes.io/de/docs/setup/minikube/">Minikube</a> haben wir in <a href="https://blog.eumelnet.de/blogs/blog8.php/joomla-installation-mit-kubernetes-und-helm-1">diesem Posting</a> schon behandelt.

K3S
---

<a href="https://k3s.io/">K3S</a> von Rancher ist ein Leichtgewicht-Kubernetes, nutzbar auch auf Edge oder Raspberry PI. Die Installationsroutine ist auf der Webseite beschrieben:

```
curl -sfL https://get.k3s.io | sh -
```

Hinzufügen von Nodes:

```
$ curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=XXX sh -
```

Der Token steht auf dem Master in der Datei /var/lib/rancher/k3s/server/node-token

Kubeadm
-------

<a href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/">Kubeadm</a> ist ein Deployment-Tool fuer Kubernetes-Cluster. Auf der Webseite ist die Installation auf Ubuntu beschrieben:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" >  /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Neues Cluster initialisieren:

```
$kubeadm init
```

Anschliessend passendes Netzwerk-Plugin installieren, zum Beispiel flannel:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Node Status vom master pruefen:

```bash
$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
quickstart   Ready    master   8m26s   v1.16.3
```

Hinzufügen von Nodes:

```bash
$ kubeadm join 192.168.1.240:6443 --token gtlhtc.csy63766tya0plk8 \
    --discovery-token-ca-cert-hash sha256:2b2c547e4ed85b2cea1ac23c55ecc3eb203a1d915924657dcc2fd5a3d5af8fb9
```

Das Kommando wird am Ende der Installation von kubeadm init ausgegeben

MicroK8S
--------

<a href="https://ubuntu.com/kubernetes/install#single-node">MicroK8S </a> ist ein Produkt von Canonical. Die Installation laut Webseite erfolgt mit Snap:

```bash
$ sudo snap install microk8s --classic
```

Auch der kubernetes-client ist in diesem Snap mit enthalten:

```bash
$ microk8s.kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
quickstart   Ready    none   2m39s   v1.16.3
```

Eine vollstaendige Liste der Loesungen findet man auf  [https://kubernetes.io/de/docs/setup/](https://kubernetes.io/de/docs/setup)
