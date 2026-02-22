# llm_infra

> **Automated GitOps deployment of an LLM infrastructure on k3s (Ubuntu)**
>
> Ansible provisions a k3s cluster and bootstraps Argo CD.  
> Argo CD then uses the **App of Apps** pattern to deploy and keep in sync:
> Apache APISIX, vLLM, Prometheus/Grafana, **Loki/Promtail (log storage)**, and Headlamp.

---

## Table of Contents

1. [Architecture overview](#architecture-overview)
2. [Repository layout](#repository-layout)
3. [Prerequisites](#prerequisites)
4. [Quickstart](#quickstart)
   - [1. Adapt the inventory](#1-adapt-the-inventory)
   - [2. Set SSH access](#2-set-ssh-access)
   - [3. Run the playbook](#3-run-the-playbook)
5. [Accessing services after deployment](#accessing-services-after-deployment)
   - [Argo CD](#argo-cd)
   - [Grafana](#grafana)
   - [Logs – Loki + Promtail](#logs--loki--promtail)
   - [vLLM (OpenAI-compatible API)](#vllm-openai-compatible-api)
   - [Apache APISIX](#apache-apisix)
   - [Headlamp](#headlamp)
6. [How Argo CD keeps everything in sync](#how-argo-cd-keeps-everything-in-sync)
7. [Customising Helm values](#customising-helm-values)
8. [Enabling an Ingress later](#enabling-an-ingress-later)
9. [Variables reference](#variables-reference)
10. [Troubleshooting](#troubleshooting)

---

## Architecture overview

```
Control machine (you)
       │
       │  ansible-playbook
       ▼
┌──────────────────────────────────────────────────┐
│                 k3s cluster (Ubuntu)             │
│                                                  │
│  ┌──────────────┐   ┌──────────────┐  ┌──────┐  │
│  │  k3s-master  │   │ k3s-worker1  │  │  …   │  │
│  │              │   │              │  │      │  │
│  │  Argo CD     │   │  workloads   │  │      │  │
│  └──────────────┘   └──────────────┘  └──────┘  │
└──────────────────────────────────────────────────┘
       │
       │  GitOps sync (App of Apps)
       ▼
GitHub: tmattern/llm_infra
  deploy/argocd/apps/
    ├── apisix.yaml       → Apache APISIX (API Gateway)
    ├── vllm.yaml         → vLLM (LLM inference)
    ├── monitoring.yaml   → Prometheus + Grafana
    ├── logging.yaml      → Loki + Promtail (pod log storage)
    └── headlamp.yaml     → Headlamp (Kubernetes UI)
```

Services are exposed via **NodePort** (no LoadBalancer required).

---

## Repository layout

```
llm_infra/
├── ansible/
│   ├── inventories/prod/
│   │   ├── hosts.ini               # node IPs / hostnames (edit me)
│   │   └── group_vars/all.yml      # all variables & version pins
│   ├── playbooks/
│   │   └── site.yml                # main entry-point
│   └── roles/
│       ├── k3s_cluster/            # installs k3s server + agents
│       ├── argocd/                 # installs Argo CD via upstream manifest
│       └── argocd_app/             # applies root App-of-Apps
├── deploy/
│   ├── argocd/
│   │   ├── root-app/
│   │   │   └── application.yaml    # root Application (reference copy)
│   │   └── apps/
│   │       ├── apisix.yaml
│   │       ├── vllm.yaml
│   │       ├── monitoring.yaml
│   │       ├── logging.yaml
│   │       └── headlamp.yaml
│   └── helm-values/
│       ├── apisix/values.yaml
│       ├── vllm/values.yaml
│       ├── monitoring/values.yaml  ← also wires Loki datasource into Grafana
│       ├── logging/values.yaml     ← Loki + Promtail configuration
│       └── headlamp/values.yaml
└── README.md
```

---

## Prerequisites

| Tool | Version | Where |
|------|---------|--------|
| Ansible | ≥ 2.14 | control machine |
| Python | ≥ 3.9 | control machine |
| `kubectl` | ≥ 1.28 | control machine (optional, for post-install ops) |
| Ubuntu nodes | 22.04 LTS | k3s master + workers |
| SSH key-pair | any | control machine → nodes |

Install Python dependencies on the control machine:

```bash
pip install ansible kubernetes
```

---

## Quickstart

### 1. Adapt the inventory

Edit `ansible/inventories/prod/hosts.ini` and replace the example IP addresses:

```ini
[master]
k3s-master  ansible_host=<MASTER_IP>  ansible_user=ubuntu

[workers]
k3s-worker1 ansible_host=<WORKER1_IP>  ansible_user=ubuntu
k3s-worker2 ansible_host=<WORKER2_IP>  ansible_user=ubuntu
```

> You can use IP addresses or resolvable hostnames.  
> Add or remove worker lines as needed.

### 2. Set SSH access

Option A – set the key path in `ansible/inventories/prod/hosts.ini`:

```ini
[k3s_cluster:vars]
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Option B – pass it at runtime:

```bash
ansible-playbook ... --private-key ~/.ssh/id_rsa
```

Option C – use an SSH agent:

```bash
eval $(ssh-agent) && ssh-add ~/.ssh/id_rsa
```

### 3. Run the playbook

```bash
ansible-playbook \
  -i ansible/inventories/prod/hosts.ini \
  ansible/playbooks/site.yml
```

The playbook will:

1. Install k3s on the master node.  
2. Join worker nodes to the cluster.  
3. Fetch `~/.kube/k3s-prod.yaml` to your local machine.  
4. Install Argo CD in the `argocd` namespace.  
5. Patch the Argo CD server `Service` to `NodePort` (ports 30080/30443).  
6. Apply the root **App of Apps** `Application` resource.

From this point on, Argo CD will continuously sync all child applications.

---

## Accessing services after deployment

All services default to **NodePort** – use any node's IP address.

### Argo CD

```bash
# NodePort – open in browser
https://<ANY_NODE_IP>:30443

# Or use port-forward if NodePort is blocked:
kubectl port-forward svc/argocd-server -n argocd 8080:443 \
  --kubeconfig ~/.kube/k3s-prod.yaml
# then: https://localhost:8080
```

**Get the initial admin password:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" \
  --kubeconfig ~/.kube/k3s-prod.yaml | base64 -d; echo
```

Login:

```bash
argocd login <ANY_NODE_IP>:30443 \
  --username admin \
  --password <PASSWORD_FROM_ABOVE> \
  --insecure
```

> ⚠️ Change the admin password after first login:  
> `argocd account update-password`

### Grafana

```
http://<ANY_NODE_IP>:30300
```

Default credentials: `admin` / `changeme`  
(Change `grafana.adminPassword` in `deploy/helm-values/monitoring/values.yaml`)

### Logs – Loki + Promtail

Loki is the log aggregation backend; Promtail is a DaemonSet that automatically
collects logs from every pod on every node and ships them to Loki.

**Architecture:**

```
[Pod stdout/stderr]
      │
      ▼  (tail /var/log/pods/**/*.log)
  Promtail (DaemonSet – one pod per node)
      │
      ▼  (HTTP push)
  Loki (ClusterIP – internal only)
      │
      ▼  (Grafana datasource: http://loki-stack.logging.svc.cluster.local:3100)
  Grafana → Explore → Loki
```

**No external port is required** – Loki is queried by Grafana via its internal
ClusterIP.  If you need direct CLI access, use port-forward:

```bash
kubectl port-forward svc/loki-stack -n logging 3100:3100 \
  --kubeconfig ~/.kube/k3s-prod.yaml
# then query via LogCLI:
logcli query '{namespace="vllm"}' --addr http://localhost:3100
```

**Query logs in Grafana:**

1. Open Grafana → **Explore** → select the **Loki** datasource.
2. Use the **Log browser** or type a [LogQL](https://grafana.com/docs/loki/latest/query/) query:

```logql
# All logs from the vllm namespace
{namespace="vllm"}

# Filter for errors in any pod
{namespace="vllm"} |= "error"

# APISIX access log lines containing a specific route
{namespace="apisix"} |= "upstream_uri"

# Count log lines per minute for a pod
count_over_time({pod=~"vllm-.*"}[1m])
```

**Key labels automatically added by Promtail:**

| Label | Example |
|-------|---------|
| `namespace` | `vllm` |
| `pod` | `vllm-abc12` |
| `container` | `vllm` |
| `node_name` | `k3s-worker1` |
| `app` | `vllm` |

**Retention:** logs are kept for 30 days by default.  
Change `loki.config.limits_config.retention_period` in
`deploy/helm-values/logging/values.yaml`.

### vLLM (OpenAI-compatible API)

```bash
curl http://<ANY_NODE_IP>:30800/v1/models
```

Set the model in `deploy/helm-values/vllm/values.yaml` → `model:`.  
For gated Hugging Face models create a Kubernetes Secret:

```bash
kubectl create secret generic vllm-hf-token \
  --from-literal=token=<HF_TOKEN> \
  -n vllm \
  --kubeconfig ~/.kube/k3s-prod.yaml
```

Then reference it in `deploy/helm-values/vllm/values.yaml`:

```yaml
huggingFaceTokenSecret:
  name: vllm-hf-token
  key: token
```

### Apache APISIX

```
http://<ANY_NODE_IP>:30090   # gateway
http://<ANY_NODE_IP>:30091   # dashboard
```

### Headlamp

```
http://<ANY_NODE_IP>:30190
```

---

## How Argo CD keeps everything in sync

```
Argo CD
  └── root-app  (watches deploy/argocd/apps/)
        ├── apisix      (Helm chart + values from this repo)
        ├── vllm        (Helm chart + values from this repo)
        ├── monitoring  (kube-prometheus-stack + values)
        ├── logging     (loki-stack: Loki + Promtail)
        └── headlamp    (Helm chart + values)
```

Every `git push` to `main` is automatically picked up by Argo CD
(`syncPolicy.automated`).  To trigger a manual sync:

```bash
argocd app sync root-app --kubeconfig ~/.kube/k3s-prod.yaml
# or sync a specific child app:
argocd app sync monitoring
```

---

## Customising Helm values

All per-application Helm value overrides live under `deploy/helm-values/`.  
Edit the relevant `values.yaml`, commit, and push – Argo CD will reconcile.

```
deploy/helm-values/
├── apisix/values.yaml
├── vllm/values.yaml       ← model name, GPU limits, …
├── monitoring/values.yaml ← Grafana password, retention, Loki datasource wiring
├── logging/values.yaml    ← Loki retention, storage size, Promtail scrape config
└── headlamp/values.yaml
```

---

## Enabling an Ingress later

If you add an Ingress controller (e.g. Nginx, Traefik, or APISIX Ingress):

1. Remove `--disable traefik` from `k3s_server_extra_args` in  
   `ansible/inventories/prod/group_vars/all.yml` (or install Nginx separately).
2. Change `service.type` from `NodePort` to `ClusterIP` in each `values.yaml`.
3. Add an `Ingress` resource (or `HTTPRoute` for Gateway API) in the relevant  
   `deploy/helm-values/<app>/` directory.
4. Commit and push – Argo CD will apply the changes.

---

## Variables reference

| Variable | File | Default | Description |
|----------|------|---------|-------------|
| `k3s_version` | `group_vars/all.yml` | `v1.29.3+k3s1` | k3s release to install |
| `k3s_server_extra_args` | `group_vars/all.yml` | `--disable traefik --disable servicelb` | Extra k3s server flags |
| `kubeconfig_local_path` | `group_vars/all.yml` | `~/.kube/k3s-prod.yaml` | Where kubeconfig is saved locally |
| `argocd_version` | `group_vars/all.yml` | `v2.10.3` | Argo CD release to install |
| `argocd_namespace` | `group_vars/all.yml` | `argocd` | Namespace for Argo CD |
| `argocd_server_nodeport` | `group_vars/all.yml` | `30080` | HTTP NodePort for Argo CD server |
| `argocd_server_nodeport_https` | `group_vars/all.yml` | `30443` | HTTPS NodePort for Argo CD server |
| `argocd_repo_url` | `group_vars/all.yml` | `https://github.com/tmattern/llm_infra.git` | Git repo Argo CD watches |
| `argocd_apps_path` | `group_vars/all.yml` | `deploy/argocd/apps` | Path of child Application manifests |

---

## Troubleshooting

**k3s install fails – curl not found**

```bash
sudo apt-get install -y curl
```

**Workers cannot join the cluster**

- Ensure port `6443` is open between workers and master (firewall / security group).
- Verify the master's `ansible_host` is reachable from workers.

**Argo CD pods not starting**

```bash
kubectl get pods -n argocd --kubeconfig ~/.kube/k3s-prod.yaml
kubectl describe pod <POD> -n argocd --kubeconfig ~/.kube/k3s-prod.yaml
```

**Argo CD root-app shows `Unknown` or `OutOfSync`**

Argo CD may take a minute after startup to clone the repo.  
Run `argocd app sync root-app` or wait for the next auto-sync.

**vLLM OOMKilled**

Increase `resources.limits.memory` in `deploy/helm-values/vllm/values.yaml`  
or assign GPU resources and enable GPU scheduling (`nvidia.com/gpu`).

**Loki pods not starting / PVC Pending**

k3s ships with the `local-path` provisioner which creates PVCs automatically.
If the PVC stays `Pending`, check:

```bash
kubectl get pvc -n logging --kubeconfig ~/.kube/k3s-prod.yaml
kubectl describe pvc loki-stack -n logging --kubeconfig ~/.kube/k3s-prod.yaml
```

**No logs visible in Grafana Explore**

1. Confirm Promtail is running on every node:
   ```bash
   kubectl get pods -n logging -o wide --kubeconfig ~/.kube/k3s-prod.yaml
   ```
2. Check Promtail logs for connection errors:
   ```bash
   kubectl logs -n logging -l app=promtail --tail=50 \
     --kubeconfig ~/.kube/k3s-prod.yaml
   ```
3. Verify the Loki datasource URL in Grafana → **Configuration** → **Data Sources**.  
   It should be `http://loki-stack.logging.svc.cluster.local:3100`.
