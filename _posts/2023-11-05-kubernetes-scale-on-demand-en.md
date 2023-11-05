---
layout: post
Tag: en
title: Scaling Kubernetes on demand
subtitle: Keda and Keda HTTP Add-On
date: 2023-11-05
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Sustainable computing
Climate change is on everyone's lips. Meanwhile, the world's data centers are happily humming and humming along. We will address the topic of sustainable computing in the next article. First of all, this is about scaling on demand, so our workload in the Kubernetes cluster is scaled as needed using the [Horizontal Pod Autoscaler](https://kubernetes.io/de/docs/tasks/run-application/horizontal-pod -autoscale/) from 1 to infinity.
But what if that 0 to infinity were possible?

# Keda
[Keda](https://keda.sh) - Kubernetes Event Driven Autoscaling. A pithy term and just as brilliant. I installed my workload in the cluster and scaled the deployment to 0. Everything is “ready to go”. The go then comes from an event, old school would now be a cron job that scales the workload up at 8 a.m. and down again at 6 p.m. Anyway. But what would be the scenario of a service that is rarely used, such as a website that someone only visits once in a while, like this one? Okay, let's ignore all the spam and bots, we can worry about those later. Maybe one or two visitors come by here during the day. And for them we open our store by starting a pod after the first request in the browser, which houses an Nginx web server, which then delivers this content here and shows it to the visitor in the browser with a short delay. When the page is loaded, Keda waits a while and scales the deployment back to 0. So we save computer resources and electricity and thus protect the environment.

# Preparation
Our Kubernetes cluster is a K3S that manages itself with Rancher. K3S comes with Traefik as an ingress controller as standard. We have to make a few adjustments there:

```bash
kubectl -n kube-system edit helmchartconfigs.helm.cattle.io traefik
```

We add the following lines:

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

We'll see why in a moment.

# Install Keda and Keda HTTP Addon
One thing in advance: Keda is very clean and structured. In the [Documentation](https://keda.sh/docs/2.12/deploy/) you can quickly find the options for installing Keda. The documentation is also structured according to standards for technical writing: it goes from easy steps to complicated ones, superficial and general descriptions lead to a lot of depth, such as [the detailed description of all Helm Chart parameters](https://github.com/ kedacore/charts/tree/main/keda#general-parameters). Something like this is rare and shows a lot of love for the project. Thanks to the high security standards, hardly any adjustments are necessary. We can start with the Helm installation:

```bash
helm repo add keda https://kedacore.github.io/charts
helm repo update
helm upgrade -i keda keda/keda --namespace keda --create-namespace --set rbac.aggregateToDefaultRoles=true
helm upgrade -i http-add-on keda/keda-add-ons-http --namespace keda --set rbac.aggregateToDefaultRoles=true
```

Here we have decided on a cluster-wide installation. The CRDs and cluster-wide RBAC are installed, with the rights being aggregated to the respective default users such as `admin`. Project owners can then later manage Keda for their app in their Rancher project. Another option would be the Keda operator or namespaced installation. However, we don't want to burden the user with managing the Kedas components. He wants it to be as simple as possible, hence this approach here.

# Demo app & demo user
The demo user owns a project in Rancher and is the project owner there. In the demoapp project we create a demoapp namespace.

<img src="/images/2023-11-05_1.png"/>

The app we use is a demo app, let's use the [Flask app here](https://github.com/mcsps/use-cases/tree/master/flask). We install this into the demoapp namespace.

```bash
kubectl -n demoapp apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/flask/deployment.yaml
kubectl -n demoapp apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/flask/service.yaml
```

Now all we need is one ingress and our app would be accessible to the world. But let's look at the Keda architectural picture again:

<img src="https://raw.githubusercontent.com/kedacore/http-add-on/main/docs/images/arch.png" width="925" height="525"/>

Between Ingress and Service there is the Intereceptor through which we have to pass the traffic. Our smuggler is called `relink` and is deployed in the demoapp namespace:

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

The interceptor service runs in a different namespace. This service redirects traffic there.
The Ingress now has `relink` as backend:

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

In order for `ExternalName` to be accepted as an ingress endpoint by Traefik, the changes to Traefik's Helmchartconfig were necessary at the beginning.

# HTTPScaledObject
After our last step, our workload ends up somewhere in nirvana. It is not transparent to the project owner where he is sending the traffic. Of course, consultation with the cluster owner is necessary here.

Now to tackle scaling, we need an HTTPScaledObject:

```bash
cat <<EOF | kubectl -n demoapp apply -f -
kind: HTTPScaledObject
apiVersion: http.keda.sh/v1alpha1
metadata:
     name: demoapp
spec:
     hosts:
     - demoapp.otc.mcsps.de
     scaledown period: 10
     scaleTargetRef:
         deployment: demoapp
         service: demo app
         port: 80
     replicas:
         min: 0
         max: 10
     targetPendingRequests: 1
EOF
```

So the Keda HTTP Interceptor should forward our traffic to the demoapp service in our demoapp namespace. The minimum replica is 0, so no pods of the app are running:

```bash
$ kubectl -n demoapp get deployments.apps demoapp
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
demoapp   0/0     0            0           23h
```

We can also query the status of the Keda and it looks something like this:

```bash
kubectl -n demoapp describe httpscaledobjects.http.keda.sh demoapp
```

```yaml
Status:
   Conditions:
     Message: Identified HTTPScaledObject creation signal
     Reason: PendingCreation
     Status: Unknown
     Timestamp: 2023-11-04T16:42:14Z
     Type: Pending
     Message: App ScaledObject created
     Reason: AppScaledObjectCreated
     Status: True
     Timestamp: 2023-11-04T16:42:14Z
     Type: Created
     Message: Finished object creation
     Reason: HTTPScaledObjectIsReady
     Status: True
     Timestamp: 2023-11-04T16:42:14Z
     Type: Ready
Events: <none>
```

If we now open our demo app, it will take a moment and then the app will be available:

```bash
$ curl http://demoapp.otc.mcsps.de/a
You requested: a
```

We set the inactive timeout to 10 seconds. So the app is asleep again before we can look in the event log to see what happened:

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

That's it! The concept is adapted from [Idle Instances from Openshift](https://docs.openshift.com/container-platform/4.9/applications/idling-applications.html).

However, there are many other options for [scaling](https://keda.sh/docs/2.12/scalers/). Also worth mentioning:

# Prometheus

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
   name: demo-keda-scaledobject
spec:
   scaleTargetRef:
     apiVersion: apps/v1
     kind: Deployment
     name: demoapp
   pollingInterval: 10 # Optional. Default: 30 seconds
   cooldownPeriod: 300 # Optional. Default: 300 seconds
   minReplicaCount: 0 # Optional. Default: 0
   maxReplicaCount: 6 # Optional. Default: 100
   fallback: # Optional. Section to specify fallback options
     failureThreshold: 3 # Mandatory if fallback section is included
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

In this example, a monitoring instance with Prometheus is still running. We install another ServiceMonitor:

```bash
kubectl -n demoapp apply -f https://raw.githubusercontent.com/mcsps/use-cases/master/monitoring/servicemonitor-demoapp.yaml
```

and could then control our scaling with PromQL queries. Of course it doesn't work with the Flask metrics, because our Flask app would have to run at least once. But it may be suitable for larger applications where, for example, a smaller part runs permanently and the large Java container is only started for certain requests. This is just an idea.

# Antispam
Our on-demand scaling now works fine. If we let it loose on the Internet, it would hardly be able to calm down because there are tons of bots out there trying to spy on our app.

Here are a few ideas to curb traffic:

Traefik offers the [Middleware](https://doc.traefik.io/traefik/middlewares/http/ipwhitelist/) resource to whitelist IPs:

```yaml
apiVersion: traefik.io/v1alpha1
kind: middleware
metadata:
   name: test-ipwhitelist
spec:
   ipWhiteList:
     sourceRange:
       - 127.0.0.1/32
       - 192.168.1.7
```

Blacklists work the other way around:

```yaml
apiVersion: traefik.io/v1alpha1
kind: middleware
metadata:
   name: test-ipwhitelist
spec:
   ipWhiteList:
     ipStrategy:
       excludedIPs:
         - 127.0.0.1/32
         - 192.168.1.7
```

A deeper possibility is an application firewall in front of the ingress controller. Or you can integrate [dynamic spam lists (DNBL)](https://gist.github.com/theMiddleBlue/02142f84007a5538491e109b383f28ba) in the Nginx of the ingress controller.

# Conclusion

Scaling on demand may only be a small contribution to protecting the environment and your wallet if you have to pay for CPU and memory. But it is a start and thanks to the excellent Keda project it is also possible in open source.
