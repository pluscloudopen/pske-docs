---
title: "Einrichtung eines Vertical Pod Autoscalers (VPA)"
linkTitle: "Einrichtung eines Vertical Pod Autoscalers (VPA)"
weight: 20
date: 2023-02-21
---

Im folgenden Beispiel wird eine Applikation deployed, welche aus einem freizugänglichen Image resultiert. Dieses Beispiel dient der Veranschaulichung, wie der HPA auf Zustandsveränderungen reagiert.

## 1) Provisionieren eines Clusters
Zuerst sollte ein Kubernetes Cluster provisioniert werden.

## 2) Deployment der Applikation
### Ausrollen des Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: hpa-demo-deployment
spec:
 selector:
   matchLabels:
     run: hpa-demo-deployment
 replicas: 1
 template:
   metadata:
     labels:
       run: hpa-demo-deployment
   spec:
     containers:
     - name: hpa-demo-deployment
       image: k8s.gcr.io/hpa-example
       ports:
       - containerPort: 80
       resources:
         limits:
           cpu: 500m
         requests:
           cpu: 200m
```


```bash
$ kubectl apply -f deployment.yml
deployment.apps/hpa-demo-deployment created
```

```bash
$ kubectl -n default get pods
NAME                                  READY   STATUS    RESTARTS   AGE
hpa-demo-deployment-85bff8877-9z9p9   1/1     Running   0          3s
```
```bash
$ kubectl -n default get deployment
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
hpa-demo-deployment   1/1     1            1           6s
```

### Erstellen eines Services

```yaml
apiVersion: v1
kind: Service
metadata:
 name: hpa-demo-deployment
 labels:
   run: hpa-demo-deployment
spec:
 ports:
 - port: 80
 selector:
   run: hpa-demo-deployment
```

```bash
$ kubectl apply -f service.yaml
service/hpa-demo-deployment created
```

```bash
$ kubectl -n default get services # Kann auch svc abgekürzt werden
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hpa-demo-deployment   ClusterIP   10.122.166.51   <none>        80/TCP    5s
kubernetes            ClusterIP   10.112.0.1      <none>        443/TCP   13d
```

### Konfigurieren des HPA

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
 name: hpa-demo-deployment
spec:
 scaleTargetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: hpa-demo-deployment
 minReplicas: 1
 maxReplicas: 10
 targetCPUUtilizationPercentage: 50
```

```bash
$ kubectl apply -f service.yaml
service/hpa-demo-deployment created
```

```bash
$ kubectl -n default get horizontalpodautoscaler # Kann auch hpa abgekürzt werden
NAME                  REFERENCE                        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo-deployment   Deployment/hpa-demo-deployment   0%/50%    1         10        1          11s
```

## 3) Testing
### Status vor dem Testing

```bash
$ kubectl -n default get pods
NAME                                  READY   STATUS    RESTARTS   AGE
hpa-demo-deployment-85bff8877-9z9p9   1/1     Running   0          20m
```

```bash
$ kubectl -n default get horizontalpodautoscaler # Kann auch hpa abgekürzt werden
NAME                  REFERENCE                        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo-deployment   Deployment/hpa-demo-deployment   0%/50%    1         10        1          20m
```

### Erzeugen von Last

Mit folgendem Befehl wird eine Last simuliert:

```bash
kubectl run -i --tty load-simulation --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hpa-demo-deployment; done"
```

### Verhalten während Last


#### Nach ca. 30 Sekunden
Der HPA bemerkt, dass der Pod das Target überschritten hat und fängt damit an, dementsprechend gegenzusteuern.
Daraufhin stellt 4 Replikas zur Verfügung und wartet auf deren Readiness.

```bash
$ kubectl -n default get pods && kubectl -n default get horizontalpodautoscaler
NAME                                  READY   STATUS   RESTARTS   AGE
hpa-demo-deployment-85bff8877-5b49k   1/1     Running  0          28s
hpa-demo-deployment-85bff8877-98qdq   1/1     Running  0          58s
hpa-demo-deployment-85bff8877-9z9p9   1/1     Running  0          2m51s
hpa-demo-deployment-85bff8877-sfdjh   1/1     Running  0          58s
hpa-demo-deployment-85bff8877-vtn48   1/1     Running  0          58s
load-simulation                       1/1     Running  0          2m24s

NAME                  REFERENCE                        TARGETS    MINPODS   MAXPODS    REPLICAS   AGE
hpa-demo-deployment   Deployment/hpa-demo-deployment   241%/50%   1         10         4          24m
```

#### Nach ca. 3 Minuten
Der HPA  bemerkt, dass trotz der gewählten Replikate das Target überschritten wird und stellt deshalb ein weiteres Replikat dazu.

```bash 
$ kubectl -n default get pods && kubectl -n default get horizontalpodautoscaler
NAME                                  READY   STATUS   RESTARTS   AGE
hpa-demo-deployment-85bff8877-5b49k   1/1     Running  0          3m29s
hpa-demo-deployment-85bff8877-98qdq   1/1     Running  0          3m59s
hpa-demo-deployment-85bff8877-9z9p9   1/1     Running  0          6m42s
hpa-demo-deployment-85bff8877-gmvr2   1/1     Running  0          58s
hpa-demo-deployment-85bff8877-sfdjh   1/1     Running  0          3m59s
hpa-demo-deployment-85bff8877-vtn48   1/1     Running  0          3m59s
load-simulation                       1/1     Running  0          5m25s

NAME                  REFERENCE                        TARGETS    MINPODS   MAXPODS    REPLICAS   AGE
hpa-demo-deployment   Deployment/hpa-demo-deployment   56%/50%    1         10         6          27m
```

#### Nach ca. 2 Minuten weiteren Minuten 
Der HPA hat das CPU Target wieder erreicht und somit seine Aufgabe erfüllt

```bash 
$ kubectl -n default get pods && kubectl -n default get horizontalpodautoscaler
NAME                                  READY   STATUS   RESTARTS   AGE
hpa-demo-deployment-85bff8877-5b49k   1/1     Running  0          5m48s
hpa-demo-deployment-85bff8877-98qdq   1/1     Running  0          6m18s
hpa-demo-deployment-85bff8877-9z9p9   1/1     Running  0          8m12s
hpa-demo-deployment-85bff8877-gmvr2   1/1     Running  0          3m17s
hpa-demo-deployment-85bff8877-sfdjh   1/1     Running. 0          6m18s
hpa-demo-deployment-85bff8877-vtn48   1/1     Running  0          6m18s
load-simulation                       1/1     Running  0          7m44s

NAME                  REFERENCE                        TARGETS   MINPODS   MAXPODS    REPLICAS   AGE
hpa-demo-deployment   Deployment/hpa-demo-deployment   47%/50%   1         10         6          30m
```

## 4) Entfernen von Last

Durch den folgenden Befehl wird der Load-Simulator entfernt:

```bash
$ kubectl delete pod load-simulation
pod "load-simulation" deleted
```

{{< alert title="Wichtig!">}}
WICHTIG! Die Ramp-Down-Time liegt bei standardmäßig 5 Minuten. Das heißt, dass ein Absenken der Last erst nach 5 Minuten (+- 15 Sekunden) vom HPA mit einem absenken der Replikate quittiert wird.
{{< /alert >}}

Im Anschluss werden die Replikate entfernt, da keine Last mehr vorhanden ist.

## 5) Cleanup
Im folgenden werden der Service, das Deployment und der HPA entfernt.

```bash
$ kubectl delete -f hpa.yaml
horizontalpodautoscaler.autoscaling "hpa-demo-deployment" deleted
 
$ kubectl delete -f deployment.yaml
deployment.apps "hpa-demo-deployment" deleted
 
$ kubectl delete -f service.yaml
service "hpa-demo-deployment" deleted
```