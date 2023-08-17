---
layout: post
tag: en
title: Curl Kubernetes API
subtitle: How to call Kubernetes API with call?
date: 2023-08-17
background: '/images/k8s-cosmos.png'
twitter: 'images/cosignwebhook.png'
author: eumel8
---

Shortly came the question: How to call Kubernetes-API with curl? The Kubernetes documentation has an howto with kubectl proxy, with an external provides authentication. But there is also an innerCluster communication, the way from the Pod in the cluster to the Kubernetes-API.

For this are tokens automatically mounted in each Pod with a default ServiceAccount in the namespace. This feature can be disabled in the cluster, but neverteheless, this account has no permissions. We must provide a Role and connect with a RoleBinding this to a ServiceAccount, which we ensurly created at the same time. This runs on a StatefulSet with an image, which contains curl, of course, and a shell and jq for parsing Json outputs.

Manifest:

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: curl-client
  name: curl-client
spec:
  serviceName: curl-client
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: curl-client
  template:
    metadata:
      labels:
        app: curl-client
    spec:
      containers:
      - image: mtr.devops.telekom.de/mcsps/mysql-client:0.0.6
        imagePullPolicy: Always
        command: ['sh', '-c']
        args: ["tail -f /dev/null"]
        name: curl-client
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
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
      serviceAccountName: curl-client
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
    app: curl-client
  name: curl-client
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    app: curl-client
  name: curl-client
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app: curl-client
  name: curl-client
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: curl-client
subjects:
  - kind: ServiceAccount
    name: curl-client
```

Our ServiceAccount `curl-client` has access to query Services, Pods, and Endpoints. 

In the Pod we have a cluster certificate and a token as a secret mounted, which we can use to query:


```bash
% kubectl -n demoapp exec -it curl-client-0 -- sh
$ export TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
$ curl -v https://10.43.0.1/openapi/v2 -H "Authorization: Bearer $TOKEN" --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt 
$ curl https://10.43.0.1/api/v1/namespaces/demoapp/pods -H "Authorization: Bearer $TOKEN" --cacert /var/run/secrets/kubernetes.io
/serviceaccount/ca.crt
$ curl -s https://10.43.0.1/api/v1/namespaces/demoapp/pods -H "Authorization: Bearer $TOKEN" --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt  | jq -r '.items[].metadata.name'
demoapp-57bf45f76-bgkwb
curl-client-0
```

[Gist](https://gist.github.com/eumel8/da4cea06d1cc4dc4f167c19519246fc9)


