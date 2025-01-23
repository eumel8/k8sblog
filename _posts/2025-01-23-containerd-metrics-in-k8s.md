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


-  container_cpu_usage_seconds_total

NAMESPACE!

[cAdvisor](https://github.com/google/cadvisor)

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

# Kube State Metrics

[Kube State Metrics](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-state-metrics)
https://github.com/mcsps/docker-images

- kube_deployment_created

# Containerd metrics

- containerd_cri_image_pulls_total

- Wieviel Pulls
- Wieviel Fehler
- Wieviel Traffic
https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus/values.yaml


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
            proxy_pass http://127.0.0.1:1338/metrics;
        }
    }
```

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


```bash
# cat /etc/containerd/config.toml 
# https://github.com/containerd/containerd/blob/main/docs/ops.md
# Use config version 2 to enable new configuration fields.
# Config file is parsed as version 1 by default.
version = 2

imports = ["/etc/containerd/conf.d/*.toml"]

[debug]
  level = "debug"

[metrics]
  address = "127.0.0.1:1338"
```

Für [Crio metrics](https://github.com/cri-o/cri-o/blob/main/tutorials/metrics.md)

