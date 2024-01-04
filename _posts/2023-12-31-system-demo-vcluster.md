---
layout: post
tag: de
title: System Demo Vcluster
subtitle: Istio Service Mesh mit Vcluster
date: 2023-12-31
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro
Vcluster ist eine Technologie, um virtuelle Kubernetes Cluster im Kubernetes Cluster abzubilden. Was sind die Vorteile?
Hier ein Beispiel eines Istio Service Mesh mit zwei Vcluster-Instanzen in einem Kubernetes Cluster.

<img src="/images/systemdemo-vcluster.png"/>

# Vcluster Installation
Für die Installation steht uns ein von Rancher verwalteter Kubernetes Cluster zur Verfügung. Wir haben ein Projekt angelegt und im Projekt zwei Namespaces für zwei Vcluster. Wir brauchen als Werkzeuge Helm CLI, vcluster, kubectl, openssl, istioctl, make.

Die Installation ist denkbar einfach und sollte ohne jegliche Änderung der Default-Values funktionieren. Wir deaktivieren in der Demo aber die NetworkPolicy Isolation und setzen einen extra TLS-SAN, damit das Vcluster-Zertifikat für unseren späteren externen Endpunkt gültig ist. Diese Option brauchen wir auch nur bei Traefik-Ingress-Controller, bei Ingress-Aktivierung im Helm-Chart wird diese automatisch gesetzt (siehe nächstes Kapitel):

```bash
helm -n vc1 upgrade -i vc1 --set isolation.networkPolicy.enabled=false --set syncer.extraArgs={--tls-san=vc1.otc.mcsps.de} --version v0.15.7 oci://mtr.devops.telekom.de/caas/charts/vcluster
helm -n vc2 upgrade -i vc2 --set isolation.networkPolicy.enabled=false --set syncer.extraArgs={--tls-san=vc2.otc.mcsps.de} --version v0.15.7 oci://mtr.devops.telekom.de/caas/charts/vcluster
```

Kontrolle:

```bash
vcluster -n vc1 list

  NAME | CLUSTER | NAMESPACE | STATUS  | VERSION | CONNECTED |            CREATED            |    AGE     | DISTRO
-------+---------+-----------+---------+---------+-----------+-------------------------------+------------+---------
  vc1  | local   | vc1       | Running | v0.15.7 |           | 2023-12-21 17:09:16 +0100 CET | 201h30m22s | OSS

vcluster -n vc2 list

  NAME | CLUSTER | NAMESPACE | STATUS  | VERSION | CONNECTED |            CREATED            |    AGE     | DISTRO
-------+---------+-----------+---------+---------+-----------+-------------------------------+------------+---------
  vc2  | local   | vc2       | Running | v0.15.7 |           | 2023-12-21 17:09:16 +0100 CET | 201h30m22s | OSS
```

Standardmässig kann Netzwerkverkehr verboten sein. Dieser muss mit NetworkPolicies explizit freigeschalten werden. Hier eine In/Out any/any Rule für die  Demo.

vc1: 

```bash
cat <<EOF | kubectl -n vc1 apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-vc-allow-all
spec:
  egress:
  - {}
  ingress:
  - {}
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

vc2: 

```bash
cat <<EOF | kubectl -n vc2 apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-vc-allow-all
spec:
  egress:
  - {}
  ingress:
  - {}
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

Um den Vcluster zu erreichen, kann man mit `vcluster connect` entweder kubectl aufrufen oder mit bash einen Kommandoprompt ausführen, um dann von dort mit kubectl/helm weiter zu arbeiten.

# Vcluster KubeAPI Zugriff

Um auf die KubeAPI vom Vcluster zugreifen zu können, müssen wir einen Service exposen. Das machen wir beim Traefik Ingress Controller mit einer IngressRouteTCP:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: vc1
  namespace: vc1
spec:
  entryPoints:
  - websecure
  routes:
  - match: HostSNI("vc1.otc.mcsps.de")
    services:
    - name: vc1
      port: 443
  tls:
    passthrough: true
EOF
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: vc2
  namespace: vc2
spec:
  entryPoints:
  - websecure
  routes:
  - match: HostSNI("vc2.otc.mcsps.de")
    services:
    - name: vc2
      port: 443
  tls:
    passthrough: true
EOF
```

Beim Nginx Ingress Controller, aktivieren wir das Feature bei der Vcluster Installation:

```bash
helm -n vc1 upgrade -i vc1 --set isolation.networkPolicy.enabled=false --set ingress.enabled=true --set ingress.host=vc1.otc.mcsps.de --set ingress.annotation="nginx.ingress.kubernetes.io/ssl-passthrough=true" --set ingress.annotation="nginx.ingress.kubernetes.io/backend-protocol=HTTPS" --version v0.15.7  oci://mtr.devops.telekom.de/caas/charts/vcluster
helm -n vc2 upgrade -i vc2 --set isolation.networkPolicy.enabled=false --set ingress.enabled=true --set ingress.host=vc2.otc.mcsps.de --set ingress.annotation="nginx.ingress.kubernetes.io/ssl-passthrough=true" --set ingress.annotation="nginx.ingress.kubernetes.io/backend-protocol=HTTPS" --version v0.15.7  oci://mtr.devops.telekom.de/caas/charts/vcluster
```

Jetzt brauchen wir noch einen Benutzer mit Cluster-Admin-Rechten.

Dafür legen wir ein ClusterRoleBinding an:

```bash
vcluster -n vc1 connect vc1 -- bash
source  <(kubectl completion bash)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vc-istio-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- name: u-istio
  kind: User
  apiGroup: rbac.authorization.k8s.io
EOF
exit
```

```bash
vcluster -n vc2 connect vc2 -- bash
source  <(kubectl completion bash)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vc-istio-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- name: u-istio
  kind: User
  apiGroup: rbac.authorization.k8s.io
EOF
exit
```

Jetzt erstellen wir ein Client-Zertifikat für diesen User:

```bash
openssl genpkey -out u-istio.key -algorithm Ed25519
openssl req -new -key u-istio.key -out u-istio.csr -subj "/CN=u-istio/O=admin"
cat u-istio.csr | base64 | tr -d "\n"
```

Schicken ein CertificateSigningRequest:

```bash
vcluster -n vc1 connect vc1 -- bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: u-istio
spec:
  request: $(cat u-istio.csr | base64 | tr -d "\n")
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000  # 365 day
  usages:
  - client auth
EOF
```

Genehmigen diesen

```bash
kubectl certificate approve u-istio
```

Kriegen das genehmigte Zertifikat, verschlüsseln den Zertifikatsschlüssel mit base64 und holen uns noch das Serverzertifikat vom Vcluster, was wir ebenfalls base64 verschlüsseln:

```bash
kubectl get csr/u-istio -o jsonpath="{.status.certificate}"
exit
```

auf vc2:

```bash
vcluster -n vc2 connect vc2 -- bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: u-istio
spec:
  request: $(cat u-istio.csr | base64 | tr -d "\n")
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000  # 365 day
  usages:
  - client auth
EOF
kubectl certificate approve u-istio
kubectl get csr/u-istio -o jsonpath="{.status.certificate}"
exit
```

der Schlüssel:

```bash
cat u-istio.key | base64 -w 0
```

Kontext Host-Cluster:

```bash
kubectl -n vc1 exec -it vc1-0 -- cat  /data/server/tls/server-ca.crt| base64 -w 0
kubectl -n vc2 exec -it vc2-0 -- cat  /data/server/tls/server-ca.crt| base64 -w 0
```

Die 5 Fragmente basteln wir in eine KUBECONFIG Datei:

```bash
cat <<EOF > /tmp/vc-config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <server-ca.crt vc1>
    server: https://vc1.otc.mcsps.de:443
  name: vc1
- cluster:
    certificate-authority-data: <server-ca.crt vc2>
    server: https://vc2.otc.mcsps.de:443
  name: vc2
contexts:
- context:
    cluster: vc1
    user: u-istio-1
  name: vc1
- context:
    cluster: vc2
    user: u-istio-2
  name: vc2
current-context: vc2
kind: Config
preferences: {}
users:
- name: u-istio-1
  user:
    client-certificate-data: <u-istio cert vc1>
    client-key-data: <u-istio key>
- name: u-istio-2
  user:
    client-certificate-data: <u-istio cert vc2>
    client-key-data: <u-istio key>
EOF
```

Kontrolle:

```bash
$ export KUBECONFIG=/tmp/vc-config
$ kubectl config use-context vc1
$ kubectl get ns
NAME                   STATUS   AGE
default                Active   12h
kube-system            Active   12h
kube-public            Active   12h
kube-node-lease        Active   12h
$ kubectl config use-context vc2
$ kubectl get ns
NAME                   STATUS   AGE
default                Active   12h
kube-system            Active   12h
kube-public            Active   12h
kube-node-lease        Active   12h
```

Wenn wir nicht alle Kommandozeilenwerkzeuge parat haben oder uns Netzwerkverbindung zum Ingres fehlt, können wir uns auch einen POD als Arbeitsumgebung installieren:


```bash
cat <<EOF > vcluster-client.yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: vc-client
  name: vc-client
  namespace: vc1
spec:
  serviceName: vc-client
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: vc-client
  template:
    metadata:
      labels:
        app: vc-client
    spec:
      containers:
      - image: mtr.devops.telekom.de/caas/k8s-tools:latest
        imagePullPolicy: Always
        command: ['sh', '-c']
        args: ["tail -f /dev/null"]
        name: vc-client
        resources:
          limits:
            cpu: 1000m
            memory: 1024Mi
          requests:
            cpu: 100m
            memory: 128Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          privileged: false
          readOnlyRootFilesystem: true
          runAsUser: 1000
          runAsGroup: 1000
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - name: workdir
          mountPath: /home/appuser
        - name: tmp
          mountPath: /tmp
      dnsPolicy: ClusterFirst
      hostNetwork: false
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1000
        supplementalGroups:
        - 1000
      terminationGracePeriodSeconds: 3
      serviceAccountName: vc-client
      volumes:
      - name: workdir
        emptyDir: {}
      - name: tmp
        emptyDir:
          medium: Memory
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app: vc-client
  name: vc-client
  namespace: vc1
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app: vc-client
  name: vc-client
  namespace: vc1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
  - kind: ServiceAccount
    name: vc-client
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app: vc-client
  name: vc-client
  namespace: vc2
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
  - kind: ServiceAccount
    name: vc-client
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  annotations:
  name: np-vc-client
  namespace: vc1
spec:
  egress:
  - {}
  podSelector:
    matchLabels:
      app: vc-client
  policyTypes:
  - Egress
EOF
```

Dieser Client läuft im Namespace vc1 und hat Zugriffsrechte auf die Namespace vc1 und vc2


```bash
kubectl apply -f vcluster-client.yaml
kubectl -n vc1 cp /tmp/vc-config vc-client-0:/tmp/vc-config
```

# Kubernetes Dashboard

Als Beifang können wir uns an dieser Stelle das [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) installieren. Eine lauffähige Version befindet sich [hier](https://gist.githubusercontent.com/eumel8/0f6d0bc19a25376ff541344e601a1d65/raw/bbdb47868ac077a60eaecd6ec75224ee2bc8c6e8/dashboard.yaml).

Zur Installation können wir jetzt unsere neue KUBECONFIG Datei benutzen:

```bash
export KUBECONFIG=/tmp/vc-config 
kubectl config use-context vc2
kubectl apply -f https://gist.githubusercontent.com/eumel8/0f6d0bc19a25376ff541344e601a1d65/raw/bbdb47868ac077a60eaecd6ec75224ee2bc8c6e8/dashboard.yaml
kubectl config use-context vc1
kubectl apply -f https://gist.githubusercontent.com/eumel8/0f6d0bc19a25376ff541344e601a1d65/raw/bbdb47868ac077a60eaecd6ec75224ee2bc8c6e8/dashboard.yaml
```

Das Kubernetes Dashboard bietet leider keine Two-Factor-Auth oder ein anderes zentral externes Authentifizierunssystem an.
Wenn das Dashboard installiert ist, müssen wir uns zum Login einen Token generieren:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```
Den admin-user haben wir bei der Installation des Dashboard schon mit angelegt.

Der Zugriff von aussen wieder über eine IngressRouteTCP im Context des Host-Clusters

```bash
cat <<EOF | kubectl apply -f -
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: vc1-dashboard
  namespace: vc1
spec:
  entryPoints:
  - websecure
  routes:
  - match: HostSNI("vc1-dashboard.otc.mcsps.de")
    services:
    - name: kubernetes-dashboard-x-kubernetes-dashboard-x-vc1
      port: 443
  tls:
    passthrough: true
EOF
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: vc2-dashboard
  namespace: vc2
spec:
  entryPoints:
  - websecure
  routes:
  - match: HostSNI("vc2-dashboard.otc.mcsps.de")
    services:
    - name: kubernetes-dashboard-x-kubernetes-dashboard-x-vc2
      port: 443
  tls:
    passthrough: true
EOF
```

# Istio Installation

Wir verwenden Istio mit zwei Master Instanzen im selben Service-Mesh und im selben Netzwerk. Eine Pod-zu-Pod-Kommunikation ist möglich. Das funktioniert nur, wenn sich beide Instanzen im selben Rancher-Projekt befinden und keine Networkpolicy den Netzwerkverkehr verbietet. Eine Kommunikation über zwei unterschiedliche Netzwerk wäre auch möglich, bedarf aber eines East-West-Gateways, was nicht Thema dieser System-Demo sein soll.

Die Installation von Istio beginnt auf beiden VClusters mit dem Anlegen der Namepaces und einiger Secrets.
Dazu klonen wir uns das Istio Repo für ein Demo-Setup:

```bash
mkdir /tmp/vc-demo
cd /tmp/vc-demo
git clone https://github.com/istio/istio.git
```

```bash
kubectl config use-context vc1
kubectl create namespace istio-system
kubectl config use-context vc2
kubectl create namespace istio-system
```

Anlegen von Serverzertifikaten von einer Demo-CA:

```bash
make -f istio/tools/certs/Makefile.selfsigned.mk root-ca
make -f istio/tools/certs/Makefile.selfsigned.mk vc1-cacerts
make -f istio/tools/certs/Makefile.selfsigned.mk vc2-cacerts

kubectl config use-context vc1
kubectl -n istio-system create secret generic cacerts --from-file=vc1/ca-cert.pem --from-file=vc1/ca-key.pem --from-file=vc1/cert-chain.pem --from-file=vc1/root-cert.pem
kubectl config use-context vc2
kubectl -n istio-system create secret generic cacerts --from-file=vc2/ca-cert.pem --from-file=vc2/ca-key.pem --from-file=vc2/cert-chain.pem --from-file=vc2/root-cert.pem

```

Von unserer zusammengebastelten KUBECONFIG erstellen wir remote secrets. Diese sind wichtig zur Service Discovery in Istio

```bash
istioctl create-remote-secret --context=vc1 --name=vc1 | kubectl apply -f - --context=vc2
istioctl create-remote-secret --context=vc2 --name=vc2 | kubectl apply -f - --context=vc1 
```

Nun installieren wir die IstioOperator, nicht zu verwechseln mit dem Operator - den installieren wir hier nicht, weswegen wir `ignoreReconcile` auf `true` stellen. Wir müssen also unsere Resource immer manuell ändern bzw. neu deployen.


```bash
kubectl config use-context vc1
cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  annotations:
    install.istio.io/ignoreReconcile: "true"
  name: istiooperator
  namespace: istio-system
spec:
  components:
    base:
      enabled: true
    cni:
      enabled: false
    egressGateways:
    - enabled: false
      name: istio-egressgateway
    ingressGateways:
    - enabled: true
      name: istio-ingressgateway
      k8s:
        securityContext:
          fsGroup: 2000
          runAsGroup: 2000
          runAsUser: 2000
          supplementalGroups:
            - 2000
    istiodRemote:
      enabled: false
    pilot:
      enabled: true
      k8s:
        securityContext:
          fsGroup: 2000
          runAsGroup: 2000
          runAsUser: 2000
          supplementalGroups:
            - 2000
  hub: mtr.devops.telekom.de/istio
  meshConfig:
    defaultConfig:
      proxyMetadata: {}
    enablePrometheusMerge: true
  profile: minimal
  tag: 1.20.1
  values:
    base:
      enableCRDTemplates: false
      validationURL: ""
    defaultRevision: ""
    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
        env: {}
        name: istio-egressgateway
        secretVolumes:
        - mountPath: /etc/istio/egressgateway-certs
          name: egressgateway-certs
          secretName: istio-egressgateway-certs
        - mountPath: /etc/istio/egressgateway-ca-certs
          name: egressgateway-ca-certs
          secretName: istio-egressgateway-ca-certs
        type: ClusterIP
      istio-ingressgateway:
        autoscaleEnabled: false
        env: {}
        name: istio-ingressgateway
        secretVolumes:
        - mountPath: /etc/istio/ingressgateway-certs
          name: ingressgateway-certs
          secretName: istio-ingressgateway-certs
        - mountPath: /etc/istio/ingressgateway-ca-certs
          name: ingressgateway-ca-certs
          secretName: istio-ingressgateway-ca-certs
        type: ClusterIP
    global:
      configValidation: true
      defaultNodeSelector: {}
      defaultPodDisruptionBudget:
        enabled: true
      defaultResources:
        requests:
          cpu: 10m
      imagePullPolicy: ""
      imagePullSecrets: []
      istioNamespace: istio-system
      istiod:
        enableAnalysis: false
      jwtPolicy: third-party-jwt
      logAsJson: false
      logging:
        level: default:error
      meshID: mesh1
      meshNetworks: {}
      mountMtlsCerts: false
      multiCluster:
        clusterName: vc1
        enabled: true
      network: network1
      omitSidecarInjectorConfigMap: false
      oneNamespace: false
      operatorManageWebhooks: false
      pilotCertProvider: istiod
      proxy:
        autoInject: enabled
        clusterDomain: cluster.local
        componentLogLevel: misc:error
        enableCoreDump: false
        excludeIPRanges: ""
        excludeInboundPorts: ""
        excludeOutboundPorts: ""
        image: proxyv2
        includeIPRanges: '*'
        logLevel: warning
        privileged: false
        readinessFailureThreshold: 4
        readinessInitialDelaySeconds: 0
        readinessPeriodSeconds: 15
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 100m
            memory: 128Mi
        startupProbe:
          enabled: true
          failureThreshold: 600
        statusPort: 15020
        tracer: zipkin
      proxy_init:
        image: proxyv2
      useMCP: false
    pilot:
      autoscaleEnabled: false
      image: pilot
    telemetry:
      enabled: false
EOF
``` 

```bash
kubectl config use-context vc2
cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  annotations:
    install.istio.io/ignoreReconcile: "true"
  name: istiooperator
  namespace: istio-system
spec:
  components:
    base:
      enabled: true
    cni:
      enabled: false
    egressGateways:
    - enabled: false
      name: istio-egressgateway
    ingressGateways:
    - enabled: true
      name: istio-ingressgateway
      k8s:
        securityContext:
          fsGroup: 2000
          runAsGroup: 2000
          runAsUser: 2000
          supplementalGroups:
            - 2000
    istiodRemote:
      enabled: false
    pilot:
      enabled: true
      k8s:
        securityContext:
          fsGroup: 2000
          runAsGroup: 2000
          runAsUser: 2000
          supplementalGroups:
            - 2000
  hub: mtr.devops.telekom.de/istio
  meshConfig:
    defaultConfig:
      proxyMetadata: {}
    enablePrometheusMerge: true
  profile: minimal
  tag: 1.20.1
  values:
    base:
      enableCRDTemplates: false
      validationURL: ""
    defaultRevision: ""
    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
        env: {}
        name: istio-egressgateway
        secretVolumes:
        - mountPath: /etc/istio/egressgateway-certs
          name: egressgateway-certs
          secretName: istio-egressgateway-certs
        - mountPath: /etc/istio/egressgateway-ca-certs
          name: egressgateway-ca-certs
          secretName: istio-egressgateway-ca-certs
        type: ClusterIP
      istio-ingressgateway:
        autoscaleEnabled: false
        env: {}
        name: istio-ingressgateway
        secretVolumes:
        - mountPath: /etc/istio/ingressgateway-certs
          name: ingressgateway-certs
          secretName: istio-ingressgateway-certs
        - mountPath: /etc/istio/ingressgateway-ca-certs
          name: ingressgateway-ca-certs
          secretName: istio-ingressgateway-ca-certs
        type: ClusterIP
    global:
      configValidation: true
      defaultNodeSelector: {}
      defaultPodDisruptionBudget:
        enabled: true
      defaultResources:
        requests:
          cpu: 10m
      imagePullPolicy: ""
      imagePullSecrets: []
      istioNamespace: istio-system
      istiod:
        enableAnalysis: false
      jwtPolicy: third-party-jwt
      logAsJson: false
      logging:
        level: default:error
      meshID: mesh1
      meshNetworks: {}
      mountMtlsCerts: false
      multiCluster:
        clusterName: vc2
        enabled: true
      network: network1
      omitSidecarInjectorConfigMap: false
      oneNamespace: false
      operatorManageWebhooks: false
      pilotCertProvider: istiod
      proxy:
        autoInject: enabled
        clusterDomain: cluster.local
        componentLogLevel: misc:error
        enableCoreDump: false
        excludeIPRanges: ""
        excludeInboundPorts: ""
        excludeOutboundPorts: ""
        image: proxyv2
        includeIPRanges: '*'
        logLevel: warning
        privileged: false
        readinessFailureThreshold: 4
        readinessInitialDelaySeconds: 0
        readinessPeriodSeconds: 15
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 100m
            memory: 128Mi
        startupProbe:
          enabled: true
          failureThreshold: 600
        statusPort: 15020
        tracer: zipkin
      proxy_init:
        image: proxyv2
      useMCP: false
    pilot:
      autoscaleEnabled: false
      image: pilot
    telemetry:
      enabled: false
EOF
``` 

Die Funktionsweise von Istio können wir erstmal anhand der Logs überprüfen:

```bash
kubectl -n istio-system logs -l app=istiod -f
kubectl -n istio-system logs -l app=istio-ingressgateway -f
```

Weitere Funktionstest vom Multi-Service-Mesh mit der Istio-Demo-App. Bis zu dieser Stelle kommen wir ohne Root-Zugriff oder erweiterten Schreibrechten bei der Installation aus. Jetzt möchte Istio aber den Netzwerkverkehr von Applikationen umleiten. Das macht er mit `istio-iptables`, ein Wrapper für `iptables`. Das läuft entweder als root im Injection-Sidecar der Apps oder als CNI-Plugin mit HostPath.
Für ersteres sind folgende securitySettings notwendig:

```yaml
      allowPrivilegeEscalation: false                
      capabilities:                                  
        add:                                                 
        - NET_ADMIN                                         
        - NET_RAW                                    
        drop:                                        
        - ALL                                        
      privileged: false                                                                                                                                 
      readOnlyRootFilesystem: false                                               
      runAsGroup: 0                      
      runAsNonRoot: false               
      runAsUser: 0 
```

Diese werden durch Templates aus der Configmap `istio-sidecar-injector` im istio-system Namespace gezogen. Es lohnt sich aber nicht, diese anzupassen. Es sind wirklich die minimalsten Rechte.

Wenn man Gatekeeper verwendet, kann man die Ausname mit `exempt_images: - mtr.devops.telekom.de/istio/proxyv2:1.20.1` auf die Constraints Capabilities, AllowedUsers und ReadOnlyRootFilesystem einschränken. Das heisst, Container nur mit diesem Image, welches man unter Kontrolle hat, darf diese 3 Sicherheitsregeln brechen. Das Image hat auch das `istio-iptables` Kommando geladen und wirkt sich, ohne HostNetwork nur auf den Netzwerkbereich des Pods aus. Trotzdem wird das verbleibende Sicherheitsrisiko hier erwähnt.


# Helloworld v1/v2

Istio stellt auch eine [Demo App zur Verfügung](https://istio.io/latest/docs/setup/install/multicluster/verify/), die in Versuin 1 auf Vcluster1 und in Version 2 auf VCluster 2 zu deployen ist. Eine [angepasste Version](https://github.com/mcsps/use-cases/blob/master/istio/helloworld.yaml)

```bash
kubectl config use-context vc1
kubectl create ns sample
kubectl label namespace sample istio-injection=enabled
kubectl -n sample apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/istio/helloworld.yaml -l version=v1
kubectl -n sample apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/istio/helloworld.yaml -l service=helloworld
```

```bash
kubectl config use-context vc2
kubectl create ns sample
kubectl label namespace sample istio-injection=enabled
kubectl -n sample apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/istio/helloworld.yaml -l version=v2
kubectl -n sample apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/istio/helloworld.yaml -l service=helloworld
```

An der Stelle könnten wir noch einen Ingress deployen, um auf die App zugreifen zu können.

Istio liefert noch eine curl app dazu, um die Verbindung überprüfen zu können. Machen wir es so auf Vcluster1

```bash
kubectl config use-context vc1
kubectl -n sample apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/istio/sleep.yaml
kubectl -n sample get pods
```

Wenn alles so geklappt hat, sollte wechselweise, je nach Antwortzeit die v1 Version vom Vcluster1 oder die v2 Version vom Vcluster2 antworten:

```bash
kubectl config use-context vc1
$ for i in {1..12}; do kubectl -n sample exec -it sleep-76cc9846f7-vtm4r -- curl -sS helloworld.sample:5000/hello;done
Hello version: v1, instance: helloworld-v1-54864596f9-7897x
Hello version: v1, instance: helloworld-v1-54864596f9-7897x
Hello version: v2, instance: helloworld-v2-c4b799cd-4zq7h
Hello version: v2, instance: helloworld-v2-c4b799cd-4zq7h
Hello version: v1, instance: helloworld-v1-54864596f9-7897x
Hello version: v1, instance: helloworld-v1-54864596f9-7897x
Hello version: v2, instance: helloworld-v2-c4b799cd-4zq7h
Hello version: v1, instance: helloworld-v1-54864596f9-7897x
Hello version: v2, instance: helloworld-v2-c4b799cd-4zq7h
Hello version: v1, instance: helloworld-v1-54864596f9-7897x
Hello version: v2, instance: helloworld-v2-c4b799cd-4zq7h
Hello version: v2, instance: helloworld-v2-c4b799cd-4zq7h
```

