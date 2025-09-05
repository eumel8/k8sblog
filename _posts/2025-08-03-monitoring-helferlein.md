---
layout: post
tag: de
title: storagecheck und andere Monitoring Helferleins
subtitle: Prometheus Metriken in anderen Programmen nutzen
1ate: 2025-08-03
background: '/images/helferlein.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

Jetzt läuft also endlich unser Monitoring-System mit Prometheus. Mit dem Helm Chart [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/) wird meist zusammen Prometheus, Alertmanager und Grafana installiert. Alles schön und übersichtlich. Dennoch kann man das noch bisschen schöner machen.

# Storagecheck

Wenn man nicht gerade lokalen Speicher im Kubernetes Cluster benutzt, hat man immer das Problem: Funktioniert mein CSI-Treiber noch, bzw. das dahinterliegende Backend. Da hilft der [storagecheck](https://github.com/eumel8/storagecheck). Das Programm läuft selbst im Cluster, sucht sich die erste StorageClass raus, erstellt damit einen PVC, erstellt einen Pod, mountet den PVC und schreibt "Hello World" da drauf. Wenn das klappt, räumt er alles wieder ab, und startet neu nach einer festgelegten Zeit, etwa eine halbe Stunde. 
Die einzelnen Arbeitsschritte werden als Prometheus Metriken zur Verfügung gestellt, also funktioniert der Prozess überhaupt, wie lange dauert das, gibt's Fehler. So erkennt man recht schnell, wenn etwas mit dem Storage nicht mehr stimmt.

<img src="https://raw.githubusercontent.com/eumel8/storagecheck/main/storagecheck-1.png"/>

<img src="https://raw.githubusercontent.com/eumel8/storagecheck/main/storagecheck-2.png"/>


# Prometheus Dashboard

Manchmal ist es etwas zuviel, einen ganzen Grafana zu installieren. Man will vielleicht nur den Status von ein paar
Metriken wissen, wie etwa: Ist mein Service verfügbar, wieviel Traffik mache ich gerade oder wieviel Kunden sind auf der Plattform?
Dagegen hilft [prometheus-dashboard](https://github.com/eumel8/prometheus-dashboard). Die Anwendung ist zweigeteilt. Das Backend stellt eine API bereit, die wiederum Daten aus einer existierenden Prometheus Instance bezieht. Das Frontend ist ein Webserver mit einer index.html, in der man dann in Javascript die gewünschten Queries definieren kann. Sieht dann etwa so aus:

<img src="https://raw.githubusercontent.com/eumel8/prometheus-dashboard/main/screenshot.png"/>


# Cluster App

Eine Desktop Anwendung für Linux amd64 und arm, entsprechende Binaries kann man sich [hier](https://github.com/eumel8/cluster-app/releases) runterladen. Dann ist ein bisschen Konfiguration notwendig. Man muss ein paar Variablen setzen und eine `metrics.json` zusammenbasteln. Steht alles auf der [Projekt Seine](https://github.com/eumel8/cluster-app/releases)

Das Ergebnis ist eine Anwendung die zyklisch im Hintergrund die Metriken von einem Prometheus pollt und den Status in 
einem Fenster darstellt. Dabei ändert sich die Farbe der Anwendung von grün (alles ok), zu gelb (bisschen kaputt) und rot (ganz kaputt):

<img src="https://raw.githubusercontent.com/eumel8/cluster-app/main/cluster-app-1.png"/>

<img src="https://raw.githubusercontent.com/eumel8/cluster-app/main/cluster-app-2.png"/>

Das Programm kann man nebenbei laufen lassen, beliebig verkleinern und kriegt dennoch mit, wenn einer der wichtigsten Dienste gerade nicht funktioniert.

Dasselbe Programm gibt's auch auf der [carbon-app](https://github.com/eumel8/carbon-app/). Die läuft auf dem Raspberry im Vollbild Mode für genau eine Metrik. In dem Fall ist es die Ökostrom-Generierung in Deutschland, ist aber eine Prometheus-Metrik wie jede andere.

<img src="https://blog.eumel.de/images/2024-06-19_1.jpg" width="585" height="386"/>

# Cluster Check

[Cluster Check](https://github.com/eumel8/clustercheck) ist ein Kommandozeilenwerkzeug, was gegen den aktuellen Kubernetes Kontext agiert. Dann brauch das Programm ein Prometheus Endpunkt, die mit Umgebungsvariable `PROMETHEUS_URL` zu konfigurieren ist. Von den tausenden von Metriken, die für einen Cluster generiert werden, suchen wir uns die 10 wichtigsten raus und gebe den Status aus:


```
% clustercheck                              
APISERVER 🟢 OK (1) 
CLUSTER 🟢 OK (1) 
FLUENTBITERRORS 🔴 FAIL (0) 
FLUENTDERRORS 🟢 OK (1) 
GOLDPINGER 🔴 FAIL (0) 
KUBEDNS 🟢 OK (1) 
KUBELET 🟢 OK (1) 
NETWORKOPERATOR 🟢 OK (1) 
NODE 🟢 OK (1) 
STORAGECHECK 🟢 OK (1) 
PROMETHEUSAGENT 🔴 FAIL (0) 
SYSTEMPODS 🟢 OK (1) 
```

Die Metriken lassen sich im Programmcode konfigurieren, somit sind keine Parameter oder Konfigurationsdateien notwendig.


Viel Spass!
