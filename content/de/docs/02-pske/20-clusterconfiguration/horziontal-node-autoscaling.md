---
title: "PSKE - Horizontal Node Autoscaling"
linkTitle: "Horizontal Node Autoscaling"
weight: 10
date: 2023-02-21
---

Der Horizontal Node Autoscaler (HNA) ist ein Tool, welches automatisch die Anzahl an Worker Nodes eines Kubernetes Clusters anpasst, wenn folgende Bedingungen zutreffen:

- Es gibt Pods, die aufgrund von mangelnden Ressourcen nicht gestartet werden können <br>
- Es gibt Worker Nodes, die für einen bestimmten Zeitraum (Default: 30m) zu gering ausgelastet sind und die Pods auf anderen Worker Nodes verteilt werden können

## Voraussetzung

Damit der Horizontal Node Autoscaler in ein Shoot Cluster installiert wird müssen die Angaben "Autoscaler Min." und "Autoscaler Max." in mindestens einer Worker Group definiert sein

- "Autoscaler Min." definiert die minimale Anzahl an Worker Nodes innerhalb der Worker Group <br>
- "Autoscaler Max." definiert die maximale Anzahl an Worker Nodes, die der Horizontal Node Autoscaler bei Ressourcen-Engpässen innerhalb einer Worker Group bereitstellen wird

![HNA](/images/content/02-pske/30-clusterconfiguration/hna.png)

## Simulation
Aktuell existiert in dem Shoot Cluster eine Worker Node:

```bash
kubectl describe node shoot--ldtivqit95-worker-jh07p-z1-7d897-cgrw6
Capacity:
  cpu:                2
  ephemeral-storage:  50633164Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             4030532Ki
  pods:               110
Allocatable:
  cpu:                1920m
  ephemeral-storage:  49255941901
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             2879556Ki
  pods:               110
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                1047m (54%)   0 (0%)
  memory             1120Mi (39%)  18788Mi (668%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
```

Es wird ein NGINX Deployment erstellt:

```yaml
kubectl apply -f deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "2048Mi"
          #   cpu: "500m"
          limits:
            memory: "2048Mi"
            # cpu: "500m"
```

Die Ressourcen des bestehenden Worker Nodes reichen nicht aus und es wird der cluster-autoscaler (Horizontal Node Autoscaler) getriggert:

```bash
k describe pod nginx-54c7fd947f-b2k67
Events:
  Type     Reason            Age   From                Message
  ----     ------            ----  ----                -------
  Warning  FailedScheduling  37s   default-scheduler   0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
  Normal   TriggeredScaleUp  26s   cluster-autoscaler  pod triggered scale-up: [{shoot--ldtivqit95-worker-jh07p-z1 1->2 (max: 3)}]
```

Nach Löschung des Deployments wurde der zusätzliche Worker Node nach 30m vom Cluster Autoscaler (Horizontal Node Autoscaler) wieder deprovisioniert.

## Best Practices
- Modifizeren Sie keine Nodes via Hand, die Teil einer Autoscaling Gruppe sind. Alle Nodes in der selben Node Group sollten die selben Kapazitätsangaben und Labels besitzen
- Verwenden Sie Requests für die Container/Pods!
- Verwenden Sie PodDisruptionBudgets um zu verhindern, dass Pods zu abrupt gelöscht werden (falls benötigt)
- Überprüfen Sie, ob das Quota des Cloud Anbieters groß genug ist, bevor das min/max des Horizontal Node Autoscalers gesetzt wird
- Verwenden Sie keine zusätzlichen Node Group Autoscaler (auch nicht von Ihrem Cloud Anbieter)

## Fazit
Der Horizontal Node Autoscaler funktioniert wie eingangs beschrieben. Wichtig ist, dass die Best Practices eingehalten werden, damit der Cluster Autoscaler wie vorgesehen funktioniert.
Der HNA ist per Default in PSKE aktiviert und steht Ihnen zur Verfügung.