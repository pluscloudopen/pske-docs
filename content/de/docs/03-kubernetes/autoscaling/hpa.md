---
title: "Horizontal Pod Autoscaler (HPA)"
linkTitle: "Horizontal Pod Autoscaler (HPA)"
weight: 20
date: 2023-02-21
---

Eine der Formen der automatischen Skalierung, ist der HPA, welcher bei der die Anzahl der Pods in zum Beispiel einem Deployment oder einem Replica Set auf der Grundlage verschiedener Metriken ( wie z.B. der RAM-, CPU-Auslastung, Tageszeit oder Traffic auf einem LB ) erhöht oder verringert wird - die Skalierung ist horizontal, da sie die Anzahl der Instanzen und nicht die einem einzelnen Pod zugewiesenen Ressourcen betrifft.

Der HPA kann Skalierungsentscheidungen auf der Grundlage von benutzerdefinierten oder extern bereitgestellten Metriken ( über z.B. Prometheus ) treffen und arbeitet nach der Erstkonfiguration automatisch. Es müssen lediglich die MIN- und MAX-Anzahl der Replikate festgelegt werden.

Nach der Konfiguration überprüft der Horizontal Pod Autoscaler Controller die Metriken in zyklischen Abständen und skaliert die Pods entsprechend nach oben oder unten. Standardmäßig überprüft HPA die Metriken alle 15 Sekunden.

Zur Überprüfung der Metriken ist der HPA auf eine andere Kubernetes-Ressource, den Metrics Server, angewiesen. Der Metrics Server liefert standardmäßige Messdaten zur Ressourcennutzung, indem er Daten der "kubernetes.summary_api" erfasst, wie z. B. CPU- und Speichernutzung für Nodes und Pods. Er kann auch Zugriff auf benutzerdefinierte Metriken bieten (die von einer externen Quelle gesammelt werden können), wie z. B. die Anzahl der aktiven Sitzungen auf einem Load Balancer, oder den Load eines Backends.

Dadurch wird sichergestellt, dass eine Anwendung auch unter Last, immer verfügbar ist und ihre Funktion erfüllen kann. Genau so, wird dafuer gesorgt, dass eine Applikation nicht mehr als die momentan erforderlichen Replikate ausfuehrt und somit die Ressourcennutzung und "Kosten in beide Richtungen" optimiert. Da die Kosten in den Meisten Setups (Workernodes) aber eher statisch sind und nur in seltetenen Fällen automatisch mitskalieren, ist die Skalierung natürlich nur im Rahmen der clusterweiten verfügbaren Rechenleistung möglich.

### Wie funktioniert der HPA ?

![HPA](/images/content/03-kubernetes/autoscaling/hpa.png)

Der HPA fragt in regelmäßigen Abständen bei der Kubernetes-API die Ressourcennutzung eines Pods ab und passt dann die Anzahl der Replikate nach Bedarf an, um ein Zielniveau für die Ressourcennutzung zu erreichen.

Im Detail funktioniert dies wie folgt:

HPA überwacht die in der Config angegebene Metrik. (Standard alle 15 Sekunden)
Auf der Grundlage der erfassten Metrik berechnet HPA die gewünschte Anzahl der erforderlichen Replikate.
Der HPA skaliert dann bei bedarf den Pod auf die gewünschte Anzahl von Replikaten.
Da HPA eine kontinuierliche Überwachung durchführt, wiederholt sich der Prozess ab Schritt 1.
Weitere Infos zum Algorithmus findet man hier: https://kubernetes.io/de/docs/tasks/run-application/horizontal-pod-autoscale/#details-zum-algorithmus

### Wie wird der HPA genutzt ?

Um den HPA zu nutzen, muss man eine Kubernetes-Ressource des Typs HorizontalPodAutoscaler erstellen. In dieser Ressource gibt man das zu skalierende Deployment oder RC, die minimale und maximale Anzahl der Replikate sowie die Zielressourcennutzung oder benutzerdefinierte Metriken an.

Hier ein Beispiel für die Konfiguration eines HPA für ein Deployment, auf Basis von Kubernetes-Metriken:

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: my-deployment-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
```

In dieser Konfiguration wird die minimale Anzahl der Replikate auf 1 und die maximale Anzahl der Replikate auf 10 festgelegt, und es wird eine CPU-Auslastung von 80 % angestrebt. HPA skaliert die Anzahl der Replikate der "my-deployment"-Einrichtung automatisch auf der Grundlage der CPU-Auslastung der Pods.

Wenn man den HPA auf Basis benutzerdefinierten Metriken agieren lassen moechte, muss er in der Lage sein, Metriken von einem Prometheus-Endpunkt abzurufen. Dazu kann man Prometheus so konfigurieren, dass es Metriken von den Pods abruft, oder man verwenden einen k8s-Dienst, der Metriken im Prometheus-Format exportiert.
Sobald man die Metriken in Prometheus zur Verfügung hat, kann ein Prometheus-Adapter verwenddet werden, um HPA für die Verwendung der Metriken zu konfigurieren.

Hier ein Beispiel für die Konfiguration einer HPA zur Skalierung auf der Grundlage einer benutzerdefinierten Metrik:

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: my-deployment-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: custom_metric
      targetAverageValue: 100
```

In dieser Konfiguration wird die Mindestanzahl der Replikate auf 1 und die Höchstanzahl der Replikate auf 10 beispielhaft festgelegt, sowie eine benutzerdefinierte Metrik namens "custom_metric" mit einem Durchschnittswert von 100 vorgegeben. Der HPA skaliert die Anzahl der Replikate des entsprechenden Deployments automatisch auf der Grundlage des Werts der "custom_metric".

Es ist auch erwähnenswert, dass der HPA einen Control-Loop verwendet, um sicherzustellen, dass die Replikate die gewünschte Anzahl erreichen. Dies geschieht durch Abfrage des API-Servers nach der jeweiligen Konformität der Anforderungen.

### Limitierungen des Horizontal Pod Autoscalers
HPA ist zwar ein leistungsfähiges Tool, aber nicht für jeden Anwendungsfall ideal und kann nicht jedes Problem mit Cluster-Ressourcen lösen. Es gibt dabei einige Dinge zu beachten:

**Der HPA ...**

... funktioniert nur mit stateless Applications, die eine Replikation vertragen oder Stateful Sets, welche eine persistenz ermöglichen

... funktioniert nicht mit Deamon Sets

... kann, wenn man die Grenzen für die Metriken der Pods nicht effizient festlegt, Pods häufig beenden oder Ressourcen verschwenden

... kann nicht zusammen mit dem VPA verwendet werden auf Basis der selben Metriken

... kann nicht skalieren, wenn die Gesamtkapazität des Clusters erschöpft ist, bis neue Nodes zum Cluster hinzugefügt werden