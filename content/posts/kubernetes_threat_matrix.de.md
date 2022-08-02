---
title: "Kubernetes Threat Matrix"
date: 2022-08-01
draft: false
author: "Benjamin Koltermann"
authorlink: "/authors/benjamin-koltermann/"
featuredImage: "/images/kubernetes-threat-matrix/preview.png"
description: "In diesem Blog wird eine Kubernetes Threat Matrix im Stil einer MITRE ATT&CK Matrix vorgestellt."
tags: [kubernetes, threat matrix]
categories: [Threat Matrix]
---

<!--more-->

## Was ist eine MITRE ATT&CK Matrix

Die Threat Matrizen, werden mithilfe verschiedener Techniken aus dem [MITRE ATT&CK framework](https://attack.mitre.org/) erstellt. Diese Techniken werden Kategorisiert und in der sogenannten Threat Matrix festgehalten. MITRE selbst, stellt verschiedene Matrizen zur Verfügung um diese in einer auf Threats basierenden Analyse zu nutzen. So gibt es zu verschiedenen "Produkten" wie Windows, Linux, MacOS, Cloud oder Containern eine Matrix um die verschiedenen Threats besser einordnen zu können. Für Kubernetes gibt es jedoch noch keine offizielle Matrix von MITRE selber, weshalb wir im laufe unserer Beratungen eine eigene Matrix erstellt haben. Außerdem hat [Microsoft](https://www.microsoft.com/security/blog/2021/03/23/secure-containerized-environments-with-updated-threat-matrix-for-kubernetes/) eine Threat Matrix für Kubernetes erstellt, diese ist jedoch aus meiner Sicht nicht ganz vollständig.

### Warum sollte man eine MITRE ATT&CK Matrix nutzen?

Um einen Angriff erfolgreich abzuschliessen, durchläuft ein Angreifer meistens die verschiedenen Phasen einer MITRE ATT&CK Matrix. Die Threat Matrix hilft beim identifizieren und einordnen verschiedener Threats und welche Techniken angewendet werden könnten um diese zu exploiten.

## Kubernetes Threat Matrix

{{< image src="/images/kubernetes-threat-matrix/threat_matrix.png" caption="AVOLENS Kubernetes Threat Matrix" >}}

### Reconnaissance

Unter [Reconnaissance](https://attack.mitre.org/tactics/TA0043/) versteht man das beschaffen von Informationen, bevor dem eigentlichen Angriff.

#### Public Kubernetes API Endpoint

Ein öffentlicher Kubernetes API Endpunkt kann genutzt werden um Informationen über ein Kubernetes Cluster zu erlangen, z.B. indem ein Angreifer den `/version` Endpunkt anspricht. Der `/version` Endpunkt ist standardmäßig in Kubernetes aktiv und kann ohne Authentifizierung angesprochen werden. Um dies zu verhindern, sollte die Kubernetes API nur aus bestimmten Netzwerken erreichbar sein.

{{< admonition type=info title="Kubernetes Beispiel Version Endpunkt" open=true >}}

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

[Initial Access](https://attack.mitre.org/tactics/TA0001/) beschreibt Methoden, wie ein Angreifer in ein Netzwerk gelangen kann.

#### Supply Chain Compromise

Ein Angreifer, dem es gelingt die Supply Chain eines Kubernetes Clusters zu infizieren, kann Zugriff auf verschiedene Cluster Ressourcen erhalten, indem er z.b. Container Images austauscht. Dies könnte er beim erstellen von Container Images in Jenkins oder Gitlab CI/CD machen. Der Angreif muesste nur einen extra Build Step hinzufügen, in welchem er seinen Schadcode in das Image einbauen lässt.

### Execution

In der [Execution](https://attack.mitre.org/tactics/TA0002/) Phase, versucht ein Angreifer Schadcode auszuführen.

#### Malicious operator

Ein Operator automatisiert administrative Aufgaben. Manche Applikationen kommen ohne Operator nicht mehr aus und sind somit auf diesen angewiesen. Da ein Operator aus Code besteht, kann ein Angreifer den Operator als Vorwand nutzen um bösartigen Code in ein Kubernetes Cluster einzuschleusen. Operator laufen meistens mit erhöhten Rechten, da Sie Ressourcen erstellen und löschen müssen.

### Persistence

Der Angrifer möchte, dass Sein Angriff [Persitent](https://attack.mitre.org/tactics/TA0003/) also auch nach Neustarts oder Updates weiter erfolgreich ist.

#### Malicous Pause Container

Sollte ein Angreifer in der Lage sein, einen eigenen Pause Container in das Cluster einzuschleusen, so läuft dieser Container in jedem Kubernetes Pod. Ein Administrator hat es enorm schwer herauszufinden, dass das Pause Container Image geändert wurde, was vorallem daran liegt, dass der Pause Container nur sichtbar ist, wenn man auf der Node die Container auflistet.

#### Static Pod

Da ein statischer Pod nicht vom `kubelet` gelöscht werden kann, ist ein statischer Pod in der Hand eines Angreifers nicht zu unterschätzen. Ein Administrator muss die entsprechende Yaml Datei von der Node löschen, damit ein statischer Pod endgültig gelöscht werden kann.

### Defense Evasion

Unter [Defense Evasion](https://attack.mitre.org/tactics/TA0005/) fallen Techniken , welche verhindern sollen, dass ein Angreifer entdeckt wird.

#### Delete Kubernetes Events

Events dienen dazu nachzuvollziehen, wann etwas im Cluster passiert ist. Ein Angreifer, der Kubernetes Events löschen kann, kann unbemerkt Schaden im Cluster anrichten.

### Credential Access

[Credential Access](https://attack.mitre.org/tactics/TA0006/) beschreibt den vorgang wie ein Angreifer Passwörter und Nutzernamen klauen kann.

#### Malicious Admission Controller

Sollte ein Angreifer einen MutatingAdmissionWebhook Admission Controller kontrollieren können, so kann er mit diesem sämtliche Secrets und Service Account Tokens auslesen. Somit erhält der Angreifer Zugang zu weiteren Accounts und Passwörtern innerhalb des Kubernetes Clusters.

### Lateral Movement

Um von einem Pod zum nächsten zu gelangen nutzt ein Angreifer [Lateral Movement](https://attack.mitre.org/tactics/TA0008/) Techniken.

#### Container Service Account

Jeder Pod in Kubernetes hat einen Service Account zugewiesen. Das heisst, ist ein Pod kompromittiert, hat ein Angreifer Zugriff auf valide Kubernetes API Zugangsdaten. Aus diesem Grund sollten Service Accounts immer nach dem `Least Privilege Principal` erstellt werden.

### Exfiltration

[Exfiltration](https://attack.mitre.org/tactics/TA0010/) Techniken wendet ein Angreifer an um Daten aus dem Netzwerk zu entwenden.

#### Malicious Admission Controller

Ein Angreifer hat die Möglichkeit Daten aus einem Cluster zu extrahieren, indem er einen Admission Controller erstellt und den entsprechenden `AdmissionReview` an einem von Ihm kontrollierten Server sendet. So kann ein Angreifer z.B. alle neu erstellten oder geupdateten Secrets auslesen.
