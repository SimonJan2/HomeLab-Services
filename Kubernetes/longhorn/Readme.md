## Table Of Content
---
- [[#Getting Started|Getting Started]]
- [[#Resources|Resources]]
	- [[#Resources#Helm|Helm]]
	- [[#Resources#longhornctl (preflight checker)|longhornctl (preflight checker)]]
- [[#Prerequisites Verification|Prerequisites Verification]]
	- [[#Prerequisites Verification#Cluster connectivity|Cluster connectivity]]
	- [[#Prerequisites Verification#Kubernetes version|Kubernetes version]]
	- [[#Prerequisites Verification#iSCSI / NFS / disk on every node|iSCSI / NFS / disk on every node]]
- [[#Preflight Check (optional, recommended)|Preflight Check (optional, recommended)]]
	- [[#Preflight Check (optional, recommended)#Loading the kernel modules (recommended, do on every node)|Loading the kernel modules]]
- [[#Installing Longhorn|Installing Longhorn]]
	- [[#Installing Longhorn#Add the Longhorn Helm repository|Add the Longhorn Helm repository]]
	- [[#Installing Longhorn#The values file|The values file]]
	- [[#Installing Longhorn#Install with Helm|Install with Helm]]
	- [[#Installing Longhorn#Confirm the deployment|Confirm the deployment]]
- [[#Verifying the Installation|Verifying the Installation]]
- [[#Setting Longhorn as the Default StorageClass|Setting Longhorn as the Default StorageClass]]
- [[#Exposing the UI with Traefik + cert-manager|Exposing the UI with Traefik + cert-manager]]
	- [[#Exposing the UI with Traefik + cert-manager#Directory layout|Directory layout]]
	- [[#Exposing the UI with Traefik + cert-manager#TLS certificate|TLS certificate]]
	- [[#Exposing the UI with Traefik + cert-manager#Basic auth secret|Basic auth secret]]
	- [[#Exposing the UI with Traefik + cert-manager#Middlewares|Middlewares]]
	- [[#Exposing the UI with Traefik + cert-manager#IngressRoute|IngressRoute]]
	- [[#Exposing the UI with Traefik + cert-manager#Visit the UI|Visit the UI]]
- [[#Sample Workload|Sample Workload]]
	- [[#Sample Workload#Prove persistence|Prove persistence]]
- [[#K3s-Specific Notes|K3s-Specific Notes]]
- [[#Troubleshooting|Troubleshooting]]
	- [[#Troubleshooting#API server / kube-vip VIP unreachable (`connection refused` to `10.100.102.175:6443`)|API server / kube-vip VIP unreachable]]
	- [[#Troubleshooting#Wrong `csi.kubeletRootDir` — CSI driver never registers (PVCs stuck, pods stuck `ContainerCreating`)|Wrong kubeletRootDir]]
- [[#Uninstall (optional)|Uninstall (optional)]]

## Getting Started
---
This guide installs **[Longhorn](https://longhorn.io/) v1.12.1** — a cloud-native distributed block storage engine — on the K3s cluster built by my [Ansible playbook](https://github.com/SimonJan2/Ansible).

The playbook already did the heavy lifting that Longhorn normally requires:
- Installed `open-iscsi`, `nfs-common`, and `util-linux` on every node
- Enabled and started the `iscsid` daemon
- Formatted `/dev/sdb` (60 GB) as ext4 and mounted it persistently at **`/var/lib/longhorn`** on all 6 nodes

> `/var/lib/longhorn` is exactly Longhorn's default data path, so **no disk preparation is needed in this guide** — we just point Longhorn at it.

> ⚠️ **Unlike Traefik, K3s does NOT ship with Longhorn.** There is nothing to disable before installing. K3s's only built-in StorageClass is `local-path` (Rancher Local Path Provisioner), which we will demote so Longhorn becomes the default. See [[#Setting Longhorn as the Default StorageClass|Setting Longhorn as the Default StorageClass]].

### Cluster at a glance
| Role | Nodes | IPs | RAM | OS disk | Longhorn disk |
|---|---|---|---|---|---|
| Control-plane (etcd) | `kube-master-1/2/3` | `10.100.102.168–170` | 1.6 Gi | `/dev/sda` 50 GB | `/dev/sdb` 60 GB → `/var/lib/longhorn` |
| Workers | `kube-node-1/2/3` | `10.100.102.171–173` | 3.3 Gi | `/dev/sda` 50 GB | `/dev/sdb` 60 GB → `/var/lib/longhorn` |

- **K3s version:** `v1.36.2+k3s1` (Kubernetes 1.36 — above Longhorn v1.12.1's v1.34+ requirement)
- **Control-plane VIP (kube-vip):** `10.100.102.175`
- **MetalLB pool:** `10.100.102.180–185`
- **Traefik** (custom helm install, `traefik-external` IngressClass, MetalLB IP `10.100.102.181`)
- **cert-manager** v1.21.1 with a Cloudflare DNS-01 `letsencrypt-production` `ClusterIssuer` issuing a wildcard `*.local.simonlab.xyz`

This guide reuses all of the above — **we will not reinstall Traefik, cert-manager, or MetalLB**.

## Resources
---
> All resources for this tutorial live in the `services/longhorn/` folder next to this guide.

<iframe width="560" height="315" src="https://www.youtube.com/embed/q95g9IuCNjg?si=PLACEHOLDER" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Official documentation: <https://longhorn.io/docs/1.12.1/>

### Helm
---
Helm is already installed on this cluster's control node (used for Traefik and cert-manager). Verify:
```shell
helm version
```
You should see something like:
```shell
version.BuildInfo{Version:"v3.x.x", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.x"}
```

If you need to (re)install Helm:
```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

### longhornctl (preflight checker)
---
Longhorn ships a CLI that checks the nodes for missing prerequisites. Download it (amd64):
```shell
curl -sSfL -o longhornctl \
  https://github.com/longhorn/cli/releases/download/v1.12.1/longhornctl-linux-amd64
chmod +x longhornctl
sudo mv longhornctl /usr/local/bin/
```

Verify:
```shell
longhornctl version
```

We will use it in [[#Preflight Check (optional, recommended)|Preflight Check]].

## Prerequisites Verification
---
### Cluster connectivity
---
Verify you can talk to the cluster:
```shell
kubectl get nodes -o wide
```
You should see all 6 nodes `Ready`:
```shell
NAME            STATUS   ROLES                AGE   VERSION        INTERNAL-IP      EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION             CONTAINER-RUNTIME
kube-master-1   Ready    control-plane,etcd   13h   v1.36.2+k3s1   10.100.102.168   <none>        Ubuntu 26.04 LTS   7.0.0-30-generic (amd64)   containerd://2.3.2-k3s2
kube-master-2   Ready    control-plane,etcd   13h   v1.36.2+k3s1   10.100.102.169   <none>        Ubuntu 26.04 LTS   7.0.0-27-generic (amd64)   containerd://2.3.2-k3s2
kube-master-3   Ready    control-plane,etcd   13h   v1.36.2+k3s1   10.100.102.170   <none>        Ubuntu 26.04 LTS   7.0.0-29-generic (amd64)   containerd://2.3.2-k3s2
kube-node-1     Ready    <none>               13h   v1.36.2+k3s1   10.100.102.171   <none>        Ubuntu 26.04 LTS   7.0.0-30-generic (amd64)   containerd://2.3.2-k3s2
kube-node-2     Ready    <none>               13h   v1.36.2+k3s1   10.100.102.172   <none>        Ubuntu 26.04 LTS   7.0.0-27-generic (amd64)   containerd://2.3.2-k3s2
kube-node-3     Ready    <none>               13h   v1.36.2+k3s1   10.100.102.173   <none>        Ubuntu 26.04 LTS   7.0.0-30-generic (amd64)   containerd://2.3.2-k3s2
```

### Kubernetes version
---
Longhorn v1.12.1 requires Kubernetes **>= v1.34**.
```shell
kubectl version
```
You should see `Server Version: ... v1.36.2+k3s1` — well above the minimum.

### iSCSI / NFS / disk on every node
---
The Ansible `prepare-nodes` role installed everything Longhorn needs. Spot-check on any node (example shows `kube-master-1`):
```shell
ssh kube-master-1
```
```shell
# iscsid must be active
systemctl is-active iscsid
# -> active

# Packages must be present
dpkg -l | grep -E 'open-iscsi|nfs-common|util-linux'
# -> ii  open-iscsi ...
# -> ii  nfs-common ...
# -> ii  util-linux ...

# /dev/sdb must be mounted at /var/lib/longhorn
df -h /var/lib/longhorn
# -> /dev/sdb   59G   98M   56G   1% /var/lib/longhorn

lsblk
# -> sdb   8:16   0   60G  0 disk
```
Repeat for the other 5 nodes (or trust the Ansible playbook — it ran idempotently across all of them).

## Preflight Check (optional, recommended)
---
Run the official preflight checker against the cluster. It spins up a temporary pod on each node and verifies `iscsid`, NFS4 support, the required packages, and (for V2) kernel modules / HugePages.

> ⚠️ **The checker needs the `longhorn-system` namespace to exist** (it runs its check pods there) and an explicit kubeconfig path (`longhornctl` does NOT auto-discover `~/.kube/config` the way `kubectl` does). Create the namespace first, then run the check:
```shell
kubectl create namespace longhorn-system
```
```shell
longhornctl check preflight --kubeconfig=/home/simonj/.kube/config
```
> If you prefer, export `KUBECONFIG` once and drop the flag from subsequent `longhornctl` commands:
> ```shell
> export KUBECONFIG=/home/simonj/.kube/config
> longhornctl check preflight
> ```

Expected output for this cluster (V1 engine) — one block per node, all 6 identical:
```shell
INFO[...] Initializing preflight checker
INFO[...] Cleaning up preflight checker
INFO[...] Running preflight checker
INFO[...] Retrieved preflight checker result:
kube-master-1:
  error:
  - '[KernelModules] nfs is not loaded. (exit code: 1)'
  - '[KernelModules] dm_crypt is not loaded. (exit code: 1)'
  info:
  - '[IscsidService] Service iscsid is running'
  - '[NFSv4] NFS4 is supported'
  - '[Packages] nfs-common is installed'
  - '[Packages] open-iscsi is installed'
  - '[Packages] cryptsetup is installed'
  - '[Packages] dmsetup is installed'
  warn:
  - '[KubeDNS] Kube DNS "coredns" is set with fewer than 2 replicas; consider increasing replica count for high availability'
  - '[MultipathService] multipathd.service is running. Please refer to https://longhorn.io/kb/troubleshooting-volume-with-multipath/ for more information.'
... (one identical block per node)
INFO[...] Cleaning up preflight checker
INFO[...] Completed preflight checker
```
> The checker exits `0` (success) even with the items above — they are **non-blocking for a basic V1 install**. Here is what each means and whether you should act on it:

| Item | Severity | Needed for | Action |
|---|---|---|---|
| `nfs` kernel module not loaded | error → non-blocking | **RWX (ReadWriteMany) volumes only** | Load it (see below) if you plan to use RWX volumes |
| `dm_crypt` kernel module not loaded | error → non-blocking | **Encrypted volumes (LUKS) only** | Load it if you plan to use volume encryption |
| `multipathd.service` running | warn | — | No action; informational (see the [KB article](https://longhorn.io/kb/troubleshooting-volume-with-multipath/) if you hit multipath volume issues) |
| coredns < 2 replicas | warn | — | No action for a homelab; K3s runs 1 coredns replica by default |

### Loading the kernel modules (recommended, do on every node)
---
Both modules are available on the nodes (`modinfo nfs` / `modinfo dm_crypt` succeed) — they're just not loaded by default. Load them now and persist across reboots so RWX volumes and encrypted volumes work when you need them.

On **each of the 6 nodes** (log in and run):
```shell
# Load immediately (takes effect now)
sudo modprobe nfs
sudo modprobe dm_crypt

# Persist across reboots
echo -e "nfs\ndm_crypt" | sudo tee /etc/modules-load.d/longhorn.conf

# Verify
lsmod | grep -E '^nfs|dm_crypt'
```

> 💡 **Tip — do it with Ansible instead of logging into 6 nodes.** This is a natural addition to the `prepare-nodes` role. Add a task like:
> ```yaml
> - name: Load Longhorn kernel modules
>   community.general.modprobe:
>     name: "{{ item }}"
>     state: present
>     persistent: present
>   loop:
>     - nfs
>     - dm_crypt
>   tags: longhorn
> ```
> Then `ansible-playbook site.yaml --tags longhorn` applies it to all nodes at once.

Re-run the preflight check after loading the modules — the `nfs` / `dm_crypt` errors should be gone:
```shell
longhornctl check preflight --kubeconfig=/home/simonj/.kube/config
```

## Installing Longhorn
---
### Add the Longhorn Helm repository
---
```shell
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

### The values file
---
> ⚠️ **Critical: verify your kubelet root dir BEFORE installing.** K3s's kubelet root directory has changed across versions/OSes — some setups use the K3s-specific `/var/lib/rancher/k3s/agent/kubelet`, but **K3s v1.36.2 on Ubuntu 26.04 uses the standard `/var/lib/kubelet`**. Getting this wrong causes the CSI driver to silently fail to register (see [[#Troubleshooting|Troubleshooting]] for the full symptom list and fix). Check on any node BEFORE running `helm install`:
> ```shell
> # Whichever of these has real pod directories (pods/<uuid>/...) is the real kubelet root.
> sudo ls /var/lib/kubelet/pods 2>&1 | head -3
> sudo ls /var/lib/rancher/k3s/agent/kubelet/pods 2>&1 | head -3
> ```
> Use whichever path actually contains pod UUID directories as `csi.kubeletRootDir` below.

The full file is at `services/longhorn/longhorn-values.yaml`. The key decisions baked in:
> - **Longhorn is the default StorageClass** (`persistence.defaultClass: true`)
> - **V1 Data Engine** (`persistence.dataEngine: v1`) — low resource use, fits the 1.6–3.3 GB nodes
> - **3 replicas** for the default StorageClass (across the 6 nodes)
> - **`reclaimPolicy: Retain`** — PVs survive PVC deletion (safer for a homelab)
> - **`csi.kubeletRootDir: /var/lib/kubelet`** — set explicitly. ⚠️ **Verify this on YOUR cluster before installing** — see the callout below.
> - **`defaultSettings.defaultDataPath: /var/lib/longhorn/`** — matches the Ansible `/dev/sdb` mount
> - Telemetry / upgrade-checker disabled
> - The chart's built-in Ingress is **disabled** — we expose the UI with our own Traefik `IngressRoute` (see [[#Exposing the UI with Traefik + cert-manager|below]])

```yaml
# longhorn-values.yaml
---
persistence:
  createStorageClass: true
  defaultClass: true
  dataEngine: v1
  defaultFsType: ext4
  defaultClassReplicaCount: 3
  defaultDataLocality: disabled
  reclaimPolicy: Retain
  volumeBindingMode: "Immediate"
  disableRevisionCounter: "true"
  migratable: false
  nfsOptions: ""

csi:
  # K3s kubelet root dir — set explicitly for deterministic CSI registration.
  kubeletRootDir: "/var/lib/rancher/k3s/agent/kubelet"

defaultSettings:
  defaultDataPath: "/var/lib/longhorn/"
  storageMinimalAvailablePercentage: 25
  storageOverProvisioningPercentage: 100
  autoSalvage: true
  guaranteedInstanceManagerCPU: false
  upgradeChecker: false
  allowCollectingLonghornUsageMetrics: false
  allowRecurringJobWhileVolumeDetached: false

longhornUI:
  replicas: 2

ingress:
  enabled: false

httproute:
  enabled: false

enablePSP: false
```

### Install with Helm
---
> The `longhorn-system` namespace was already created in [[#Preflight Check (optional, recommended)|Preflight Check]]. `--create-namespace` is kept below for safety (it's idempotent — it won't error if the namespace exists).
```shell
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.12.1 \
  --values longhorn-values.yaml
```
> ℹ️ You may see a stream of `Warning: unrecognized format "int64"` lines during the install. **These are harmless** — they come from a CRD schema-validation quirk in Kubernetes 1.36 against Longhorn's CRDs. The install still succeeds (`STATUS: deployed`) and Longhorn works normally. They can be ignored.
>
> ℹ️ On memory-constrained control-plane nodes (this cluster's masters have 1.6 GB RAM), the simultaneous startup of Longhorn's DaemonSet pods on the masters can briefly starve the kube-apiserver / kube-vip and cause a momentary `connection refused` to the VIP (`10.100.102.175:6443`). It self-recovers within ~10–30s as pods finish starting. If it does not recover, see [[#Troubleshooting|Troubleshooting]] → *API server / kube-vip VIP unreachable*.

### Confirm the deployment
---
Wait for everything to come up (the CSI plugins take a little longer than the manager):
```shell
kubectl -n longhorn-system get pods -w
```
You should eventually see all pods `Running`:
```shell
NAME                                                READY   STATUS    RESTARTS   AGE
longhorn-ui-xxxxxxxxx-yyyyy                         1/1     Running   0          2m
longhorn-manager-abcde                              1/1     Running   0          2m     # one per node (DaemonSet)
longhorn-manager-fghij                              1/1     Running   0          2m
... (6 longhorn-manager pods total, one per node)
longhorn-driver-deployer-xxxxxxxxx-yyyyy            1/1     Running   0          2m
instance-manager-xxxxxxxxxxxxxxxxxxxxxxxxxx         1/1     Running   0          90s    # one per node
engine-image-ei-xxxxxxxx-yyyyy                      1/1     Running   0          90s    # one per node
longhorn-csi-plugin-xxxxx                           2/2     Running   0          90s    # one per node
csi-attacher-xxxxxxxxx-yyyyy                        1/1     Running   0          90s    # x3
csi-provisioner-xxxxxxxxx-yyyyy                     1/1     Running   0          90s    # x3
csi-resizer-xxxxxxxxx-yyyyy                         1/1     Running   0          90s    # x3
csi-snapshotter-xxxxxxxxx-yyyyy                     1/1     Running   0          90s    # x3
```
> The `longhorn-manager`, `instance-manager`, `engine-image-*`, and `longhorn-csi-plugin` pods are DaemonSets — you get one per node (6 of each). The `csi-*` controller pods and `longhorn-ui` are Deployments with 3 / 3 / 3 / 3 / 2 replicas respectively.

## Verifying the Installation
---
Check the StorageClasses:
```shell
kubectl get sc
```
You will likely see:
```shell
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  14h
longhorn (default)     driver.longhorn.io      Retain          Immediate              true                   5m
longhorn-static        driver.longhorn.io      Delete          Immediate              true                   5m
```
> ⚠️ **Two StorageClasses are marked `(default)`!** Helm adds `storageclass.kubernetes.io/is-default-class=true` to the `longhorn` SC but does **NOT** remove it from K3s's pre-existing `local-path`. With two defaults, a PVC with no `storageClassName` is ambiguous (provisioning can fail or pick the wrong one). **You must demote `local-path` manually** — see [[#Setting Longhorn as the Default StorageClass|the next section]].

> The extra `longhorn-static` SC is created automatically by Longhorn for **statically-provisioned** volumes (pre-existing PVs you import by hand). It has `reclaimPolicy: Delete` and is NOT marked default — leave it alone; it does not affect normal dynamic PVCs.

Confirm the Longhorn nodes all show their 60 GB disk (the manager reports to the UI/API; you can also check via the CRDs):
```shell
kubectl -n longhorn-system get nodes.longhorn.io
kubectl -n longhorn-system get nodes.longhorn.io -o wide
```
You should see all 6 nodes `Ready` and `Schedulable`:
```shell
NAME            READY   ALLOWSCHEDULING   SCHEDULABLE   AGE
kube-master-1   True    true              True          4m
kube-master-2   True    true              True          4m
kube-master-3   True    true              True          5m
kube-node-1     True    true              True          5m
kube-node-2     True    true              True          4m
kube-node-3     True    true              True          3m
```

Check the CSI driver registered with the kubelet:
```shell
kubectl get csidrivers | grep longhorn
```
You should see `driver.longhorn.io` registered:
```shell
NAME                 ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   ...   AGE
driver.longhorn.io   true             true             false             ...   4m
```
> The exact columns vary by Kubernetes version; the important thing is that `driver.longhorn.io` appears. This means the kubelet found Longhorn's CSI plugin (using the `csi.kubeletRootDir` we set in the values). If it is missing, see [[#Troubleshooting|Troubleshooting]].

## Setting Longhorn as the Default StorageClass
---
Setting `persistence.defaultClass: true` in the values file makes Longhorn annotate itself as a default StorageClass — but Helm's hook **does NOT remove the `is-default-class` annotation from K3s's pre-existing `local-path`**, so you end up with **two** defaults (as seen above). This is ambiguous and must be fixed.

Demote `local-path` (and confirm `longhorn` is the only default):
```shell
# Demote local-path
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Verify:
```shell
kubectl get sc
```
You should now see **only** `longhorn (default)`:
```shell
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path             rancher.io/local-path   Delete          WaitForFirstConsumer   false                  14h
longhorn (default)     driver.longhorn.io      Retain          Immediate              true                   5m
longhorn-static        driver.longhorn.io      Delete          Immediate              true                   5m
```

> `longhorn` was already annotated `true` by the install, so you normally only need the demote command above. If `longhorn` is ever missing the annotation, promote it explicitly:
> ```shell
> kubectl patch storageclass longhorn \
>   -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
> ```

> **Why keep `local-path` at all?** Some lightweight workloads (logs, scratch space, single-replica caches) are fine on `local-path` and don't need replication. To use it for a specific PVC, set `storageClassName: local-path` on the PVC.

## Exposing the UI with Traefik + cert-manager
---
Longhorn installs a `longhorn-frontend` `Service` (ClusterIP, port 80) in `longhorn-system`. The UI has **no built-in authentication**, so we front it with Traefik basic-auth and terminate TLS with cert-manager, exactly like the Traefik dashboard.

### Directory layout
---
```shell
simonj@kube-master-1:~/services/longhorn$ tree
.
├── certificate.yaml
├── longhorn-values.yaml
├── secret-tls.yaml
├── sample
│   ├── nginx-deployment.yaml
│   ├── nginx-ingressroute.yaml
│   ├── nginx-pvc.yaml
│   └── nginx-svc.yaml
└── ui
    ├── ingressroute.yaml
    ├── middleware.yaml
    └── secret-basic-auth.yaml

2 directories, 9 files
```

Here's the Ubuntu command to recreate that directory structure (already present in this repo):
```shell
mkdir -p ~/services/longhorn/ui
mkdir -p ~/services/longhorn/sample
touch ~/services/longhorn/certificate.yaml
touch ~/services/longhorn/longhorn-values.yaml
touch ~/services/longhorn/secret-tls.yaml
touch ~/services/longhorn/ui/ingressroute.yaml
touch ~/services/longhorn/ui/middleware.yaml
touch ~/services/longhorn/ui/secret-basic-auth.yaml
touch ~/services/longhorn/sample/nginx-pvc.yaml
touch ~/services/longhorn/sample/nginx-deployment.yaml
touch ~/services/longhorn/sample/nginx-svc.yaml
touch ~/services/longhorn/sample/nginx-ingressroute.yaml
```

### TLS certificate
---
The Traefik `IngressRoute` needs the TLS secret `local-simonlab-xyz-tls` to exist **in the `longhorn-system` namespace** (Traefik looks up the TLS secret in the same namespace as the IngressRoute by default). There are two ways to get it there:

#### Approach A — Copy the existing secret (recommended, works immediately)
---
You already have a valid wildcard `*.local.simonlab.xyz` TLS secret in the `default` namespace (created during the Traefik/cert-manager setup). Simply copy it into `longhorn-system`:

```shell
kubectl get secret local-simonlab-xyz-tls -n default -o yaml \
  | sed 's/namespace: default/namespace: longhorn-system/' \
  | kubectl apply -f -
```

> The pre-made `secret-tls.yaml` file in this directory is a copy of this secret, so you can also just:
> ```shell
> kubectl apply -f secret-tls.yaml
> ```

Verify:
```shell
kubectl get secret local-simonlab-xyz-tls -n longhorn-system
```
You should see:
```shell
NAME                     TYPE                DATA   AGE
local-simonlab-xyz-tls   kubernetes.io/tls   2      0s
```

> ⚠️ **This copied secret does NOT auto-renew.** When cert-manager renews the original cert in the `default` namespace (~30 days before expiry), re-run the copy command above to refresh this one. Alternatively, use Approach B below once the rate limit clears, and cert-manager will handle renewal automatically.

#### Approach B — Issue a fresh Certificate via cert-manager (auto-renews, but rate-limited)
---
> `certificate.yaml` issues the wildcard `*.local.simonlab.xyz` **into the `longhorn-system` namespace** via the existing `letsencrypt-production` ClusterIssuer. This auto-renews, so it's the cleaner long-term approach.

```yaml
# certificate.yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: local-simonlab-xyz
  namespace: longhorn-system
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

Apply and watch it become `Ready`:
```shell
kubectl apply -f certificate.yaml
kubectl get certificate -n longhorn-system -w
```
You should see:
```shell
NAME                 READY   SECRET                   AGE
local-simonlab-xyz   True    local-simonlab-xyz-tls   30s
```

> ⚠️ **Let's Encrypt rate limit.** If you already issued 5 certificates for this exact set of identifiers (`*.local.simonlab.xyz` + `local.simonlab.xyz`) in the last 168 hours (7 days) — e.g. during the Traefik/cert-manager setup — the Certificate will stay `False` with:
> ```
> 429 too many certificates (5) already issued for this exact set of identifiers
> in the last 168h0m0s, retry after <timestamp>
> ```
> This is expected. **Use Approach A (copy the secret) instead**, and come back to Approach B after the retry window opens. You can check the exact retry time with:
> ```shell
> kubectl describe certificate local-simonlab-xyz -n longhorn-system | grep "retry after"
> ```
> If you hit the rate limit, clean up the failed Certificate so cert-manager stops retrying:
> ```shell
> kubectl delete certificate local-simonlab-xyz -n longhorn-system
> ```

> If the Certificate stays `False` for a reason OTHER than the rate limit, check `kubectl describe certificate local-simonlab-xyz -n longhorn-system` and the cert-manager logs: `kubectl logs -n cert-manager -l app.kubernetes.io/name=cert-manager`. The most common cause is a stale/invalid Cloudflare API token in the `cloudflare-token-secret` in `cert-manager`.

#### Which approach did this guide use?
---
This cluster hit the Let's Encrypt rate limit (5 certs for `*.local.simonlab.xyz` already issued in the last 7 days during the Traefik setup), so we used **Approach A** (copy the secret). The `certificate.yaml` file is still provided for when the rate limit clears — switch to it later by deleting the copied secret and applying the Certificate.

### Basic auth secret
---
Install `htpasswd` (if missing):
```shell
sudo apt-get update
sudo apt-get install apache2-utils
```

Generate a credential line for user `simonj`:
```shell
htpasswd -nb simonj 'YOUR_PASSWORD_HERE'
```
Example output:
```shell
simonj:$apr1$abcdEFGH$1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d
```

**Option A (recommended) — let kubectl build the Secret directly:**
```shell
kubectl -n longhorn-system create secret generic basic-auth \
  --from-file=auth=<(htpasswd -nb simonj 'YOUR_PASSWORD_HERE')
```

**Option B — base64-encode the line and put it in `ui/secret-basic-auth.yaml`:**
```shell
echo -n 'simonj:$apr1$abcdEFGH$1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d' | base64
```
Paste the output as the value of `auth:` in `ui/secret-basic-auth.yaml`, then:
```shell
kubectl apply -f ui/secret-basic-auth.yaml
```

Verify:
```shell
kubectl get secret -n longhorn-system basic-auth
# -> basic-auth   Opaque   1   30s
```
> ⚠️ The shipped `ui/secret-basic-auth.yaml` contains a **placeholder** (`simonj:REPLACE_ME`) — it will NOT authenticate. You must replace it with one of the two options above before the UI is reachable.

### Middlewares
---
> `ui/middleware.yaml` defines two Traefik Middlewares in `longhorn-system`: `longhorn-auth` (basic-auth from the secret) and `longhorn-buffering` (raises the request body limit to 10 GB so backing-image uploads from the UI work — per the official Longhorn Traefik ingress guide).

```yaml
# ui/middleware.yaml
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: longhorn-auth
  namespace: longhorn-system
spec:
  basicAuth:
    secret: basic-auth
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: longhorn-buffering
  namespace: longhorn-system
spec:
  buffering:
    maxRequestBodyBytes: 10485760000
```

Apply:
```shell
kubectl apply -f ui/middleware.yaml
kubectl get middleware -n longhorn-system
```
You should see:
```shell
NAME                AGE
longhorn-auth       5s
longhorn-buffering  5s
```

### IngressRoute
---
> `ui/ingressroute.yaml` is a Traefik `IngressRoute` (the CRD form, matching the dashboard) — NOT the standard `networking.k8s.io/v1` Ingress the chart would create. We use the `kubernetes.io/ingress.class: traefik-external` annotation so it is picked up by the custom Traefik instance.

```yaml
# ui/ingressroute.yaml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: longhorn-frontend
  namespace: longhorn-system
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`longhorn-ui.local.simonlab.xyz`)
      kind: Rule
      services:
        - name: longhorn-frontend
          port: 80
      middlewares:
        - name: longhorn-auth
          namespace: longhorn-system
        - name: longhorn-buffering
          namespace: longhorn-system
  tls:
    secretName: local-simonlab-xyz-tls
```

Apply:
```shell
kubectl apply -f ui/ingressroute.yaml
kubectl get ingressroute -n longhorn-system
```
You should see:
```shell
NAME               AGE
longhorn-frontend  5s
```

### Visit the UI
---
Make sure `*.local.simonlab.xyz` resolves to the Traefik MetalLB IP (`10.100.102.181`) — your DNS / `dnsmasq` / Cloudflare already handles this if the Traefik dashboard works.

Quick auth check from the CLI:
```shell
# Without credentials -> 401
curl -Ik https://longhorn-ui.local.simonlab.xyz
# -> HTTP/2 401
# -> Www-Authenticate: Basic realm="traefik"

# With credentials -> 200
curl -Iku simonj https://longhorn-ui.local.simonlab.xyz
# -> HTTP/2 200
```

Open in a browser:
```
https://longhorn-ui.local.simonlab.xyz
```
You should be prompted for credentials, then see the Longhorn dashboard with all 6 nodes and their 60 GB `/var/lib/longhorn` disks listed under **Node**.

## Sample Workload
---
A tiny nginx deployment that uses a Longhorn PVC, proving dynamic provisioning + replication work.

```shell
simonj@kube-master-1:~/services/longhorn$ tree sample
sample
├── nginx-deployment.yaml
├── nginx-ingressroute.yaml
├── nginx-pvc.yaml
└── nginx-svc.yaml
```

`nginx-pvc.yaml`
```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-longhorn-pvc
  namespace: default
spec:
  storageClassName: longhorn
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

`nginx-deployment.yaml`
```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx-longhorn
  namespace: default
  labels:
    app: nginx-longhorn
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx-longhorn
  template:
    metadata:
      labels:
        app: nginx-longhorn
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          persistentVolumeClaim:
            claimName: nginx-longhorn-pvc
```

`nginx-svc.yaml`
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-longhorn
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx-longhorn
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

`nginx-ingressroute.yaml`
```yaml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-longhorn
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nginx-longhorn.local.simonlab.xyz`)
      kind: Rule
      services:
        - name: nginx-longhorn
          port: 80
      middlewares:
        - name: default-headers
          namespace: default
  tls:
    secretName: local-simonlab-xyz-tls
```

Apply the whole folder at once:
```shell
kubectl apply -f sample/
```

Verify the PVC was dynamically provisioned by Longhorn:
```shell
kubectl get pvc -n default nginx-longhorn-pvc
```
You should see:
```shell
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nginx-longhorn-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   2Gi        RWO            longhorn       10s
```

Check the PV was created with 3 replicas (the `longhorn.io/volume-revision-count` annotation / the UI's volume view):
```shell
kubectl get pv
kubectl describe pv $(kubectl get pvc nginx-longhorn-pvc -o jsonpath='{.spec.volumeName}') | grep -i longhorn
```
In the Longhorn UI → **Volume**, you will see `pvc-xxxx` with **3/3 healthy replicas** spread across 3 different nodes.

Hit the demo over the ingress:
```
https://nginx-longhorn.local.simonlab.xyz
```
You should see the default nginx welcome page.

### Prove persistence
---
Write a file into the mounted volume, then delete the pod and confirm the file survives (a new pod reattaches the same Longhorn PVC):
```shell
# Get the pod name
POD=$(kubectl get pod -l app=nginx-longhorn -o jsonpath='{.items[0].metadata.name}')

# Write a marker file into the Longhorn-backed volume
kubectl exec "$POD" -- sh -c 'echo "hello from $(date)" > /usr/share/nginx/html/index.html'

# Fetch it through the ingress (or via port-forward)
curl -k https://nginx-longhorn.local.simonlab.xyz
# -> hello from <timestamp>

# Force a pod recreation — the Recreate strategy guarantees a fresh pod
kubectl delete pod "$POD"

# Wait for the new pod
kubectl get pod -l app=nginx-longhorn -w

# The file is still there (Longhorn replicated it)
curl -k https://nginx-longhorn.local.simonlab.xyz
# -> hello from <the original timestamp>
```
> Try deleting the pod multiple times — Longhorn reattaches the same volume with its 3 replicas intact. To really stress it, you can also `kubectl cordon` the node the pod lands on, delete the pod, and watch Longhorn fail over to a replica on another node.

Clean up the demo when done:
```shell
# Capture the underlying volume name BEFORE deleting the PVC (you need it below)
VOLUME=$(kubectl get pvc nginx-longhorn-pvc -o jsonpath='{.spec.volumeName}')

kubectl delete -f sample/
```
> ⚠️ **`reclaimPolicy: Retain` means the PV and the actual data are KEPT**, not deleted, when the PVC is removed — you'll see the PV go to `Released` and the Longhorn volume stick around as `detached`. This is intentional (it protects you from accidental data loss), but it also means **the 2Gi of disk space is NOT freed** until you clean up both objects manually:
> ```shell
> # 1. Delete the PV
> kubectl delete pv "$VOLUME"
>
> # 2. Delete the underlying Longhorn volume — this is what actually frees the disk space
> kubectl -n longhorn-system delete volumes.longhorn.io "$VOLUME"
>
> # Verify both are gone
> kubectl get pv
> kubectl -n longhorn-system get volumes.longhorn.io
> ```
> If you forgot to capture `$VOLUME` before deleting the PVC, find it from the retained PV instead:
> ```shell
> kubectl get pv -o jsonpath='{range .items[?(@.spec.claimRef.name=="nginx-longhorn-pvc")]}{.metadata.name}{"\n"}{end}'
> ```

## K3s-Specific Notes
---
- **Kubelet root directory.** This is the single most important K3s-specific setting and the one most likely to bite you. Some K3s versions/distros use a K3s-specific kubelet root (`/var/lib/rancher/k3s/agent/kubelet`), but **on this cluster (K3s v1.36.2, Ubuntu 26.04), the kubelet uses the standard `/var/lib/kubelet`**. Longhorn's CSI driver needs the correct path to register with the kubelet and mount volumes into pods; get it wrong and the CSI driver silently fails to register on every node (`kubectl get csinode` shows `0` drivers) while the pods themselves still report `Running`. We hit exactly this during setup — see [[#Troubleshooting|Troubleshooting]] for the full fix. Always verify on a node before installing/upgrading:
  ```shell
  # The path with real pod UUID directories is the real kubelet root.
  ls /var/lib/kubelet/pods | head -3
  ls /var/lib/rancher/k3s/agent/kubelet/pods | head -3
  ```
  If a future K3s version changes this path, update `csi.kubeletRootDir` and follow the full recreation procedure in [[#Troubleshooting|Troubleshooting]] (a plain `helm upgrade` is not sufficient — the CSI DaemonSet and 4 sidecar Deployments must be deleted and recreated).
- **Default data path.** The Ansible playbook mounted `/dev/sdb` at `/var/lib/longhorn`, which is Longhorn's default `defaultDataPath`. We set it explicitly in the values for documentation, but it would work out-of-the-box.
- **iSCSI.** K3s does not manage iSCSI — it relies on the host's `iscsid`, which the Ansible playbook enabled. If a node reboots and `iscsid` is not running, Longhorn volumes on that node will fail to attach.
- **No ServiceLB conflict.** K3s's ServiceLB is disabled (MetalLB handles LoadBalancers), so there is no port conflict with Longhorn's services (which are all ClusterIP anyway).
- **Flannel CNI.** K3s's default Flannel CNI is fully supported by Longhorn v1.12.1; no CNI changes needed.

## Troubleshooting
---
### API server / kube-vip VIP unreachable (`connection refused` to `10.100.102.175:6443`)
On clusters with low-RAM control-plane nodes (this cluster's masters have 1.6 GB), the simultaneous startup of Longhorn's DaemonSets (`longhorn-manager`, `engine-image`, `instance-manager`) on the masters can briefly starve the kube-apiserver or kube-vip, causing a momentary `dial tcp 10.100.102.175:6443: connect: connection refused`. It usually self-recovers within 10–30s.

If it does **not** recover, check from a master:
```shell
# Is kube-vip holding the VIP?
ip a | grep 10.100.102.175

# Is k3s-server alive?
sudo systemctl status k3s-server

# Memory pressure?
free -mh
```
Mitigations (in order of effort):
- Wait ~1 min and retry `kubectl get nodes` — usually it comes back.
- Restart k3s on the node currently holding the VIP: `sudo systemctl restart k3s-server` (do ONE master at a time to keep etcd quorum).
- Keep Longhorn's heavier DaemonSets off the control-plane (run Longhorn only on the workers) by setting `nodeSelector` / tolerations in `longhorn-values.yaml` and labelling worker nodes.
- Add RAM to the master nodes (the real fix for a busy homelab).

### Wrong `csi.kubeletRootDir` — CSI driver never registers (PVCs stuck, pods stuck `ContainerCreating`)
This is the single most disruptive misconfiguration on K3s. **Symptoms, in order of appearance:**
1. `kubectl get csinode` shows `DRIVERS: 0` on every node (the real smoking gun):
   ```shell
   kubectl get csinode
   ```
2. Pods using a Longhorn PVC hang in `ContainerCreating` with this event:
   ```shell
   kubectl describe pod <pod-name> | grep -A2 Events
   # Warning  FailedAttachVolume  ...  CSINode <node> does not contain driver driver.longhorn.io
   ```
3. `longhorn-csi-plugin` pods (the DaemonSet) look healthy (`3/3 Running`) — **this is misleading**, the plugin process is up but its registration socket is in a directory the kubelet isn't watching.
4. If you also see `csi-attacher` / `csi-provisioner` / `csi-resizer` / `csi-snapshotter` pods crash-looping with:
   ```
   grpc: addrConn.createTransport failed ... dial unix /csi/csi.sock: connect: no such file or directory
   ```
   these controller-side sidecars are looking for the socket in the OLD path too.

**Root cause:** `csi.kubeletRootDir` in `longhorn-values.yaml` doesn't match the kubelet's actual root directory on this K3s version/OS. Confirm the real path:
```shell
# Whichever path has real pod UUID directories is correct — the other will be empty.
sudo ls /var/lib/kubelet/pods | head -3
sudo ls /var/lib/rancher/k3s/agent/kubelet/pods | head -3
```

**Fix — this requires more than `helm upgrade` alone.** The `longhorn-driver-deployer` skips re-deploying CSI components it thinks are already current, so simply changing the value and running `helm upgrade` is **not enough** — you must force it to recreate them:
```shell
# 1. Fix the value in longhorn-values.yaml, then upgrade
helm upgrade longhorn longhorn/longhorn -n longhorn-system --version 1.12.1 --values longhorn-values.yaml

# 2. Delete the CSI node plugin DaemonSet so the deployer recreates it with the new path
kubectl -n longhorn-system delete daemonset longhorn-csi-plugin

# 3. Delete the 4 controller-side CSI sidecar Deployments (they cache the OLD socket path too)
kubectl -n longhorn-system delete deployment csi-attacher csi-provisioner csi-resizer csi-snapshotter

# 4. Restart the driver-deployer to recreate everything with the corrected path
kubectl -n longhorn-system delete pod -l app=longhorn-driver-deployer

# 5. Wait, then confirm all nodes now show 1 driver
sleep 30
kubectl get csinode
kubectl -n longhorn-system get pods
```
You should see `DRIVERS: 1` on every node, and all `csi-*` / `longhorn-csi-plugin` pods `Running` with `0` restarts.

**Clean up anything created while the driver was broken.** PVCs/pods created before the fix may be stuck with stale `VolumeAttachment`s, `Released` PVs, or orphaned Longhorn `Volume` CRs:
```shell
# Remove any stale VolumeAttachments left over from failed attach attempts
kubectl get volumeattachment
kubectl delete volumeattachment --all

# If old PVs are stuck Released with a finalizer, clear the finalizer then delete
kubectl patch pv <pv-name> -p '{"metadata":{"finalizers":null}}'
kubectl delete pv <pv-name>

# If the Longhorn Volume CR itself won't delete normally, delete it directly
kubectl -n longhorn-system delete volumes.longhorn.io <volume-name>
```
Then re-apply your workload (e.g. `kubectl apply -f sample/`) — it should attach and become `Running` within seconds.

### `longhorn-manager` pods `CrashLoopBackOff`
Check the logs:
```shell
kubectl -n longhorn-system logs -l app=longhorn-manager --tail=50
```
Common causes:
- `iscsid` not running on the node → `sudo systemctl start iscsid` on the offending node
- `/var/lib/longhorn` not mounted → `mount /dev/sdb /var/lib/longhorn` (or reboot the node; fstab should remount it)
- Out of disk → `df -h /var/lib/longhorn` on the node

### Certificate stuck `False`
```shell
kubectl describe certificate local-simonlab-xyz -n longhorn-system
kubectl get order,challenge -A | grep simonlab
kubectl logs -n cert-manager -l app.kubernetes.io/name=cert-manager --tail=50
```
First, check for the **Let's Encrypt rate limit** (the most common cause if you already set up Traefik/cert-manager on this domain):
```shell
kubectl describe certificate local-simonlab-xyz -n longhorn-system | grep "rateLimited\|retry after"
```
If you see `429 ... too many certificates (5) already issued for this exact set of identifiers in the last 168h`, you've hit the 5-certs-per-week limit. **Use the copy-secret approach** (Approach A in the TLS section above) instead of issuing a new cert:
```shell
kubectl delete certificate local-simonlab-xyz -n longhorn-system   # stop retrying
kubectl get secret local-simonlab-xyz-tls -n default -o yaml \
  | sed 's/namespace: default/namespace: longhorn-system/' \
  | kubectl apply -f -
```

If it is NOT the rate limit, the next most common cause is a stale/invalid Cloudflare API token in `cloudflare-token-secret` (in `cert-manager` ns). Regenerate a Cloudflare API token with `Zone:DNS:Edit` for `simonlab.xyz` and update the secret:
```shell
kubectl -n cert-manager create secret generic cloudflare-token-secret \
  --from-literal=cloudflare-token='YOUR_NEW_TOKEN' --dry-run=client -o yaml | kubectl apply -f -
```

### UI returns `404` or `503`
- `404` from Traefik → the `IngressRoute` host doesn't match, or DNS points at the wrong IP. Confirm `dig longhorn-ui.local.simonlab.xyz` resolves to `10.100.102.181` (the Traefik MetalLB IP).
- `503` from Traefik → the `longhorn-frontend` Service has no ready endpoints:
  ```shell
  kubectl -n longhorn-system get endpoints longhorn-frontend
  kubectl -n longhorn-system get pods -l app=longhorn-ui
  ```

### Basic auth prompts but then `401`
The `basic-auth` secret's `auth` key does not match what you typed. Re-create it:
```shell
kubectl -n longhorn-system delete secret basic-auth
kubectl -n longhorn-system create secret generic basic-auth \
  --from-file=auth=<(htpasswd -nb simonj 'YOUR_PASSWORD_HERE')
```
The key MUST be named `auth` (Traefik's basicAuth middleware looks for exactly that key).

### PVC stuck `Pending`
```shell
kubectl describe pvc <name>
kubectl -n longhorn-system get pods -l app=longhorn-csi-plugin
```
If the CSI plugin is fine, check that the Longhorn nodes are schedulable and have space:
```shell
kubectl -n longhorn-system get nodes.longhorn.io -o wide
```
In the UI → **Node**, each node should show a green disk with >25% free space.

## Uninstall (optional)
---
> ⚠️ Uninstalling Longhorn deletes all Longhorn volumes. Back up any data first.

Follow the official uninstall steps — Longhorn requires deleting all volumes and backups before the chart can be cleanly removed: <https://longhorn.io/docs/1.12.1/deploy/uninstall/>

```shell
# 1. Delete all workloads using Longhorn PVCs
kubectl delete -f sample/   # and any other workloads

# 2. Delete all remaining PVCs backed by Longhorn
kubectl delete pvc -A -l storageclass=longhorn 2>/dev/null || true
# Or delete by storage class:
kubectl get pvc -A -o json | jq -r '.items[] | select(.spec.storageClassName=="longhorn") | .metadata.namespace + "/" + .metadata.name' | xargs -r -n1 kubectl delete pvc -n
```
> ⚠️ **`reclaimPolicy: Retain` means deleting the PVCs above does NOT delete the PVs or the underlying Longhorn volumes/data.** They'll be left behind as `Released` PVs and `detached` Longhorn volumes. Clean those up too, or the Helm/CRD uninstall below may hang waiting on them:
> ```shell
> # List anything left over
> kubectl get pv
> kubectl -n longhorn-system get volumes.longhorn.io
>
> # Delete every Longhorn PV and its underlying volume
> for pv in $(kubectl get pv -o jsonpath='{range .items[?(@.spec.csi.driver=="driver.longhorn.io")]}{.metadata.name}{"\n"}{end}'); do
>   kubectl -n longhorn-system delete volumes.longhorn.io "$pv" --ignore-not-found
>   kubectl delete pv "$pv" --ignore-not-found
> done
> ```
```shell
# 3. Uninstall the Helm release
helm uninstall longhorn -n longhorn-system

# 4. Clean up CRDs (optional — only if you will not reinstall)
kubectl delete -f https://raw.githubusercontent.com/longhorn/longhorn/v1.12.1/deploy/longhorn.yaml

# 5. Remove the namespace
kubectl delete namespace longhorn-system

# 6. Remove the /var/lib/longhorn data on each node (optional, irreversible)
#    ssh kube-master-1 'sudo rm -rf /var/lib/longhorn/*'
```
