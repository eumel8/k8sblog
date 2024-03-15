---
layout: post
tag: de
title: Vcluster Backup
subtitle: Erweiterung der Crossplane Composition
date: 2024-03-14
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

Zu Vcluster und Crossplane gab es in diesem [Blog schon Experimente](https://k8sblog.eumel.de/2022/12/14/vcluster-in-rancher-mit-crossplane.html). Wir können eine Resource `vcluster` erstellen und Crossplane wird ein Helm-Chart mit einem Vcluster-Deployment ausrollen und diesen Cluster in Rancher registrieren. Dazu haben wir als Unterbau K3S mit der eingebetteten SQLite-DB genommen, da auch nur eine Single-Instanz verfügbar ist. Gut genug für Tests und Entwicklung.

Der Haken bei der Sache ist: auch bei hochverfügbarem Storage für die Datenbank kann schon mal was kaputtgehen. In der SQLite ist das ganze Leben des Vclusters drin. Gute Idee, das zu backupen.

# Lösung 1 - der Cronjob

Hier ist das [Gist](https://gist.github.com/eumel8/686471197180d6f844191393c946a4db):

<script src="https://gist.github.com/eumel8/686471197180d6f844191393c946a4db.js"></script>

Es wird also im Kubernetes-Cluster ein Cronjob installiert, der periodisch einen Job startet, der den PVC der K3S-Instanz mountet und dort mit sqlite ein Backup initiert. Im Beispiel ist dann auch noch ein Job für einen Restore aufgeführt.

Zwei Probleme treten hier auf: Der PVC-Name muss manuell gesetzt werden (kriegt man in der Automatisierung, wenn man kustomize oder Helm verwendet, auch noch irgendwie hin. Und, je nach StorageClass (hier Longhorn), muss der Cronjob auf demselben Workernode laufen wie der K3S-Pod. Sonst gibts "multi-attach error". Auch das könnte man irgendwie lösen, mit noch mehr Fummelei.

# Lösung 2 - vcluster-backup Programm

Ein kleines [Go-Programm](https://github.com/eumel8/vcluster-backup/), welches permanent eine Datei (state.db) verschlüsselt in einen S3-Speicher ablegt.

```bash
$ ./vcluster-backup -h
Usage of ./vcluster-backup:
  -accessKey string
    	S3 accesskey.
  -backupFile string
    	Sqlite database of K3S instance. (default "/data/server/db/state.db")
  -backupInterval int
    	Interval in minutes for backup. (default 2)
  -bucketName string
    	S3 bucket name. (default "k3s-backup")
  -decrypt
    	Decrypt the file
  -encKey string
    	S3 encryption key.
  -endpoint string
    	S3 endpoint.
  -list
    	List S3 objects
  -region string
    	S3 region. (default "default")
  -secretKey string
    	S3 secretkey.
  -trace string
    	Trace S3 API calls
  -insecure string
    	Insecure S3 backend
```

Das Programm läuft im Vordergrund und je nach `backupInterval` Zeit wird alle paar Minuten die Datei state.db gesichert-

Wozu ist das jetzt gut? Nun, was oben als Job lief, läuft hier quasi nebenan. Um diesen Sidecar-Container ins Helm-Chart von Vcluster zu integrieren, bedarf es [dieses PR](https://github.com/loft-sh/vcluster/pull/1593). Fortan wird also auf denselben Speicher vom K3s zugegriffen und die Backups erstellt. Ein Beispiel für ein Deployment ist [hier](https://github.com/eumel8/vcluster-backup/blob/main/sidecar-value.yaml) (`sidecar` wurde in `sidecarContainer` umbenannt). Die Arbeit lässt sich am stdout log beobachten.

# Crossplane Erweiterungen

Unsere Vcluster werden mit Crossplane und dem Helm-Provider deployt. Um das Backup und deren Einrichtung zu automatisieren, bedarf es einiger Erweiterungen.

Gegeben sei ein S3-Backend. Wir verwenden eine Minio-Instanz und ich verweise auch immer wieder auf die [Version mit den Security-Fixes](https://github.com/eumel8/minio/tree/fix/securitycontext). Leicht mit Helm zu installieren. Wenn es im selben Cluster läuft, bräuchten wir nicht mal einen Ingress und können den internen Service-Endpunkt benutzen.

Um einen separaten User und Bucket für jeden erstellten Vcluster zu erstellen, bedarf es eines [weiteren Helm-Charts: s3-register](https://github.com/mcsps/helm-charts/tree/master/charts/s3-register). Dieses benötigt die Admin-Credentials und den Endpunktnamen. Bucketname und Username werden aus dem Vcluster-Namen hergeleitet, das Passwort wird generiert und in ein Secret gespeichert. Von dort kann es dann vcluster-backup abrufen.

Schauen wir uns die gesamte Composition dazu an:


<script src="https://gist.github.com/eumel8/bfa1df538741f2fba9b2d84c7f80a3b2.js"></script>
