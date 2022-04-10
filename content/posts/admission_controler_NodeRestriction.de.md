---
title: "Admission Controller NodeRestriction in Aktion"
date: 2022-01-10T15:55:00+01:00
draft: false
author: "Benjamin Koltermann"
authorlink: "/authors/benjamin-koltermann/"
featuredImage: "/images/admission-controller-noderestriction/preview.png"
description: "Dieser Blogeintrag zeigt wie der Admission Controller NodeRestriction ein Cluster vor dem tieferen Eindringen eines Angreifers schützen kann."
tags: [kubernetes, admission controller, video]
categories: [Admission Controller]
---

<!--more-->

## Was ist ein Admission Controller

Ein Admission Controller prüft eingehende Anfragen an den API-Server, nachdem diese authentifiziert und autorisiert wurden, auf Gültigkeit. Kubernetes bringt viele unterschiedliche Admission Controller, welche verschiedene Aufgaben übernehmen, in der `kube-apiserver` Binary mit. Diese müssen aktiviert werden um innerhalb eines Kubernetes Clusters zu funktionieren. Sollte eine API-Anfrage von einem Admission Controller blockiert werden, erhält der Nutzer eine Fehlermeldung.

Eine vollständige Liste mit allen Admission Controllern und weiteren Informationen findet man [hier](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/).

### Der Admission Controller NodeRestriction

Der NodeRestriction Admission Controller ist vorallem interessant, weil er einen Angreifer dabei hindern kann ein Kubernetes Cluster weiter zu exploiten, sollte dieser eine Node unter seiner Kontrolle haben. Er sorgt dafür, dass ein kubelet nur noch Node und Pod API Objekte verwenden darf, welche an ihre Node gebunden sind. Konkret kann ein Angreifer mit einer gehackten Node nur die gehackte Node, nicht aber andere Nodes beeinflussen.

Ausserdem hindert er das kubelet am modifizieren von allen Labeln mit dem Präfix `node-restriction.kubernetes.io/`, was zur Folge hat, dass ein Deployment mit einem `nodeSelector` Feld, welches auf eine gelabelte Node mit Labelpräfix `node-restriction.kubernetes.io/`, verweist, auch immer auf diese Node deployt wird. Der Angreifer hat keine Möglichkeit, die Label zu ändern, sodass das Deployment auf einer anderen Node landet aufgrund des `nodeSelector` Feld.

{{< admonition type=warning title="Wichtig" open=true >}}

Damit der NodeRestriction Admission Controller richtig funktioniert, muss jedes kubelet Zugangsdaten aus der `system:nodes` Gruppe in der Form `system:node:<nodeName>` nutzen. Hierzu empfiehlt es sich, die [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/) mit dem Parameter `--authorization-mode=Node` zu aktivieren.

{{< /admonition >}}

## Aktivieren des NodeRestriction Admission Controllers

Um den Admission Controller zu aktivieren, muss der API-Server mit dem Argument `--enable-admission-plugins=...,NodeRestriction,...` gestartet werden. Das Argument kann in dem Manifest des API-Servers ergänzt werden. Hier zu sehen in Zeile 10.

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

Im Anschluss muss der API-Server neu gestartet werden um die Konfiguration zu übernehmen. Ein einfaches `systemctl restart kubelet` auf der entsprechenden Node reicht hierfür aus.

## Beispiel Szenario

Als Demonstration dient ein Kubernetes Cluster mit einer Control Plane sowie zwei Worker Nodes (Worker-1 und Worker-2). Ein Administrator erstellt ein Deployment, welches auf Worker-1 deployed werden soll. Mithilfe des `nodeSelector` PodSpec Feld im Deployment werden die Nodes vom `kube-scheduler` nach dem entsprechenden Label untersucht. Anschliessend wird das Deployment auf der Node geplant, welche das entsprechende Label besitzt.

### Normales Deployment

{{< admonition type=info title="Szenario Beschreibung" open=true >}}

Node Selektor vom Deployment: `apps: web`

Kompromitierte Node: --

Deployment geplant: Worker-1

Deployment geplant nach Angriff: Worker-1

| Label | Node/Deployment | Vorher | Nachher |
| --- | ------- | ----------- | ------------ |
| apps  | Worker-1        | web    | web     |
| apps  | Worker-2        | web    | web     |
| apps  | Deployment      | web    | web     |

{{< /admonition >}}

{{< image src="/images/admission-controller-noderestriction/normal.svg" caption="Kubernetes Deployment auf Worker-1 mithilfe eines Node Selektors" >}}

### Angriff ohne NodeRestriction Admission Controller

{{< admonition type=info title="Szenario Beschreibung" open=true >}}

Node Selektor vom Deployment: `apps: web`

Kompromitierte Node: Worker-2

Deployment geplant: Worker-1

Deployment geplant nach Angriff: Worker-2

| Label | Node/Deployment | Vorher | Nachher |
| --- | ------- | ----------- | ------------ |
| apps  | Worker-1   | web    | false   |
| apps  | Worker-2   | db     | web     |
| apps  | Deployment | web    | web     |

{{< /admonition >}}

{{< image src="/images/admission-controller-noderestriction/hacked.svg" caption="Angreifer nutzt eine gekaperte Node aus und lässt das Deployment auf seine Node deployen" >}}

### Angriff mit NodeRestriction Admission Controller

{{< admonition type=info title="Szenario Beschreibung" open=true >}}

Node Selektor vom Deployment: `apps: web`

Kompromitierte Node: Worker-2

Deployment geplant: Worker-1

Deployment geplant nach Angriff: Worker-1

| Label | Node/Deployment | Vorher | Nachher |
| --- | ------- | ----------- | ------------ |
| worker-1.node-restriction.kubernetes.io/apps | Worker-1   | web    | web   |
| worker-2.node-restriction.kubernetes.io/apps | Worker-2   | db     | db    |
| worker-1.node-restriction.kubernetes.io/apps | Deployment | web    | web   |

{{< /admonition >}}

{{< image src="/images/admission-controller-noderestriction/secure.svg" caption="Der Admission Controler NodeRestriction verhindert, dass das Deployment auf die Node des Angreifers deployed wird" >}}

{{< admonition type=question title="Was macht der Admission Controller?" open=false >}}

Der Angreifer kann sich von der gehackten Node aus, der Control Plane gegenüber, als Worker-2 Node authentifizieren. Die gehackte Node hatte keine Rechte um das Label von Worker-1 so zu ändern, dass das Deployment auf Worker-2 geplant wird. Der Admission Controller NodeRestriction, hat erfolgreich das Updaten der Labels einer von Worker-1 verhindert.

{{< /admonition >}}

## Demo Angriff Video

{{< youtube eDI-ALRoO4Q >}}
