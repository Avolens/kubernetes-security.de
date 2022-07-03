---
title: "VMware Carbon Black Kubernetes Image Scanning"
date: 2022-05-10
draft: false
author: "Benjamin Koltermann"
authorlink: "/authors/benjamin-koltermann/"
featuredImage: "/images/vmware-carbon-black-kubernetes-image-scanning/preview.png"
description: "In diesem Artikel geht es um das scannen und erkennen von Schwachstellen von Container Images mit der VMware Carbon Black Lösung. Außerdem wird die Integration des Container Scanning mithilfe von Policies in ein Kubernetes Cluster beschrieben."
tags: [vmware, carbon black, image scanning]
categories: [VMware]
---

<!--more-->

## Die Testumgebung und Installation

Als Testumgebung, haben wir ein einfaches 3 Node Cluster, mit 2 Worker und einer Master Node, bestehend aus den folgenden Komponenten gewählt. 

```bash
bash:~$ kubectl get nodes
master     Ready    control-plane,master   101d   v1.23.3
worker-1   Ready    worker                 101d   v1.23.3
worker-2   Ready    worker                 101d   v1.23.3
```

### Carbon Black Kubernetes Sensor

Carbon Black oder genauer der Kubernetes Sensor für Carbon Black, wurde nach der offiziellen [Carbon Black Installationsanleitung](https://docs.vmware.com/en/VMware-Carbon-Black-Cloud/services/cbc-sensor-installation-guide/GUID-D2D621D7-E341-4F16-88AF-B5919958B142.html) installiert und konfiguriert. Nach dem der Sensor installiert ist, sollten alle Pods mit der Zeit starten.

```bash
bash:~$ kubectl get pods -n carbon-black
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

Zum scannen der Container Images wird die Carbon Black CLI benötigt. Diese kann in Carbon Black unter **Inventory** --> **Kubernetes** --> **Clusters** --> **CLI Config** eingerichtet und gedownloaded werden. Die Carbon Black Binary sollte anschließend einsatzbereit sein.

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

Dieser Artikel Beschäftigt sich **nicht** intensiv mit den Carbon Black Policies, weshalb nur der Prozess zum Erstellen von Carbon Black Kubernetes Policies für Image Scanning beschrieben wird. Die offizielle Dokumentation von VMware zu den Kubernetes Policies befindet sich [hier](https://docs.vmware.com/en/VMware-Carbon-Black-Cloud/services/carbon-black-cloud-user-guide/GUID-AE6ED527-C2DF-471B-92F8-0C9269975C2B.html).

{{< /admonition >}}

Die Carbon Black Kubernetes Image scanning Policiy, zählt zu den Hardening Policies. Somit muss in der Carbon Black Umgebung eine Policiy unter **Enforce** --> **K8s Policies** --> **Hardening Policies** angelegt werden. Nachdem der Policiy ein Name und Scope zugewiesen wurde findet man unter **Container Images** eine Regel mit dem Namen `Image not scanned`. Diese Regel fügen wir unserer Policy mit der Aktion `Block` hinzu. Anschließend bekommt man die Möglichkeit Außnahmen für bereits existierende Workloads innerhalb des gewählten Scopes zu definieren. Nachdem die Policy erstellt wurde, kann es eine halbe Minute dauern, bis diese greift.

{{< admonition type=info title="Kubernetes Scopes" open=true >}}

Die Wahl des Richtigen Kubernetes Scopes kann entscheidend sein. Da Kubernetes Scopes nicht in diesem Artikel behandelt werden, findet man eine Übersicht [hier](https://docs.vmware.com/en/VMware-Carbon-Black-Cloud/services/carbon-black-cloud-user-guide/GUID-14E951FE-F4DA-49EB-9FF5-B655BB0490DC.html).

{{< /admonition >}}

## Image Scanning mit der Carbon Black CLI

Das scannen eines Images lässt sich schnell und einfach durchführen. In der Carbon Black CLI wird der `image scan` Befehl genutzt, um ein Image zu scannen. Im Anschluss wird das zu scannende Image angegeben, die Ergebnisse des Scanns erhält man direkt als Ausgabe der CLI, man kann sich diese aber auch in der Carbon Black Cloud ansehen. Der Upload des Scanergebnis in die Carbon Black Cloud erfolgt automatisch, nach Abschluss des Image scan.

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

In der Cloud Oberfläche erhält man eine Zusammenfassung über den Container Image Scan, sowie eine detaillierte Übersicht unter den Reiter `Vulnerabilities`.

{{< image src="/images/vmware-carbon-black-kubernetes-image-scanning/cb-image-scan-overview-cloud.png" caption="Übersicht über den Image Scan des Ubuntu Image" >}}

{{< /admonition >}}

### Test Deployment ohne gescanntes Images

Sollte ein Container Image nicht vor dem Deployen gescannt sein, erhält man eine Fehlermeldung, dass das Deployment von einer Kubernetes Security Policy geblockt wurde.

```bash
bash:~$ kubectl create deployment ubuntu-bionic-20220315 -n secure --image ubuntu:bionic-20220315
error: failed to create deployment: admission webhook "resources.validating-webhook.cbcontainers" denied the request: Blocked by Kubernetes security policies
```
