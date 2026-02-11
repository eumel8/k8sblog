---
layout: post
tag: de
title: Gitlab CI K8s
subtitle: Ende zu Ende Tests für Deine Applikation
1ate: 2026-02-10
background: '/images/gitlab-ci-k8s.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

Man muss mal seine alten Dämonen bezwingen. Seit Jahren verfolgt mich dieser Schmerz von Gitlab CI/CD Pipelines. Für das, was man bei Github Action mühelos in paar Minuten hinkleckst, kann man bei Gitlab CI/CD unter Umständen Stunden oder Tage verbringen. Man versucht sich in Pipelines, Commit für Commit, es ist erbarmungslos. 
Es wird Zeit, mal ein ultimatives Job-Template zu erstellen.

# Ende zu Ende Tests

Ende zu Ende Tests sind so etwas wie die Königsklasse der Softwareentwicklung, bedeutet es doch wie der Name schon sagt, eine Applikation von vorn bis hinten zu testen. In unserem Fall sind das meist Apps, die auf Kubernetes deployt werden. Oder ganze Helm Charts und ob diese sich fehlerfrei installieren lassen. Wenn man sowas automatisiert macht, kann man auch Matrixen bauen und etwa die App oder das Helm Chart mit verschiedenen Kubernetes-Versionen testen und schauen, "ob da was brickt", wie der Fachmann sagt.
Die Aufgabe ist es also, eine Pipeline zu erstellen, die zum einen einen vollwertigen Kubernetes Cluster bereitstellt, die App installiert und je nach Ergebnis grün meldet und fehlschlägt. 
Für Kubernetes bietet Gitlab dazu einen extra Service an, aber das bedarf eines externen Clusters. Und der muss nach jedem Pipeline-Lauf ja wieder zurückgesetzt werden, wenn man etwa CRDs oder andere cluster-nahe Sachen installiert. 
Also muss das alles in den Runner.

# Gitlab Runner

Gitlab Runner waren früher extra Maschinen, auf denen ein Runner-Agent lief, der Job-Aufträge entgegennahm und auf der Shell ausführte. Nicht besonders sicher, wenn man vor allem Shared Runner zwischen verschiedenen Projekten verwendet. Ausserdem brauch man recht viel Resourcen und es skaliert nicht richtig.

Die nächste Generation waren die Docker-Runner, dort läuft also ein Docker-Container auf einer VM oder physikalischen Maschine. Immer noch recht aufwendig. 

Die letzte Generation sind die Kubernetes-Runner. Da läuft der Gitlab-Job in einem Pod, der vom Runner-Pod gestartet wird. Sehr sicher, sehr isoliert, um einem Kubernetes-Cluster da drin zu installieren, nicht unbedingt geeignet. Meine Idee war da vCluster, wir hatten hier mal eine [System Demo](https://github.com/caas-team/system-demos/blob/main/2024-01/vcluster/vcluster-vc1.md), aber das bedarf einer vCluster-Infrastruktur, aber die hat man nicht unbedingt zur Verfügung. Man könnte also nur in der Pipeline als pre-job einen Cluster oder vCluster irgendwo spawnen, was dauert, Zeit kostet und fehleranfällig ist. Fokus ist immer noch der Ende zu Ende Test unserer Applikation.

Schauen wir uns den Docker-Runner nochmal genauer an. Es gibt ein paar etablierte Mini-Kubernetes-Installationen, die auf DIND basieren, Docker in Docker. Als da wären

* kind
* k3d (mit k3s)

Auf einer Ubuntu-VM können wir zum Beispiel starten:

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" -o script.deb.sh
sudo bash script.deb.sh
sudo apt install gitlab-runner
```

Das installiert uns das Gitlab-Runner-Paket. In der Gitlab-Weboberfläche können wir in den CI/CD-Einstellungen einen neuen Runner registrieren. Wir kriegen dazu einen Token und haben die Gitlab-Url. Damit können wir den Runner von Hand registrieren:

```bash
sudo gitlab-runner register -n \
  --url "http://192.168.0.41/" \
  --registration-token glrt-xxxxxxx \
  --executor docker \
  --description "dind" \
  --tag-list "dind" \
  --docker-image "docker:29.2-cli" \
  --docker-privileged \
  --docker-volumes "/certs/client"
```

Der Knackpunkt ist vielleicht schon ersichtlich: `--docker-privileged`. Das wird auf shared Umgebungen immer deaktiviert. Bis docker:28-dind kommt man aber auch ohne dem Parameter zurecht. Danach wird's schwierig.

# gitlab-ci.yml

Hier ist das [Gist](https://gist.github.com/eumel8/db8eb4bf32e551eb8ca96a9c5b3372ff):

<script src="https://gist.github.com/eumel8/db8eb4bf32e551eb8ca96a9c5b3372ff.js"></script>

Was haben wir hier? Die Pipeline startet mit docker:dind, installiert uns kubectl, helm und kind.

Der Kind-Cluster wird jetzt in dem vom Runner gestarteten Docker-Image einen Docker starten und dort drin dann Kubernetes installieren. 
Damit man an die API rankommt, wird die Listen-Address auf 0.0.0.0 gestellt. Nicht besonders sicher, aber es ist ja nur eine Pipeline.
Und damit wir vom Docker host des Runners die API über SSL erreichen, brauchen wir einen gültigen SAN im Zertifikat, was wir bei der Cluster-Erstellung mit angeben. In der extrahierten Kube-Config müssen wir dann diesen Hostnamen mit sed wieder ersetzen, um letztlich Zugriff zu erlangen.

Der K3D ist wesentlich wartungsärmer. Runterladen, starten, fröhlich sein. Er dient auch nur als Hülle für den K3S Cluster da drin. Deswegen darf man sich nicht von den Versionen irre machen lassen:

* [Kind](https://github.com/kubernetes-sigs/kind/releases) Version v0.31.0 (Kubernetes 1.35)
* [K3D](https://github.com/k3d-io/k3d/releases/) Version v5.8.3 ([K3S](https://github.com/k3s-io/k3s/releases) Kubernetes v1.35.0+k3s3)

Beim Kind Cluster ist noch zu beachten, dass dieser erheblich mehr Resourcen brauch als der K3S Cluster im K3D. Vom DIND wird nur 1 CPU durchgereicht, oder man muss das im Gitlab-Runner anpassen

```toml
[runners.docker]
  cpus = "4"
  memory = "8g"
```

Alternativ kann man im Kind Manifest die Anzahl der Nodes erhöhen, was einerseits eh ganz gut ist, um etwa Daemonsets zu testen. Andererseits die Sache auch schon wieder unnötig komplexer macht. Muss man selber wissen.

# Matrix

In der Matrix werden die Jobs in der Pipeline mit der jeweiligen Variable ausgeführt. Die Syntax wurde der von Github Action angepasst:

```yaml

k3d-cluster:
  stage: deploy
  parallel:
    matrix:
      - K8S_VERSION: ["v1.35.0-k3s3", "v1.34.3-k3s3"]
```

Pipeline im Gitlab:

<img src="/images/k8s-ci-cd-pipeline.png"/>

# Ende zu Ende

Zum Schluss noch eine Idee für ein Ende zu Ende Test eines Helm Charts. Kann man sicher noch parametisieren, damit man es wiederverwenden kann. Eine Grundidee ist ein eignes `values-e2e.yaml`, in dem ich Resourcen anpassen oder Features an- und ausschalten kann. Auch kann man mit `end2end: true` Konditionen in den Templates setzen, die nur bei den Ende zu Ende Tests ausgeführt werden.

```
#!/bin/bash
set -e

# Configuration
NAMESPACE="observability-system"
RELEASE_NAME="monitoring-observability"
SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
CHART_PATH="${SCRIPT_DIR}/.."
VALUES_FILE="${CHART_PATH}/values-e2e.yaml"

echo "🚀 Starting E2E Test for monitoring-observability..."
echo "📂 Chart Path: $CHART_PATH"
echo "📄 Values File: $VALUES_FILE"

# Check Prerequisites
if ! command -v helm &> /dev/null; then
    echo "❌ Helm is not installed."
    exit 1
fi

if ! command -v kubectl &> /dev/null; then
    echo "❌ Kubectl is not installed."
    exit 1
fi

# 1. Install Flux CRDs ---
echo "Installing Flux CRDs..."
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml

# Wait for Flux CRDs to be ready
echo "Waiting for Flux CRDs to be established..."
kubectl wait --for condition=established --timeout=120s crd/helmreleases.helm.toolkit.fluxcd.io crd/helmrepositories.source.toolkit.fluxcd.io


# 2. Prepare Cluster
echo "🔧 Preparing Namespaces..."
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
kubectl config set-context --current --namespace=$NAMESPACE

# 3. Dependency Build
echo "📦 Building Helm dependencies..."
helm dependency build

# 4. Install/Upgrade Chart
echo "🚀 Installing Helm Chart..."
helm upgrade --install $RELEASE_NAME $CHART_PATH \
  -f $VALUES_FILE \
  -n $NAMESPACE \
  --create-namespace \
  --wait \
  --timeout 5m

# 5. Verification
echo "🔍 Verifying Deployments..."
kubectl get pods -n $NAMESPACE --show-labels
kubectl get events -n $NAMESPACE

# Check critical components
COMPONENTS=("goldpinger" "prometheus" "kube-state-metrics" "prometheus-node-exporter" "storagecheck")

FAILED=0
for app in "${COMPONENTS[@]}"; do
    if kubectl get pods -n $NAMESPACE -l "app.kubernetes.io/name=$app" | grep -q "Running"; then
        echo "✅ Component '$app' is Running"
    else
        echo "⚠️  Warning: Component '$app' not found or not running immediately (check logs)."
        # We don't fail immediately here because names might vary slightly in charts
    fi
done

if kubectl get pods -n kube-system -l "app=containerd-metrics-proxy" | grep -q "Running"; then
   echo "✅ Component 'containerd-metrics-proxy' is Running"
else
   echo "⚠️  Warning: Component 'containerd-metrics-proxy' not found or not running immediately (check logs)."
fi

echo "✅ E2E Test Installation Phase Completed!"
echo "➡️  You can now check specific functionality (e.g., via port-forward)."
```

ref: [https://blog.cesc.cool/guide-for-a-gitlab-job-and-kind](https://blog.cesc.cool/guide-for-a-gitlab-job-and-kind)
