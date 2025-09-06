---
layout: post
tag: de
title: "Kubernetes Admission Controller im Eigenbau"
subtitle: "Kubernetes Admission Controller für Cosign Image Verifizierung gibt es schon am Markt. Wenn diese aber nicht den eigenen Ansprüchen genügen, baut man einen selber. Wie das geht, erfährst Du in diesem Artikel"
date: 2023-01-07
background: '/images/admission-control.png'
twitter: 'images/cosignwebhook.png'
author: eumel8
---

Neues Jahr, neues Projekt, könnte man sagen. Aber dieses begann am Ende des letzten, ist dennoch ein guter Start ins Neue.


# Docker-Container als solches
Die Verwendung von Docker-Containern in Kubernetes ist eine bekannte Notwendigkeit, 
auch wenn [dockerd](https://docker.io) mittlerweile durch [containerd](https://containerd.io/) ersetzt wurde. 
Das Vorgehen ist auch weit etabliert: Es gibt ein [Dockerfile](https://docs.docker.com/engine/reference/builder/) mit einer
Definition zu einem Basis-Image oder man startet `From: scratch` und dann bringt man die Programmlogik rein und installiert
Pakete.

Das Problem dabei ist, im Resultat weiss man nicht genau, was in dem Docker-Container drin ist, wenn man ihn
nicht selber erstellt hat. Dieses Problem hat mittlerweile einen Namen bekommen: Supply Chain Attack. Es ist nicht klar,
was der Container alles kann und welche Backdoors vielleicht eingebaut worden sind.

Praktisches Beispiel zur Hand: [https://hub.docker.com/r/gnuu/busybox](https://hub.docker.com/r/gnuu/busybox) - ein Container aus unserem [Gnuu](https://www.gnuu.de)
Projekt, was wir im Projekt selber vielleicht 100 mal runtergeladen haben. Docker Hub weist aber Downloadraten weit
über 10.000 auf! Selbst [https://hub.docker.com/r/gnuu/postfix](https://hub.docker.com/r/gnuu/postfix) hat mehr als 2500 Downloads, obwohl in der Beschreibung steht,
dass dieser Container ein spezielles Konstrukt ist und nur unter bestimmten Konfigurationen funktioniert. Hat jeder diese
Beschreibung gelesen? Hat jeder das referenzierte Dockerfile auf Github und die Build-Pipeline überprüft? Wahrscheinlich
nicht. Der Name ist cool, ich brauche ein Busybox-Image, und schon hat man sich irgendein Dreckszeug eingefangen.

# Container Signierung
Die Lösung des Problems des Unbekannten Containers existiert schon sehr lange: [Docker Content Trust](https://docs.docker.com/engine/security/trust/). Die Container werden mit einem Schlüssel signiert und die Signaturen in einem [Notariat](https://github.com/notaryproject/notary) abgelegt. Das Verfahren hatte seit seiner Erfindung schon starke Schlagseite. Die Signaturen waren nicht geschützt - ja, jedermann konnte sie sogar einfach löschen. Ausserdem war es nicht bis zu Ende gedacht, denn die Container mit den Signaturen mussten vor der Benutzung ja mal irgendwie verifiziert werden. Dazu gab es keine Software.

# Nächste Generation Container Signierung
Zum Glück hat die Containersicherheit in den letzten Jahren stark an Bedeutung gewonnen. Nicht nur, dass [Notary V2](https://notaryproject.dev/) ins Leben gerufen wurde, es entwickelten sich weitere Standards um den [OCI Layer](https://opencontainers.org/), die es ermöglichten, weitere Informationen zum Docker-Container am selben Platz zu speichern wie etwa seine Signaturen

# Sigstore
[Cosign Sigstore](https://www.sigstore.dev/) hat sich als neuer Standard für Container Signierung etabliert.
Der ganze Prozess aus Signierung und Verifizierung besteht aus 3 Kommandos:

```bash
$ cosign generate-key-pair
$ cosign sign --key cosign.key  mtr.devops.telekom.de/eumel8/test1:signed
$ cosign verify --key cosign.pub  mtr.devops.telekom.de/eumel8/test1:signed
```

Fertig!

# Admission Controller
Kubernetes Admission Controller sind Werkzeuge aus dem Zugriffs- und Authorisierungsmanagement, die Aufgaben im
Cluster sortieren und lenken. Verschiedene Admission Controller benutzen wir vielleicht schon, ohne es zu wissen:
ServiceAccounts, PodSecurity, PodSecurityPolicy, Priority, ResourceQuota - alles Admission Controller.

Man unterscheidet `ValidationWebhookConfiguration`, um Sachen zu validieren. Und `MutatingWebhookConfiguration`, um Sachen auch zu verändern.

Im [Cert Manager](https://github.com/cert-manager/cert-manager) gibt es zum Beispiel beides: Den ValidationWebhook, um Zertfikate und deren Gültigkeit zu verifizieren und MutationWebhook, um neue auszustellen.

Schematische Darstellung eines POD-Lebenszyklus mit Admission Controller:

<img src="/images/2023-01-07-1.png" width="585" height="386" />

# Admission Controller für Image Verifizierung
Die Überprüfung von Signaturen von Containern ist eine prädistinierte Aufgabe für einen Admission Controller.
Deswegen kommt er auch in zahlreichen Tools zum Einsatz:

## [Kubewarden](https://www.kubewarden.io/)
Ein Admission Controller, der einen ganzen [Hub](https://hub.kubewarden.io/) von Plugins für einen Policy Server bereithält.
Einer davon ist zur Verifizierung von Images zuständig. Der User kann Policies erstellen, die dann von dem Policy Server
durch eine neue Instanz umgesetzt werden. Da ist dann auch der Knackpunkt: ist eine Policy fehlerhaft, funktioniert cluster-weit der ganze Updateprozess nicht mehr. Ausserdem hat man ganz schönen Overhead, wenn man nur dieses eine Plugin nutzen will.
Und noch ein Manko: diese Policies haben wieder ein bestimmtes Mime-Format, die nicht von allen Registries unterstützt werden.

## [Sigstore Policy Controller](https://github.com/sigstore/policy-controller/)
Dieser versucht die Interaktionen mit Usern völlig zu umgehen, indem er einfach nur ClusterImagePolicy anbietet, die
dann nur von einem cluster-admin verwaltet werden dürfen. Über eine namespaced Lösung denkt man noch nach.

## [Kyverno](https://kyverno.io/policies/other/verify_image/)
Auch ein Werkzeug mit vielen Policy-Plugins. Eines ist zur Verifizierung von Images. Bereitgestellt werden diese in
ClusterPolicy, auch eine nicht-namespaced Resource.
Jetzt könnte man darauf kommen, dass jeder User sein eigenes Tool im shared Cluster installiert. Aber das funktioniert
natürlich nicht, wenn zentrale Resourcen wie ClusterRoles oder besagte WebHooks angelegt werden. Man könnte das alles ausnanderfieseln, die Webhooks noch mit Namespace-Selektoren versehen (technisch wäre das alles drin), aber wer will sowas verwalten.

## [Connaisseur](https://github.com/sse-secure-systems/connaisseur/)
Bei diesem Werkzeug werden die Policies im Helm-Chart mit installiert. In der Single-Instanz sehr praktisch, 
aber die zweite Installation im selben Cluster scheitert schon an gemeinsam genutzen Resourcen (siehe oben).

## [Open Policy Agent](https://github.com/sigstore/cosign-gatekeeper-provider)
Open Policy Agent(OPA) ist auch eher eine Werkzeugsammlung, in der ich mit einer Script-Sprache Policies
selbst entwerfen kann. Für Cosign gibt es zwar schon einen Provider, aber der überprüft nur, ob es eine Container-Signatur
gibt und nicht ob diese gültig ist. Aber an diesem könnte man anknüpfen, wenn man sich mit dieser Scriptsprache etwas
beschäftigt und solche Anwendungsfälle wie Signatur-Speicherort und private Images beachtet.

# Cosign Webhook
An dieser Stelle beginnt die Geschichte unseres eigenen Admission Controllers. Dieser beginnt mit einer Konfiguration:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: cosignwebhook
webhooks:
  - admissionReviewVersions:
    - v1
    name: cosignwebhook.caas.telekom.de
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["cosignwebhook", "kube-system", "cattle-system", "default"]
    clientConfig:
      service:
        name: cosignwebhook
        namespace: cosignwebhook
        path: "/validate"
      caBundle: "LS0..."
    rules:
      - operations: ["CREATE","UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    failurePolicy: Fail
    sideEffects: None
```

Wir schauen also wie in dem Schaubild oben nach Events im Cluster die PODs erstellen oder aktualisieren möchten.
Diese Information wird an einen Service `cosignwebhook` im Namespace `cosignwebhook` gesendet. Einige Namespaces sind
von der Validierung ausgenommen, wie etwa der Controller selber und  einige System-Namespaces.

Im `cosignwebook` Namespace haben wir also einen Service und einen dahinterliegenden Pod, bereitgestellt durch ein
Deployment mit dem eigentlichen Admission-Controller.
Es ist ein Webservice, also brauchen wir erstmal einen Webserver, der auf diese Anfragen antwortet:


```golang
func (cs *CosignServerHandler) serve(w http.ResponseWriter, r *http.Request) {
	var body []byte
	if r.Body != nil {
		if data, err := io.ReadAll(r.Body); err == nil {
			body = data
		}
	}
	if len(body) == 0 {
		glog.Error("empty body")
		http.Error(w, "empty body", http.StatusBadRequest)
		return
	}

	if r.URL.Path != "/validate" {
		glog.Error("no validate")
		http.Error(w, "no validate", http.StatusBadRequest)
		return
	}
```

Ziel-URL ist also `/validate`. Dort soll dann etwas passieren.

Das Objekt ist `AdmissionReview` (siehe [API-Beschreibung](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#validatingwebhook-v1-admissionregistration-k8s-io) und dort haben wir ein Pod-Objekt zur Überprüfung eingebettet:

```golang
arRequest := v1.AdmissionReview{}
	if err := json.Unmarshal(body, &arRequest); err != nil {
		glog.Error("incorrect body")
		http.Error(w, "incorrect body", http.StatusBadRequest)
		return
	}

	raw := arRequest.Request.Object.Raw
	pod := corev1.Pod{}
	if err := json.Unmarshal(raw, &pod); err != nil {
		glog.Error("error deserializing pod")
		return
	}
```

Zur Signatur-Überprüfung brauchen wir den Cosign-Public-Key. Der ist im POD in einer Environment-Variable abgelegt:

```golang
	pubKey := ""
	for i := 0; i < len(pod.Spec.Containers[0].Env); i++ {
		value := pod.Spec.Containers[0].Env[i].Value
		if pod.Spec.Containers[0].Env[i].Name == cosignEnvVar {
			pubKey = value
		}
	}
```

Dann brauchen wir den Image-Namen:

```golang
	image := pod.Spec.Containers[0].Image
	refImage, err := name.ParseReference(image)
```

Das Image könnte nicht-öffentlich sein. Dann brauchen wir die ImagePullSecrets:

```golang
	imagePullSecrets := make([]string, 0, len(pod.Spec.ImagePullSecrets))
	for _, s := range pod.Spec.ImagePullSecrets {
		imagePullSecrets = append(imagePullSecrets, s.Name)
	}
	opt := k8schain.Options{
		Namespace:          pod.Namespace,
		ServiceAccountName: pod.Spec.ServiceAccountName,
		ImagePullSecrets:   imagePullSecrets,
	}
```

Das Herzstück sind dann bloss noch ein paar Zeilen, die die Original-Cosign-Routine zum Verifizieren von
Signaturen verwendet. [k8schain](https://pkg.go.dev/github.com/google/go-containerregistry/pkg/authn/k8schain) ist eine nette Bibliothek von Google, die sich um das Zusammensammeln
der Secrets und das Einloggen in der Container-Registry kümmert, damit wir dort Zugriff auf das Image und
deren Signatur haben:

```golang
	kc, err := k8schain.NewInCluster(context.Background(), opt)
	remoteOpts := []ociremote.Option{ociremote.WithRemoteOptions(remote.WithAuthFromKeychain(kc))}
	_, _, err = cosign.VerifyImageSignatures(
		context.Background(),
		refImage,
		&cosign.CheckOpts{
			RegistryClientOpts: remoteOpts,
			SigVerifier:        cosignLoadKey,
		})
```

Das Ergebnis ist etwas komisch. Wenn kein Fehler zurückkommt, ist alles in Ordnung und das Image verifiziert.
Der Admission-Controller gibt einen Admission-Response `Success` zurück:

```golang
v1.AdmissionReview{
		TypeMeta: metav1.TypeMeta{
			Kind:       admissionKind,
			APIVersion: admissionApi,
		},
		Response: &v1.AdmissionResponse{
			Allowed: admissionPermissions,
			UID:     ar.Request.UID,
			Result: &metav1.Status{
				Status:  admissionStatus,
				Message: admissionMessage,
				Code:    admissionCode,
			},
		},
```

Die Container-Signatur wurde erfolgreich überprüft. Die Anfrage wird an den Scheduler weitergeleitet, der den POD letztlich
auf einen Node verteilt und der Kubelet den POD überwacht und startet.

Interessiert an den Details? [Cosign Webhook](https://github.com/eumel8/cosignwebhook) ist auf Github verfügbar, zusammen
mit Installationsanleitungen und Downloadmöglichkeiten.

Happy Secure Containering!
