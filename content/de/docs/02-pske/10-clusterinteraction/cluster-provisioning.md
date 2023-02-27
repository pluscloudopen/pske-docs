---
title: "PSKE - Cluster provisionierung"
linkTitle: "Cluster provisionierung"
weight: 10
date: 2023-02-21
---

Im Menü "Clusters" (1) wählen Sie im Anschluss bei "Kubernetes Clusters" das Plus-Symbol (2). Die Eingabemaske für die Clustererstellung erscheint.

![1](/images/content/02-pske/10-clusterinteraction/cluster-provisioning/1.png)

Bei "Infrastructure" (1) ist unsere pluscloud open bereits für Sie ausgewählt. Im folgenden Schritt geben Sie Details zum Cluster an (2), wie etwa den Clusternamen, Kubernetes-Version und den Zweck (Purpose) des zu erstellenden Clusters. Sie haben die Möglichkeit, den vordefinierten Clusternamen (random) zu wählen, können aber auch einen eigenen, sprechenderen Namen wählen:

![2](/images/content/02-pske/10-clusterinteraction/cluster-provisioning/2.png)

Optional "YAML" (3): Hier können die IP-Netze der Worker Nodes definiert werden, welche sich mit den IP-Netzen 10.20.0.0/16, 192.168.123.0/24 nicht überlappen dürfen. Diese Auswahl kann wichtig sein, insofern Sie später Ihr Kubernetes Cluster via HybridConnector mit einer Bestandsumgebung verbinden wollen.

```yaml
spec:
  provider:
    type: openstack
    infrastructureConfig:
      networks:
        workers: 10.250.0.0/16
  networking:
    nodes: 10.250.0.0/16
```

Im Punkt "Worker" wählen Sie aus, wie Ihre Worker Nodes gesized werden sollen und wie viele Sie erstellen möchten. Die Master/Controlplane Nodes sind für Sie nicht konfigurierbar. Diese werden vom Gardener gemanagt.

Das folgende Beispiel erstellt zwei Worker Nodes mit dem Flavor "SCS-4V:8:100" (4 vCPUs, 8 GB RAM, 100 GB lokaler Storage). Es werden "Ubuntu 20.04" als Betriebssystem und  "containerd" als Container Runtime verwendet.

![3](/images/content/02-pske/10-clusterinteraction/cluster-provisioning/3.png)

Im Punkt "Maintenance" können Sie wählen, zu welchem Zeitpunkt Ihr Cluster auf Updates vom Worker-Betriebssystem und der Kubernetes-Version geprüft und ggf. geupdatet werden soll. Mit dem Punkt "Auto Update" können Sie kontrollieren, ob Updates automatisch eingespielt werden.

Der letzte Punkt "Hibernation" definiert, wann Ihr Cluster automatisch zu gewissen Zeiten heruntergefahren und die Worker-Node-Ressourcen entfernt werden. Das dient zur Kostenminimierung beim jeweiligen Cloud-Provider und ist besonders dienlich, wenn Sie ein Entwicklungscluster besitzen, das nicht permanent aktiv sein muss. Hier können Sie ein oder mehrere Zeitfenster bestimmen.

Mit einem abschließenden Klick auf "Create" wird das Cluster erstellt.