# HomeLab-Services

My complete homelab documentation — Docker & Kubernetes templates, notes, setups, and configurations covering infrastructure, applications, networking, and more.

---

**Hey, there!**

This repository holds all my homelab documentation files. Here you'll find notes, setups, and configurations for infrastructure, applications, networking, and more.

I created them as a free resource for my own reference and for anyone to adapt to their own setup. Each service lives in its own folder with a dedicated `Readme.md` walking through the install and configuration step by step.

> ⚠️ **Be aware**, products and versions change over time. I do my best to keep up with the latest changes and releases, but please understand that this won't always be the case. Always verify against the upstream documentation before applying anything in production.

## Documentation
---
The repository is organised by platform, then by service. Each service folder contains its manifests, values files, and a `Readme.md` guide.

```text
HomeLab-Services/
└── Kubernetes/
    ├── Traefik+Cert-Manager/   # Custom Traefik ingress + cert-manager (Let's Encrypt, Cloudflare DNS-01)
    └── longhorn/               # Longhorn distributed block storage
```

| Guide | Description |
|---|---|
| [Kubernetes / Traefik + Cert-Manager](Kubernetes/Traefik+Cert-Manager/Readme.md) | Installs a custom Traefik ingress controller (HTTP/3, MetalLB IP, dashboard with basic-auth) and cert-manager issuing wildcard Let's Encrypt certificates via Cloudflare DNS-01. Includes a sample nginx workload. |
| [Kubernetes / Longhorn](Kubernetes/longhorn/Readme.md) | Installs Longhorn distributed block storage on a K3s cluster, sets it as the default StorageClass, exposes the UI through Traefik + cert-manager with basic-auth, and ships a sample PVC workload to prove persistence. |

## Roadmap
---
This repository will grow over time. Planned content includes:

- **Docker** — Compose templates and application stacks
- **Kubernetes** — additional services (monitoring, DNS, GitOps, identity, etc.)
- **Networking** — reverse proxies, DNS, and VLAN/segmentation notes
- **Applications** — service-specific configurations and backups

> Items above are planned — not yet present in the repo. The index in [Documentation](#documentation) reflects what is actually checked in today.

## Conventions
---
- **One folder per service.** Each service lives in its own directory with its manifests, Helm values, and a `Readme.md` guide.
- **Secrets are never committed.** Real secret manifests are gitignored; each ships an `*-example.yaml` template with placeholders instead. Copy the example, fill in your real values, and apply it locally.
- **Helm values are checked in** so installs are reproducible — `helm install ... --values <service>-values.yaml`.

## Related Resources
---
- [Ansible](https://github.com/SimonJan2/Ansible) — my Ansible playbook for provisioning the K3s cluster referenced by the Kubernetes guides
- [techno-tim/launchpad](https://github.com/techno-tim/launchpad) — inspiration and original resources for the Traefik + cert-manager setup
- [k3s-etcd Ansible — techno-tim](https://technotim.live/posts/k3s-etcd-ansible/) — walkthrough for installing a K3s HA cluster with etcd

## Contribution
---
As this is my personal homelab documentation, I don't accept contributions. But feel free to fork this repository and use it as a starting point for your own documentation.
