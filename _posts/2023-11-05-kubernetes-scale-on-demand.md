---
layout: post
tag: de
title: Kubernetes auf Nachfrage skalieren
subtitle: Keda und Keda HTTP Add-On
date: 2023-11-05
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Nachhaltiges Computern
Der Klimawandel ist in aller Munde. Währenddessen summt und brummt es in den Rechenzentren dieser Welt fröhlich vor sich hin. Zum Thema Nachhaltiges Computern werden wir uns im nächsten Beitrag widmen. Hier geht es erstmal um Skalieren auf Nachfrage, also unsere Workload im Kubernetes Cluster wird je nach Bedarf skaliert, mit dem [Horizontal Pod Autoscaler](https://kubernetes.io/de/docs/tasks/run-application/horizontal-pod-autoscale/) von 1 auf unendlich.
Aber was wäre jetzt, wenn das 0 auf unendlich möglich wäre?

# Keda
[Keda](https://keda.sh) - Kubernetes Event Driven Autoscaling. Ein markiger Begriff und ebenso genial. Ich habe im Cluster meine Workload installiert und das Deployment auf 0 skaliert. Alles ist "ready to go". Das Go kommt dann von einem Event, old school wäre jetzt ein Cronjob, der die Workload um 8 Uhr hoch und um 18 Uhr wieder runterskaliert. Immerhin. Was wäre jetzt aber das Szenario eines wenig benutzen Dienstes wie etwa eine Webseite, die nur abundzu jemand besucht, wie etwa diese hier? Okay, klammern wir den ganzen Spam und Bots mal aus, um die können wir uns später kümmern. Am Tag kommen vielleicht ein oder zwei Besucher hier vorbei. Und für die öffnen wir unseren Laden, indem wir nach der ersten Anfrage im Browser einen Pod starten, welcher einen Nginx-Webserver beherbergt, der dann diesen Content hier ausliefert und dem Besucher im Browser mit geringer Verzögerung manifestiert. Wenn die Seite geladen ist, warten Keda ein Weilchen und skaliert das Deployment wieder auf 0. Wir sparen also Computer Resourcen, Strom und schonen somit die Umwelt.

# Vorbereitung
Unser Kubernetes Cluster ist ein K3S, der sich selbst mit Rancher verwaltet. K3S kommt standardmässig mit Traefik als Ingress-Controller. Dort müssen wir schon mal ein paar Anpassungen durchführen:

```bash
kubectl -n kube-system edit helmchartconfigs.helm.cattle.io  traefik
```

Wir fügen folgende Zeilen hinzu:

```yaml
spec:
  valuesContent: |-
    providers:
      kubernetesCRD:
        allowemptyservices: true
        allowExternalNameServices: true
      kubernetesIngress:
        allowemptyservices: true
        allowExternalNameServices: true
```

Warum, das sehen wir gleich.

# Keda und Keda HTTP Addon installieren
Eins vorweg: Keda ist sehr sauber und strukturiert aufgebaut. In der [Dokumentation](https://keda.sh/docs/2.12/deploy/) findet man schnell die Möglichkeiten, um Keda zu installieren. Die Dokumentation ist auch nach Standards für technische Dokumentation aufgebaut: Es geht von leichten Schritten bis zu komplizierten, oberflächliche und allgemeingültige Beschreiben führen zu sehr viel Tiefe, wie etwa [die detailierte Beschreibung aller Helm Chart Parameter](https://github.com/kedacore/charts/tree/main/keda#general-parameters). Sowas findet man selten und zeugt von sehr viel Liebe zum Projekt. Dank der [hohen Sicherheitsstandards](https://github.com/kedacore/charts/tree/main/keda#keda-is-secure-by-default) sind kaum Anpassungen notwendig. Wir können beginnen mit der Helm-Installation:

```bash
helm repo add keda https://kedacore.github.io/charts
helm repo update
helm upgrade -i keda keda/keda --namespace keda --create-namespace --set rbac.aggregateToDefaultRoles=true
helm upgrade -i http-add-on keda/keda-add-ons-http --namespace keda --set rbac.aggregateToDefaultRoles=true
```

Wir haben uns hier zur cluster-weiten Installation entschieden. Es werden die CRDs installiert und cluster-weites RBAC, wobei die Rechte an die jeweiligen Default-User wie `admin` aggregiert werden. Damit können dann später Projekt-Owner in ihrem Rancher-Projekt Keda für ihre App verwalten. Eine andere Möglichkeit wäre der Keda-Operator oder die Namespaced Installation. Wir wollen den Benutzer aber nicht mit der Verwaltung der Kedas-Komponenten belasten. Er soll es so einfach wie möglich haben, deswegen dieser Ansatz hier.

# Demoapp & Demouser
Der Demouser besitzt im Rancher ein Projekt und ist dort Projektowner. In dem demoapp Projekt erstellen wir einen demoapp Namespace.

<img src="/images/2023-11-05_1.png"/>

Als App dient uns eine Demo-App, nehmen wir die [Flask App hier](https://github.com/mcsps/use-cases/tree/master/flask). Diese installieren wir in den demoapp Namespace.

```bash
kubectl -n demoapp apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/flask/deployment.yaml
kubectl -n demoapp apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/flask/service.yaml
``` 

Jetzt bräuchten wir nur noch einen Ingress und unsere App wäre von der Welt erreichbar. Aber schauen wir uns nochmal das Keda Architekturbild an:

<img src="https://raw.githubusercontent.com/kedacore/http-add-on/main/docs/images/arch.png" width="925" height="525"/>

Zwischen Ingress und Service gibt es den Interceptor, durch den wir den Verkehr schleusen müssen. Unser Schleuser heisst `relink` und wird mit im demoapp Namespace deployt:

```bash
cat <<EOF | kubectl -n demoapp apply -f -
apiVersion: v1
kind: Service
metadata:
  name: relink
spec:
  type: ExternalName
  externalName: keda-add-ons-http-interceptor-proxy.keda.svc.cluster.local
EOF
```

Der Interceptor Service läuft in einem anderen Namespace. Dieser Service leitet den Verkehr dorthin um.
Der Ingress hat jetzt `relink` als Backend:

```bash
cat <<EOF | kubectl -n demoapp apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demoapp
spec:
  ingressClassName: traefik
  rules:
  - host: demoapp.otc.mcsps.de
    http:
      paths:
      - backend:
          service:
            name: relink
            port:
              number: 8080
        path: /
        pathType: Prefix
EOF
```

Damit `ExternalName` als Ingress Endpunkt von Traefik akzeptiert wird, waren die Änderungen an der Helmchartconfig von Traefik am Anfang notwendig.

# HTTPScaledObject

Nach unserem letzten Arbeitsschritt landet unsere Workload irgendwo im Nirvana. Es ist für den Projekt-Owner nicht transparent, wohin er den Verkehr sendet. Hier ist natürlich Rücksprache mit dem Cluster-Owner notwendig.

Um jetzt die Skalierung anzugehen, brauchen wir ein HTTPScaledObject:

```bash
cat <<EOF | kubectl -n demoapp apply -f -
kind: HTTPScaledObject
apiVersion: http.keda.sh/v1alpha1
metadata:
    name: demoapp
spec:
    hosts:
    - demoapp.otc.mcsps.de
    scaledownPeriod: 10
    scaleTargetRef:
        deployment: demoapp
        service: demoapp
        port: 80
    replicas:
        min: 0
        max: 10
    targetPendingRequests: 1
EOF
```

Der Keda HTTP Interceptor soll also unseren Verkehr auf den demoapp Service in unserem demoapp Namespace weiterleiten. Das minimale Replica ist 0, also es laufen keine Pods der app:

```bash
$ kubectl -n demoapp get deployments.apps demoapp
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
demoapp   0/0     0            0           23h
```

Den Status vom Keda können wir auch abfragen und sieht dann ungefähr so aus:

```bash
kubectl -n demoapp describe httpscaledobjects.http.keda.sh demoapp
```

```yaml
Status:
  Conditions:
    Message:    Identified HTTPScaledObject creation signal
    Reason:     PendingCreation
    Status:     Unknown
    Timestamp:  2023-11-04T16:42:14Z
    Type:       Pending
    Message:    App ScaledObject created
    Reason:     AppScaledObjectCreated
    Status:     True
    Timestamp:  2023-11-04T16:42:14Z
    Type:       Created
    Message:    Finished object creation
    Reason:     HTTPScaledObjectIsReady
    Status:     True
    Timestamp:  2023-11-04T16:42:14Z
    Type:       Ready
Events:         <none>
```

Wenn wir jetzt unsere Demoapp aufrufen, dauert es einen kleinen Moment und dann ist die App verfügbar:

```bash
$ curl http://demoapp.otc.mcsps.de/a
You requested: a
```

Das Inaktiv-Timeout haben wir auf 10 Sekunden eingestellt. Die App schläft also schon wieder, ehe wir im Event-Log nachschauen können, was passiert ist:

```bash
$ kubectl -n demoapp get events -w=1
LAST SEEN   TYPE     REASON                       OBJECT                          MESSAGE
77s         Normal   KEDAScaleTargetActivated     scaledobject/demoapp            Scaled apps/v1.Deployment demoapp/demoapp from 0 to 1
77s         Normal   ScalingReplicaSet            deployment/demoapp              Scaled up replica set demoapp-6b6f4bc684 to 1 from 0
77s         Normal   SuccessfulCreate             replicaset/demoapp-6b6f4bc684   Created pod: demoapp-6b6f4bc684-s4gng
77s         Normal   Scheduled                    pod/demoapp-6b6f4bc684-s4gng    Successfully assigned demoapp/demoapp-6b6f4bc684-s4gng to k3s-test-server-2
77s         Normal   Pulled                       pod/demoapp-6b6f4bc684-s4gng    Container image "mtr.devops.telekom.de/mcsps/mcsps-python:latest" already present on machine
77s         Normal   Created                      pod/demoapp-6b6f4bc684-s4gng    Created container demoapp
77s         Normal   Started                      pod/demoapp-6b6f4bc684-s4gng    Started container demoapp
66s         Normal   KEDAScaleTargetDeactivated   scaledobject/demoapp            Deactivated apps/v1.Deployment demoapp/demoapp from 1 to 0
66s         Normal   ScalingReplicaSet            deployment/demoapp              Scaled down replica set demoapp-6b6f4bc684 to 0 from 1
66s         Normal   SuccessfulDelete             replicaset/demoapp-6b6f4bc684   Deleted pod: demoapp-6b6f4bc684-s4gng
66s         Normal   Killing                      pod/demoapp-6b6f4bc684-s4gng    Stopping container demoapp
```

Das wars! Das Konzept ist den [Idle Instances von Openshift](https://docs.openshift.com/container-platform/4.9/applications/idling-applications.html) adaptiert. 

Es bieten sich aber noch viele andere Möglichkeiten des [Skalierens](https://keda.sh/docs/2.12/scalers/) an. Erwähnt sei noch:

# Prometheus

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: demo-keda-scaledobject
spec:
  scaleTargetRef:
    apiVersion:    apps/v1
    kind:          Deployment
    name:          demoapp
  pollingInterval:  10                               # Optional. Default: 30 seconds
  cooldownPeriod:   300                              # Optional. Default: 300 seconds
  minReplicaCount:  0                                # Optional. Default: 0
  maxReplicaCount:  6                                # Optional. Default: 100
  fallback:                                          # Optional. Section to specify fallback options
    failureThreshold: 3                              # Mandatory if fallback section is included
    replicas: 1
  advanced: # Optional. Section to specify advanced options
    horizontalPodAutoscalerConfig: # Optional. Section to specify HPA related options
      behavior: # Optional. Use to modify HPA's scaling behavior
        scaleDown:
          stabilizationWindowSeconds: 150
          policies:
            - type: Percent
              value: 100
              periodSeconds: 15
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-operated.demoapp:9090/
        metricName: flask_http_request_duration_seconds # Note: name to identify the metric, generated value would be `prometheus-http_requests_total`
        query: sum(rate(flask_http_request_duration_seconds_count{path="a"}[1m])) # Note: query must return a vector/scalar single element response
        threshold: '1'
        # Optional fields:
        ignoreNullValues: "true" # Default is `true`, which means ignoring the empty value list from Prometheus. Set to `false` the scaler will return error when Prometheus target is lost
```

In diesem Beispiel läuft noch eine Monitoring-Instanz mit Prometheus. Wir installieren noch einen ServiceMonitor:

```bash
kubectl -n demoapp apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/monitoring/servicemonitor-demoapp.yaml
```

und könnten dann mit PromQL Abfragen unsere Skalierung steuern. Klappt natürlich nicht mit den Flask-Metriken, denn dazu müsste unsere Flask-App mindestens einmal laufen. Aber es ist vielleicht für grössere Applikationen geeignet, wo zum Beispiel ein kleinerer Teil permanent läuft und der grosse Java-Container nur bei bestimmten Requests gestartet wird. Das nur so als Idee.

# Antispam
Unsere Skalierung nach Bedarf funktioniert jetzt einwandfrei. Wenn wir ihn im Internet loslassen, würde er aber kaum zur Ruhe kommen, da Unmengen von Bots unterwegs sind, die unsere App ausspionieren wollen. 

Hier ein paar Ideen, um den Verkehr einzudämmen:

Traefik bietet die [Middleware](https://doc.traefik.io/traefik/middlewares/http/ipwhitelist/) Resource an, um IPs zu whitelisten:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: test-ipwhitelist
spec:
  ipWhiteList:
    sourceRange:
      - 127.0.0.1/32
      - 192.168.1.7
``` 

Umgekehr geht Blacklisten:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: test-ipwhitelist
spec:
  ipWhiteList:
    ipStrategy:
      excludedIPs:
        - 127.0.0.1/32
        - 192.168.1.7
```
Eine tiefere Möglichkeit ist eine Applikations-Firewall vor dem Ingress-Controller. Oder man bindet [dynamische Spamlisten (DNBL)](https://gist.github.com/theMiddleBlue/02142f84007a5538491e109b383f28ba) im Nginx des Ingress Controllers ein.

# Fazit

Skalieren nach Bedarf mag nur ein kleiner Beitrag sein, um die Umwelt und den Geldbeutel zu schonen, wenn man für CPU und Memory bezahlen muss. Es ist aber ein Anfang und Dank des hervorragenden Keda Projekts auch in Open Source möglich.
