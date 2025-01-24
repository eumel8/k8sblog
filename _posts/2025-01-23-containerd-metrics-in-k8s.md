---
layout: post
tag: de
title: containerd Metriken in Kubernetes
subtitle: Messen was im Hafen los ist
1ate: 2025-01-23
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

Heute wollen wir uns wieder mal mit Monitoring beschäftigen. Bei Kubernetes interessiert vor allem der Resourcenverbrauch von Pods und Containern. Alles messen kann man mit Prometheus.

# cAdvisor

[cAdvisor](https://github.com/google/cadvisor) ist ein Projekt der Firma Google, wie auch das ganze Kubernetes Projekt von Google stammt. Es geht um Messwerte und Kapazitäten von laufenden Containern, um eine bessere Resourcenplanung im Ökoprojekt hinzukriegen. Diese Metriken werden über eine API zur Verfügung gestellt. Eine typische Metrik ist zum Beispiel:

`container_cpu_usage_seconds_total`

Eine [scrape_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config) sieht im Prometheus etwa so aus:

```
      - job_name: 'kubernetes-nodes-cadvisor'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
```

Es wird also direkt auf die Kubernetes-API im Scope der Nodes zugegriffen, um dort die Messwerte zu bekommen. Diese entstehen im Linux-Kernel des Nodes und werden durch die jeweilige Container-Engine geleitet. Ein Umweg, wie es scheint, in [CRI Pods und Container Metriken](https://kubernetes.io/docs/reference/instrumentation/cri-pod-container-metrics/) soll es einen direkten Weg geben, diese Metriken abzurufen. Da wir hier auf Hostebene sind, gibt es keinen Kubernetes-Namespace als Label, da nur [Linux Namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) existieren, die mit dem Kubernetes-Namensvetter nix zutun haben. Schade eigentlich, denn so kann man die Scrape-Config nicht weiter eingrenzen, um etwa nur Metriken von einem Kubernetes-Namespace zu bekommen. Der Weg dazu ist etwas umständlich: Man müsste in der TSDB über die [Prometheus Admin API](https://prometheus.io/docs/prometheus/latest/querying/api/#delete-series) nicht gewünschte Metriken löschen.

# Kube State Metrics

[Kube State Metrics](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-state-metrics)
ist neben cAdvisor der zweite Teil vom [Prometheus Community Helm Chart](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus/values.yaml), um Container Metriken zu bekommen. Eine typische Metrik ist hier etwa

`kube_deployment_created`

Es wird also viel über den Status von Resourcen im Kubernetes-Cluster erzählt. Interessant ist das für die Alarmierung, wenn man etwa wissen will, ob ein Deployment nicht läuft oder in einem komischen Zustand ist. Hier sind wir vollständig in der Kubernetes-Welt angekommen. Ich kann also die Messwerte auf Kubernetes-Namespaces einschränken.

# Containerd metrics

Als letztes und am wenigsten beachtet gibt es Metriken der Container-Engine. Ein typischer Wert ist hier

`containerd_cri_image_pulls_total`

Andere Werte sind noch:

- Wieviel Pulls gibt es pro Container
- Wieviel Fehler gibt es dabei
- Wieviel Traffic verursachen die Pulls

Alles interessante Werte, die es zu scrapen gibt. Bloss dazu brauch man erstmal einen Scrape-Punkt im Prometheus. Normalerweise gibt es da Exporter, bloss für diesen Dienst ist die Auswahl rar. Entweder weil es zu wenig Interesse an diesen Werte gibt, oder weil es zu trivial ist, diese zu sammeln.

In der Container-Engine muss man das Sammeln der Metriken und Ausgabe über einen Endpunkt erstmal einschalten.
Bei containerd sieht das etwa so aus:

```bash
# cat /etc/containerd/config.toml 
# https://github.com/containerd/containerd/blob/main/docs/ops.md

[metrics]
  address = "127.0.0.1:1338"
```

Für [Crio metrics](https://github.com/cri-o/cri-o/blob/main/tutorials/metrics.md) funktioniert das ähnlich.
Wenn man grosszügig ist, ersetzt man `127.0.0.1` durch `0.0.0.0` und schon lauscht der Dienst auf allen Netzwerkinterfaces des Nodes.
Wir bräuchten dann nur noch ein `daemonset` um einen Pod auf allen Nodes zu starten, der uns durch einen Proxy Zugang zu diesen Port gewährt:

<details>
{% highlight yaml %}
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: metrics-proxy
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: metrics-proxy
  template:
    metadata:
      labels:
        app: metrics-proxy
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: metrics-proxy
              topologyKey: kubernetes.io/hostname
            weight: 1
      containers:
      - name: nginx
        image: ghcr.io/mcsps/nginx-non-root:1.0.0
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
            - CAP_NET_RAW
          privileged: false
          readOnlyRootFilesystem: true
          runAsUser: 101
          runAsGroup: 101
          seccompProfile:
            type: RuntimeDefault
        volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
        - name: tmp
          mountPath: /tmp
      hostNetwork: true
      securityContext:
        fsGroup: 1000
        runAsNonRoot: true
        supplementalGroups:
        - 1000
        seccompProfile:
          type: RuntimeDefault
      tolerations:
      - effect: NoSchedule
        operator: Exists
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-config
      - name: tmp
        emptyDir:
          medium: Memory
          sizeLimit: 500Mi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: kube-system
data:
  nginx.conf: |
    server {
        listen 8080;

        location /v1/metrics {
            proxy_pass http://127.0.0.1:1338/v1/metrics;
        }
    }
```
{% endhighlight %}
</details>

Ist natürlich ein Denkfehler. Wir sind auf dem Localhost des Pods und nicht des Hosts, auch mit `hostNetwork`

Wir brauch also schon etwas, was den Verkehr vom Pod-Network auf den Host weiterleitet. Das geht mit [Socat](https://packages.debian.org/de/sid/socat):

<details>
{% highlight yaml %}
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: containerd-metrics-proxy
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: containerd-metrics-proxy
  template:
    metadata:
      labels:
        app: containerd-metrics-proxy
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: containerd-metrics-proxy
              topologyKey: kubernetes.io/hostname
            weight: 1
      containers:
      - name: nginx
        image: ghcr.io/mcsps/socat:1.0.1
        command: ["socat", "TCP-LISTEN:8080,fork", "TCP:127.0.0.1:1338"]
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
            - CAP_NET_RAW
          privileged: false
          readOnlyRootFilesystem: true
          runAsUser: 101
          runAsGroup: 101
          seccompProfile:
            type: RuntimeDefault
      hostNetwork: true
      securityContext:
        fsGroup: 1000
        runAsNonRoot: true
        supplementalGroups:
        - 1000
        seccompProfile:
          type: RuntimeDefault
      tolerations:
      - effect: NoSchedule
        operator: Exists
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: containerd-metrics-proxy
  namespace: kube-system
spec:
  selector:
    app: containerd-metrics-proxy
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```
{% endhighlight %}
</details>

Voila, auf Port 8080 haben wir die versionierten Container Metriken und können die mit einer `scrape_config` abholen.

```yaml
      - job_name: 'containerd-metrics-proxy'
        kubernetes_sd_configs:
          - role: endpoints
            namespaces:
              names:
                - kube-system
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_name]
            separator: ;
            regex: metrics-proxy
            replacement: $1
            action: keep
          - source_labels: [__meta_kubernetes_service_name]
            target_label: job
          - source_labels: [__meta_kubernetes_endpoint_port_name]
            target_label: port
        metrics_path: /v1/metrics
```


Übrigens: Alle verwendeten Container Images gibt es auf [https://github.com/mcsps/docker-images](https://github.com/mcsps/docker-images).

Viel Spass!
