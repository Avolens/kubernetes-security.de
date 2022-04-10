---
title: "Admission Controller NodeRestriction in Action"
date: 2022-01-10T15:55:00+01:00
draft: false
author: "Benjamin Koltermann"
authorlink: "/en/authors/avolens/benjamin_koltermann/"
featuredImage: "/images/admission-controller-noderestriction/preview.png"
description: "This blog post demonstrates how the Admission Controller NodeRestriction can protect a cluster from deeper penetration by an attacker."
tags: [kubernetes, admission controller, video]
categories: [Admission Controller]
---

<!--more-->

## What is an Admission Controller

An Admission Controller checks incoming requests to the API server for validity after they have been authenticated and authorized. Kubernetes has many different built in Admission Controllers, which perform different tasks. These are already compiled in the `kube-apiserver` binary, but must be enabled to function within a Kubernetes cluster. If an API request is blocked by an Admission Controller, the user will receive an error message.

A complete list of all Admission Controllers and more information can be found [here](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/).

### The NodeRestriction Admission Controller

The NodeRestriction Admission Controller is interesting because it can prevent an attacker from further exploiting a Kubernetes cluster if they have a node under their control. It ensures that a kubelet can only use node and pod API objects that are bound to their node. Specifically, an attacker with a hacked node can only affect the hacked node, but not other nodes.

It also prevents the kubelet from modifying any labels with the prefix `node-restriction.kubernetes.io/`, which means that a deployment with a `nodeSelector` field pointing to a labeled node with label prefix `node-restriction.kubernetes.io/` will always be deployed to that node. The attacker has no way to change the labels, that the deployment ends up on a different node because of the `nodeSelector` field.

{{< admonition type=warning title="Important" open=true >}}

For the NodeRestriction Admission Controller to work properly, each kubelet must use credentials from the `system:nodes` group in the form `system:node:<nodeName>`. For this, it is recommended to enable [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/) with the parameter `--authorization-mode=Node`.

{{< /admonition >}}

## Aktivieren des NodeRestriction Admission Controllers

To enable the Admission Controller, the API server must be started with the argument `--enable-admission-plugins=...,NodeRestriction,...`. The argument can be added to the API server's manifest. Here you can see it in line 10.

```yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
...
```

Afterwards the API server must be restarted to apply the configuration. A simple `systemctl restart kubelet` on the corresponding node is sufficient for this.

## Example scenario

A Kubernetes cluster with a control plane and two worker nodes (worker-1 and worker-2) serves as a demonstration. An administrator creates a deployment to be deployed on worker-1. Using the `nodeSelector` PodSpec field in the deployment, the `kube-scheduler` scans the nodes for the appropriate label. Then the deployment is scheduled on the node that has the corresponding label.

### Normal deployment

{{< admonition type=info title="Scenario description" open=true >}}

Node selector from deployment: `apps: web`

Compromised node: --

Deployment planned: worker-1

Deployment planned after attack: worker-1

| Label | Node/Deployment | Before | After |
| --- | ------- | ----------- | ------------ |
| apps  | worker-1        | web    | web     |
| apps  | worker-2        | web    | web     |
| apps  | Deployment      | web    | web     |

{{< /admonition >}}

{{< image src="/images/admission-controller-noderestriction/normal.svg" caption="Kubernetes Deployment on worker-1 Using a Node Selector" >}}

### Attack without NodeRestriction Admission Controller

{{< admonition type=info title="Scenario description" open=true >}}

Node selector from deployment: `apps: web`

Compromised Node: worker-2

Deployment planned: worker-1

Deployment planned after attack: worker-2

| Label | Node/Deployment | Before | After |
| --- | ------- | ----------- | ------------ |
| apps  | worker-1   | web    | false   |
| apps  | worker-2   | db     | web     |
| apps  | Deployment | web    | web     |

{{< /admonition >}}

{{< image src="/images/admission-controller-noderestriction/hacked.svg" caption="Attackers exploit a hijacked node to let the deployment schedule to their node" >}}

### Attack with NodeRestriction Admission Controller

{{< admonition type=info title="Scenario description" open=true >}}

Node selector from deployment: `apps: web`

Compromised Node: worker-2

Deployment planned: worker-1

Deployment planned after attack: worker-1

| Label | Node/Deployment | Before | After |
| --- | ------- | ----------- | ------------ |
| worker-1.node-restriction.kubernetes.io/apps | worker-1   | web    | web   |
| worker-2.node-restriction.kubernetes.io/apps | worker-2   | db     | db    |
| worker-1.node-restriction.kubernetes.io/apps | Deployment | web    | web   |

{{< /admonition >}}

{{< image src="/images/admission-controller-noderestriction/secure.svg" caption="The Admission Controller NodeRestriction prevents the deployment from being scheduled to the attacker's node" >}}

{{< admonition type=question title="What does the Admission Controller do?" open=false >}}

The attacker can authenticate from the hacked node to the control plane as a Worker-2 node. The hacked node had no privileges to change the label of Worker-1 to schedule deployment to Worker-2. The Admission Controller NodeRestriction, successfully prevented the updating of the labels of one of Worker-1.

{{< /admonition >}}

## Demo Attack Video

The demo video is only available in German.

{{< youtube eDI-ALRoO4Q >}}
