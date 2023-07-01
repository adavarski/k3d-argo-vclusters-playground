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
INFO[0000] Deleting cluster 'argo-vcluster'             
INFO[0011] Deleting cluster network 'k3d-argo-vcluster' 
INFO[0012] Deleting 1 attached volumes...               
INFO[0012] Removing cluster details from default kubeconfig... 
INFO[0012] Removing standalone kubeconfig file (if there is one)... 
INFO[0012] Successfully deleted cluster argo-vcluster!  
WARN[0000] No node filter specified                     
WARN[0000] No node filter specified                     
WARN[0000] No node filter specified                     
INFO[0000] portmapping '8080:80' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy] 
INFO[0000] Prep: Network                                
INFO[0000] Created network 'k3d-argo-vcluster'          
INFO[0000] Created image volume k3d-argo-vcluster-images 
INFO[0000] Starting new tools node...                   
INFO[0001] Creating node 'k3d-argo-vcluster-server-0'   
INFO[0005] Starting Node 'k3d-argo-vcluster-tools'      
INFO[0007] Creating LoadBalancer 'k3d-argo-vcluster-serverlb' 
INFO[0011] Using the k3d-tools node to gather environment information 
INFO[0015] HostIP: using network gateway 172.29.0.1 address 
INFO[0015] Starting cluster 'argo-vcluster'             
INFO[0015] Starting servers...                          
INFO[0015] Starting Node 'k3d-argo-vcluster-server-0'   
INFO[0023] All agents already running.                  
INFO[0023] Starting helpers...                          
INFO[0025] Starting Node 'k3d-argo-vcluster-serverlb'   
INFO[0034] Injecting records for hostAliases (incl. host.k3d.internal) and for 2 network members into CoreDNS configmap... 
INFO[0036] Cluster 'argo-vcluster' created successfully! 
INFO[0036] You can now use it like this:                
kubectl cluster-info
"ingress-nginx" already exists with the same configuration, skipping
NAME: ingress-nginx
LAST DEPLOYED: Sat Jul  1 18:05:09 2023
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace ingress-nginx get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
"argo" already exists with the same configuration, skipping
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "juicefs-csi-driver" chart repository
...Successfully got an update from the "kong" chart repository
...Successfully got an update from the "istio" chart repository
...Successfully got an update from the "argo" chart repository
...Successfully got an update from the "prometheus-community" chart repository
...Successfully got an update from the "crossplane-stable" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
namespace/argocd created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-applicationset-controller created
serviceaccount/argocd-notifications-controller created
serviceaccount/argocd-repo-server created
serviceaccount/argocd-server created
secret/argocd-notifications-secret created
secret/argocd-secret created
configmap/argocd-cm created
configmap/argocd-cmd-params-cm created
configmap/argocd-gpg-keys-cm created
configmap/argocd-notifications-cm created
configmap/argocd-rbac-cm created
configmap/argocd-ssh-known-hosts-cm created
configmap/argocd-tls-certs-cm created
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
clusterrole.rbac.authorization.k8s.io/argocd-application-controller created
clusterrole.rbac.authorization.k8s.io/argocd-repo-server created
clusterrole.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-repo-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-server created
role.rbac.authorization.k8s.io/argocd-application-controller created
role.rbac.authorization.k8s.io/argocd-applicationset-controller created
role.rbac.authorization.k8s.io/argocd-notifications-controller created
role.rbac.authorization.k8s.io/argocd-repo-server created
role.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
rolebinding.rbac.authorization.k8s.io/argocd-notifications-controller created
rolebinding.rbac.authorization.k8s.io/argocd-repo-server created
rolebinding.rbac.authorization.k8s.io/argocd-server created
service/argocd-application-controller-metrics created
service/argocd-applicationset-controller created
service/argocd-repo-server-metrics created
service/argocd-repo-server created
service/argocd-server-metrics created
service/argocd-server created
service/argocd-redis-metrics created
service/argocd-redis created
deployment.apps/argocd-applicationset-controller created
deployment.apps/argocd-notifications-controller created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
deployment.apps/argocd-redis created
statefulset.apps/argocd-application-controller created
resource mapping not found for name: "argocd-application-controller" namespace: "" from "STDIN": no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1"
ensure CRDs are installed first
resource mapping not found for name: "argocd-repo-server" namespace: "" from "STDIN": no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1"
ensure CRDs are installed first
resource mapping not found for name: "argocd-server" namespace: "" from "STDIN": no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1"
ensure CRDs are installed first
resource mapping not found for name: "argocd-redis" namespace: "" from "STDIN": no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1"
ensure CRDs are installed first
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io condition met
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io condition met
ingress.networking.k8s.io/argocd-ingress created
application.argoproj.io/applications created
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
vcluster connect team-1 -n team-1 --server=https://team-1.
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
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80

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

```

## Monitoring

With our monitoring stack, we can monitor our `vcluster`, very comfortably. Just head over to the Grafana and browse the dashboard you need.


## Some links

- https://github.com/dirien/vcluster-webinar
- https://www.vcluster.com/
- https://taskfile.dev/
- https://argoproj.github.io/argo-cd/
- https://www.scaleway.com/

