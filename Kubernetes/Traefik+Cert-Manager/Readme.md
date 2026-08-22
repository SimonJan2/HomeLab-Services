## Table Of Content
---
- [[#Getting Started|Getting Started]]
- [[#Resources|Resources]]
	- [[#Resources#Helm|Helm]]
- [[#Installing|Installing]]
- [[#Traefik|Traefik]]
	- [[#Traefik#middleware|middleware]]
	- [[#Traefik#dashboard|dashboard]]
- [[#Sample Workload|Sample Workload]]
- [[#cert-manager|cert-manager]]
	- [[#cert-manager#staging|staging]]
	- [[#cert-manager#production|production]]

## Getting Started
---
If you need to install a new kubernetes cluster you can use my [Ansible Playbook](https://technotim.live/posts/k3s-etcd-ansible/) to install one.

> ⚠️ **K3s ships with its own built-in Traefik.** This guide installs a second, fully custom Traefik (with HTTP/3, a pinned MetalLB IP, dashboard, etc.), which will conflict with the built-in one — both try to create a cluster-scoped `IngressClass` named `traefik`, and `helm install` will fail with an ownership error like:
> ```
> Error: INSTALLATION FAILED: unable to continue with install: IngressClass "traefik" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-namespace" must equal "traefik": current value is "kube-system"
> ```
> Ideally, disable it when you first provision the cluster by passing `--disable traefik --disable servicelb` to the K3s installer (the Ansible playbook above has an option for this). If K3s is already running and you skipped that, see [[#Disabling the built-in K3s Traefik|Disabling the built-in K3s Traefik]] below before continuing.

### Disabling the built-in K3s Traefik
---
If you already have a running K3s cluster with the built-in Traefik enabled, disable it on **every server/master node**, one at a time (don't restart more than one `k3s-server` at once — you need to keep etcd quorum).

On each master:
```shell
sudo cp /etc/rancher/k3s/config.yaml /etc/rancher/k3s/config.yaml.bak
sudo sed -i '/^disable:/a\  - traefik' /etc/rancher/k3s/config.yaml
cat /etc/rancher/k3s/config.yaml
```
Confirm `traefik` appears under `disable:`, then restart that node's K3s server and wait for all nodes to show `Ready` before moving to the next master:
```shell
sudo systemctl restart k3s-server
kubectl get nodes -w
```

Once all masters have been updated and restarted, K3s's deploy-controller automatically removes the built-in Traefik's `HelmChart` resources — along with its `Deployment`, `Service`, `IngressClass`, and even the `traefik.io` CRDs. Confirm everything is gone before installing your own:
```shell
helm list -A
kubectl get ingressclass
kubectl get crd | grep traefik.io
```
All three commands should return nothing related to Traefik. The `traefik/traefik` Helm chart you install below bundles its own CRDs (in its `crds/` folder) and will install them automatically, so nothing further is needed here.

## Resources
---
> You can find all of the resources for this tutorial [here](https://github.com/techno-tim/launchpad/tree/master/kubernetes/traefik-cert-manager)

<iframe width="560" height="315" src="https://www.youtube.com/embed/G4CmbYL9UPg?si=q0B41HPNr6vJycgi" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Helm
---
```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```
For other ways to install Helm see the installation docs [here](https://helm.sh/docs/intro/install)

## Installing
---
Verify you can communicate with your cluster
```shell
kubectl get nodes
```

You should see
```shell
NAME     STATUS   ROLES                       AGE   VERSION
k3s-01   Ready    control-plane,etcd,master   10h   v1.23.4+k3s1
k3s-02   Ready    control-plane,etcd,master   10h   v1.23.4+k3s1
k3s-03   Ready    control-plane,etcd,master   10h   v1.23.4+k3s1
k3s-04   Ready    <none>                      10h   v1.23.4+k3s1
k3s-05   Ready    <none>                      10h   v1.23.4+k3s1
```

Verify helm is installed
```shell
helm version
```

You should see
```shell
version.BuildInfo{Version:"v3.8.0", GitCommit:"d14138609b01886f544b2025f5000351c9eb092e", GitTreeState:"clean", GoVersion:"go1.17.5"}
```

## Traefik
---
> These [resources](https://github.com/techno-tim/launchpad/tree/master/kubernetes/traefik-cert-manager) are in the `launchpad/kubernetes/traefik-cert-manager/traefik/` folder

```shell
simonj@kube-m:~/traefik$ tree
.
├── cert
│   └── secret-to-copy.yaml
├── dashboard
│   ├── ingress.yaml
│   ├── middleware.yaml
│   └── secret-dashboard.yaml
├── default-headers.yaml
└── traefik-values.yaml

3 directories, 6 files
```

Here's the Ubuntu command to create that entire directory structure with all folders and files:
```shell
mkdir -p ~/traefik/cert
mkdir -p ~/traefik/dashboard
touch ~/traefik/cert/secret-to-copy.yaml
touch ~/traefik/dashboard/ingress.yaml
touch ~/traefik/dashboard/middleware.yaml
touch ~/traefik/dashboard/secret-dashboard.yaml
touch ~/traefik/default-headers.yaml
touch ~/traefik/traefik-values.yaml
```

secret-to-copy.yaml
```yaml
apiVersion: v1
data:
  tls.crt: REDACTED
  tls.key: REDACTED
kind: Secret
metadata:
  annotations:
    cert-manager.io/alt-names: '*.local.simonlab.xyz,local.simonlab.xyz'
    cert-manager.io/certificate-name: local-simonlab-xyz
    cert-manager.io/common-name: '*.local.simonlab.xyz'
    cert-manager.io/ip-sans: ""
    cert-manager.io/issuer-group: ""
    cert-manager.io/issuer-kind: ClusterIssuer
    cert-manager.io/issuer-name: letsencrypt-staging
    cert-manager.io/uri-sans: ""
  labels:
    controller.cert-manager.io/fao: "true"
  name: local-simonlab-xyz-staging-tls
  namespace: traefik
type: kubernetes.io/tls
```

ingress.yaml
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
  # Revert to using the annotation for the ingress class
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`kube-traefik.local.simonlab.xyz`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
      middlewares:
        - name: traefik-dashboard-basicauth
          namespace: traefik
  tls:
    # Reference the secret, which will now be in the same namespace (traefik)
    secretName: local-simonlab-xyz-staging-tls

# apiVersion: traefik.io/v1alpha1
# kind: IngressRoute
# metadata:
#   name: traefik-dashboard
#   namespace: traefik
#   annotations:
#     kubernetes.io/ingress.class: traefik-external
# spec:
#   entryPoints:
#     - websecure
#   routes:
#     - match: Host(`kube-traefik.local.simonlab.xyz`)
#       kind: Rule
#       middlewares:
#         - name: traefik-dashboard-basicauth
#           namespace: traefik
#       services:
#         - name: api@internal
#           kind: TraefikService
# # tls:
# #   secretName: traefik-dashboard-tls
```

middleware.yaml
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: traefik-dashboard-basicauth
  namespace: traefik
spec:
  basicAuth:
    secret: traefik-dashboard-auth
```

secret-dashboard.yaml
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-auth
  namespace: traefik
type: Opaque
data:
  users: REDACTED
```

default-headers.yaml
```yaml
# This Middleware is written to be compatible with an older Traefik v2/early-v3 CRD schema.
# It does NOT use the modern 'securityHeaders' block.

apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: default-headers
  namespace: default
spec:
  headers:
    # --- Security Headers (flat structure for older CRDs) ---
    browserXssFilter: true
    contentTypeNosniff: true
    stsIncludeSubdomains: true
    stsPreload: true
    stsSeconds: 31536000
    customFrameOptionsValue: "SAMEORIGIN"
    # 'forceSTSHeader' is not needed when stsSeconds is set.

    # --- Custom Headers ---
    customRequestHeaders:
      X-Forwarded-Proto: "https"
    
    customResponseHeaders:
      X-Powered-By: ""
```

traefik-values.yaml
```yaml
global:
  sendAnonymousUsage: false
  checkNewVersion: false

additionalArguments:
  - "--serversTransport.insecureSkipVerify=true"
  - "--log.level=INFO"

deployment:
  enabled: true
  replicas: 3
  annotations: {}
  podAnnotations: {}
  additionalContainers: []
  initContainers: []

ports:
  web:
    http:
      redirections:
        entryPoint:
          to: websecure
          priority: 10
  websecure:
    http3:
      enabled: true
      advertisedPort: 4443
    http:
      tls:
        enabled: true

ingressRoute:
  dashboard:
    enabled: false

providers:
  kubernetesCRD:
    enabled: true
    ingressClass: traefik-external
    allowExternalNameServices: true
  kubernetesIngress:
    enabled: true
    allowExternalNameServices: true
    publishedService:
      enabled: false

rbac:
  enabled: true

service:
  enabled: true
  type: LoadBalancer
  annotations: {}
  labels: {}
  spec:
    loadBalancerIP: 10.100.102.181 # this should be an IP in the MetalLB range
  loadBalancerSourceRanges: []
  externalIPs: []

# Command to install traefik
# helm install traefik traefik/traefik --namespace traefik --create-namespace --values traefik-values.yaml

# command to check traefik status
# kubectl get svc --all-namespaces -o wide
```

Add repo
```shell
helm repo add traefik https://helm.traefik.io/traefik
```

Update repo
```shell
helm repo update
```

Create our namespace
```shell
kubectl create namespace traefik
```

Get all namespaces
```shell
kubectl get namespaces
```

We should see
```shell
NAME              STATUS   AGE
default           Active   21h
kube-node-lease   Active   21h
kube-public       Active   21h
kube-system       Active   21h
metallb-system    Active   21h
traefik           Active   12s
```

Install traefik
```shell
helm install traefik traefik/traefik --namespace traefik --create-namespace --values traefik-values.yaml
```

Check the status of the traefik ingress controller service
```shell
kubectl get svc --all-namespaces -o wide
```

We should see traefik with the specified IP
```shell
NAMESPACE        NAME              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE   SELECTOR
default          kubernetes        ClusterIP      10.43.0.1       <none>          443/TCP                      16h   <none>
kube-system      kube-dns          ClusterIP      10.43.0.10      <none>          53/UDP,53/TCP,9153/TCP       16h   k8s-app=kube-dns
kube-system      metrics-server    ClusterIP      10.43.182.24    <none>          443/TCP                      16h   k8s-app=metrics-server
metallb-system   webhook-service   ClusterIP      10.43.205.142   <none>          443/TCP                      16h   component=controller
traefik          traefik           LoadBalancer   10.43.156.161   192.168.30.80   80:30358/TCP,443:31265/TCP   22s   app.kubernetes.io/instance=traefik,app.kubernetes.io/name=traefik
```

Get all pods in `traefik` namespace
```shell
kubectl get pods --namespace traefik
```

We should see pods in the `traefik` namespace
```shell
NAME                       READY   STATUS    RESTARTS   AGE
traefik-76474c4d47-l5z74   1/1     Running   0          11m
traefik-76474c4d47-xb282   1/1     Running   0          11m
traefik-76474c4d47-xx5lw   1/1     Running   0          11m
```

### middleware
---
Apply middleware
```shell
kubectl apply -f default-headers.yaml
```

Get middleware
```shell
kubectl get middleware
```

We should see our headers
```shell
NAME              AGE
default-headers   25s
```
### dashboard
---
Install `htpassword`
```shell
sudo apt-get update
sudo apt-get install apache2-utils
```

Generate a credential / password that’s base64 encoded
```shell
htpasswd -nb simonj password | openssl base64
```

Apply secret
```shell
kubectl apply -f secret-dashboard.yaml
```

Get secret
```
kubectl get secrets --namespace traefik
```

Apply middleware
```shell
kubectl apply -f middleware.yaml
```

Apply dashboard
```shell
kubectl apply -f ingress.yaml
```

Visit `https://traefik.local.example.com`

## Sample Workload
---
> These [resources](https://github.com/techno-tim/launchpad/tree/master/kubernetes/traefik-cert-manager) are in the `launchpad/kubernetes/traefik-cert-manager/nginx/` folder

```shell
simonj@kube-m:~/nanntechworld/nginx-cert$ tree
.
├── deployment.yaml
├── ingress.yaml
└── service.yaml

1 directory, 3 files
```

Here's the Ubuntu command to create that nginx-cert directory structure:
```shell
mkdir -p ~/nginx-cert
touch ~/nginx-cert/deployment.yaml
touch ~/nginx-cert/ingress.yaml
touch ~/nginx-cert/service.yaml
```

deployment.yaml
```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 3
  progressDeadlineSeconds: 600
  revisionHistoryLimit: 2
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```

ingress.yaml
```yaml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx
  namespace: default
  annotations: 
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.nginx-cert.local.simonlab.xyz`)
      kind: Rule
      services:
        - name: nginx
          port: 80
    - match: Host(`nginx-cert.local.simonlab.xyz`)
      kind: Rule
      services:
        - name: nginx
          port: 80
      middlewares:
        - name: default-headers
  tls:
    secretName: local-simonlab-xyz-tls
```

service.yaml
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  # CHANGE THIS from LoadBalancer to ClusterIP
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```shell
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

Or you can apply an entire folder at once!
```shell
kubectl apply -f nginx
```

## cert-manager
---
> These [resources](https://github.com/techno-tim/launchpad/tree/master/kubernetes/traefik-cert-manager) are in the `launchpad/kubernetes/traefik-cert-manager/cert-manager/` folder

```shell
simonj@kube-m:~/cert-manager$ tree
.
├── certificates
│   ├── production
│   │   └── local-simonlab-xyz.yaml
│   └── staging
│       └── local-simonlab-xyz.yaml
├── issuers
│   ├── letsencrypt-production.yaml
│   ├── letsencrypt-staging.yaml
│   └── secret-cf-token.yaml
└── values.yaml

5 directories, 6 files
```

Here's the Ubuntu command to create that cert-manager directory structure:
```shell
mkdir -p ~/services/cert-manager/certificates/production
mkdir -p ~/services/cert-manager/certificates/staging
mkdir -p ~/services/cert-manager/issuers
touch ~/services/cert-manager/certificates/production/local-simonlab-xyz.yaml
touch ~/services/cert-manager/certificates/staging/local-simonlab-xyz.yaml
touch ~/services/cert-manager/issuers/letsencrypt-production.yaml
touch ~/services/cert-manager/issuers/letsencrypt-staging.yaml
touch ~/services/cert-manager/issuers/secret-cf-token.yaml
touch ~/services/cert-manager/values.yaml
```

│   ├── production
│   │   └── local-simonlab-xyz.yaml
```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: local-simonlab-xyz
  namespace: default
spec:
  secretName: local-simonlab-xyz-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: "*.local.simonlab.xyz"
  dnsNames:
  - "local.simonlab.xyz"
  - "*.local.simonlab.xyz"
```

│   └── staging
│       └── local-simonlab-xyz.yaml
```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: local-simonlab-xyz
  namespace: default
spec:
  secretName: local-simonlab-xyz-staging-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: "*.local.simonlab.xyz"
  dnsNames:
  - "local.simonlab.xyz"
  - "*.local.simonlab.xyz"
```

letsencrypt-production.yaml
```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: simonjan2@hotmail.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - dns01:
          cloudflare:
            email: simonjan2@hotmail.com
            apiTokenSecretRef:
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "simonlab.xyz"
```

letsencrypt-staging.yaml
```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: simonjan2@hotmail.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - dns01:
          cloudflare:
            email: simonjan2@hotmail.com
            apiTokenSecretRef:
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "simonlab.xyz"
```

secret-cf-token.yaml
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  cloudflare-token: REDACTED # be sure you are generating an API token and not a global API key https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/#api-tokens
```

values.yaml
```yaml
installCRDs: false
replicaCount: 3
extraArgs:
  - --dns01-recursive-nameservers=1.1.1.1:53,9.9.9.9:53
  - --dns01-recursive-nameservers-only
podDnsPolicy: None
podDnsConfig:
  nameservers:
    - 1.1.1.1
    - 9.9.9.9
```

Add repo
```shell
helm repo add jetstack https://charts.jetstack.io
```

Update it
```shell
helm repo update
```

Create our namespace
```shell
kubectl create namespace cert-manager
```

Get all namespaces
```shell
kubectl get namespaces
```

We should see
```shell
NAME              STATUS   AGE
cert-manager      Active   12s
default           Active   21h
kube-node-lease   Active   21h
kube-public       Active   21h
kube-system       Active   21h
metallb-system    Active   21h
traefik           Active   4h35m
```

Apply crds
> _Note: Be sure to change this to the [latest version](https://cert-manager.io/docs/installation/supported-releases/) of `cert-manager`_
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.1/cert-manager.crds.yaml
```

Install with helm
```shell
helm install cert-manager jetstack/cert-manager --namespace cert-manager --values=values.yaml --version v1.21.1
```

Apply secrets
> Be sure to generate the correct token if using Cloudflare.This is using an [API Token](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/#api-tokens) and not a global key.

From `issuers` folder
```shell
kubectl apply -f secret-cf-token.yaml
```

Apply staging `ClusterIssuer`

From `issuers` folder
```shell
kubectl apply -f letsencrypt-staging.yaml
```

Create certs

### staging
---

From `certificates/staging` folder
```shell
kubectl apply -f local-example-com.yaml
```

Check the logs
```shell
kubectl logs -n cert-manager -f cert-manager-877fd747c-fjwhp
```

Get `challenges`
```shell
kubectl get challenges
```

Get more details
```shell
kubectl describe order local-technotim-live-frm2z-1836084675
```

### production
---

Apply production `ClusterIssuer`

From `issuers` folder
```shell
kubectl apply -f letsencrypt-production.yaml
```

From `certificates/production` folder
```shell
kubectl apply -f local-example-com.yaml
```