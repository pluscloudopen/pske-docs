---
title: "Einrichtung eines Horizontal Pod Autoscalers (HPA)"
linkTitle: "Einrichtung eines Horizontal Pod Autoscalers (HPA)"
weight: 20
date: 2023-02-21
---

Im folgenden Beispiel wird eine Applikation deployed, welche aus einem freizugänglichen Image resultiert. Dieses Beispiel dient der Veranschaulichung, wie der VPA auf Zustandsveränderungen reagiert.

## 1) Provisionieren eines Clusters
Zuerst sollte ein Kubernetes Cluster provisioniert werden.

## 2) Installation des VPA
Anders als beim HPA muss der VPA nachinstalliert werden. Dies erfolgt über das Repository https://github.com/kubernetes/autoscaler.

### Clonen des Repositories

```bash
$ git clone https://github.com/kubernetes/autoscaler.git
```

### Wechsel in das richtige Directory

```bash
$ cd autoscaler/vertical-pod-autoscaler/
```

### Installation des VPA

```bash
$ ./hack/vpa-up.sh
```

### Überprüfen der Installation

``` bash
$ kubectl get pods -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
...
metrics-server-7b236j497-bnw9s              1/1     Running   0          67d
vpa-admission-controller-3ns8d8777d-pps3w   1/1     Running   0          12s
vpa-recommender-6fcsnm26j5-s7lw0            1/1     Running   0          23s
vpa-updater-7sm51h55c-a9smw                 1/1     Running   0          23s
...
```

## 3) Deployment der Applikation

### Ausrollen des Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: vpa-demo-deployment
spec:
 selector:
   matchLabels:
     run: vpa-demo-deployment
 replicas: 1
 template:
   metadata:
     labels:
       run: vpa-demo-deployment
   spec:
     containers:
     - name: vpa-demo-deployment
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
deployment.apps/vpa-demo-deployment created

$ kubectl -n default get pods
NAME                                  READY   STATUS    RESTARTS   AGE
vpa-demo-deployment-85bff8877-9z9p9   1/1     Running   0          3s

$ kubectl -n default get deployment
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
vpa-demo-deployment   1/1     1            1           6s
```

### Erstellen eines Services

```yaml
apiVersion: v1
kind: Service
metadata:
 name: vpa-demo-deployment
 labels:
   run: vpa-demo-deployment
spec:
 ports:
 - port: 80
 selector:
   run: vpa-demo-deployment
```

```bash
$ kubectl apply -f service.yaml
service/vpa-demo-deployment created
$ kubectl -n default get services # Kann auch svc abgekürzt werden
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
vpa-demo-deployment   ClusterIP   10.122.166.51   <none>        80/TCP    5s
kubernetes            ClusterIP   10.112.0.1      <none>        443/TCP   13d
```

### Konfigurieren des VPA

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-deployment-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: vpa-demo-deployment
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        controlledResources:
          - cpu
          - memory
        maxAllowed:
          cpu: 1
          memory: 500Mi
        minAllowed:
          cpu: 100m
          memory: 50Mi
  updatePolicy:
    updateMode: "Auto"
```
```bash
$ kubectl apply -f vpa.yaml
verticalpodautoscaler.autoscaling.k8s.io/my-deployment-vpa created
```

## 3) Testing

Status vor dem Testing

```bash
$ kubectl -n default get pods
NAME                                  READY   STATUS    RESTARTS   AGE
vpa-demo-deployment-85bff8877-9z9p9   1/1     Running   0          20m
```

```bash
$ kubectl -n default get verticalpodautoscaler # Kann auch hpa abgekürzt werden
NAME                  REFERENCE                        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
vpa-demo-deployment   Deployment/vpa-demo-deployment   0%/50%    1         10        1          20m
```

### Erzeugen von Last

Mit folgendem Befehl wird eine Last simuliert:

```bash
kubectl run -i --tty load-simulation --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://vpa-demo-deployment; done"
```

### Verhalten während Last

Das Verhalten während der Last ist individuell und abhängig von der Applikation.

## 4) Entfernen von Last

Durch den folgenden Befehl wird der Load-Simulator entfernt:

```bash
$ kubectl delete pod load-simulation
pod "load-simulation" deleted
```

{{< alert title="Wichtig!">}}
WICHTIG! Die Ramp-Down-Time liegt bei standardmäßig 5 Minuten. Das heißt, dass ein Absenken der Last erst nach 5 Minuten (+- 15 Sekunden) vom VPA mit einem absenken der Replikate quittiert wird.
{{< /alert >}}

Im Anschluss werden die Replikate entfernt, da keine Last mehr vorhanden ist.

## 5) Cleanup

Im folgenden werden der Service, das Deployment und der VPA entfernt.

```bash
$ kubectl delete -f vpa.yaml
horizontalpodautoscaler.autoscaling "vpa-demo-deployment" deleted
 
$ kubectl delete -f deployment.yaml
deployment.apps "vpa-demo-deployment" deleted
 
$ kubectl delete -f service.yaml
service "vpa-demo-deployment" deleted
```