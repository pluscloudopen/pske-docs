---
title: "PSKE - Terraform Cluster Provisionierung"
linkTitle: "Terraform Cluster Provisionierung"
weight: 10
date: 2023-10-20
---

# Kubectl Provider

Der Terraform Kubectl Provider ermöglich die Provsionierung von Kubernetes Clustern innerhalb der PSKE (Gardener) mittels, wie bereits erwähnt, Terraform.

## Notwendige Komponenten und Zugänge

Folgendes ist erforderlich:

- Zugangstoken zum PSKE (Gardener) Dashboard
- lokal installierten Terraform Client (https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
- der kubectl Terraform Provider, welcher aber durch Terraform später selbstständig installiert wird (https://github.com/gavinbunney/terraform-provider-kubectl)

## Vorbereitung Terraform und Einrichtung des Providers

Als erstes erstellt man sich ein leeres Arbeitsverzeichnis für den Test oder man cloned dieses Repository und wechselt in das Verzeichnis.

Je nachdem ob man ein leeres Verzeichnis erstellt hat oder das Repo verwendet, legt man sich eine "versions.tf" an, welche folgendes beinhaltet:

```terraform
terraform {
  required_version = ">= 0.13"

  required_providers {
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = ">= 1.7.0"
    }
  }
}
```

Hier wird definiert, welche(r) Provider in welcher Version benötigt wird und seitens dem Terraform Tool automatisch bei dem folgenden "init" bezogen werden soll.

Grundsätzlich ist der name "versions.tf" vollkommen gleich, man kan die Datei nennen wie man will, nur die Endung .tf ist wichtig, da Terraform bei der Ausührung alle .tf files in dem aktuellen Verzeichnis such und ausführt. Es macht aber Sinn, es nach einem entsprechenden Schema zu benennen, damit man den Überblick behält.

## Anlage der PSKE Gardener Konfiguration

Im nächsten Schritt legen wir eine "gardener.tf" an (wie erwähnt, der Name ist ansich egal, solange die Datei eine .tf Endung hat), in der wir zunächst folgenden Block defineren:

```terraform
provider "kubectl" {
  config_path    = "PFAD/KUBECONFIGDATEINAME"
}
```

Hier wird definiert, das wir den "kubectl" Terraform Provider nutzen wollen, welchen wir zuvor in der versions.tf angegeben haben. Des weiteren benötigt er die Angabe des Pfades zu der kubeconfig für den PSKE/Gardener Service Accounts, welcher die Berechtigung hat, Cluster anzulegen und zu löschen.

Hierzu loggt man sich auf das entsprechende PSKE Dashboard ein, in diesem konkreten Fall https://dashboard.prod.gardener.get-cloud.io.

Dort klickt man links auf Members und sucht dann den Service-Account des Projects raus, in  meinem konkreten Fall wäre das MA-24, welcher auch als Rolle "Service Account Manager" hat, und lädt sich von diesem die kubeconfig-Datei runter.

Diese Datei muss dann entsprechen in der zuvor angelegten versions.tf referenziert werden. Bei diesem simplen Beispiel wäre das bei mir folgendermaßen:

```terraform
provider "kubectl" {
  config_path    = "~/Downloads/kubeconfig-2.yaml"
}
```

Somit ist schonmal sichergestellt, das der kubectl-Provider die notwendige Konfiguration hat, um sich mit der PSKE (Gardener) verbinden und entsprechende Aktionen ausführen zu können.

## Definition des Kubernetes Clusters

Nachdem wir Terraform eingerichtet haben und den kubectl Provider soweit konfiguriert haben, das er mit der PSKE kommunizieren kann, kommen wir zu dem eigentlichen Punkt, die Provisionierung eines Kubernetes Clusters.

Hierzu legen wir z.B. eine cluster.tf an, welche grundsätzlich folgenden Inhalt haben muss:

```terraform
resource "kubectl_manifest" "NAMEDERRESOURCE" {
    yaml_body = <<YAML
<HIER KOMMT DIE EIGENTLICHE CONFIG REIN>
YAML
}
```

NAMEDERRESOURCE ist nur für Terraform relevant und spiegelt nicht den Namen des eigentlichen Clusters wieder, aber offen gestanden macht es Sinn die entsprechen zu benennen, damit man diese bei einer wachensenden Konfiguration später wieder zuordenen kann.

Hiermit übergibt man der Gardener API die gewünschte Konfiguration im YAML-Format, welche man selbstverständlich komplett per Hand schreiben, man aber auch von dem Dashboard für sich erledigen lassen kann.

Hierzu spielt man quasi die Anlage eines Kubernetes Clusters mittels des entsprechenden PSKE Dashboards durch, konfiguriert sich den Cluster so wie man ihn benötigt mit allen Einstellungen die man benötigt, die es Anzahl und Größe der Worker-Nodes, Maintenance&Hibernation Schedules usw usw, aber man führt am Ende die Erstellung nicht aus, sonder klickt oben in der Leiste neben "Overview" auf "YAML". Dort erhält man dann die komplette Defintion der gewünschten Konfiguration im YAML-Format.

Diese Konfiguration kopiert man sich nun und fügt diese in der cluster.tf anstelle des <HIER KOMMT DIE EIGENTLICHE CONFIG REIN> ein, also zwischen die beiden Zeilen mit YAML.

In meinem Beispiel würde dies wie folgt aussehen, wie man aber an der Metadata sieht, muss dies entsprechend für die Umgebung wo es laufen soll angepasst werden. Daher nehmt lieber euren eigenen Extrakt.

```terraform
resource "kubectl_manifest" "tf_test_shoot" {
    yaml_body = <<YAML
kind: Shoot
apiVersion: core.gardener.cloud/v1beta1
metadata:
  namespace: garden-ma-24
  name: terraform-test
spec:
  provider:
    type: openstack
    infrastructureConfig:
      apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
      kind: InfrastructureConfig
      networks:
        workers: 10.250.0.0/16
      floatingPoolName: ext01
    controlPlaneConfig:
      apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
      kind: ControlPlaneConfig
      loadBalancerProvider: amphora
    workers:
      - name: k8sworker
        minimum: 1
        maximum: 2
        maxSurge: 1
        machine:
          type: SCS-2V:4:100
          image:
            name: ubuntu
            version: 22.4.2
          architecture: amd64
        zones:
          - nova
        cri:
          name: containerd
        volume:
          size: 50Gi
  networking:
    nodes: 10.250.0.0/16
    type: cilium
  cloudProfileName: pluscloudopen-hire
  secretBindingName: my-openstack-secret
  region: RegionOne
  purpose: evaluation
  kubernetes:
    version: 1.27.5
    enableStaticTokenKubeconfig: false
  addons:
    kubernetesDashboard:
      enabled: false
    nginxIngress:
      enabled: false
  maintenance:
    timeWindow:
      begin: 010000+0200
      end: 020000+0200
    autoUpdate:
      kubernetesVersion: true
      machineImageVersion: true
  hibernation:
    schedules:
      - start: 00 17 * * 1,2,3,4,5
        location: Europe/Berlin
  controlPlane:
    highAvailability:
      failureTolerance:
        type: node
YAML
}
```

Hinweis: wenn man den Cluster nur mittels Terraform anlegen, aber nicht wieder löschen möchte, dann muss man hier nichts weiter an der Definition anpassen. Soll aber Terraform später auch wieder in der Lage sein, den angelegten Cluster mittels eines "terraform destroy" auch wieder dem Erdboden gleich zu machen, dann benötigt man hier noch folgende Anpassung:

```terraform
metadata:
  namespace: garden-ma-24
  name: terraform-test
  annotations:
    confirmation.gardener.cloud/deletion: "true"
```

Also der obige metadata-Block muss um folgendes ergänzt werden:

```terraform
  annotations:
    confirmation.gardener.cloud/deletion: "true"
```

Wichtig, auf die Einrückung achten, es ist YAML!!!

## Erstellung des Kubernetes Clusters

Wenn man alles oben entsprechend gemacht hat, kommen wir nun endlich zur eigentlich Erstellung des Cluster.

Wir wechseln mit unserer Shell in das Arbeitsverzeichnis, wo wir die versions.tf, cluster.tf und gardener.tf angelegt haben und initialisieren erstmal das Terraform und lassen es den kubectl Provider installieren:

```bash
terraform init
```

Die Ausgabe sollte hier folgendem ähneln:

```bash
Initializing the backend...

Initializing provider plugins...
- Finding gavinbunney/kubectl versions matching ">= 1.7.0"...
- Installing gavinbunney/kubectl v1.14.0...
- Installed gavinbunney/kubectl v1.14.0 (self-signed, key ID AD64217B5ADD572F)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/cli/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Als nächstes macht man ein "terraform plan", was sich die zuvor erstellten .tf Konfigurationen anschaut und überprüft, ob das soweit Hand und Fuss hat, ob er alles kennt und ob man keine Syntax-Fehler gemacht hat

```bash
terraform plan
```

In meinem Fall würde das wie folgt aussehen:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # kubectl_manifest.tf_test_shoot will be created
  + resource "kubectl_manifest" "tf_test_shoot" {
      + api_version             = "core.gardener.cloud/v1beta1"
      + apply_only              = false
      + force_conflicts         = false
      + force_new               = false
      + id                      = (known after apply)
      + kind                    = "Shoot"
      + live_manifest_incluster = (sensitive value)
      + live_uid                = (known after apply)
      + name                    = "terraform-test"
      + namespace               = "garden-ma-24"
      + server_side_apply       = false
      + uid                     = (known after apply)
      + validate_schema         = true
      + wait_for_rollout        = true
      + yaml_body               = (sensitive value)
      + yaml_body_parsed        = <<-EOT
            apiVersion: core.gardener.cloud/v1beta1
            kind: Shoot
            metadata:
              annotations:
                confirmation.gardener.cloud/deletion: "true"
              name: terraform-test
              namespace: garden-ma-24
            spec:
              addons:
                kubernetesDashboard:
                  enabled: false
                nginxIngress:
                  enabled: false
              cloudProfileName: pluscloudopen-hire
              controlPlane:
                highAvailability:
                  failureTolerance:
                    type: node
              hibernation:
                schedules:
                - location: Europe/Berlin
                  start: 00 17 * * 1,2,3,4,5
              kubernetes:
                enableStaticTokenKubeconfig: false
                version: 1.27.5
              maintenance:
                autoUpdate:
                  kubernetesVersion: true
                  machineImageVersion: true
                timeWindow:
                  begin: 010000+0200
                  end: 020000+0200
              networking:
                nodes: 10.250.0.0/16
                type: cilium
              provider:
                controlPlaneConfig:
                  apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
                  kind: ControlPlaneConfig
                  loadBalancerProvider: amphora
                infrastructureConfig:
                  apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
                  floatingPoolName: ext01
                  kind: InfrastructureConfig
                  networks:
                    workers: 10.250.0.0/16
                type: openstack
                workers:
                - cri:
                    name: containerd
                  machine:
                    architecture: amd64
                    image:
                      name: ubuntu
                      version: 22.4.2
                    type: SCS-2V:4:100
                  maxSurge: 1
                  maximum: 2
                  minimum: 1
                  name: k8sworker
                  volume:
                    size: 50Gi
                  zones:
                  - nova
              purpose: evaluation
              region: RegionOne
              secretBindingName: my-openstack-secret
        EOT
      + yaml_incluster          = (sensitive value)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
```

Falls ihm soweit alles zusagt, dann sollte es besagen, das er hier eine Ressource (den Kubernetes Cluster) anlegen möchte mit den zuvor definierten Werten.

Stimmt man dem zu, dann kann man dies mittels "terraform apply" ausführen lassen. Er zeigt einem dann nochmal alles an, was er machen würde und benötigt die Angabe eines "yes", damit er es dann auch wirklich ausführt. Diese Abfrage kann man umgehen, indem man ein "-auto-approve" an das Kommando anhängt, aber man sollte hier wirklich, wirklich sicher sein, das man dies auch wirklich so möchte.

```bash
terraform apply
```

Was in meinem Fall dann wiederum so aussehen würde:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # kubectl_manifest.tf_test_shoot will be created
  + resource "kubectl_manifest" "tf_test_shoot" {
      + api_version             = "core.gardener.cloud/v1beta1"
      + apply_only              = false
      + force_conflicts         = false
      + force_new               = false
      + id                      = (known after apply)
      + kind                    = "Shoot"
      + live_manifest_incluster = (sensitive value)
      + live_uid                = (known after apply)
      + name                    = "terraform-test"
      + namespace               = "garden-ma-24"
      + server_side_apply       = false
      + uid                     = (known after apply)
      + validate_schema         = true
      + wait_for_rollout        = true
      + yaml_body               = (sensitive value)
      + yaml_body_parsed        = <<-EOT
            apiVersion: core.gardener.cloud/v1beta1
            kind: Shoot
            metadata:
              annotations:
                confirmation.gardener.cloud/deletion: "true"
              name: terraform-test
              namespace: garden-ma-24
            spec:
              addons:
                kubernetesDashboard:
                  enabled: false
                nginxIngress:
                  enabled: false
              cloudProfileName: pluscloudopen-hire
              controlPlane:
                highAvailability:
                  failureTolerance:
                    type: node
              hibernation:
                schedules:
                - location: Europe/Berlin
                  start: 00 17 * * 1,2,3,4,5
              kubernetes:
                enableStaticTokenKubeconfig: false
                version: 1.27.5
              maintenance:
                autoUpdate:
                  kubernetesVersion: true
                  machineImageVersion: true
                timeWindow:
                  begin: 010000+0200
                  end: 020000+0200
              networking:
                nodes: 10.250.0.0/16
                type: cilium
              provider:
                controlPlaneConfig:
                  apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
                  kind: ControlPlaneConfig
                  loadBalancerProvider: amphora
                infrastructureConfig:
                  apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
                  floatingPoolName: ext01
                  kind: InfrastructureConfig
                  networks:
                    workers: 10.250.0.0/16
                type: openstack
                workers:
                - cri:
                    name: containerd
                  machine:
                    architecture: amd64
                    image:
                      name: ubuntu
                      version: 22.4.2
                    type: SCS-2V:4:100
                  maxSurge: 1
                  maximum: 2
                  minimum: 1
                  name: k8sworker
                  volume:
                    size: 50Gi
                  zones:
                  - nova
              purpose: evaluation
              region: RegionOne
              secretBindingName: my-openstack-secret
        EOT
      + yaml_incluster          = (sensitive value)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

kubectl_manifest.tf_test_shoot: Creating...
kubectl_manifest.tf_test_shoot: Creation complete after 2s [id=/apis/core.gardener.cloud/v1beta1/namespaces/garden-ma-24/shoots/terraform-test]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Wenn er hier keine Fehler anzeigen sollte, dann hat er die Erstellung des Clusters gestartet. Man kann nun in das PSKE Dashboard gehen und dem Status folgen, bis die Erstellung erfolgt ist.

## Änderung des Kubernetes Clusters

Möchte man Änderungen an dem Kubernetes Cluster vornehmen, bedarf es hier der Anpassung der von uns zuvor vorgenommenen Definition in der "cluster.tf".

Sagen wir mal, das wir z.B. die Anzahl der Worker-Nodes anpassen wollen, also z.B. von

```terrform
    workers:
      - name: k8sworker
        minimum: 2
        maximum: 3
        maxSurge: 1
```

auf

```terrform
    workers:
      - name: k8sworker
        minimum: 2
        maximum: 3
        maxSurge: 1
```

Nachdem man die obige Änderung gemacht hat, kann man diese wieder mittels "terraform plan" schauen, ob es von der Syntax her korrekt ist und mit einem "terraform apply" kann man die oben definierte Änderung durchführen lassen.

Wichtig hier ist zu bedenken, das nur wenn die Syntax korrekt ist, heisst es nicht, das es nicht bei einem "apply" fehlschlagen kann. Das kann multiple Gründe haben, bespielsweise:

1. Die gewünschten Ressourcen übersteigen das, was im Cluster verfügbar ist. Also das mehr CPU, Ram usw. anfordert wird. Der Plan wird hier zwar ausgeführt, Gardener wird aber während der Ausführung irgendwann einen Fehler produzieren, das die Ressourcen erschöpft sind.

2. Man definiert etwas, das es nicht gibt wie z.B. unter machine / type, wo wir in unserem Beispiel den folgenden Typ "SCS-2V:4:100" ausgewählt haben. Wenn man hier nun einen Maschinentyp angibt, der in dieser Form nicht bereitgestellt wird, z.B. "SCS-1V:2:100", also nur VCPU-Kern und 2GB Ram, dann wird es zwar bei "plan" als alles in Ordnung angezeigt werden, aber bei einem "apply" wird ein Fehler ausgegeben:

```bash
.workers[0].machine.type: Unsupported value: "SCS-1V:2:100": supported values: "SCS-16V:32:100", "SCS-16V:64:100", "SCS-2V:16:50", "SCS-2V:4:100", "SCS-2V:8:100", "SCS-4V:16:100", "SCS-4V:32:100", "SCS-4V:8:100", "SCS-8V:16:100", "SCS-8V:32:100", "SCS-8V:8:100"]
```

Es gibt noch weitaus mehr Gründe, warum ein Plan nicht ausgeführt werden kann, was aber in der Regel über eine relativ klare Fehlermeldung erklärt wird.

## Löschung des Kubernetes Clusters

Falls man den Cluster wieder löschen möchte via Terraform, musste man vor der Erstellung wie oben erwähnt bei der Konfiguration folgenden Block ergänzen:

```terraform
  annotations:
    confirmation.gardener.cloud/deletion: "true"
```

Hat man dies getan, ist man nun auch mit Terraform wieder in der Lage, den zuvor erstellten Cluster wieder mittels eines "terraform destroy" abzureissen. Auch hier Bedarf es wieder einer Bestätigung mittels der Eingabe eines "yes", was man natürlich auch wieder entsprechend mittels des Anhängen eines "-auto-approve" umgehen kann. Aber auch hier gilt, seit euch sicher das es das ist was ihr wollt und ihr es auf die richtige Umgebung anwendet.

```bash
terraform destroy
```

Das sieht in meinem Fall dann wieder so aus:

```bash
kubectl_manifest.tf_test_shoot: Refreshing state... [id=/apis/core.gardener.cloud/v1beta1/namespaces/garden-ma-24/shoots/terraform-test]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # kubectl_manifest.tf_test_shoot will be destroyed
  - resource "kubectl_manifest" "tf_test_shoot" {
      - api_version             = "core.gardener.cloud/v1beta1" -> null
      - apply_only              = false -> null
      - force_conflicts         = false -> null
      - force_new               = false -> null
      - id                      = "/apis/core.gardener.cloud/v1beta1/namespaces/garden-ma-24/shoots/terraform-test" -> null
      - kind                    = "Shoot" -> null
      - live_manifest_incluster = (sensitive value)
      - live_uid                = "42edffa5-dacd-4c23-a879-a8034b175f7c" -> null
      - name                    = "terraform-test" -> null
      - namespace               = "garden-ma-24" -> null
      - server_side_apply       = false -> null
      - uid                     = "42edffa5-dacd-4c23-a879-a8034b175f7c" -> null
      - validate_schema         = true -> null
      - wait_for_rollout        = true -> null
      - yaml_body               = (sensitive value)
      - yaml_body_parsed        = <<-EOT
            apiVersion: core.gardener.cloud/v1beta1
            kind: Shoot
            metadata:
              annotations:
                confirmation.gardener.cloud/deletion: "true"
              name: terraform-test
              namespace: garden-ma-24
            spec:
              addons:
                kubernetesDashboard:
                  enabled: false
                nginxIngress:
                  enabled: false
              cloudProfileName: pluscloudopen-hire
              controlPlane:
                highAvailability:
                  failureTolerance:
                    type: node
              hibernation:
                schedules:
                - location: Europe/Berlin
                  start: 00 17 * * 1,2,3,4,5
              kubernetes:
                enableStaticTokenKubeconfig: false
                version: 1.27.5
              maintenance:
                autoUpdate:
                  kubernetesVersion: true
                  machineImageVersion: true
                timeWindow:
                  begin: 010000+0200
                  end: 020000+0200
              networking:
                nodes: 10.250.0.0/16
                type: cilium
              provider:
                controlPlaneConfig:
                  apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
                  kind: ControlPlaneConfig
                  loadBalancerProvider: amphora
                infrastructureConfig:
                  apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
                  floatingPoolName: ext01
                  kind: InfrastructureConfig
                  networks:
                    workers: 10.250.0.0/16
                type: openstack
                workers:
                - cri:
                    name: containerd
                  machine:
                    architecture: amd64
                    image:
                      name: ubuntu
                      version: 22.4.2
                    type: SCS-2V:4:100
                  maxSurge: 1
                  maximum: 2
                  minimum: 1
                  name: k8sworker
                  volume:
                    size: 50Gi
                  zones:
                  - nova
              purpose: evaluation
              region: RegionOne
              secretBindingName: my-openstack-secret
        EOT -> null
      - yaml_incluster          = (sensitive value)
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

kubectl_manifest.tf_test_shoot: Destroying... [id=/apis/core.gardener.cloud/v1beta1/namespaces/garden-ma-24/shoots/terraform-test]
kubectl_manifest.tf_test_shoot: Destruction complete after 1s

Destroy complete! Resources: 1 destroyed.
```

Wenn er hier nun ebenfalls wieder keine Fehler ausgibt und am Ende sagt, das er 1 Ressource zerstört hat, dann kann man wieder auf das PSKE Dashboard zurück wechseln und der Löschung zuschauen.
