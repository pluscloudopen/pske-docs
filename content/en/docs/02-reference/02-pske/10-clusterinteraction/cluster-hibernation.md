---
title: "PSKE - Cluster hibernation"
linkTitle: "Cluster hibernation"
weight: 20
date: 2023-02-21
---

Das Feature "Hibernation" bietet Ihnen die Möglichkeit automatisiert oder auf Knopfdruck einen oder mehrere PSKE-Cluster in den Modus "hibernated" zu versetzen, um somit Kosten der Cloud-Ressourcen, die nicht 24/7 benötigt werden, einzusparen.

Bsp.: Test oder Development PSKE-Cluster, welche nur zur regulären Arbeitszeit betrieben werden.

## Hibernation

Durch Nutzung des Features "Hibernation" werden die folgenden PSKE-Cluster Komponenten heruntergefahren:

Workload
Worker Nodes des PSKE Clusters
Control Plane (kube-apiserver, etcd, kube-scheduler, kube-controller-manager, cloud-controler-manager)

Ausgenommen hiervon sind Floating IP-Adressen, Load Balancer und Persistent Volumes, welche weiter berechnet werden.

Da die Daten der Controlplane persistent vorgehalten werden und wir weiterhin Compute Ressourcen für den Cluster reservieren, wird die Clusterstunde weiterberechnet.

## Wake-Up

Beim "Aufwecken" wird das PSKE-Cluster mit seinen Komponenten inkl. Floating IP-Adressen, Load Balancer und Persistent Volumes

in seinen vorigen Ursprungszustand versetzt und kann somit wieder wie gewohnt genutzt werden.

## Konfiguration von Hibernation
### (1) Im PSKE-Dashboard
### (1.1) Manuell via YAML Cluster Manifest

Wenn Sie Ihren PSKE-Cluster manuell über das YAML Cluster Manifest in den Modus "hibernated" versetzen wollen, können Sie dies unter dem Punkt "spec" konfigurieren.

```yaml
spec:
  hibernation:
    enabled: true
    schedules:
      - start: "00 20 * * *"       # Start hibernation every day at 8PM
        end: "0 6 * * *"           # Stop hibernation every day at 6AM
      location: "Europe/Berlin"  # Specify a location for the cron to run in
```

Der Start- und Endpunkt orientiert sich an der bekannten Cron Syntax aus der Crontab in der Unix/Linux Welt.

### (1.2) Manuell via Knopfdruck

Sollten Sie Ihren PSKE-Cluster in den Modus "hibernated" versetzen wollen, können Sie über Clusters (1) den entsprechenden PSKE-Cluster auswählen und über die drei Punkte (2) den Menüpunkt "Hibernate Cluster" (3) auswählen.

![1](/images/content/02-pske/10-clusterinteraction/cluster-hibernation/1.png)

![2](/images/content/02-pske/10-clusterinteraction/cluster-hibernation/2.png)

### (1.3) Automatisiert via Hibernation Schedule

Wenn Sie Ihren PSKE-Cluster automatisiert in den Modus "hibernated" versetzen wollen, können Sie dazu den Hibernation

Schedule verwenden und Start- und Endpunkt sowie Wochentag separat je PSKE-Cluster konfigurieren.

#### Neues PSKE-Cluster

![3](/images/content/02-pske/10-clusterinteraction/cluster-hibernation/3.png)

Es können ein oder mehrere Hibernation Schedule Tasks konfiguriert werden. Erstellen Sie dazu unter (1) einen neuen PSKE-Cluster und scrollen Sie bis zum Ende der Seite. Die Wochentage, Start- und Endpunkt sowie auch ohne Startpunkt (Wake up at) können Sie über (2) konfigurieren. Sie können ebenfalls auch weitere Hibernation Schedule Tasks (3) erstellen und mit (4) die PSKE-Cluster Erstellung abschließen. 

#### Bestehendes PSKE-Cluster

Wählen Sie dazu unter (1) das entsprechende PSKE-Cluster aus und klicken Sie auf den Namen (2) um zum Punkt Hibernation (3) zu kommen.

![4](/images/content/02-pske/10-clusterinteraction/cluster-hibernation/4.png)

![5](/images/content/02-pske/10-clusterinteraction/cluster-hibernation/5.png)

Die Wochentage, Start- und Endpunkt sowie auch ohne Startpunkt (Wake up at) können Sie über (1) konfigurieren, Sie können aber auch weitere Hibernation Schedule Tasks (2) erstellen und mit (3) die Erstellung der Hibernation Schedule Tasks für das ausgewähle PSKE-Cluster speichern.

![6](/images/content/02-pske/10-clusterinteraction/cluster-hibernation/6.png)