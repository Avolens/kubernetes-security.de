---
title: "VMware Carbon Black Kubernetes Image Scanning"
date: 2022-01-10T15:55:00+01:00
draft: false
author: "Benjamin Koltermann"
authorlink: "/authors/benjamin-koltermann/"
featuredImage: "/images/admission-controller-noderestriction/preview.png"
description: "In diesem Artikel geht es um das scannen und erkennen von Schwachstellen von Container Images mit der VMware Carbon Black Lösung. Außerdem wird die Integration des Container Scanning mithilfe von Policies in ein Kubernetes Cluster beschrieben."
tags: [vmware, carbon black, image scanning]
categories: [VMware]
---

<!--more-->

## Die Testumgebung und Installation

Als Testumgebung, haben wir ein einfaches 3 Node Cluster, mit 2 Worker und einer Master Node, bestehend aus den folgenden Komponenten gewählt. 

```bash
bash:~$ kubectl get nodes

```

### Carbon Black Kubernetes Sensor

Carbon Black oder genauer der Kubernetes Sensor für Carbon Black, wurde nach der offiziellen [Carbon Black Installationsanleitung](https://docs.vmware.com/en/VMware-Carbon-Black-Cloud/services/cbc-sensor-installation-guide/GUID-D2D621D7-E341-4F16-88AF-B5919958B142.html) installiert und konfiguriert. Nach dem der Sensor installiert ist, sollten alle Pods mit der Zeit starten.

```bash
bash:~$ kubectl get pods -n carbon-black

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

## Test Deployment

### Ohne gescanntes Images

### Mit gescanntem Image
