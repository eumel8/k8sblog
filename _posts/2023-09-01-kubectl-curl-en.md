---
layout: post
tag: en
title: Kubectl Curl
subtitle: Self-made plugin to query internal web services with curl
date: 2023-09-01
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Kubectl
In the last article we dealt with command line tools. Kubectl is of course the be-all and end-all when administering the Kubernetes cluster. The program comes with many functions from home. But there is also a plugin feature that lets you do a lot more.

# Curl
Curl as a command line tool for a wide variety of services, especially web services, has become an integral part of everyday life. Be it to check whether a web server is available and responding, or to quickly query something or download a file, all of this is done very quickly with curl. A `curl -n demoapp http://demoapp-57bf45f76-bgkwb:8080/` usually does not work in the Kubernetes cluster, since a pod is only connected via the overlay network and the service network is only connected internally in the cluster functions. You would need an ingress or a service with NodePort or an external load balancer.

# Kubectl Curl
The solution to the dilemma is called [Kubectl Curl](https://github.com/segmentio/kubectl-curl). The plugin starts a port forward from a local port via the Kubernetes API to the target pod. The inventor then went to great lengths to process all curl options in the plugin as a Go package. Too much trouble I think, but it didn't work in the Rancher environment. It had taken a while to figure that out. It then ended up in this [fork](https://github.com/eumel8/kubectl-curl), which on the one hand makes the package superfluous and then also works in rancher clusters.
The installation is very easy if you download a corresponding binary from the `Release` page and copy it to the same directory as kubectl. Anyone who has installed Go will also find a command for installing via Go in the README.
Prerequisite for use is a local `curl`. `jq` further refines the json output.

# Examples
What can we do with kubectl-curl:

- Calling a web server:

```bash
$ kubectl curl -n demoapp  http://demoapp-57bf45f76-bgkwb:8080/
```

- metrics of programs:

```bash
$ kubectl curl -n kube-logging http://kube-logging-logging-operator-f874c54f8-wcd6g:8080/metrics
```

- Asking Prometheus:

```bash
$ kubectl curl -n cattle-monitoring-system http://prometheus-rancher-monitoring-prometheus-0:9090/api/v1/query?query=fluentbit_uptime | jq .
```

- Health check of services:

```bash
$ kubectl curl -n kube-system http://traefik-rmtth:9000/ping
```

Have a lot of fun!
