---
layout: post
tag: en
title: Vcluster Backup
subtitle: Enhancement of Crossplane Composition
date: 2024-03-14
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Intro

There have already been experiments on Vcluster and Crossplane in this [blog](https://k8sblog.eumel.de/2022/12/14/vcluster-in-rancher-mit-crossplane-en.html). We can create a resource `vcluster` and Crossplane will roll out a Helm chart with a vcluster deployment and register that cluster in Rancher. To do this, we used K3S with the embedded SQLite DB as a base, as only a single instance is available. Good enough for testing and development.

The catch is that even with highly available storage for the database, something can break. The entire life of the Vcluster is contained in the SQLite. Good idea to backup that.

# Solution 1 - the Cronjob

Here is the [Gist](https://gist.github.com/eumel8/686471197180d6f844191393c946a4db):

<script src="https://gist.github.com/eumel8/686471197180d6f844191393c946a4db.js"></script>

A cron job is installed in the Kubernetes cluster, which periodically starts a job that mounts the PVC of the K3S instance and initiates a backup there with sqlite. The example also lists a job for a restore.

Two problems arise here: The PVC name has to be set manually (you can still do it somehow in the automation if you use kustomize or Helm). And, depending on the StorageClass (here Longhorn), the cron job has to run on the same worker node like the K3S pod. Otherwise there is a “multi-attach error”. This could also be solved somehow, with even more fiddling.

# Solution 2 - vcluster-backup program

A small [Go-program](https://github.com/eumel8/vcluster-backup/), which periodically encrypt and save a file (state.db) on S3 storage.

```bash
$ ./vcluster-backup -h
Usage of ./vcluster-backup:
  -accessKey string
    	S3 accesskey.
  -backupFile string
    	Sqlite database of K3S instance. (default "/data/server/db/state.db")
  -backupInterval int
    	Interval in minutes for backup. (default 2)
  -bucketName string
    	S3 bucket name. (default "k3s-backup")
  -decrypt
    	Decrypt the file
  -encKey string
    	S3 encryption key.
  -endpoint string
    	S3 endpoint.
  -list
    	List S3 objects
  -region string
    	S3 region. (default "default")
  -secretKey string
    	S3 secretkey.
  -trace string
    	Trace S3 API calls
  -insecure string
    	Insecure S3 backend
```

The program runs in the foreground and depending on the `backupInterval` time, the state.db file is backed up every few minutes-

What is this good for now? Well, what was a job upfront is basically running next door here. To integrate this sidecar container into Vcluster's Helm chart, [this PR](https://github.com/loft-sh/vcluster/pull/1593) is required. From now on, the same storage is accessed by the K3s and the backups are created. An example deployment is [here](https://github.com/eumel8/vcluster-backup/blob/main/sidecar-value.yaml) (`sidecar` has been renamed to `sidecarContainer`). The work can be watched in the stdout log.

# Crossplane Enhancement

Our Vclusters are deployed with Crossplane and the Helm provider. In order to automate the backup and its setup, some extensions are required.

Given an S3 backend. We use a Minio instance and I always refer to the [version with the security fixes](https://github.com/eumel8/minio/tree/fix/securitycontext). Easy to install with Helm. If it runs in the same cluster, we wouldn't even need an ingress and can use the internal service endpoint.

In order to create a separate user and bucket for each Vcluster created, an [additional Helm chart: s3-register](https://github.com/mcsps/helm-charts/tree/master/charts/s3-register ) is used. This requires the admin credentials and the endpoint name. Bucket name and username are derived from the Vcluster name, the password is generated and stored in a secret. From there it can then retrieve vcluster-backup.

Let's look at the entire composition:

<script src="https://gist.github.com/eumel8/bfa1df538741f2fba9b2d84c7f80a3b2.js"></script>
