---
title: "Vertical Pod Autoscaler (VPA)"
linkTitle: "Vertical Pod Autoscaler (VPA)"
weight: 20
date: 2023-02-21
---

Der Vertical Pod Autoscaler (VPA) ist eine Erweiterung der Kubernetes API, welche eine vertikale Skalierung für Kubernetes-Controller wie Deployments bzw. deren Pods bietet. Das Funktionsprinzip ist etwas aufwändiger als beim HPA. Indem er die Ressourcenanforderungsparameter (resource requests) der Pods, auf Grundlage der Analyse, der von den Arbeitslasten gesammelten Metriken optimiert. "requests" sind die deklarative Angabe der minimal erforderlichen Ressourcen für einen (oder mehrere) Container, aus denen ein Pod besteht; je höher der Wert, desto mehr Zugriff hat der geplante Pod auf CPU oder RAM. Wenn festgestellt wird, dass ein Workload mehr Ressourcen verbraucht als in seiner Spezifikation festgelegt, berechnet der VPA einen neuen, Satz angemessener Werte, im Bereich seiner vorgegebenen Limits.

Der VPA ermöglicht zwei verschiedene Arten von Ressourcen, die für jeden Container eines Pods angegeben werden können: Requests + Limits.


{{< alert title="Limits / Requests" >}}
***Was ist ein Request?***
Requests definieren das Minimum an Ressourcen, die Container benötigen. Eine Anwendung kann beispielsweise mehr als 256 MB Arbeitsspeicher benötigen, aber Kubernetes garantiert dem Container ein Minimum von 256 MB, wenn seine Anforderung 256 MB RAM beträgt.

***Was ist ein Limit?***
Ein Limit definiert die maximale Menge an Ressourcen, die ein bestimmter Container zugewiesen bekommt. Die Anwendung benötigt vielleicht mindestens 256 MB Arbeitsspeicher, dennoch möchte man unter hoher Last sicherstellen, dass sie nicht mehr als 512 MB RAM verbraucht.
{{< /alert >}}

### Wie funktioniert der VPA ?

Der VPA fragt in regelmäßigen Abständen bei der Kubernetes-API die Ressourcennutzung eines Pods ab und passt dann die Anzahl der Replikate nach Bedarf an, um ein Zielniveau für die Ressourcennutzung zu erreichen.

Im Detail funktioniert dies wie folgt:

1. Der VPA Recommender liest die VPA-Konfiguration und die Metriken zur Ressourcenauslastung vom Metrik-Server.
2. VPA Recommender liefert Empfehlungen für Pod-Ressourcen an den VPA.
3. VPA Updater nimmt die Empfehlung für die Pod-Ressourcen entgegen.
4. VPA Updater leitet die Beendigung des Pods ein.
5. Das Deployment stellt fest, dass der Pod beendet wurde, und erstellt den Pod neu, damit es seinem State entspricht.
6. Wenn sich der Pod im Wiederherstellungsprozess befindet, erhält der VPA Admission Controller die Pod-Ressourcenempfehlung. Da Kubernetes keine dynamische Änderung der Ressourcengrenzen eines laufenden Pods unterstützt, kann VPA bestehende Pods nicht mit neuen Grenzen aktualisieren. Es beendet Pods, die veraltete Grenzwerte verwenden. Wenn der Controller des Pods die Ersetzung vom Kubernetes-API-Dienst anfordert, injiziert der VPA Admission Controller die aktualisierten Ressourcenanforderungen und Grenzwerte in die Spezifikation des neuen Pods.
7. Schließlich überschreibt der VPA Admission Controller die Empfehlungen für den Pod.

{{< alert title="updateMode" >}}
Je nachdem, wie der VPA konfiguriert wird, beherrscht er folgende Modi:

- Die Empfehlungen direkt anwenden, indem die Pods aktualisiert/neu erstellt werden (updateMode = auto).
- Die empfohlenen Werte als Referenz speichern (updateMode = off).
- Die empfohlenen Werte nur auf neu erstellte Pods anwenden (updateMode = initial).
{{< /alert >}}

### Wie wird der VPA genutzt ?

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-deployment-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-deployment
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


### Limitierungen des VPA
Wie beim HPA ist der VPA zwar ein leistungsfähiges Tool, unterliegt aber ebenfalls einigen Einschränkungen und ist nicht für jeden Anwendungsfall ideal. Auch kann er nicht jedes Problem mit Cluster-Ressourcen lösen. Es gibt dabei einige Dinge zu beachten:

**Der VPA ....**

**.... kann**, wenn man die Grenzen für die Metriken der Pods nicht effizient festlegt, Pods häufig beenden oder Ressourcen verschwenden <br>
**.... kann** nicht zusammen mit dem VPA verwendet werden auf Basis der selben Metriken <br>
**.... kann** nicht skalieren, wenn die Gesamtkapazität des Clusters erschöpft ist, bis neue Nodes zum Cluster hinzugefügt werden <br>
**.... kann** mehr Ressourcen empfehlen, als im Cluster verfügbar sind, was dazu führt, dass der Pod (aufgrund unzureichender Ressourcen) keinem Knoten zugewiesen wird und daher nie läuft. Um diese Einschränkung zu umgehen, ist es eine gute Idee, den LimitRange auf die maximal verfügbaren Ressourcen zu setzen. Dadurch wird sichergestellt, dass die Pods nicht mehr Ressourcen anfordern, als der LimitRange definiert.

Testing
Es gibt mehrere Möglichkeiten, den Kubernetes Vertical Pod Autoscaler (VPA) zu testen.

Verwendung eines Load-Testing-Tools
Eine Möglichkeit, den VPA zu testen, besteht darin, ein Load-Testing-Tool wie Apache JMeter oder Gatling zu verwenden, um Last auf einer Anwendung zu erzeugen. Man kann dann beobachten, wie der VPA reagiert, indem die Anzahl der Replikate basierend auf dem Ressourcenverbrauch der Pods erhöht oder verringert wird.

Verwenden des Kubernetes-Befehls "kubectl"
Man kann "kubectl" verwenden, um die Anzahl der Replikate der Pods manuell zu erhöhen oder zu verringern und beobachten, wie der VPA darauf reagiert. Als genaueres Beispiel dient hier "kubectl scale", um die Anzahl der Replikate eines Deployments oder RCs zu erhöhen.



Gleichzeitige Verwendung von VPA und HPA
Der HPA und VPA können miteinander in Konflikt geraten - wenn man beispielsweise bei beiden den RAM als Basismetrik der Skalierung verwendet. Dadurch kann es passieren, dass beide versuchen, Workloads gleichzeitig vertikal und horizontal zu skalieren, was unvorhersehbare Folgen haben kann. Um solche Konflikte zu vermeiden, ist es Best Practice, dass HPA und VPA auf unterschiedliche Metriken Metriken achten.


In der Regel wird der VPA so eingestellt, dass die Skalierung auf der Basis von CPU oder RAM erfolgt, und benutzerdefinierte Metriken für HPA verwendet werden.



Überwachung
Sobald der VPA eingerichtet ist, kann dieser mittels der Kubernetes-API oder mit einem Monitoringtool wie Prometheus, Grafana oder Kubernetes Dashboard überwacht werden.

Links und Quellen
https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html