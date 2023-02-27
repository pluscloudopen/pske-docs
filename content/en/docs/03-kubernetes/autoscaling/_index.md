---
title: "Kubernetes - Autoscaling"
linkTitle: "Kubernetes - Autoscaling"
weight: 20
date: 2023-02-21
description: >
  
---

## Formen von Kubernetes Autoscaling

Autoscaling ist eine Methode zur automatischen Skalierung von K8s-Workloads nach oben oder unten auf Basis/Erwartung der Ressourcennutzung. Autoscaling in Kubernetes hat drei Dimensionen:

{{< alert >}}
***Horizontal Pod Autoscaler (HPA)***: Passt die Anzahl der Replikate eines Pods an.

***Cluster-Autoscaler***: Passt die Anzahl der Nodes eines Clusters an.

***Vertical Pod Autoscaler (VPA)***: Passt die Ressourcenanforderungen und -grenzen eines Containers an.
{{< /alert >}}

Die verschiedenen Autoscaler arbeiten auf einer von zwei Kubernetes-Ebenen

**Pod-Ebene**: Die HPA- und VPA-Methoden finden auf der Pod-Ebene statt. Sowohl HPA als auch VPA skalieren die verfügbaren Ressourcen oder Instanzen des Pods, sowohl auf, als auch ab.

**Clusterebene**: Der Cluster-Autoscaler fällt unter die Clusterebene und skaliert die Anzahl der Nodes innerhalb Ihres Clusters nach oben oder unten.

{{< alert color="warning" title="Warning">}}
Gleichzeitige Verwendung von VPA und HPA
Der HPA und VPA können miteinander in Konflikt geraten - wenn man beispielsweise bei beiden den RAM als Basismetrik der Skalierung verwendet. Dadurch kann es passieren, dass beide versuchen, Workloads gleichzeitig vertikal und horizontal zu skalieren, was unvorhersehbare Folgen haben kann. Um solche Konflikte zu vermeiden, ist es Best Practice, dass HPA und VPA auf unterschiedliche Metriken Metriken achten.


In der Regel wird der VPA so eingestellt, dass die Skalierung auf der Basis von CPU oder RAM erfolgt, und benutzerdefinierte Metriken für HPA verwendet werden.
{{< /alert >}}