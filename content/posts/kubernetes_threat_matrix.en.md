---
title: "Kubernetes Threat Matrix"
date: 2022-08-01
draft: false
author: "Benjamin Koltermann"
authorlink: "/en/authors/benjamin-koltermann/"
featuredImage: "/images/kubernetes-threat-matrix/preview.png"
description: "This blog presents a Kubernetes threat matrix in the style of a MITRE ATT&CK matrix"
tags: [kubernetes, threat matrix]
categories: [Threat Matrix]
---

<!--more-->

## What is a MITRE ATT&CK Matrix

The threat matrices, are created using various techniques from the [MITRE ATT&CK framework](https://attack.mitre.org/). These techniques are categorized and recorded in the so-called Threat Matrix. MITRE itself provides different matrices to be used in a threat based analysis. Thus, there is a matrix for various "products" such as Windows, Linux, MacOS, Cloud or Containers in order to better classify the various threats. For Kubernetes, however, there is not yet an official matrix from MITRE itself, which is why we have created our own matrix in the course of our consulting work.

### Why you should use a MITRE ATT&CK Matrix

To successfully complete an attack, an attacker usually goes through the different phases of a MITRE ATT&CK matrix. The threat matrix helps to identify and classify different threats and which techniques could be used to exploit them.

## Kubernetes Threat Matrix

{{< image src="/images/kubernetes-threat-matrix/threat_matrix.png" caption="AVOLENS Kubernetes Threat Matrix" >}}

### Reconnaissance

[Reconnaissance](https://attack.mitre.org/tactics/TA0043/) is the gathering of information before the actual attack.

#### Public Kubernetes API Endpoint

A public Kubernetes API endpoint can be used to gain information about a Kubernetes cluster, e.g. by an attacker addressing the `/version` endpoint. The `/version` endpoint is active by default in Kubernetes and can be accessed without authentication. To prevent this, the Kubernetes API should only be accessible from certain networks.

{{< admonition type=info title="Kubernetes example version endpoint" open=true >}}

```json
{
  "major": "1",
  "minor": "23",
  "gitVersion": "v1.23.4",
  "gitCommit": "e6c093d87ea4cbb530a7b2ae91e54c0842d8308a",
  "gitTreeState": "clean",
  "buildDate": "2022-03-06T21:32:53Z",
  "goVersion": "go1.17.7",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

{{< /admonition >}}

### Initial Access

[Initial Access](https://attack.mitre.org/tactics/TA0001/) describes methods for an attacker to gain entry into a network.

#### Supply Chain Compromise

An attacker who manages to infect the supply chain of a Kubernetes cluster can gain access to various cluster resources, e.g. by exchanging container images. He could do this when creating container images in Jenkins or Gitlab CI/CD. The attacker would only need to add an extra build step in which he injects his malicious code into the image.

### Execution

In the [Execution](https://attack.mitre.org/tactics/TA0002/) phase, an attacker attempts to execute malicious code.

#### Malicious operator

An operator automates administrative tasks. Some applications can not run without an operator and are therefore dependent on it. Since an operator consists of code, an attacker can use the operator as a pretext to inject malicious code into a Kubernetes cluster. In addition Operators usually run with more privileges because they need to create and delete resources.

### Persistence

The attacker wants his attack [Persitent](https://attack.mitre.org/tactics/TA0003/) to continue to be successful even after reboots or updates.

#### Malicious Pause Container

If an attacker is able to inject their own Pause container into the cluster, this container will run in every Kubernetes pod. It is extremely difficult for an administrator to find out that the Pause container image has been changed, mainly because the Pause container is only visible when one list the running containers on the node.

#### Static Pod

Since a static pod cannot be deleted by the `kubelet`, a static pod in the hands of an attacker is not to be underestimated. An administrator must delete the corresponding Yaml file from the node for a static pod to be permanently deleted.

### Defense Evasion

[Defense Evasion](https://attack.mitre.org/tactics/TA0005/) includes techniques designed to prevent an attacker from being detected.

#### Delete Kubernetes Events

Events are used to track when something happened in the cluster. An attacker who can delete Kubernetes events can cause damage to the cluster undetected.

### Credential Access

[Credential Access](https://attack.mitre.org/tactics/TA0006/) describes how an attacker can steal passwords and usernames.

#### Malicious Admission Controller

If an attacker can control a MutatingAdmissionWebhook Admission Controller, he can use it to read all secrets and service account tokens. This gives the attacker access to additional accounts and passwords within the Kubernetes cluster.

### Lateral Movement

To move from one Pod to the next, an attacker uses [Lateral Movement](https://attack.mitre.org/tactics/TA0008/) techniques.

#### Container Service Account

Each pod in Kubernetes has a service account assigned to it. This means that if a pod is compromised, an attacker has access to valid Kubernetes API credentials. For this reason, service accounts should always be created according to the `least privilege principal`.

### Exfiltration

[Exfiltration](https://attack.mitre.org/tactics/TA0010/) Techniques an attacker uses to steal data from the network.

#### Malicious Admission Controller

An attacker has the ability to extract data from a cluster by creating an Admission Controller and sending the corresponding `AdmissionReview` to a server under his control. For example, an attacker can read all newly created or updated secrets.
