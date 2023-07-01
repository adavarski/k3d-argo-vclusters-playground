Cloud Native (GitOps)


## Introduction

In this repo, we will show you how to create build a cloud native Kubernetes cluster vending machine with vcluster.

## Prerequisites

- `kubectl`, `helm`, `vcluster`, `k3d` ust be installed.

Install vlcuster cli:
```
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64" && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster
```

### Setup environment

```
cd scripts/ && ./init-k3d-demo-env.sh

Example Output:

$ ./init-k3d-demo-env.sh 
.....
ArcgoCD admin password:
V7JBqouePrCmtzwj
```

## Deploy The Application, Using GitOps

via GitOps through ArgoCD. We define the ArgoCD application like this:

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
    repoURL: https://github.com/adavarski/k3d-argo-vclusters-playground.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
````

Pointing to our `kustomization.yaml` file in the `applications` directory.

There we composite the applications, we want to deploy. You could have a separate GItOps repository for this too.

We install following applications:

- vcluster (only this application is needed for this playground)

Note: for production we can add additiona applications like:

- argocd
- cert-manager
- external-dns
- ingress-nginx
- kube_prometheus_stack

 Note: To use Ingress for the `vcluster`, we need to enable passthrough-mode in the `ingress-nginx` Helm chart.

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

### ArgoCD waves (for Production)

It's important here to mention, that we are using the `argocd.argoproj.io/sync-wave` annotation to define the different waves.

This is important, because we want to deploy the applications in the right order. This important, if we have some dependencies between the applications.
For example Prometheus with the ServiceMonitor.

To define the waves, we use the `argocd.argoproj.io/sync-wave` annotation and chose the wave number. The wave number is the order in which the applications will be deployed.

Example: 

```yaml
metadata:
  name: kube-prometheus-stack
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

## vcluster

Virtual clusters are fully working Kubernetes clusters that run on top of other Kubernetes clusters. Compared to fully separate "real" clusters, virtual clusters do not have their own node pools. Instead, they are scheduling workloads inside the underlying cluster while having their own separate control plane.

If you want to know more, I recomend to watch this latest session of `Containers from the Couch` with Justin, Rich and Lukas

%[https://www.youtube.com/watch?v=a8fIyUd9438]

We created a folder called `vcluster`. And here we are going to use the `App of apps` approach again. You probably spotted the
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
    repoURL: https://github.com/adavarski/k3d-argo-vclusters-playground.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
````

Inside the `vcluster` folder we define different ArgoCD applications. Depending on the type of `vcluster` we want to deploy.
We use `kusomization.yaml` again to glue the ArgoCD applications together. The structure of the folders is completely up to you.

Currently `vcluster` needs, when installed via `helm` the CIDR range provided by the `serviceCIDR` flag.

To get the CIDR range, we use following target in our `taskfile`:

```bash
kubectl create service clusterip test --clusterip 1.1.1.1 --tcp=80:80

error: failed to create ClusterIP service: Service "test" is invalid: spec.clusterIPs: Invalid value: []string{"1.1.1.1"}: failed to allocate IP 1.1.1.1: the provided IP (1.1.1.1) is not in the valid range. The range of valid IPs is 10.43.0.0/16

```

The valid IP range will be displayed in the `taskfile` output. Here it is `10.43.0.0/16`

Here an example of a `vcluster` application, using `k0s`:

````yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: team-1
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
        serviceCIDR: 10.43.0.0/16
        ingress:
          enabled: true
    chart: vcluster
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
````

As we deployed an ingress controller, we can use this to access the `vcluster`.

This is absolut brilliant, as we can now order via git pull requests new cluster.

### Accessing the vcluster

I am going to use the `vcluster` cli here. But you could also get the `kubeconfig` from your Ops Team

```bash
$ vcluster connect team-1 -n team-1
done √ Switched active kube context to vcluster_team-1_team-1_k3d-argo-vcluster
warn   Since you are using port-forwarding to connect, you will need to leave this terminal open
- Use CTRL+C to return to your previous kube context
- Use `kubectl get namespaces` in another terminal to access the vcluster
Forwarding from 127.0.0.1:10317 -> 8443
Forwarding from [::1]:10317 -> 8443



Note: vcluster connect team-1 -n team-1 --server=https://team-1.example.com
[done] √ Virtual cluster kube config written to: ./kubeconfig.yaml. You can access the cluster via `kubectl --kubeconfig ./kubeconfig.yaml get namespaces`
```

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
kubectl run nginx --image=nginx -n team-1
kubectl expose pod nginx --port=80 -n team-1

kubectl apply -f ingresses/ingress-team-1-nginx.yaml -n team-1

curl http://team1-nginx.192.168.1.99.nip.io:8080
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

Check:
davar@devops:~/CROSSPLAIN/TRY-1/VCLUSTER/k3d-argo-vclusters-playground$ kubectl get svc -n team-1
NAME                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
team-1-headless                          ClusterIP   None            <none>        443/TCP                  16m
team-1                                   ClusterIP   10.43.156.154   <none>        443/TCP                  16m
kube-dns-x-kube-system-x-team-1          ClusterIP   10.43.200.71    <none>        53/UDP,53/TCP,9153/TCP   14m
team-1-node-k3d-argo-vcluster-server-0   ClusterIP   10.43.141.230   <none>        10250/TCP                14m
nginx                                    ClusterIP   10.43.240.253   <none>        80/TCP                   3m35s
davar@devops:~/CROSSPLAIN/TRY-1/VCLUSTER/k3d-argo-vclusters-playground$ kubectl get all -n team-1
NAME                                                  READY   STATUS    RESTARTS   AGE
pod/team-1-0                                          2/2     Running   0          16m
pod/coredns-6ff5b88b48-q8hqr-x-kube-system-x-team-1   1/1     Running   0          15m
pod/nginx                                             1/1     Running   0          4m23s

NAME                                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
service/team-1-headless                          ClusterIP   None            <none>        443/TCP                  16m
service/team-1                                   ClusterIP   10.43.156.154   <none>        443/TCP                  16m
service/kube-dns-x-kube-system-x-team-1          ClusterIP   10.43.200.71    <none>        53/UDP,53/TCP,9153/TCP   15m
service/team-1-node-k3d-argo-vcluster-server-0   ClusterIP   10.43.141.230   <none>        10250/TCP                15m
service/nginx                                    ClusterIP   10.43.240.253   <none>        80/TCP                   4m5s

NAME                      READY   AGE
statefulset.apps/team-1   1/1     16m
davar@devops:~/CROSSPLAIN/TRY-1/VCLUSTER/k3d-argo-vclusters-playground$ kubectl get ing -n team-1
NAME             CLASS    HOSTS                             ADDRESS      PORTS   AGE
team-1           <none>   vcluster.local                    172.29.0.2   80      16m
argocd-ingress   nginx    team1-nginx.192.168.1.99.nip.io   172.29.0.2   80      5m17s



```


Screenshots:
```
k3d-argo-vcluster-playground-apps-team-1.png
k3d-argo-vcluster-playground-apps-vcluster.png
k3d-argo-vcluster-playground-apps-applications.png
```


## Monitoring

With our monitoring stack, we can monitor our `vcluster`, very comfortably. Just head over to the Grafana and browse the dashboard you need.


## Some links

- https://github.com/dirien/vcluster-webinar
- https://www.vcluster.com/
- https://taskfile.dev/
- https://argoproj.github.io/argo-cd/
- https://www.scaleway.com/

