---
layout: post
tag: en
title: "Self-Made Kubernetes Admission Controller"
subtitle: "Kubernetes Admission Controller for Cosign image verification are many. But if these do not meet your own requirements, you build one yourself. You can find out how to do this in this article"
date: 2023-01-07
background: '/images/k8s-cosmos.png'
twitter: 'images/cosignwebhook.png'
author: eumel8
---

New Year, new project, just saying. But it started at the end of the last, and it's still a good start in new New Year.

# Docker container like that
The usage of Docker container in Kubernetes is a known requirement also when [dockerd](https://docker.io) is replaced by
[containerd](https://containerd.io/). 
The workflow is established: There is a [Dockerfile](https://docs.docker.com/engine/reference/builder/) with a definition
to a base image or we start `From: scratch` and bring the program logic into and installing packages.

The problem is, at the end there is no knowledge what's in the Docker container if you don't build them by yourself.
The problem has meanwhile a name: Supply Chain Attack. It's not clear, which features the container has and which
backdoor are built in.

Practical example: [https://hub.docker.com/r/gnuu/busybox](https://hub.docker.com/r/gnuu/busybox) - a container from
our [Gnuu](https://www.gnuu.de) project, which we downloaded around 100 times. Docker Hub shows a download rate over
10.000! Also [https://hub.docker.com/r/gnuu/postfix](https://hub.docker.com/r/gnuu/postfix) has more then 2500 downloads,
although in the description is highlighted: this container is a special and works only with a special configuration.
Has anyone read this description? Has anyone reviewed the refered Dockerfile on Github or the build pipeline?
Probably not. It has a cool name, I need a Busybox image, and you've caught some dirty stuff.

# Container Signing
The solution of the problem of the Unknown Container exists for many time: [Docker Content Trust](https://docs.docker.com/engine/security/trust/). The container will signed by a key and the signature is stored in a [Notary](https://github.com/notaryproject/notary). The process has a strong oblique position since inventing. The signatures are not secured - yeah, everyone is able to delete them. In addition, it was not thought through to the end, because the containers with the signatures had to be verified somehow before they could be used. There was no software for this.

# Next Generation Container Signing
Thankfully, container security has grown in importance in recent years. Not only [Notary V2](https://notaryproject.dev/) brought to life, more standard are developed around [OCI Layer](https://opencontainers.org/) with the possibility to store more information on the same place as Docker container, like signatures.

# Sigstore
[Cosign Sigstore](https://www.sigstore.dev/) is established to the new standard for Container Signing.
The whole process of signing and verification has only 3 commands:

```bash
$ cosign generate-key-pair
$ cosign sign --key cosign.key  mtr.devops.telekom.de/eumel8/test1:signed
$ cosign verify --key cosign.pub  mtr.devops.telekom.de/eumel8/test1:signed
```

Ready!

# Admission Controller
Kubernetes Admission Controller are tools from access- and authorization management to sort and guide tasks in the cluster.
We are already using different Admission Controller, without the knowledge of of their existence::
ServiceAccounts, PodSecurity, PodSecurityPolicy, Priority, ResourceQuota - all Admission Controller.

One differentiates `ValidationWebhookConfiguration`, to validate things. And `MutatingWebhookConfiguration`, to change things.

For example, im [Cert Manager](https://github.com/cert-manager/cert-manager) are both:  ValidationWebhook, to verify and validate certificates and MutationWebhook, to create new ones.

Schema POD lifecycle with Admission Controller:

<img src="/images/2023-01-07-1.png" width="585" height="386" />

# Admission Controller for Image Verification
Verifying container signatures is a predestined task for an Admission Controller.
That is why it is also used in numerous tools:

## [Kubewarden](https://www.kubewarden.io/)
An Admission Controller, which has a completly [Hub](https://hub.kubewarden.io/) of plugins for a Policy Server.
One of them is vor verifying of images. The user can create Policies, which will be deployed by a new
instance of the Policy Server. And that's the bad point: is one Policy faulty, the cluster-wide update process is broken.
Beside that we have a big overhead for only this one plugin.
And another bad point: These Policies have again a specific Mime-format, which are not full supported by any registries.

## [Sigstore Policy Controller](https://github.com/sigstore/policy-controller/)
This tries to completely bypass the interactions with users by simply offering ClusterImagePolicy, which
can then only be managed by a cluster admin. A namespaced solution is still being considered.

## [Kyverno](https://kyverno.io/policies/other/verify_image/)
Also a tool with many Policy plugins. One of them is for verifying of images. Provided by a ClusterPolicy,
again a non-namespaced resource.
Now one could come up with the fact that each user installs his own tool in the shared cluster. But that works
of course not if central resources such as ClusterRoles or as mentioned WebHooks. You could crack it all up, add namespace selectors to the webhooks (technically everything would be possible), but who wants to manage something like that.

## [Connaisseur](https://github.com/sse-secure-systems/connaisseur/)
In this tool Policies are installed by the installing Helm Chart. In a single instance very practical, but a second
installation lacks on the same used resourced (see above).

## [Open Policy Agent](https://github.com/sigstore/cosign-gatekeeper-provider)
Open Policy Agent(OPA) is also a tool framework with a script language to create your own policies.
For Cosign is also a provider present, but there is only a validation if a signature is presengt, not if the signature
is valid. But you could build on this if you familiarize yourself with this script language
and pays attention to such use cases as signature location and private images.

# Cosign Webhook
On this place starts the story about our own Admission Controllers. In the beginning is a configuration:

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

So, like in the diagram above, we look for events in the cluster that want to create or update PODs.
This information send to a service `cosignwebhook` in namespace `cosignwebhook`. Some namespaces are
excluded like controller itself ans some system namespaces.

In `cosignwebook` namespace we have a service and a POD behind, provided by a deployment with the Admission Controller.

It's a web service, then we need a web server to answer the requests:

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

Target url is `/validate`. Something should happen there.

The object is `AdmissionReview` (see [API description](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#validatingwebhook-v1-admissionregistration-k8s-io)) and there we have a POD object included for review:

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

To verify the signature we need the Cosign public key. For that we have in the POD the environment variable:

```golang
	pubKey := ""
	for i := 0; i < len(pod.Spec.Containers[0].Env); i++ {
		value := pod.Spec.Containers[0].Env[i].Value
		if pod.Spec.Containers[0].Env[i].Name == cosignEnvVar {
			pubKey = value
		}
	}
```

Then we need the image name:

```golang
	image := pod.Spec.Containers[0].Image
	refImage, err := name.ParseReference(image)
```

The image can be not-public. Therefore we need the ImagePullSecrets:

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

The core are then just a few more lines that the original cosign routine for verifying signatures used.
[k8schain](https://pkg.go.dev/github.com/google/go-containerregistry/pkg/authn/k8schain) is a smart library from
Google to collect secrets, log into container registry. to get access to the image and the signature:

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

The result is a little weird. If no error comes back, everything is fine and the image is verified.
The Admission Controller returns an admission response of `Success`:

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

The container signature was verified successfully. The request is forwarded to the scheduler, which ultimately completes the POD
distributed to a node and the kubelet monitors and starts the POD.

Interested in details? [Cosign Webhook](https://github.com/eumel8/cosignwebhook) is available on Github, with installation
instruction and download links.

Happy Secure Containering!
