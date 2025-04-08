---
layout: post
tag: de
title: "Cloud Nativer DNS mit CoreDNS in Kubernetes"
subtitle: Wir haben uns bei Kubernetes OpenStack External DNS mit Designate schon mit dem Thema DNS beschaeftigt, indem wir OpenStack Designate API angesprochen haben, um im Kubernetes unsere Dienste zu verwalten.
date: 2019-10-21
background: '/images/k8s-cosmos.png'
---

Dabei gibt es einfachere Moeglichkeiten unsere /etc/hosts Datei cloud native auszubauen.

<a href="https://coredns.io/">CoreDNS</a> ist ein ein weiteres Projekt der <a href="https://cncf.io/">Cloud Native Foundation</a> und stellt, wer haette das gedacht, einen DNS zur Verfuegung. Nun ist es ja so, dass /etc/hosts immer noch ein legitimes Mittel der DNS-Verwaltung ist. Verbinden wir doch beides.

<strong>Installation CoreDNS</strong>

Helm haben wir in <a href="https://blog.eumelnet.de/blogs/blog8.php/joomla-installation-mit-kubernetes-und-helm-1">diesem Posting</a> schon erklaert . Von <a href="https://github.com/helm/charts/">Helm Chart Repo</a>  deployen wir

```
helm install --name coredns --namespace=kube-system stable/coredns
```

Danach sollte wir folgenden Status haben:

```
NAMESPACE: kube-system
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME             AGE
coredns-coredns  28m

==> v1/ClusterRoleBinding
NAME             AGE
coredns-coredns  28m

==> v1/ConfigMap
NAME             DATA  AGE
coredns-coredns  1     28m

==> v1/Deployment
NAME             READY  UP-TO-DATE  AVAILABLE  AGE
coredns-coredns  1/1    1           1          28m

==> v1/Pod(related)
NAME                              READY  STATUS   RESTARTS  AGE
coredns-coredns-64c4958684-vxpc6  1/1    Running  0         28m

==> v1/Service
NAME             TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)        AGE
coredns-coredns  ClusterIP  10.43.212.227  none       53/UDP,53/TCP  13s
```

Das Helm Chart bietet in der Ausgabe noch eine Testmoeglichkeit an

```
kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools

dnstools# host kubernetes 10.43.212.227
Using domain server:
Name: 10.43.212.227
Address: 10.43.212.227#53
Aliases:

kubernetes.default.svc.cluster.local has address 10.43.0.1

$exit
```

Nun koennen wir die values.yaml nach unseren Wuenschen erweitern:

<pre> - zone: hello.world
  port: 53
  plugins:
  - name: hosts /etc/coredns/hello.world hello.world
  - name: log
zoneFiles:
  - filename: hello.world
    domain: hello.world
    contents: |
      192.168.0.100    hallo.hello.world

</pre>

Wir haben also eine Datei wie <ins>/etc/hosts</ins> mit dem Eintrag eines Hostnamens zur IP-Adresse 192.168.0.100. Diese haengen wir mit dem <a href="https://coredns.io/plugins/hosts/">hosts plugin</a> in das <ins>Corefile</ins>

Test:

```

$ kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools
If you don't see a command prompt, try pressing enter.
dnstools# host www.eumel.de 10.43.212.227
Using domain server:
Name: 10.43.212.227
Address: 10.43.212.227#53
Aliases:

www.eumel.de is an alias for vip.eumelnet.de.
vip.eumelnet.de is an alias for zuni.eumelnet.de.
zuni.eumelnet.de has address 217.79.184.96
zuni.eumelnet.de has IPv6 address 2001:4ba0:ffff:dc::1
dnstools# host hallo.hello.world 10.43.212.227
Using domain server:
Name: 10.43.212.227
Address: 10.43.212.227#53
Aliases:

hallo.hello.world has address 192.168.0.100
$ exit

```

Durch die Root-Zone "." werden Anfragen durch die Resolver in /etc/resolv.conf aufgeloest. Und unser /etc/hosts Eintrag wird durch den zweiten Test bestaetigt und ausserdem noch geloggt.

Ansicht unserer ConfigMap:

`kubectl edit cm coredns-coredns  -n kube-system`

```yaml
ApiVersion: v1
data:
  Corefile: |-
    .:53 {
        cache 30
        errors
        health
        ready
        kubernetes cluster.local
        loadbalance round_robin
        prometheus 0.0.0.0:9153
        forward . /etc/resolv.conf
    }
    hello.world:53 {
        hosts /etc/coredns/hello.world hello.world
        fallthrough
        log
    }
  hello.world: |
    192.168.0.100    hallo.hello.world
kind: ConfigMap
```

Unser CoreDNS Service IP muss als <ins>cluster_dns_server</ins> im Kubelet eingetragen sein. Erst dann nutzen interne Dienste unseren CoreDNS, der wiederum durch das <a href="https://coredns.io/plugins/kubernetes/">Kubernetes Plugin</a> die Namensaufloesung im Cluster verwaltet.

Um nun diesesn Dienst nach aussen in die Welt zu oeffnen, gaebe es verschiedene Moeglichkeiten. Hier verlassen wir die Helm-gestuetzte Installation und koennten den coredns-coredns Service vom Typ ClusterIP in LoadBalancer umwanden, einen NodePorrt 53 hinzufuegen (Achtung: der Port muss durch die NodePort-Range in der Kubernetes Cluster Konfiguration abgedeckt sein) und die NodeIPs auflisten.

Konfig Schnippsel:

```yaml
spec:
  clusterIP: 10.43.11.117
  externalIPs:
  - 80.158.77.1
  - 80.158.58.17
  - 80.158.211.99
  externalTrafficPolicy: Cluster
  ports:
  - name: udp-53
    nodePort: 53
    port: 53
    protocol: UDP
    targetPort: 53
  selector:
    app.kubernetes.io/instance: coredns
    app.kubernetes.io/name: coredns
    k8s-app: coredns
  sessionAffinity: None
  type: LoadBalancer
```

Das waere jetzt mal ein Beispiel bei eine Cluster mit mehreren Nodes. Wenn ich nur einen habe, kann ich natuerlich nur eine IP eintragen.

Leider hat die Sache jetzt einen kleinen Haken: DNS arbeitet sowohl mit UDP- als auch TCP-Protokoll. Jetzt koennte man annehmen, fuege einfach noch eine Port-Konfiguration fuer TCP hinzu. Aber mitnichten - das erlaubt Kubernetes mit Loadbalancer und nodePort nicht! Einfach aus dem Grund, weil es in der Cloud auch sonst keine UDP-Loadbalancer gibt. Fuer das Problem gibt es Workaround und neue Anforderungen,die demnaechst viellecht umgesetzt werden:

[fuer Services](https://github.com/kubernetes/kubernetes/issues/20092
[fuer Loadbalancer](https://github.com/kubernetes/kubernetes/issues/23880

Auch der Nginx-Ingress-Controller hilft uns hier nicht weiter, da er von Hause aus als Webserver mit solchen Diensten und Protokollen nichts zu tun hat. Aber auch dazu hier ein Workaround: https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/

Und zu guter Letzt: Debug Infos zu DNS in Kubernetes:

[von Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
[von AlibabaCloud](https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/dns-best-practice)
