---
title: "VMware Carbon Black Kubernetes Image Scanning"
date: 2022-05-10
draft: false
author: "Benjamin Koltermann"
authorlink: "/authors/benjamin-koltermann/"
featuredImage: "/images/admission-controller-noderestriction/preview.png"
description: "This article is about integrating VMWare Carbon Black's container scanning solution into a Kubernetes cluster using container image scanning policies."
tags: [vmware, carbon black, image scanning]
categories: [VMware]
---

<!--more-->

## Test environment and installation

As a test environment, we have chosen a simple three node cluster, with two workers and a master node, consisting of the following components.

```bash
bash:~$ kubectl get nodes
master     Ready    control-plane,master   101d   v1.23.3
worker-1   Ready    worker                 101d   v1.23.3
worker-2   Ready    worker                 101d   v1.23.3
```

### Carbon Black Kubernetes Sensor

Carbon Black, or more specifically the Kubernetes sensor for Carbon Black, has been installed and configured according to the official [Carbon Black Installation Guide](https://docs.vmware.com/en/VMware-Carbon-Black-Cloud/services/cbc-sensor-installation-guide/GUID-D2D621D7-E341-4F16-88AF-B5919958B142.html). After the sensor is installed, all pods should start over time.

```bash
bash:~$ kubectl get pods -n cbcontainers-dataplane
cbcontainers-hardening-enforcer-5df47c77c9-c7brn         1/1     Running   4          66d
cbcontainers-hardening-state-reporter-6f74d96cf4-dgvhr   1/1     Running   4          66d
cbcontainers-image-scanning-reporter-c99d75559-n64wh     1/1     Running   4          66d
cbcontainers-monitor-6bb786cdcf-q6mk8                    1/1     Running   4          66d
cbcontainers-node-agent-fsxqz                            2/2     Running   8          66d
cbcontainers-node-agent-mtvq7                            2/2     Running   9          66d
cbcontainers-operator-7869594cf6-gf7z4                   2/2     Running   10         79d
cbcontainers-runtime-resolver-5b4cc65849-vxrg4           1/1     Running   4          66d
```

### Carbon Black CLI

The Carbon Black CLI is required to scan the container images. This can be set up and downloaded in the Carbon Black environment under **Inventory** --> **Kubernetes** --> **Clusters** --> **CLI Config**. The Carbon Black binary should then be ready to use.

{{< admonition type=example title="Carbon Black CLI Beispiel" open=false >}}

```bash
bash:~$ ./cbctl-linux-amd64 version
        __         __  __
  _____/ /_  _____/ /_/ /
 / ___/ __ \/ ___/ __/ /
/ /__/ /_/ / /__/ /_/ /
\___/_.___/\___/\__/_/

Application: cbctl
Version:     v1.0.0
BuildDate:   2021/04/21
Platform:    linux/amd64
GoVersion:   go1.15.6
Compiler:    gc
```

{{< /admonition >}}

## Carbon Black Image Scanning Policy

{{< admonition type=info title="Carbon Black Policies" open=true >}}

This article does **not** deal with Carbon Black Policies in depth, only the process to create Carbon Black Kubernetes Policies for Image Scanning is described. VMware's official documentation on Kubernetes policies can be found [here](https://docs.vmware.com/en/VMware-Carbon-Black-Cloud/services/carbon-black-cloud-user-guide/GUID-AE6ED527-C2DF-471B-92F8-0C9269975C2B.html).

{{< /admonition >}}

The Carbon Black Kubernetes image scanning policy is one of the hardening policies. Therefore, a policy must be created in the Carbon Black environment under **Enforce** --> **K8s Policies** --> **Hardening Policies**. After assigning a name and scope to the policy, we find a rule named `Image not scanned` under **Container Images**. We add this rule to our policy with the action `Block`. Then we get the possibility to define exceptions for already existing workloads within the selected scope. After the policy is created, it may take half a minute for it to take effect.

{{< admonition type=info title="Kubernetes Scopes" open=true >}}

Choosing the right Kubernetes scope can be critical. Since Kubernetes scopes are not covered in this article, you can find an overview [here](https://docs.vmware.com/en/VMware-Carbon-Black-Cloud/services/carbon-black-cloud-user-guide/GUID-14E951FE-F4DA-49EB-9FF5-B655BB0490DC.html).

{{< /admonition >}}

## Image Scanning with the Carbon Black CLI

Scanning an image can be done quickly and easily. In the Carbon Black CLI the `image scan` command is used to scan an image. The image to be scanned is then specified, and the results of the scan are displayed directly as output from the CLI, but can also be viewed in the Carbon Black Cloud. The upload of the scan result to the Carbon Black Cloud is done automatically after the image scan is finished.

{{< admonition type=example title="Carbon Black CLI Ubuntu Image Beispiel Scan" open=false >}}

```bash
bash:~$ ./cbctl-linux-amd64 image scan ubuntu
 ⠿ Copied image      [copying image on disk]
 ⠿ Parsed image
 ⠿ Cataloged image   [92 packages]
 ⠿ Scanned image     [fetching result]

Scan result for docker.io/library/ubuntu:latest (sha256:e1afa64d6afd381bdcec761b92c77abf6a86a7812fba35df25a6c8d883197837):
+------------------+-------------------------+------+----------+------------------+---------+---------+
|     VULN ID      |         PACKAGE         | TYPE | SEVERITY |  FIX AVAILABLE   | CVSS V2 | CVSS V3 |
+------------------+-------------------------+------+----------+------------------+---------+---------+
| CVE-2022-1271    | liblzma5-5.2.4-1ubuntu1 | dpkg | MEDIUM   | 5.2.4-1ubuntu1.1 |     0.0 |     0.0 |
+------------------+-------------------------+------+----------+------------------+---------+---------+
| CVE-2022-1271    | gzip-1.10-0ubuntu4      | dpkg | MEDIUM   | 1.10-0ubuntu4.1  |     0.0 |     0.0 |
+------------------+-------------------------+------+----------+------------------+---------+---------+
| CVE-2016-2781    | coreutils-8.30-3ubuntu2 | dpkg | LOW      |                  |     2.1 |     6.5 |
+------------------+-------------------------+------+----------+------------------+---------+---------+
| CVE-2018-1000654 | libtasn1-6-4.16.0-2     | dpkg | LOW      |                  |     7.1 |     5.5 |
+------------------+-------------------------+------+----------+------------------+---------+---------+

Detailed report can be found at
https://defense-eu.conferdeploy.net/kubernetes/image/sha256:e1afa64d6afd381bdcec761b92c77abf6a86a7812fba35df25a6c8d883197837/overview
```

In the cloud interface, you get a summary of the container image scan, as well as a detailed overview under the tab 'Vulnerabilities'.

{{< image src="/images/vmware-carbon-black-kubernetes-image-scanning/cb-image-scan-overview-cloud.png" caption="Overview of the Ubuntu Image Scan" >}}

{{< /admonition >}}

### Deployment without scanned images

If a container image is not scanned before deployment, you will receive an error message that the deployment was blocked by a Kubernetes security policy.

```bash
bash:~$ kubectl create deployment ubuntu-bionic-20220315 -n secure --image ubuntu:bionic-20220315
error: failed to create deployment: admission webhook "resources.validating-webhook.cbcontainers" denied the request: Blocked by Kubernetes security policies
```
