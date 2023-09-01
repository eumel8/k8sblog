---
layout: post
tag: de
title: Kubectl Curl
subtitle: Ein Plugin selbst geschrieben, um interne Web-Services mit curl abzufragen
date: 2023-09-01
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Kubectl
Im letzten Artikel haben wir uns etwas mit Kommandozeilen-Werkzeuge beschäftigt. Kubectl ist da natürlich das A und O bei der Administration des Kubernetes-Clusters. Das Programm bringt von Hause aus viele Funktionen mit. Aber es gibt auch eine Plugin-Funktion, mit der man noch viel mehr machen kann.

# Curl
Curl als Kommandozeilen-Werkzeug für verschiedenste Dienste, insbesondere Web-Dienste, ist aus dem Alltag nicht mehr wegzudenken. Sei es, um zu überprüfen, ob ein Webserver erreichbar ist und antwortet, oder mal schnell etwas abzufragen oder eine Datei herunterzuladen, all das geht mit curl sehr schnell. Ein `curl -n demoapp http://demoapp-57bf45f76-bgkwb:8080/` funktioniert im Kubernetes-Cluster in der Regel nicht, da ein Pod nur über das Overlay-Network angebunden ist und auch das Service-Network nur intern im Cluster funktioniert. Da bräuchte es schon eines Ingress oder eine Service mit NodePort oder externen LoadBalancer. 

# Kubectl Curl
Die Lösung für das Dilemma heisst [Kubectl Curl](https://github.com/segmentio/kubectl-curl). Das Plugin startet einen Port-Forward von einem lokalen Port über die Kubernetes API auf den Ziel-Pod. Der Erfinder hat sich dann viel Mühe gegeben, um alle Optionen von curl in dem Plugin als Go-Package zu verarbeiten. Zuviel der Mühe, finde ich, dafür funktionierte es nicht in der Rancher-Umgebung. Es hatte eine Weile gedauert, das rauszufinden. Es endete dann in diesem [Fork](https://github.com/eumel8/kubectl-curl), was zum einen das Package überflüssig macht und dann auch in Rancher-Clustern funktioniert.
Die Installation ist denkbar einfach, wenn man in der `Release`-Seite ein entsprechendes Binary herunterlädt und in dasselbe Verzeichnis wie kubectl kopiert. Wer Go installiert hat, findet in der README auch ein Kommando zur Installation über Go.
Voraussetzung zur Nutzung ist ein lokales `curl`. `jq` verfeinert noch die Ausgabe von json.

# Beispiele
Zu was kann man kubectl-curl jetzt alles verwenden:

- Webserver Aufruf:

```bash
$ kubectl curl -n demoapp  http://demoapp-57bf45f76-bgkwb:8080/
```

- Metriken von Programmen
```bash
$ kubectl curl -n kube-logging http://kube-logging-logging-operator-f874c54f8-wcd6g:8080/metrics
```

- Prometheus abfragen
```bash
$ kubectl curl -n cattle-monitoring-system http://prometheus-rancher-monitoring-prometheus-0:9090/api/v1/query?query=fluentbit_uptime | jq .
```

- Healthcheck von Diensten
```bash
$ kubectl curl -n kube-system http://traefik-rmtth:9000/ping
```

Viel Spass!
