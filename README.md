> Cloud Native Islamabad meetup (27.01.2022)


## Introduction

In this blog entry, I will show you how to create build a cloud native Kubernetes cluster vending machine with vcluster.

## Prerequisites

- `kubectl` and `helm` cli must be installed.


## Deploy The Rest Of The Application, Using GitOps

Now you can call:


this will deploy the missing applications, via GitOps through ArgoCD. We define the ArgoCD application like this:

````yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: applications
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "99"
spec:
  destination:
    name: in-cluster
    namespace: argocd
  project: default
  source:
    path: applications
    repoURL: https://github.com//vcluster-webinar.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
````

Pointing to our `kustomization.yaml` file in the `applications` directory.

There we composite the applications, we want to deploy. You could have a separate GItOps repository for this too.

We install following applications:

- argocd
- cert-manager
- external-dns
- ingress-nginx
- kube_prometheus_stack
- vcluster

> Note: To use Ingress for the `vcluster`, we need to enable passthrough-mode in the `ingress-nginx` Helm chart.

````yaml
...
helm:
  values: |-
    ...
    controller:
      extraArgs:
        enable-ssl-passthrough: ""
      ...
chart: ingress-nginx
...
````

We follow here the `App of Apps` approach. You can find more about this interesting pattern here -> https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643747987777/Lkw3CtCu5j.png)

### ArgoCD waves

It's important here to mention, that we are using the `argocd.argoproj.io/sync-wave` annotation to define the different waves.

This is important, because we want to deploy the applications in the right order. This important, if we have some dependencies between the applications.
For example Prometheus with the ServiceMonitor.

To define the waves, we use the `argocd.argoproj.io/sync-wave` annotation and chose the wave number. The wave number is the order in which the applications will be deployed.

```yaml
metadata:
  name: kube-prometheus-stack
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

## vcluster

Virtual clusters are fully working Kubernetes clusters that run on top of other Kubernetes clusters. Compared to fully separate "real" clusters, virtual clusters do not have their own node pools. Instead, they are scheduling workloads inside the underlying cluster while having their own separate control plane.

If you get Inception vibes, welcome to the party!
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643749164726/GUnnylnMH.png)

If you want to know more, I recomend to watch this latest session of `Containers from the Couch` with Justin, Rich and Lukas

%[https://www.youtube.com/watch?v=a8fIyUd9438]

I created a folder called `vcluster`. And here we are going to use the `App of apps` approach again. You probably spotted the
`vcluster` application in the `applications` folder. Here we are pointing to the `kustomization.yaml` file, this time in the `vcluster` folder.

````yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vcluster
  annotations:
    argocd.argoproj.io/sync-wave: "99"
spec:
  destination:
    name: in-cluster
    namespace: argocd
  project: default
  source:
    path: vcluster
    repoURL: https://github.com/dirien/vcluster-webinar.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
````

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643748035435/YmUHobM2s.png)

Inside the `vcluster` folder we define different ArgoCD applications. Depending on the type of `vcluster` we want to deploy.
We use `kusomization.yaml` again to glue the ArgoCD applications together. The structure of the folders is completely up to you.


Currently `vcluster` needs, when installed via `helm` the CIDR range provided by the `serviceCIDR` flag.

To get the CIDR range, we use following target in our `taskfile`:

```bash
kubectl create service clusterip test --clusterip 1.1.1.1 --tcp=80:80

error: failed to create ClusterIP service: Service "test" is invalid: spec.clusterIPs: Invalid value: []string{"1.1.1.1"}: 
failed to allocate IP 1.1.1.1: the provided IP (1.1.1.1) is not in the valid range. The range of valid IPs is 10.32.0.0/12
```

The valid IP range will be displayed in the `taskfile` output. Here it is `10.32.0.0/12`

Here an example of a `vcluster` application, using `k0s`:

````yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: team-4
  labels:
    team: team-4
    department: development
    project: backend
    distro: k3s
    version: v1.21.4-k3s1
  annotations:
    argocd.argoproj.io/sync-wave: "99"
spec:
  destination:
    name: in-cluster
    namespace: team-4
  project: default
  source:
    repoURL: 'https://charts.loft.sh'
    targetRevision: 0.6.0-alpha.7
    helm:
      values: |-
        rbac:
          clusterRole:
            create: true
          role:
            create: true
            extended: true
        syncer:
          extraArgs:
            - --tls-san=team-4.ediri.cloud
            - --out-kube-config-server=https://team-4.ediri.cloud
        serviceCIDR: 10.32.0.0/12
        ingress:
          enabled: true
          host: team-4.ediri.cloud
          annotations:
            external-dns.alpha.kubernetes.io/hostname: team-4.ediri.cloud
            external-dns.alpha.kubernetes.io/ttl: "60"
    chart: vcluster
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
````

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643748121500/XtoGzSoAv.png)

As we deployed an ingress controller and external DNS, we can use this to access the `vcluster`.

This is absolut brilliant, as we can now order via git pull requests new cluster.

### Accessing the vcluster

I am going to use the `vcluster` cli here. But you could also get the `kubeconfig` from your Ops Team

```bash
vcluster connect team-4 -n team-4 --server=https://team-4.ediri.cloud
[done] √ Virtual cluster kube config written to: ./kubeconfig.yaml. You can access the cluster via `kubectl --kubeconfig ./kubeconfig.yaml get namespaces`
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643749033377/8WqkxazQX.png)

Now we can access the cluster as usual, with the `kubectl` command.

```bash
kubectl --kubeconfig ./kubeconfig.yaml get namespaces
NAME              STATUS   AGE
default           Active   5d1h
kube-system       Active   5d1h
kube-public       Active   5d1h
kube-node-lease   Active   5d1h
```

Or schedule our workload:

```bash
kubectl run nginx --image=nginx
pod/nginx created

kubectl port-forward pod/nginx 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080


curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Monitoring

With our monitoring stack, we can monitor our `vcluster`, very comfortably. Just head over to the Grafana and browse the dashboard you need.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643748267309/mCJEsGKTT.png)

## Some links

- https://github.com/dirien/vcluster-webinar
- https://www.vcluster.com/
- https://taskfile.dev/
- https://argoproj.github.io/argo-cd/
- https://www.scaleway.com/

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643749091015/wzybDIS81.png)
