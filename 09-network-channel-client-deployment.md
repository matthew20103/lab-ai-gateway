# 9. Deploying the AI Red Teaming Network Channel Client (NCC)

**Advanced / optional module** — do this only after completing the core lab ([01-03](00-prerequisites.md#03-lab-map)). Independent of docs 04-08; does not require the chart force-upgrade in [doc 07](07-force-upgrading-gateway-image.md).

This guide documents installing Palo Alto's **AI Red Teaming Network Channel Client (NCC)** — the agent that lets the AI Red Teaming SaaS service reach this lab's privately-hosted AI Gateway as a scan target — on the **same Ubuntu server already running the AI Gateway's k3s cluster**, without disrupting the gateway or its pod networking.

## 9.1 What the NCC Actually Does (and Why It's Safe to Co-Locate)

Per Palo Alto's *AI Red Teaming* guide ("Network Channels" chapter): the NCC opens an **outbound-only** WebSocket connection from this host to Palo Alto's server. It never listens for inbound connections itself — the SaaS side forwards red-teaming requests to the client over that existing outbound connection, which the client then relays to the target (here, the AI Gateway). Functionally it behaves like a reverse tunnel client (comparable to a Cloudflare Tunnel or Tailscale Funnel agent), not a proxy or network appliance.

Three deployment methods are documented upstream: **Kubernetes (Helm)**, **Docker Compose**, and **Standalone Binary**. All three only require *outbound* access — nothing listens for inbound traffic, so none of them need a UFW change on this host (already `default allow outgoing`, [01-ubuntu-server-preparation.md §1.7](01-ubuntu-server-preparation.md#17-firewall-ufw)).

**This lab uses the Helm method**, installed into its **own namespace** in the existing k3s cluster — deliberately avoiding Docker Compose. Rationale:

- **Kubernetes (Helm)** rides on k3s's existing CNI (Flannel) and containerd — the same networking stack the AI Gateway itself already uses. No new network stack is introduced.
- **Docker Compose** requires installing a separate Docker Engine on the host. This lab's server does not run Docker anywhere else — only k3s with its embedded containerd ([01-ubuntu-server-preparation.md §1.6](01-ubuntu-server-preparation.md#16-install-k3s)). Installing Docker fresh alongside an already-running k3s is a known source of pod-networking breakage: Docker's install creates its own `docker0` bridge and `DOCKER`/`DOCKER-USER` iptables chains, which have historically conflicted with Flannel/kube-proxy's `FORWARD`-chain rules on the same host. There's no other reason to need Docker here, so this risk isn't worth taking on.
- **Standalone Binary** is equally network-stack-safe (just a plain process making outbound HTTPS/WSS calls) and is a reasonable alternative if you'd rather keep the NCC entirely outside Kubernetes — see §9.7 for the equivalent commands.

Resource footprint for the Helm chart is small: **100m CPU / 128Mi memory requested, 200m CPU / 256Mi memory limit** — negligible next to the 2-pod gateway deployment.

## 9.2 Prerequisites

| Requirement | Notes |
|---|---|
| Working AI Gateway deployment | Complete docs 01-03 first — you need a reachable target to validate the channel against. |
| Strata Cloud Manager access | Same tenant as your AI Gateway resource ([doc 00 §0.1](00-prerequisites.md#01-create-your-own-strata-cloud-manager-account-and-ai-gateway-resource)), with the **Prisma AIRS AI Red Teaming** license. |
| `kubectl` | Already configured against this cluster from doc 01. |
| Helm CLI 3.x+ | Already installed/used in docs 02/05/07. |
| Outbound network access | To `api.sase.paloaltonetworks.com`, `auth.apps.paloaltonetworks.com`, `registry.ai-red-teaming.paloaltonetworks.com` — already permitted by this host's UFW defaults. |

## 9.3 Step 1 — Create the Network Channel in Strata Cloud Manager

1. Log in to Strata Cloud Manager.
2. Navigate to **AI Security > AI Red Teaming > Network Channels** and **Add Network Channel**.
3. Enter a **Channel Name** (e.g. `ubuntu-lab-gateway`) and optional description.
4. **Add Channel.**

The channel starts in `Draft` status until a client connects to it — that's expected until §9.6.

## 9.4 Step 2 — Create an Isolated Namespace

Install the NCC into its **own namespace**, separate from `airs-gw` — this keeps it independently upgradable/removable (`helm uninstall` or `kubectl delete namespace`) without touching anything the gateway depends on:

```bash
kubectl create namespace network-channel-client
```

## 9.5 Step 3 — Configure Registry Access

Credentials come from the channel detail page in Strata Cloud Manager (click the eye/copy icon next to the generated registry username/password — these are generated once per tenant, same as the `imageCredentials` block already in this lab's `values.yaml` for the gateway's own registry).

```bash
kubectl create secret docker-registry airs-pull-secret \
  --docker-server=registry.ai-red-teaming.paloaltonetworks.com \
  --docker-username='<REGISTRY_USERNAME>' \
  --docker-password='<REGISTRY_PASSWORD>' \
  -n network-channel-client

helm registry login registry.ai-red-teaming.paloaltonetworks.com \
  --username='<REGISTRY_USERNAME>' \
  --password='<REGISTRY_PASSWORD>'
```

## 9.6 Step 4 — Create a Service Account and Install the Client

1. In Strata Cloud Manager, create a **Service Account** (name + description), assign it the **Superuser** role, and copy/save the **Client ID** and **Client Secret** — shown only once.
2. Install the chart into the isolated namespace from §9.4:

```bash
helm install panw-network-client \
  oci://registry.ai-red-teaming.paloaltonetworks.com/<TENANT_REGISTRY_PATH>/red-teaming-onprem/charts/panw-network-client:<VERSION> \
  -n network-channel-client \
  --set config.clientId='<CLIENT_ID>' \
  --set config.clientSecret='<CLIENT_SECRET>' \
  --set config.channelId='<CHANNEL_ID>'
```

Replace `<TENANT_REGISTRY_PATH>` and `<VERSION>` with the exact values shown on the channel's setup instructions in Strata Cloud Manager (these are tenant- and release-specific, not fixed values this lab can hardcode).

## 9.7 Alternative — Standalone Binary (No Kubernetes Footprint at All)

If you'd rather not add anything to the k3s cluster, run the NCC as a plain host process instead (e.g. under a `systemd` unit) — equally safe from a networking standpoint, since it's just a process making outbound HTTPS/WSS calls:

```bash
export REGISTRY_USER='<REGISTRY_USERNAME>'
export REGISTRY_TOKEN='<REGISTRY_TOKEN>'

curl -fLO -u "$REGISTRY_USER:$REGISTRY_TOKEN" \
  "https://registry.ai-red-teaming.paloaltonetworks.com/files/<TENANT_REGISTRY_PATH>/red-teaming-onprem/stable/network-broker-client:<VERSION>-standalone-linux-amd64/airt-nb-client-linux-amd64-<VERSION>.tar.gz"

tar -xzf airt-nb-client-linux-amd64-<VERSION>.tar.gz

CLIENT_ID='<CLIENT_ID>' \
CLIENT_SECRET='<CLIENT_SECRET>' \
CHANNEL_ID='<CHANNEL_ID>' \
./nb-client
```

Wrap this in a `systemd` service if you want it to survive reboots — out of scope for this doc. Don't run both this **and** the Helm deployment against the same channel unless you intend a multi-client, round-robin setup (§9.9).

## 9.8 Step 5 — Verify the Channel Comes Online

```bash
kubectl logs -n network-channel-client -l app.kubernetes.io/name=panw-network-client -f
```

Look for a log line confirming **"Connected to the server"**. Then, in Strata Cloud Manager, either wait for the channel status to flip to `Online`, or click **Validate Channel** to confirm immediately.

## 9.9 Step 6 — Confirm the AI Gateway Is Unaffected

Don't just assume isolation — verify it:

```bash
# Gateway pods still healthy and untouched
kubectl get pods -n airs-gw -o wide

# Gateway still serving real traffic end-to-end (same check as doc 03)
curl -s http://192.168.110.161/v1/health

# NCC is in its own namespace, not mixed into the gateway's
kubectl get pods -n network-channel-client -o wide

# No new pods/resources landed inside the airs-gw namespace
kubectl get all -n airs-gw
```

Expect: `airs-gw` namespace shows exactly what it showed before this module (2 gateway pods + its own dependencies), `/v1/health` still returns successfully, and the NCC pod only exists in `network-channel-client`.

Once a target is added against this channel in Strata Cloud Manager (pointing at `http://192.168.110.161` or the gateway's internal service), a Red Teaming scan request is the real end-to-end test — but the check above already confirms the NCC didn't alter the gateway's existing pods, namespace, or health.

## 9.10 Uninstalling / Rolling Back

Fully reversible without touching the gateway at all:

```bash
helm uninstall panw-network-client -n network-channel-client
kubectl delete namespace network-channel-client
```

## 9.11 Notes

- All clients connected to the same channel are treated as homogeneous and load-balanced round-robin — so running the Helm client on this host and, say, the standalone binary on a second host against the **same channel** is a supported way to get high availability, not a conflict.
- A channel must be validated/`Online` before it can be attached to a target in the AI Red Teaming dashboard.
- This module does not modify `values.yaml`, the `airs-gw` Helm release, or HAProxy config — it is entirely additive, in a separate namespace, using a separate Helm release name.
