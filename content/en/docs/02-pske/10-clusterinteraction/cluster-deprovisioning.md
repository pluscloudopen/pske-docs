---
title: "PSKE - Cluster deprovisionierung"
linkTitle: "Cluster deprovisionierung"
weight: 30
date: 2023-02-21
---

## Kubernetes Cluster deprovisionieren

Zum Löschen eines Kubernetes Clusters wechseln Sie in der linken Menüleiste auf "CLUSTERS" (1). Anschließend können Sie über die drei Punkte (2) die Löschung des Kubernetes Clusters initiieren (3).

![11](/images/content/02-pske/10-clusterinteraction/cluster-deprovisioning/11.png)


Bestätigen Sie die Löschung des Kubernetes Clusters durch den Namen (1) und wählen Sie anschließend "Delete" (2).

![12](/images/content/02-pske/10-clusterinteraction/cluster-deprovisioning/12.png)

Das Kubernetes Cluster wird nun gelöscht. Sie können Sich den Status (1) der Löschung anzeigen lassen, um mehr Informationen (2) zu erhalten. Es werden sämtliche Ressourcen gelöscht. Das umfasst die Controlplane Nodes, Worker Nodes, PersistentVolumes, Loadbalancer und FloatingIPs.

![13](/images/content/02-pske/10-clusterinteraction/cluster-deprovisioning/13.png)

## HybridConnector deprovisionieren + Kubernetes Cluster löschen

Bevor bei "Delete Cluster" (2) das Cluster gelöscht werden kann, muss ein Ticket mit folgenden Informationen eröffnet werden:

* Vertragsnummer der pluscloud open
* Namen des Kubernetes Clusters
* Namen der Bestandsumgebung

![14](/images/content/02-pske/10-clusterinteraction/cluster-deprovisioning/14.png)

Eine Löschung des Clusters ohne vorherige Deprovisionierung des HybridConnectors führt zu folgender Fehlermeldung und der Kubernetes Cluster lässt sich nicht löschen.

{{< alert color="warning" title="Warning">}}
task "Waiting until shoot infrastructure has been destroyed" failed: Failed to delete Infrastructure shoot--sknop--demo-cluster/demo-cluster: Error deleting infrastructure: Terraform execution for command 'destroy' could not be completed:

* Error deleting openstack_networking_router_v2 <omitted>: Expected HTTP response code [] when accessing [DELETE https://intern1.api.pco.get-cloud.io:9696/v2.0/routers/<omitted>], but got 409 instead
{"NeutronError": {"type": "RouterInUse", "message": "Router <omitted> still has ports", "detail": ""}}
{{< /alert >}}