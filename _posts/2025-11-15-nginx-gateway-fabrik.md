---
layout: post
tag: de
title: Nginx Gateway Fabrik
subtitle: Und die Schlote qualmen
1ate: 2025-11-15
background: '/images/nginx-fabrik.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

Am 11.11. begann nicht nur der Karneval, auch die Kubernetes Community gab etwas Lustiges bekannt: [Das Ende vom Nginx Ingress Controller](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/). Freude dürfte auch bei denen aufgekommen sein, die erst unlängst von Bitnami auf die CNCF Version vom Nginx Ingress Controller gewechselt sind, nachdem der neue Eigentümer von Bitnami die ganzen [Lurker](https://github.com/bitnami/charts/issues/35164) nicht mehr wollte. Aber wie geht's jetzt weiter?

# Gateway API

Angepriesen als das nächste Pferd, was ins Rennen gehen soll, ist [Gateway API](https://gateway-api.sigs.k8s.io/guides/). Das Ding ist auch schnell installiert und es gibt auch einen Migrations Guide: [Migrating from Ingress](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/). Da versucht man dem Nutzer das ganze noch etwas schmachkaft zu machen, erklärt die vielen neuen Objekte und der Kenner merkt sofort: Jede Menge Arbeit.

Denn mit einem 

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

ist es leider nicht getan. Denn das installiert nur die CRDs. Für das Projekt ist der Fall erledigt und er denkt sich, es wird schon genug Leute geben, die das benutzen wollen.

# NGINX Gateway Fabric

Die [Nginx Gateway Fabric](https://github.com/nginx/nginx-gateway-fabric) ist das neue Herzstück der Anlage und ersetzt den Nginx Ingress Controller. Den haben wir beispielsweise installiert mit


```bash
helm -n ingress-nginx upgrade -i ingress-nginx --set controller.hostPort.enabled=true --set controller.service.annotations."kube-vip\.io\/loadbalancerIPs"=${public_ip} . --create-namespace
```

Achso, das ganze läuft in unserem [OTC Tofu Setup](https://github.com/eumel8/tf-kubeadm-otc/), ein Kubernetes Cluster mit Kubeadm, Kube-Vip, Cert-Manager auf Ubuntu VM auf der Open Telekom Cloud. Der Ingress Controller kann aus einem Software Shell script dort zusätzlich installiert werden.

# Cert Manager (Gateway API)

Cert Manager ist allseits beliebt beim Ausstellen von Server Zertifikaten in Zusammenhang mit Let's Encrypt. Bisher reichte eine Standard-Installation. Jetzt brauchen wir die Erweiterung, damit Cert Manager auf die neuen Objekte reagieren kann. Deswegen muss man den installieren oder upgraden mit:

```bash
helm -n cert-manager upgrade -i cert-manager jetstack/cert-manager  --set crds.enabled=true --create-namespace   --set config.apiVersion="controller.config.cert-manager.io/v1alpha1"   --set config.kind="ControllerConfiguration"   --set config.enableGatewayAPI=true
```

# NGF Installation

Die Nginx Gateway Fabric können wir wieder mit Helm installieren. Dazu müssen wir in den Service Pods Hostnetwork aktivieren und die Loadbalancer IP mitgeben. Die wird bei der OTC ausgewürfelt beim Ausführen des Tofu Deployments:

```bash
helm upgrade -i  ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway --set nginx.service.externalTrafficPolicy=Cluster --set gwAPIExperimentalFeatures=true --set gwAPIInferenceExtension=true  --set nginx.container.hostPorts[0].port=80   --set nginx.container.hostPorts[0].containerPort=80   --set nginx.container.hostPorts[1].port=443   --set nginx.container.hostPorts[1].containerPort=443 --set nginx.service.loadBalancerIP=164.30.3.60
```

Der `hostPort` bezieht sich auf den Container des Service. Also es läuft im Cluster der Nginx Gateway Fabric Controller mit einem Pod und einem Service. Dann wird pro Gateway ein weiterer Pod und Service gestartet. Der Service ist vom Typ Loadbalancer im Namespace der Applikation.

Das deployen wir unten extra, das Helm Chart vom Nginx Gateway Fabric würde das aber auch mit hergeben für eine zentrale Instanz.

Und die `loadBalancerIP` können wir direkt angeben, bislang haben wir das mit einer Annotation im Loadbalancer Service gemacht:

```yaml
  annotations:
    kube-vip.io/loadbalancerIPs: 164.30.3.60
```

# Gateway und Httproutes

Kommen wir nun zu dem, was früher mal ein `Ingress` war. Wenn man die [Gateway Spec](https://gateway-api.sigs.k8s.io/reference/spec/#gatewayspec) so ansieht, ist es dem Ingress gar nicht mal so unähnlich:


```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kubeadm-gateway
  namespace: drachenboot
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    #kube-vip.io/loadbalancerIPs: 164.30.3.60
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    hostname: drachenboot.otc.mcsps.de
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: All
    tls:
      mode: Terminate
      certificateRefs:
        - name: drachenboot-otc-mcsps-de-tls
  - name: http
    port: 80
    protocol: HTTP
    hostname: drachenboot.otc.mcsps.de
```

Wir haben das Gateway für unsere Webseite `drachenboot.otc.mcsps.de`. Die hat oben erstmal die cert-manager annotation, damit mit dem ClusterIssuer namens letsencrypt ein Zertifikat ausgestellt werden kann. Im Helm Chart vom Nginx Gateway Fabric wird ein Job mit installiert, der das dazugehörige Secret anlegt, was wir weiter unten bei den Listenern konfiguriert haben.
Die Listener selber sind relativ selbsterklärend. Sie matchen auf einen Hostnamen und dem Protokoll/Port, wobei die HTTPS Traffic auf dem Gateway terminiert wird. Das ganze gehört zur GatewayClass nginx, was vergleichbar zur IngressClass ist, womit man mehrere Controller parallel installieren kann. Standardmässig heisst das Ding nginx.

Jetzt routen wir Traffic:

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: drachenboot
  namespace: drachenboot
spec:
  parentRefs:
  - name: kubeadm-gateway
    sectionName: https
  hostnames:
  - drachenboot.otc.mcsps.de
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: drachenboot-webapp
      namespace: drachenboot
      port: 8080
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: tls-redirect
spec:
  parentRefs:
  - name: kubeadm-gateway
    sectionName: http
  hostnames:
  - drachenboot.otc.mcsps.de
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        port: 443
```

Wir referenzieren auf unser Gateway und möchten alles unter / auf unseren Service drachenboot-webapp:8080 weiterleiten. Sollte es jemand mit http als Protokoll versuchen, was das Gateway auch konfiguriert hat, soll auf https umgeleotet werden. Dazu brauchen wir die zweite Route.

Fertig! So sieht das dann aus:

```bash
$ kubectl -n drachenboot get pod,gateway,httproutes,services
NAME                                         READY   STATUS    RESTARTS   AGE
pod/drachenboot-webapp-65d44f4ff7-pphk6      1/1     Running   1          26h
pod/kubeadm-gateway-nginx-58d49cf5bf-lkhzd   1/1     Running   1          4h29m

NAME                                                CLASS   ADDRESS       PROGRAMMED   AGE
gateway.gateway.networking.k8s.io/kubeadm-gateway   nginx   164.30.3.60   True         4h29m

NAME                                               HOSTNAMES                      AGE
httproute.gateway.networking.k8s.io/drachenboot    ["drachenboot.otc.mcsps.de"]   4h29m
httproute.gateway.networking.k8s.io/tls-redirect   ["drachenboot.otc.mcsps.de"]   4h29m

NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/drachenboot-webapp      ClusterIP      10.109.189.86   <none>        8080/TCP                     26h
service/kubeadm-gateway-nginx   LoadBalancer   10.111.174.94   164.30.3.60   80:31369/TCP,443:31439/TCP   4h29m
```

Den Ingress Controller und Kube-VIP können wir jetzt löschen:

```bash
rm /etc/kubernetes/manifests/kube-vip.yaml
helm -n ingress-nginx delete ingress-nginx
```

War jetzt doch nicht so viel Raketenwissenschaft und für unsere kleine Webseite kriegt man die Umstellung auch hin. Bloss auch da gibt es Handlungsbedarf, denn so wie wir es bisher installiert haben, geht es nicht mehr:

```yaml
$ helm -n drachenboot upgrade -i drachenboot --set image.repository=ghcr.io/eumelnet/eumelnet/drachenboot --set image.tag=0.0.7 --set ingress.enabled=true --set ingress.className=nginx --set ingress.annotations."kubernetes\.io/ingress\.class"=nginx --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt --set ingress.hosts[0].host=drachenboot.otc.mcsps.de --set ingress.hosts[0].paths[0].path=/ --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific --set ingress.tls[0].hosts[0]=drachenboot.otc.mcsps.de --set ingress.tls[0].secretName=drachenboot-otc-mcsps-de-tls-secret oci://ghcr.io/eumelnet/webapp:0.1.1 --create-namespace
```

Jedes Helm Chart, was eine Ingress Konfiguration anbietet, muss angefasst werden. Die neue Funktion mit dem Gateway muss sich etablieren, wobei da verschiedene Betriebsmodelle vorgesehen sind. Die Kollegen vom Cert-Manager stellen sich in ihrer [Architektur Beschreibung](https://cert-manager.io/docs/usage/gateway/) etwa vor, dass der Cluster-Administrator das Gateway installiert und der User/Developer die HttpRoutes, was wiederum bedeutet, dass ein Helmchart HttpRoutes enthalten kann, aber kein Gateway (obwohl es dessen Namen referenzieren muss). 

Die Nginx Gateway Fabric ist nur eine von vielen Implementierungen, wenn man in die [Liste](https://gateway-api.sigs.k8s.io/implementations/#gateway-controller-implementation-status) schaut. Und jeder wird natürlich seine eigenen Vorzüge haben, da auch schon einige den GA Status erreicht haben. Bleibt abzuwarten, wie es da weitergeht. Anzumerken zum Schluss ist auch noch, dass Nginx mittlerweile zur Firma F5 gehört und die Nginx Gateway Fabric natürlich von F5-Leuten verwaltet wird. Es gibt auch Optionen, dass mit [Nginx Plus](https://www.f5.com/de_de/products/nginx/nginx-plus) zu betreiben, ein kommerzielles Produkt, welches aus der Fusion hervorgegangen ist.

Aber wer weiss, vielleicht lebt Ingress Nginx als Fork weiter und wir brauchen nichts umzustellen. Wäre ja auch mal etwas.

Bis dahin

