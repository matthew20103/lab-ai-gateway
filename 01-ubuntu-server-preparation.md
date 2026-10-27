# 1. Ubuntu Server Preparation for AI Gateway (Prisma AIRS / Portkey AIRS Gateway)

**Core lab** — required. Prerequisite: [00-prerequisites.md](00-prerequisites.md) completed (your own SCM account, AI Gateway resource, and `values.yaml` downloaded).

This guide prepares a **fresh Ubuntu server** (22.04 LTS or 24.04 LTS) to host the AI Gateway. The gateway is distributed as a **Kubernetes Helm chart** (`airs-gw-helm`), so a single-node Kubernetes runtime is required even though there's only one physical machine. We use **k3s** — a lightweight, single-binary Kubernetes distribution well suited to a single bare-metal server — and **HAProxy** as the open-source load balancer in front of the gateway pods.

> Complete this guide fully before moving on to [02-ai-gateway-deployment.md](./02-ai-gateway-deployment.md).

## 1.1 Architecture Overview

```
Internet/LAN clients
        │
        ▼  :80 / :443
   HAProxy (native systemd service on Ubuntu host)
        │  health-checks GET /v1/health
        ▼  127.0.0.1:<NodePort>  (k8s auto-assigns this; discovered in doc 2 §2.4)
   k3s Service (NodePort) — kube-proxy round-robins across pod endpoints
        │                              │
        ▼                              ▼
   Gateway Pod #1 (:8787)        Gateway Pod #2 (:8787)
        │                              │
        └──────────────┬───────────────┘
                        ▼
              Built-in Redis pod (cache store)
```

- **k3s** is installed *without* its bundled Traefik ingress controller and ServiceLB, since HAProxy takes over that role on the host.
- **HAProxy** load-balances external traffic at the host edge; Kubernetes' own `kube-proxy` load-balances across the 2 gateway pod replicas behind the NodePort. Both layers are intentional — see [02-ai-gateway-deployment.md](./02-ai-gateway-deployment.md) for why.

## 1.2 Sizing Recommendations

The chart (with defaults) runs: 2× gateway pods + 1× built-in Redis pod. (Dataservice, MinIO, and Milvus are disabled by default in the supplied `values.yaml` and are not covered here.)

| Resource | Minimum | Recommended |
|---|---|---|
| vCPU | 2 | 4 |
| RAM | 4 GB | 8 GB |
| Disk | 40 GB SSD | 80 GB+ SSD |
| OS | Ubuntu 22.04 LTS or 24.04 LTS (fresh install) | same |
| Network | Static private IP, outbound internet access (to pull images and reach upstream LLM providers) | same |

## 1.3 Initial OS Update

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget gnupg2 ca-certificates apt-transport-https software-properties-common jq
sudo reboot
```

Reconnect after reboot before continuing.

## 1.4 Hostname & Time Sync

k3s uses the hostname as the node name; keep it short and DNS-safe. Accurate time is required for TLS certificate validation and log correlation.

```bash
sudo hostnamectl set-hostname ai-gateway-01
sudo apt install -y chrony
sudo systemctl enable --now chrony
timedatectl
```

## 1.5 Disable Swap

Kubernetes (and k3s by default) expects swap to be off for predictable scheduling/memory behavior.

```bash
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
free -h   # confirm Swap: 0
```

## 1.6 Kernel Modules & sysctl for Kubernetes Networking

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## 1.7 Firewall (UFW)

Since this is a single-node cluster, only the host's own edge ports need to be open externally. Kubernetes-internal ports stay bound to loopback/cluster-internal interfaces and don't need to be exposed.

```bash
sudo apt install -y ufw

sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow OpenSSH          # 22/tcp — do this BEFORE enabling, or you'll lock yourself out
sudo ufw allow 80/tcp           # HAProxy HTTP
sudo ufw allow 443/tcp          # HAProxy HTTPS (optional TLS, see doc 2)

sudo ufw enable
sudo ufw status verbose
```

> Do **not** open 6443 (k3s API server), 10250 (kubelet), or the NodePort range (30000–32767) to the internet — those stay local-only. Only HAProxy's 80/443 are internet/LAN facing.

## 1.8 Install k3s (Single Node, No Bundled Traefik/ServiceLB)

We disable k3s's built-in `traefik` ingress controller and `servicelb` load balancer because HAProxy will own that role on the host, avoiding two overlapping load-balancer layers.

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --disable traefik \
  --disable servicelb \
  --write-kubeconfig-mode 644

sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes -o wide
```

Expect the single node listed with `STATUS: Ready`.

> **Reproducibility note (recommended for a classroom/lab setting):** the command above installs whatever k3s release is current the day you run it. If everyone in a cohort runs this on different days, people can end up on different k3s versions with subtly different behavior. To pin a specific version instead, set `INSTALL_K3S_CHANNEL` (e.g. `INSTALL_K3S_CHANNEL=v1.30` for the latest patch on that minor line) or `INSTALL_K3S_VERSION` (an exact release tag) before the `sh -s -` arguments — see the [k3s install script docs](https://docs.k3s.io/upgrades/manual) for current channel/version names.

## 1.9 kubectl Access for Your Non-Root User

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config

echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc

# Use the standalone kubectl binary that k3s installs, or alias k3s kubectl:
sudo ln -sf /usr/local/bin/k3s /usr/local/bin/kubectl 2>/dev/null || true
kubectl get nodes
```

If `kubectl` isn't on PATH as its own binary, k3s also ships `k3s kubectl <args>` — either works identically for the rest of these guides (examples use plain `kubectl`).

## 1.10 Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh

helm version
```

## 1.11 Install HAProxy

```bash
sudo apt install -y haproxy
haproxy -v
sudo systemctl enable haproxy
```

Leave the default config in place for now — it will be replaced with the AI Gateway-specific configuration in [02-ai-gateway-deployment.md](./02-ai-gateway-deployment.md).

## 1.12 Stage `values.yaml` Securely

Copy the SCM-generated `values.yaml` to the server (e.g. `scp` from your workstation, as your normal user — not with `sudo`). It contains **live secrets**: an image registry password and a `PORTKEY_CLIENT_AUTH` client-auth token. Treat it like any other credential file.

```bash
mkdir -p ~/ai-gateway
# scp values.yaml <user>@<server>:~/ai-gateway/values.yaml   (run from your workstation)

sudo chown $(id -u):$(id -g) ~/ai-gateway/values.yaml
chmod 600 ~/ai-gateway/values.yaml
ls -l ~/ai-gateway/values.yaml   # confirm -rw------- , owned by you
cat ~/ai-gateway/values.yaml     # confirm it's readable without sudo
```

The `chown` guards against the file landing as `root`-owned (which happens if any step in getting it onto the server — `scp`, an editor, or the directory creation — ran under `sudo`); without correct ownership, `chmod 600` alone leaves it unreadable to your own user, and Helm commands run as that user would then fail to read it.

**Do not** commit this file to a public git repository, paste it into chat tools, or leave it world-readable.

## 1.13 Preparation Checklist

- [ ] OS updated and rebooted
- [ ] Hostname set, time sync active (`timedatectl` shows `System clock synchronized: yes`)
- [ ] Swap disabled (`free -h` shows 0 swap)
- [ ] Kernel modules (`overlay`, `br_netfilter`) and sysctl settings applied
- [ ] UFW enabled with only 22/80/443 open
- [ ] k3s installed and node shows `Ready` (Traefik/ServiceLB disabled)
- [ ] `kubectl get nodes` works as your normal user (no `sudo`)
- [ ] `helm version` works
- [ ] HAProxy installed and enabled (not yet configured)
- [ ] `values.yaml` staged on the server with `600` permissions

Once every box is checked, continue to [02-ai-gateway-deployment.md](./02-ai-gateway-deployment.md).
