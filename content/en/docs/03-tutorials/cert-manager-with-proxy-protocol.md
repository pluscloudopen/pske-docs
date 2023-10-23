---
title: "Cert Manager With Proxy Protocol"
linkTitle: "Cert Manager With Proxy Protocol"
weight: 20
date: 2023-10-10
---

Based on the current interactions between cert manager and ingress-nginx

{{% alert title="Warning" color="warning" %}}
Ingress NGINX and NGINX Ingress are not the same deployments. Although both use NGINX as their respective reverse proxy one of them is made by Kubernetes and one is made by NGINX
{{% /alert %}}

## Clean install Ingress NGINX with helm
{{% alert title="Warning" color="warning" %}}
This part also assumes, that you are installing the NGINX into a cluster without an existing loadbalancer and that you are also familiar with the tutorial [Forward Source IP](/docs/03-tutorials/forward-source-ip/)
{{% /alert %}}

1. Add the helm repository for Ingress NGINX.

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

2. Setup its values file and make sure the following entries are not set. Please ensure that `use-proxy-protocol` and `use-forwarded-headers` are enabled as well.

- controller.service.loadBalancerIP  \
    &rarr; if it exists, keep this as an empty string.
- controller.service.annotations."loadbalancer.openstack.org/hostname" \
    &rarr; make sure this is commented out, if it exists 
- controller.service.annotations."loadbalancer.openstack.org/load-balancer-address"  \
    &rarr; make sure this is commented out, if it exists

```bash
helm show values ingress-nginx > values.yaml
```

3. Install Ingress NGINX

```bash
helm install --values values.yaml ingress-nginx ingress-nginx/ingress-nginx
```

After the installation, you will find the target values for `controller.service.loadBalancerIP`, `controller.service.annotations` and `controller.service.annotations` by locating the controller service, since we need its external ip and corresponding hostname. In this case we assume the external ip is `192.168.10.1` for demonstration purposes, despite the fact that its a private ip address.

```bash
kubectl get svc -ningress-nginx

NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP            PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.60.24.9       192.168.10.1           80:32080/TCP,443:32443/TCP   22m
ingress-nginx-controller-admission   ClusterIP      10.60.24.8       <none>                 443/TCP                      22m
```

4. Update your values-file, using the now received values:

```yaml
controller:
  service:
    loadBalancerIP: "10.60.24.9"
# [...]
    annotations:
      loadbalancer.openstack.org/hostname: "10.60.24.9.nip.io"
      loadbalancer.openstack.org/load-balancer-address: "10.60.24.9"
```

