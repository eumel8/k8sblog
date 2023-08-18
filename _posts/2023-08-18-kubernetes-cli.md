---
layout: post
tag: de
title: Kubernetes Kommandozeilenwerkzeuge
subtitle: Wie arbeite ich jeden Tag mit Kubernetes
date: 2023-08-18
background: '/images/k8s-cosmos.png'
twitter: 'images/cosignwebhook.png'
author: eumel8
---

# Kubectl
kubectl ist für die Arbeit mit Kubernetes das Werkzeug schlechthin. Es kann als Paket für verschiedene Distributionen installiert werden, oder als einzelne Datei. Details auf der [Kubernetes Webseite](https://kubernetes.io/de/docs/tasks/tools/install-kubectl/) . Um sich mit einem Kubernetes Cluster zu verbinden, bedarf es Credentials, die man als KUBECONFIG Datei von seinem Provider bekommt. Darin enthalten sind der Endpunkt des Kubernetes-Cluster und die Credentials, entweder ein Zertifikat, Passwort oder Token. Die Bedienung ist intuitiv zum Erforschen der Kubernetes-Resourcen oder Erstellen selbiger.


# Kubectl-Plugins
Wem der Funktionsumfang von kubecl nicht reicht, kann diesen mit Plugins erweitern lassen. [Krew](https://krew.sigs.k8s.io/plugins/) ist dabei ein Plugin-Manager. Man kann Plugins aber auch manuell herunterladen und verwenden. Hier eine Auswahl:

## ketall

[https://github.com/corneliusweig/ketall](https://github.com/corneliusweig/ketall)

kubectl get-all (ketall) listet wirklich alle Resourcen im Cluster oder einem Namespace, die das normale kubectl nicht listet

## kubectl-images

[https://github.com/chenjiandongx/kubectl-images](https://github.com/chenjiandongx/kubectl-images)

kubectl images listet alle Container Images eines Pods im Cluster oder einem Namespace auf.  Schön übersichtlich, wenn man mal wissen will, von welchr Registry oder Repo die Images verwendet werden.

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

kuota-calc kann die benötigten Resourcen eines Deployments unter Beachtung der Upgrade-Strategie berechnen:

```
$ cat cosignwebhook/manifests/demoapp.yaml | ./kuota-calc  --detailed
Version    Kind          Name       Replicas    Strategy         MaxReplicas    CPU     Memory    
apps/v1    Deployment    demoapp    1           RollingUpdate    2              400m    512Mi     
```


## kubectl-curl

[https://github.com/segmentio/kubectl-curl](https://github.com/segmentio/kubectl-curl)

Eine Erweiterung, um mit curl auf den HTTP-Endpunkt eines PODs zuzugreifen

## kubepug

[https://github.com/rikatz/kubepug](https://github.com/rikatz/kubepug)

findet veraltete API Endpunkte

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

ChatGPT Plugin. Erzähle ChatGPT welches Projekt Du realisieren möchtest und erhalte das passende Kubernetes Manifest dazu

<iframe width="560" height="315" src="https://www.youtube.com/embed/j6lO-zvWVdc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## kubectl-oomd

[https://github.com/jdockerty/kubectl-oomd](https://github.com/jdockerty/kubectl-oomd)

Wann ist welcher Pod wegen Speichermangel beendet worden:

```
./oomd 
POD                                                   CONTAINER           REQUEST     LIMIT     TERMINATION TIME
cae-feeder-preview-0                                  cae-feeder          2Gi         2Gi       2023-05-02 05:46:12 +0200 CEST
elastic-worker-5d6df7fbb4-lvvd5                       elastic-worker      768Mi       768Mi     2023-07-17 12:26:20 +0200 CEST
replication-live-server-delivery-environment-01-0     content-server      1Gi         1Gi       2023-08-12 13:54:49 +0200 CEST
user-changes-0                                        user-changes        768Mi       768Mi     2023-05-02 20:01:46 +0200 CEST
workflow-server-0                                     workflow-server     768Mi       768Mi     2023-06-22 09:10:44 +0200 CEST
```

Ganz schön viel, stimmt denn das:

```
$ kubectl describe pod workflow-server-0
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Wed, 26 Apr 2023 11:05:19 +0200
      Finished:     Thu, 22 Jun 2023 09:10:44 +0200
```

scheint so

Über die Release-Seite des jeweiligen Projekts kann man sich meistens das Programm herunterladen, oder mit Krew installieren.

Viel Spass!
