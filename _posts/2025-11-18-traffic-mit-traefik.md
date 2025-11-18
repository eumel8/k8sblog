---
layout: post
tag: de
title: Traffic mit Traefik
subtitle: Ingress-Ersatzverkehrsleiteinrichtung
1ate: 2025-11-18
background: '/images/traefik.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

Zweite Folge von [das Ende vom Nginx Ingress Controller](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/).
Im letzten Post haben wir uns mit [Gateway API](https://gateway-api.sigs.k8s.io/guides/) und der Nginx Gateway Fabric beschäftigt. War jetzt nicht so der Burner - neue Resourcen, viele Einschränkungen. In der [Liste](https://gateway-api.sigs.k8s.io/implementations/) der Implementierungen gibt's auch Traefik - schauen wir da mal rein.

# Traefik

[Traefik](https://traefik.io) kennen wir eigentlich schon von Ranchers [K3S](https://k3s.io/) Deployment. Dort war Traefik standardmässig als Ingress Controller mit dabei. Statt kube-vip wurde ein Dienst namens Klipper mit installiert. Bleiben wir bei kube-vip uns installieren erstmal die Traefik CRDS:

```bash
git clone https://github.com/traefik/traefik-helm-chart.git
cd traefik-helm-chart/traefik-crd
helm -n traefik upgrade -i traefik-crd . --create-namespace
cd ../traefik
```

Das Helm Chart zu Traefik kennt eine Million Parameter, von Gateway ist die Rede, aber auch Ports und Listener. Wenn man das nutzt, landet man schnell wieder in der Schleife von einem Gateway pro Webseite, die man ausliefern möchte. 

Die wesentlichen Änderungen in unserem `kubeadm-values.yaml` sind

- updateStrategy auf Recreate, weil wir hostPorts binden, die nur einmal belegt sein dürfen
- besagte hostPorts 80+443
- ändern des Metric Ports auf was anderes als 9100, weil dort schon ein Node-Exporter läuft.


```yaml
deployment:
  enabled: true
  kind: Deployment

updateStrategy:
  type: Recreate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1

ingressClass:  # @schema additionalProperties: false
  enabled: true
  isDefaultClass: true
  name: "traefik"

gateway:
  enabled: true
  listeners:
    web:
      port: 8000

gatewayClass:
  enabled: true

providers:
  kubernetesCRD:
    enabled: true
    ingressClass: "traefik"

  kubernetesIngress:
    enabled: true
    allowExternalNameServices: true
    allowEmptyServices: true

  kubernetesGateway:
    enabled: true

logs:
  general:
    level: "INFO"  # @schema enum:[TRACE,DEBUG,INFO,WARN,ERROR,FATAL,PANIC]; default: "INFO"

  access:
    enabled: true

global:
  checkNewVersion: false

ports:
  traefik:
    port: 8080
    exposedPort: 8080
    protocol: TCP
  web:
    port: 8000
    hostPort: 80
    expose:
      default: true
    exposedPort: 80
    protocol: TCP
  websecure:
    port: 8443
    hostPort: 443
    expose:
      default: true
    exposedPort: 443
    protocol: TCP
    tls:
      enabled: true
  metrics:
    port: 9102
    exposedPort: 9102

service:
  enabled: true
  single: true
  type: LoadBalancer
  annotations:
    kube-vip.io/loadbalancerIPs: 164.30.14.98
  spec:
    externalTrafficPolicy: Cluster
```

Und schon geht's los:

```bash
helm -n traefik upgrade -i traefik . -f kubeadm-values.yaml
```

Achso, wir haben oben noch unsere kube-vip Annotation für die Floating-IP angegeben, die als Loadbalancer fungiert.
Es sollte nach dem Deployment ein traefik Pod laufen und der Service sieht dann so aus:

```bash
# kubectl -n traefik get service traefik
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE
traefik   LoadBalancer   10.98.2.210   164.30.14.98   80:31383/TCP,443:32284/TCP   77m
```

Das Gateway ist auch installiert, wird aber ohne Httproutes nicht benutzt:

```bash
# kubectl -n traefik get gateway
NAME              CLASS     ADDRESS        PROGRAMMED   AGE
traefik-gateway   traefik   164.30.14.98   True         78m
```

# Ingress

Am Ingress Objekt der Applikation müssen wir gar nicht viel ändern. Ausser `ingressClassName` muss auf `traefik` geändert werden. Man könnte aber im Traefik Deployment die IngressClass auf `nginx` ändern und dann sollte alles problemlos weiter funktionieren.

# Fazit

Traefik ist die Wahl, um schnell auf eine supportete Ingress-Controller-Version zu wechseln. Das Projekt gibt es schon sehr lange und die Software ist entsprechend ausgereift, also ungefähr 10 Jahre. Firmensitz ist zwar in San Francisco (USA), aber das Herz schlägt in Frankreich (Paris und Lyon), man unterstützt also noch ein europäisches Projekt, wenn man es so möchte. Gateway API wird auch unterstützt, mehr im [Blog](https://traefik.io/blog) mit aktuellen Beiträgen. So hat man das Beste aus beiden Welten und kann seine Dienste weiter anbieten.
