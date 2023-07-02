
## Introduction

Cloud Native vcluster + argo.

###  Prerequisites

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [helm](https://helm.sh/docs/intro/install/)
- [k3d](https://k3d.io/#installation)
- [vcluster](https://www.vcluster.com/docs/getting-started/setup)

```
### Install vlcuster cli (example):
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64" && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster
```

### Setup environment

```
cd scripts/ && ./init-k3d-demo-env.sh

Example Output:

$ ./init-k3d-demo-env.sh 
....
....
....
ArcgoCD admin password:
V7JBqouePrCmtzwj
```

### Deploy The Application, Using GitOps

GitOps through ArgoCD. We define the ArgoCD application like this:

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

### Production Notes:

For Production we can add additiona applications like:

- argocd
- cert-manager
- external-dns
- ingress-nginx
- kube_prometheus_stack

To use Ingress (Production) for the `vcluster`, we need to enable passthrough-mode in the `ingress-nginx` Helm chart.

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

Note: ArgoCD waves 

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

## VCluster

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

To get the CIDR range, we can execute:

```bash
$ kubectl create service clusterip test --clusterip 1.1.1.1 --tcp=80:80

error: failed to create ClusterIP service: Service "test" is invalid: spec.clusterIPs: Invalid value: []string{"1.1.1.1"}: failed to allocate IP 1.1.1.1: the provided IP (1.1.1.1) is not in the valid range. The range of valid IPs is 10.43.0.0/16

```

The valid IP range will be displayed in the command output. Here it is `10.43.0.0/16`

Here an example of a `vcluster` application, using `team-1`:

````yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: team-1
  labels:
    team: team-1
    department: development
    project: backend
    distro: k3s
    version: v1.21.4-k3s1
  annotations:
    argocd.argoproj.io/sync-wave: "99"
spec:
  destination:
    name: in-cluster
    namespace: team-1
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
          enabled: false
    chart: vcluster
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
````

As we deployed an ingress controller via init demo script, we can use this to access the `vcluster`.

We can now order via git pull requests new cluster.

### Accessing the vcluster

We are going to use the `vcluster` cli here. But you could also get the `kubeconfig` from your Ops Team

```bash

$ vcluster list

 NAME     NAMESPACE   STATUS    CONNECTED   CREATED                          AGE      CONTEXT            
 team-3   team-3      Running               2023-07-01 19:14:00 +0300 EEST   26m28s   k3d-argo-vcluster  
 team-2   team-2      Running               2023-07-01 19:17:01 +0300 EEST   23m27s   k3d-argo-vcluster  
 team-1   team-1      Running               2023-07-01 19:06:41 +0300 EEST   33m47s   k3d-argo-vcluster 

$ vcluster connect team-1 -n team-1
done √ Switched active kube context to vcluster_team-1_team-1_k3d-argo-vcluster
warn   Since you are using port-forwarding to connect, you will need to leave this terminal open
- Use CTRL+C to return to your previous kube context
- Use `kubectl get namespaces` in another terminal to access the vcluster
Forwarding from 127.0.0.1:10317 -> 8443
Forwarding from [::1]:10317 -> 8443


$ vcluster connect team-3 -n team-3 --server=https://....
done √ Switched active kube context to vcluster_team-3_team-3_k3d-argo-vcluster
- Use `vcluster disconnect` to return to your previous kube context
- Use `kubectl get namespaces` to access the vcluster

```

Now we can access the cluster as usual, with the `kubectl` command.

```bash
$ kubectl get ns
NAME              STATUS   AGE
kube-system       Active   53m
default           Active   53m
kube-public       Active   53m
kube-node-lease   Active   53m
ingress-nginx     Active   52m
argocd            Active   49m
team-1            Active   33m
team-2            Active   33m
team-3            Active   33m

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

$ kubectl get ing -n team-1
NAME             CLASS   HOSTS                             ADDRESS      PORTS   AGE
argocd-ingress   nginx   team1-nginx.192.168.1.99.nip.io   172.30.0.2   80      17m

$ kubectl get ing --all-namespaces
NAMESPACE   NAME             CLASS    HOSTS                             ADDRESS      PORTS   AGE
argocd      argocd-ingress   nginx    argocd.192.168.1.99.nip.io        172.30.0.2   80      61m
team-3      team-3           <none>   vcluster.local                    172.30.0.2   80      35m
team-1      argocd-ingress   nginx    team1-nginx.192.168.1.99.nip.io   172.30.0.2   80      29m



$ kubectl get all -n team-1
NAME                                                  READY   STATUS    RESTARTS   AGE
pod/coredns-6ff5b88b48-tzh59-x-kube-system-x-team-1   1/1     Running   0          33m
pod/team-1-0                                          2/2     Running   0          24m
pod/nginx                                             1/1     Running   0          17m

NAME                                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
service/team-1-headless                          ClusterIP   None            <none>        443/TCP                  35m
service/team-1                                   ClusterIP   10.43.27.9      <none>        443/TCP                  35m
service/kube-dns-x-kube-system-x-team-1          ClusterIP   10.43.125.200   <none>        53/UDP,53/TCP,9153/TCP   33m
service/team-1-node-k3d-argo-vcluster-server-0   ClusterIP   10.43.132.38    <none>        10250/TCP                33m
service/nginx                                    ClusterIP   10.43.65.15     <none>        80/TCP                   17m

NAME                      READY   AGE
statefulset.apps/team-1   1/1     35m

```


### Screenshots:

<img src="pictures/k3d-argo-vcluster-playground-apps-team-1.png?raw=true" width="900">

<img src="pictures/k3d-argo-vcluster-playground-apps-vcluster.png?raw=true" width="900">

<img src="pictures/k3d-argo-vcluster-playground-apps-applications.png?raw=true" width="900">


## Some links

- https://github.com/dirien/vcluster-webinar
- https://www.vcluster.com/
- https://argoproj.github.io/argo-cd/

