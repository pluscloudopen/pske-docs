
---
title: "Forward Source IP"
linkTitle: "Forward Source IP"
weight: 4
date: 2023-02-21
---

Mit der default Konfiguration des Ingress Controllers und OpenStack Loadbalancers kommen aus Sicht des Kubernetes Clusters alle externen Webanfragen von der internen IP des Loadbalancers.

Sollte die externe IP innerhalb des Kubernetes Clusters benötigt werden, muss das sog. Proxy Protokoll aktiviert werden.

{{% alert title="Warnung" color="warning" %}}
Wenn das Proxy Protokoll aktiviert wird, kann der Ingress nicht mehr von innerhalb des Clusters angesprochen werden!
{{% /alert %}}

### Beispiel NGINX Ingress Controller
Der ConfigMap des Ingress Controllers müssen die folgenden Zeilen hinzugefügt werden:

```yaml
use-proxy-protocol: "true"
use-forwarded-headers: "true"
```

Zusätzlich muss dem dazugehörigen Loadbalancer Service für den NGINX Ingress Controller eine Annotation hinzugefügt werden:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    loadbalancer.openstack.org/proxy-protocol: "true"
```

Nun können Anwendungen innerhalb des Kubernetes Clusters die externe IP Adresse von externen Webanfragen einsehen.