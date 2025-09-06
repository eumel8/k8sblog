---
layout: post
tag: de
title: "Kubernetes an Rancher K3s und RKE2 auf OTC mit Terraform"
subtitle: Mit Kubernetes Install Quickies haben wir uns vor einiger Zeit schon mal beschäftigt. Sehr guten Eindruck machte damals schon K3S, ein einzelnes Binary, entwickelt von Rancher, zur Benutzung in schmalen Umgebungen für Kubernetes. Wie kriegen wir aber schnell eine arbeitsfähige Umgebung zum Laufen?
date: 2021-07-17
background: '/images/kubernetes-rancher.png'
---

Die Antwort lautet wieder Terraform. Mit Terraform können wir Zustände von Resourcen in der Infrastruktur beschreiben, die dann wieder durch Module oder Provider bereitgestellt werden. Um etwa Resourcen in der Open Telekom Cloud zu nutzen, gibt es den <a href="https://registry.terraform.io/providers/opentelekomcloud/opentelekomcloud/latest/docs">Terraform Provider Opentelekomcloud </a>. Dieser stellt Resourcedefinitionen bereit wie etwa ein VPC:

```
resource "opentelekomcloud_vpc_v1" "vpc" {
name  = var.environment
cidr  = var.vpc_cidr
shared = true
}
```

Sowas gibts dann auch fuer Subnet, EIP, ECS, EVS usw.. Und man kann die Resourcen miteinander verknuepfen. Ein Subnet brauch zum Beispiel die VPC-ID:

```
resource "opentelekomcloud_vpc_subnet_v1" "subnet" {
name  = var.environment
vpc_id  = opentelekomcloud_vpc_v1.vpc.id
cidr = var.subnet_cidr
gateway_ip = var.subnet_gateway_ip
primary_dns = var.subnet_primary_dns
secondary_dns = var.subnet_secondary_dns
}
```

Am Ende haben wir den kompletten Unterbau fuer unseren Kubernetes Cluster mit Rancher. Deren Logik wird mit <a href="https://cloud-init.io/">cloud-init</a> einfach als Shell-Script in die VM injected und dort lokal ausgefuehrt.

<strong>K3S</strong>
https://github.com/eumel8/tf-k3s-otc
In diesem Reporitory befindet sich die Resourcenbeschreibung in Terraform zum Installieren eines K3S Clusters in der Open Telekom Cloud. 

Vorteile:
<ul>
  <li>Sehr schmale Cluster Konfiguration mit 1,2 oder 3 Masternodes</li>
  <li>Nodes sind zustandslos, alle Daten liegen in einer RDS-Instanz (MySQL)</li>
  <li>Einzeiler zum beliebigen Erweitern mit Worker Nodes</li>
</ul>


Nachteile:
<ul>
  <li>Die RDS-Instanz als Ersatz fuer etcd ist der Dreh- und Angelpunkt des Clusters. Dadurch entstehen auch relativ hohe Betriebskosten</li>
  <li>Aenderungen an der ECS-Infrastruktur koennen dazu fuehren, dass alle Nodes gleichzeitig neu gespawned werden und der Dienst dadurch nicht verfuegbar ist. Dem wurde teilweise begegnet durch unterschiedliche Image-Versionen.</li>
</ul>


<strong>RKE2</strong>
https://github.com/eumel8/tf-rke2-otc
In diesem Repository befindet sich die Beschreibung fuer ein Terraform-Deployment in der OTC, das einen Kubernetes Cluster basierend auf RKE2 von Rancher erstellt.

Vorteile:
<ul>
  <li>Keine teure RDS-Instanz erforderlich, die Masternodes agieren unabhaengiger von der Infrastruktur</li>
  <li>Spezielles CIS Hardening (basierend auf US Governance Anforderungen) sind schon implementiert</li>
</ul>

Nachteile:
<ul>
  <li>Die Cluster-Daten liegen in der etcd auf jedem Masternode. Um die Daten nach einem Respawn nicht zu verlieren, wird ein extra Volume benoetigt, was auch einen hohen Datendurchsatz haben muss</li>
  <li>Entsprechend den Anforderungen von etcd muss der Cluster eine ungerade Anzahl von Nodes aufweisen - am wenigsten 1, besser 3</li>
  <li>Installation derzeit noch unstabil. Es sind zu viele Abhaengkeiten beim parallelen Erstellen der VMs. Eine Meldung koennte daraus resultieren, dass zu viele Mitglieder im register Status mit der etcd verbunden sind, wodurch dann erstmal gar nix mehr geht. Da bedarf es weiterer Abhaengigkeiten in Terraform</li>
</ul>

Der Vorteil bei beiden Methoden ist das "One-Binary"-Konzept.  Es bedarf also nicht allzugrosser Vorbereitungen auf dem Betriebssystemlevel. Genausoleicht wie die Installation verlaeuft auch das Loeschen, es wird auch ein Binary zum Loeschen der Umgebung mitgeliefert. Der Gesamteindruck bei beiden ist auch die unkomplizierte Handhabung.
Als Nachteil kann man das Vendor-Lock-In sehen. Beide Projekte sind zwar OpenSource, aber die Entwicklung wird von Rancher gefuehrt, die nun zu SUSE gehoeren. Der neue Produktname ist <strong>SUSE Rancher</strong>

<img src="/images/2021-07-21-1.png" width="900" height="450" />
