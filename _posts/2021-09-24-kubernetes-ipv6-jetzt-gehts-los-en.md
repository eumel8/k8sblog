---
layout: post
tag: en
title: Kubernetes IPv6 - now we starting
subtitle: Hooray! Kubernetes 1.21 is there! And now the implementation of IPv6 DualStack in K3S. IPv6 can be used together with IPv4. Well, what does that mean exactly? Let's take a look at this in practice on our home network.
date: 2021-09-24
background: '/images/ipv6-evolution.png'
---

<strong>IPv6 Basics</strong>

Once upon a time, at the end of the last millennium, a great unrest spread: Uuuh, the Internet is full! We're out of IP addresses! Each network device on the Internet needs an IP address to communicate. Protocol 0 (IP or IPv4) contains 4 octets and build a 32-bit-address. It's start with 0.0.0.0 and end with 255.255.255.255. There are 4 billion addresses, without some reserved address spaces.
Now, after 20 years we're still not out of IP addresses. Since 20 years is IPv6 available. Here are now 128-bit-addresses assigned. The available IP addresses are not 2 to the power of 22, bit 2 to the power of 128 - a very long number. And also the addresses itself are long, e.g. 2003:e9:f74a:ecf8:8088:5353:3838:eaa1. This is one IP address. The prefix is /128 (compare to IPv4: /32).
Another usable prefix is /64. Our Internet Service Provider provided for example public addresses 2003:e9:f74a:ecf8::/64.
There are 18 446.744.073.709.551.616 IP addresses between 2003:00e9:f74a:ecf8:0000:0000:0000:0000 and 2003:00e9:f74a:ecf8:ffff:ffff:ffff:ffff. On this example are two things to see: Leading zeros between colons can be removed. If the space has only zeros, it can be removed completely and separate the space with ::
Another prefix is /56, there are 256 IP addresses. And if you are not a mathematic there are <a href="https://www.internex.at/de/toolbox/ipv6">Calculators</a>.

<strong>Global address/local address(ULA)</strong>

A view on the network configuration of our DSL router:

<img src="/images/2021-09-24-1.png" width="900" height="450" />

On the bottom we have the usable IPv6 address space for the home network. The gray field means, this space is not fixed, these address space will change every 24 hours. This is not a problem for all devices which have a dynamic IPv6 address in the home network via DHCP. In the Kubernetes cluster we have another mechanism for the internal address asignment. In this case we must use Unique Local Adresses (ULA) or <a href="https://en.wikipedia.org/wiki/IPv6#Unique_Local_Unicast">Unique Local Unicast</a>. In IPv6 address space there is a prefix fc::/7. This is comparable with 10:0.0.0/8 or 192.168.0.0/16 in IPv4 space. 2 private or local networks can be overlap. But as to see in the picture, our provider assigned also a local address space. This is in the prefix fd::/7, here fd45:0a71:55a6:0001::1
This area is fixed and usable for the Kubernetes cluster.

<strong>K3S start option</strong>

Precondition is the K3S upgrade at least to Kubernetes 1.21, here 1.21.4.
Therefore we install the upgrade controller:

```shell
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.6.2/system-upgrade-controller.yaml
```

And now the upgrade plan:

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: In
      values:
      - "true"
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.21.4+k3s1
```

One IPv4 network for PODs is set by default in K3S. We overwrite this option in
`/etc/systemd/system/k3s.service`  and deactivate Flannel, because it's not IPv6 ready. IPv6 DualStack is also activate in the start option:

```shell
# ...
ExecStart=/usr/local/bin/k3s \
    server \
  --no-flannel \
  --disable servicelb \
  --kube-apiserver-arg service-cluster-ip-range=10.43.0.0/16,fd45:a71:55a6:1:2:1::/116 \
  --kube-apiserver-arg feature-gates="IPv6DualStack=true" \
  --kube-controller-manager-arg cluster-cidr=10.42.0.0/24,fd45:a71:55a6:1:2:2::/96 \
  --kube-controller-manager-arg feature-gates="IPv6DualStack=true" \
  --kube-controller-manager-arg service-cluster-ip-range=10.43.0.0/16,fd45:a71:55a6:1:2:1::/116 \
  --kube-controller-manager-arg node-cidr-mask-size-ipv4=24 \
  --kube-controller-manager-arg node-cidr-mask-size-ipv6=96 \ # 118
  --kubelet-arg feature-gates="IPv6DualStack=true" \
  --kube-proxy-arg feature-gates="IPv6DualStack=true" \
  --kube-proxy-arg cluster-cidr=10.42.0.0/24,fd45:a71:55a6:1:2:2::/96
```

Reload service config and restart service:

```shell
systemctl daemon-reload
systemctl restart k3s.service
```

<strong>Calico</strong>

<a href="https://docs.projectcalico.org/getting-started/kubernetes/">
Calico</a> is another network plugin for Kubernetes. It works internally with BGP routes and exports this also in the world.
It's IPv6-ready.
Calico is installed mostly with <a href="https://raw.githubusercontent.com/gnuu-de/k8s/master/calico_ipv6.yaml">a manifest</a>. Be careful: The delivered CRDs are versioned, dependly on the image version. If you upgrade the image, you must upgrade the CRDs too. Furthermore there are 4 parameter important:

On the first part the IP configuration of the plugin:

```yaml
          "ipam": {
              "type": "calico-ipam",
              "assign_ipv4": "true",
              "assign_ipv6": "true",
              "nat-outgoing": "false",
              "ipv4_pools": ["10.42.0.0/24"],
              "ipv6_pools": ["fd45:a71:55a6:1:2:2::/96"]
          },
```

We activate IPv6 pool and add the IP network for the cluster from the K3S start script. Below are the same definition in the deployment as environment variables:


```yaml
            - name: CALICO_IPV4POOL_CIDR
              value: "10.42.0.0/24"
            - name: CALICO_IPV6POOL_CIDR
              value: "fd45:a71:55a6:1:2:2::/96"
```

In the same area we activate IPv6 NAT outside, because we haven't outgoing IPv6 IPs:

```yaml
            - name: CALICO_IPV6POOL_NAT_OUTGOING
              value: "true"
```

The last option is to activate IPv6 in Felix. That's the agent, which runs on each node in a POD:

```yaml
            - name: FELIX_IPV6SUPPORT
              value: "true"
```

ALl other values are default. After the deployment there should be 2 PODs running in the kube-system namespace. One controller and one node POD:

```shell
# kubectl -n kube-system get pods | grep cali
calico-node-twxmf                          1/1     Running     0          34m
calico-kube-controllers-74b8fbdb46-slhxn   1/1     Running     0          34m
```

If the PODs are not running, this must be investigated before continue. Compare defined IP addresses, for example, which is mentioned in the logs.

If Calico in the cluster is working, we can check in a busybox deployment or each new started POD with a shell:

```shell
bash-5.0# ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 72:51:2B:80:AE:E4
          inet addr:10.42.0.194  Bcast:10.42.0.194  Mask:255.255.255.255
          inet6 addr: fd45:a71:55a6:1:2:3000:0:f01/128 Scope:Global
          inet6 addr: fe80::7051:2bff:fe80:aee4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1440  Metric:1
          RX packets:25 errors:0 dropped:0 overruns:0 frame:0
          TX packets:25 errors:0 dropped:1 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2674 (2.6 KiB)  TX bytes:2590 (2.5 KiB)
```

The POD has a IPv4 and a IPv6 address from the Calico address space.

```shell
bash-5.0# ping ipv4.google.com
PING ipv4.google.com (172.217.19.78): 56 data bytes
64 bytes from 172.217.19.78: seq=0 ttl=58 time=10.722 ms
64 bytes from 172.217.19.78: seq=1 ttl=58 time=10.592 ms
```

```shell
bash-5.0# ping ipv6.google.com
PING ipv6.google.com (2a00:1450:4016:80a::200e): 56 data bytes
64 bytes from 2a00:1450:4016:80a::200e: seq=0 ttl=118 time=21.472 ms
64 bytes from 2a00:1450:4016:80a::200e: seq=1 ttl=118 time=21.450 ms
```

Now we have checked name resolution and outgoing network connection to the world with IPv4 and IPv6.
If name resolution doesn't work, we should check if CoreDNS service is running (restart POD if not) and the service IP for dns (from /etc/resolv.conf) is reachable. The service network must be into the cluster network. If network connection doesn't work, you should check network connection from the node. If yes, there may something wrong with ip-forwarding rules of the kernel.

```shell
# sysctl net.ipv4.conf.all.forwarding
net.ipv4.conf.all.forwarding = 1
# sysctl net.ipv6.conf.all.forwarding
net.ipv6.conf.all.forwarding = 1
```

With `ip6tables-save -t nat` there should be a MASQUERADE rule outside.

Also to check if Calico has created 2 IP-Pools:

```shell
# kubectl get ippools.crd.projectcalico.org
NAME                  AGE
default-ipv4-ippool   39m
default-ipv6-ippool   39m
```


<strong>Traefik Ingress/Klipper Service Loadbalancer</strong>

The internal IPv6 communication is working, now we try to make services reachable from outside. As usualy this is done by an Ingress controller. In K3S we have <a href="https://doc.traefik.io/traefik/providers/kubernetes-ingress/">Traefik</a> as default. This has in K3S a very special characteristic. The Helm chart "traefik" will deploy a service type LoadBalancer. Without cloud controller there is no connection from outside, while the external ServiceIP is in the state `pending`. Rancher deploys now IN K3S a resource type DaemonSet, which creates a POD with HostNetwork and opens a port outside. Default ports are 80 and 443, extended 9000 for metrics and user defined ports. In this svclb-POD an image named `klipper-lb` is used. This runs on a loop and created two iptable rules for the loadbalancer service and port. Each port starts another container.
Two things are missing here:
1. There are only IPv4 Iptables
2. There is only an `IP->port` connection. It's only possible to bind one IP to one port. With DualStack I have one IPv4 and one IPv6. The DaemonSet denies this with duplicate entries for HostPort.
Here is a hotfix: In the start option from K3S we disable this service with `--disable servicelb`. Then deploy Traefik with DualStack option. This can be changed in the file `/var/lib/rancher/k3s/server/manifests/traefik.yaml`:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: traefik-crd
  namespace: kube-system
spec:
  chart: https://%{KUBERNETES_API}%/static/charts/traefik-crd-9.18.2.tgz
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: traefik
  namespace: kube-system
spec:
  chart: https://%{KUBERNETES_API}%/static/charts/traefik-9.18.2.tgz
  set:
    global.systemDefaultRegistry: ""
  valuesContent: |-
    logs:
      general:
        level: DEBUG
      access:
        enabled: true
    rbac:
      enabled: true
    service:
      spec:
        ipFamilies:
          - IPv4
          - IPv6
        ipFamilyPolicy: RequireDualStack
      externalIPs:
        - 192.168.0.15
        - 2003:e9:f712:a22f:a61f:72ff:fe56:1e9e
    ports:
      traefik:
        expose: true
      websecure:
        tls:
          enabled: true
    podAnnotations:
      prometheus.io/port: "8082"
      prometheus.io/scrape: "true"
    providers:
      kubernetesIngress:
        publishedService:
          enabled: true
    priorityClassName: "system-cluster-critical"
    image:
      name: "rancher/library-traefik"
    tolerations:
    - key: "CriticalAddonsOnly"
      operator: "Exists"
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"
```

With the super important option `RequireDualStack` we enable logging as well.

If the Traefik service is deployed, we get the assigned IP addresses:

```shell
$ kubectl -n kube-system get services traefik -o jsonpath="{.spec.clusterIPs}"
["10.43.206.211","fd45:a71:55a6:1:2:1:0:db1"]
```

and put them in the DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: svclb-traefik
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: svclb-traefik
  template:
    metadata:
      labels:
        app: svclb-traefik
    spec:
      containers:
      - env:
        - name: SRC_PORT
          value: "9000"
        - name: DEST_PROTO
          value: TCP
        - name: DEST_PORT
          value: "9000"
        - name: DEST_IP
          value: 10.43.206.211
        image: rancher/klipper-lb:v0.2.0
        imagePullPolicy: IfNotPresent
        name: lb-port-9000
        ports:
        - containerPort: 9000
          hostPort: 9000
          name: lb-port-9000
          protocol: TCP
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      - env:
        - name: SRC_PORT
          value: "80"
        - name: DEST_PROTO
          value: TCP
        - name: DEST_PORT
          value: "80"
        - name: DEST_IP
          value: 10.43.206.211
        - name: DEST_IP6
          value: fd45:a71:55a6:1:2:1:0:db1
        image: mtr.external.otc.telekomcloud.com/eumel8/klipper-lb:dual-stack
        imagePullPolicy: IfNotPresent
        name: lb-port-80
        ports:
        - containerPort: 80
          hostPort: 80
          name: lb-port-80
          protocol: TCP
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      - env:
        - name: SRC_PORT
          value: "443"
        - name: DEST_PROTO
          value: TCP
        - name: DEST_PORT
          value: "443"
        - name: DEST_IP
          value: 10.43.206.211
        - name: DEST_IP6
          value: fd45:a71:55a6:1:2:1:0:db1
        image: mtr.external.otc.telekomcloud.com/eumel8/klipper-lb:dual-stack
        imagePullPolicy: IfNotPresent
        name: lb-port-443
        ports:
        - containerPort: 443
          hostPort: 443
          name: lb-port-443
          protocol: TCP
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
        operator: Exists
      - key: CriticalAddonsOnly
        operator: Exists
  updateStrategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
```

The programm runs with a <a href="https://github.com/eumel8/klipper-lb/tree/dual-stack">klipper-lb fork</a>. There is an additional variable `DEST_IP6` and creates, if exists, the IPv6 iptable rules.

Voila! Our services should be available via IPv6. And our workload has connection to the IPv6 world. Easy, isn't it?

<img src="/images/2021-09-24-2.png" />
