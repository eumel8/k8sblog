---
layout: post
tag: en
title: "Kubernetes on Rancher K3s and RKE2 in OTC with Terraform"
subtitle: With Kubernetes Install Quickies we have worked in the past. K3S made a very good experience, a single binary, developed by Rancher, to use in smaller environments for Kubernetes. But how can we install a complete working space?
date: 2021-07-17
background: '/images/kubernetes-rancher.png'
---

The answer Terraform. With Terraform we make conditions for resources in an infrastructure provided with modules and provider. To use resource in Open Telekom Cloud there is a <a href="https://registry.terraform.io/providers/opentelekomcloud/opentelekomcloud/latest/docs">Terraform Provider Opentelekomcloud </a>. Provide resource definition like VPC:

```
resource "opentelekomcloud_vpc_v1" "vpc" {
name  = var.environment
cidr  = var.vpc_cidr
shared = true
}
```

There are also things for Subnet, EIP, ECS, EVS etc.. And you can combine resources with each other. A Subnet needs a VPC-ID for example:

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

At the there is a substructure for Kubernetes Cluster with Rancher. The logic is provided by <a href="https://cloud-init.io/">cloud-init</a> simple as shell script injected in the VM and exec locally.

<strong>K3S</strong>
https://github.com/eumel8/tf-k3s-otc
In this reporitory are the resource definition in Terraform to install a K3S cluster in Open Telekom Cloud. 

advantages:
<ul>
  <li>Very small cluster configuration with 1, 2, or 3 master nodes</li>
  <li>Nodes are stateless, all data are stored in a RDS instance (MySQL)</li>
  <li>One-Liner for extend with worker nodes</li>
</ul>


disadvantages:
<ul>
  <li>The RDS instance as a replacement for etcd is the key point of the cluster. The operating costs are relativly high</li>
  <li>Changes on the ECS infrastructure can cause the respawn of all nodes in parallel. The Kubernetes service are not available. Partily for this reason are variables for different image versions.</li>
</ul>


<strong>RKE2</strong>
https://github.com/eumel8/tf-rke2-otc
In this repository is a description for a Terraform deployment in OTC, which creates a Kubernetes Cluster based on RKE2 for Rancher.

advantages:
<ul>
  <li>No expensive RDS instance required, the master nodes are independent to the infrastructure</li>
  <li>specific CIS bench hardening implemented (based on US Governance requirements) </li>
</ul>

disadvantages:
<ul>
  <li>The cluster data are stored in the etcd on each master nodes. We need an extra volume to prevent data lost after respawn of the master nodes. This extra volume must have high performance like SSD</li>
  <li>Due the etcd cluster requirments we need odd nmber of nodes - at least 1, better 3</li>
  <li>The installation is currently unstable. There are too many depedencies while create the Virtual Machines in parallel. A message can be caused that there are too many members connecting to the etcd in register state. That requires more depedenies in Terraform</li>
</ul>

The advantage of both methods is the "one-binary"-concept. There is not so much preparation on the operating system level. In the same way like the installation is the deleting  included with a binary as part of the package. The overall impression of both is also the uncomplicated handling.
A disadvantage is maybe the vendor-lock-in. Both projects are Open Source, but the developement is leaded by Rancher, which is now part of SUSE company. The new product name is <strong>SUSE Rancher</strong>

<img src="/images/2021-07-21-1.png" width="900" height="450" />
