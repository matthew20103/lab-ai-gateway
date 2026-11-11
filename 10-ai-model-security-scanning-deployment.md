# 10. Deploying AI Model Security (Model Scanning)

**Advanced / optional module** — do this only after completing the core lab ([01-03](00-prerequisites.md#03-lab-map)). Independent of docs 04-09.

This guide documents installing Palo Alto's **AI Model Security** scanner (part of *AI Supply Chain Security*, Prisma AIRS) on the **same Ubuntu server already running the AI Gateway's k3s cluster and the AI Red Teaming Network Channel Client (NCC, [doc 09](09-network-channel-client-deployment.md))**, without disrupting either.

## 10.1 What AI Model Security Actually Is (and Why Co-Location Is a Different Kind of Question Than Doc 09)

Per Palo Alto's *AI Supply Chain Security* guide: AI Model Security ships as a **Python package** (`model-security-client`) offering a **CLI** (`model-security`) and an **SDK** — not a Kubernetes workload, not a Docker container, not a persistent network listener/tunnel. You invoke it on demand (or from a cron/CI job) to scan a model — from Hugging Face, local disk, or object storage (S3/GCS/Azure/Artifactory/GitLab) — against configured Security Group rules, and it exits.

That makes the risk profile **completely different from doc 09's NCC**:

- **No Kubernetes/CNI/iptables footprint at all.** It never touches k3s, Flannel, Helm, or any namespace — there is nothing here that can "break the Kubernetes network" the way an extra Docker Engine install could (doc 09 §9.1). It's a plain `pip`-installed package running as a regular process on the host.
- **Network-wise it's outbound-only HTTPS**, same class of traffic as the gateway and NCC already make — covered by this host's existing UFW `default allow outgoing` ([01-ubuntu-server-preparation.md §1.7](01-ubuntu-server-preparation.md#17-firewall-ufw)). No new firewall rules needed.
- **The real co-location risk here is resource contention, not networking.** A model scan can be CPU/RAM/disk-intensive — the vendor's own sizing guidance (§10.2) recommends 4+ CPU cores and 4GB+ RAM for a scan, and local-model scans explicitly warn they can consume **up to 4GB of memory** depending on model size. On a single-node host that's also running the 2-pod AI Gateway deployment and the NCC, an uncontrolled large scan competing for the same CPU/RAM could starve those pods rather than anything network-related. §10.5 covers how to bound this.

## 10.2 Prerequisites

| Requirement | Notes |
|---|---|
| Prisma AIRS (AI Model Security) license | Same Strata Cloud Manager tenant as docs 00/09. |
| Strata Cloud Manager Service Account | Needs a Tenant Service Group (TSG) and an IAM service account — see §10.3. |
| Outbound HTTPS access | `api.sase.paloaltonetworks.com`, `auth.apps.paloaltonetworks.com`, `*.pkg.dev` (private GCP package registry), `pypi.org`, `files.pythonhosted.org`, and `huggingface.co` if scanning HF models. All outbound — already permitted by this host's UFW defaults. |
| Python 3.11 or 3.12 | **Check this first** — see §10.4, since Ubuntu 22.04 LTS ships Python 3.10 by default, which does not meet this requirement (24.04 LTS ships 3.12 and is fine as-is). |
| `jq` | Not installed by default on Ubuntu; required by the PyPI-auth script below. |
| CPU / RAM / disk headroom | Minimum 2 CPU cores / 2GB RAM free *on top of* what k3s + the gateway pods + the NCC already use; recommended 4+ cores / 4GB+ RAM for comfortable scanning. 10GB+ free disk recommended if scanning local models (Hugging Face scans don't need local storage). |

## 10.3 Create the Service Account (Strata Cloud Manager)

1. Confirm you have at least one **Tenant Service Group (TSG)** — create one first if this is a fresh tenant, rather than using your root TSG.
2. Create a **Service Account** scoped to that TSG, via either the Strata Cloud Manager UI or the IAM API.
3. **Save the Client ID and Client Secret immediately** — the secret is shown only once and cannot be retrieved again.
4. Confirm the service account has the roles needed for AI Model Security (Assign Roles, verify the relevant custom role appears under Apps & Services).

## 10.4 Check/Install a Compatible Python

```bash
python3 --version
```

- **Ubuntu 24.04 LTS** → ships Python 3.12; nothing to do here.
- **Ubuntu 22.04 LTS** → ships Python 3.10, which is **not** supported. Install 3.11 via the `deadsnakes` PPA rather than touching the system's default `python3` (keeps `apt`/other system tooling that depends on Python 3.10 untouched):

```bash
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y python3.11 python3.11-venv
```

Either way, also install `jq` if it isn't already present (it wasn't needed by docs 01-09):

```bash
sudo apt install -y jq
```

## 10.5 Install AI Model Security in an Isolated Virtual Environment

Install into a **dedicated venv**, not the system Python — two reasons: (1) modern Ubuntu's `pip` refuses a bare system-wide install anyway (PEP 668 "externally managed environment"), and (2) it keeps this tool's pinned dependencies (`airs-schemas`, `click`, `httpx`, `pydantic`, etc.) from ever colliding with anything else installed on this host.

```bash
mkdir -p ~/ai-model-security
cd ~/ai-model-security
python3.11 -m venv .venv      # use python3.12 if that's what §10.4 left you with
source .venv/bin/activate
```

**Step 1 — save the PyPI-auth helper script** (from the vendor doc — fetches a short-lived, tenant-scoped PyPI index URL via OAuth2 client-credentials):

```bash
cat > get-pypi-url.sh <<'EOF'
#!/bin/bash
set -euo pipefail
: "${MODEL_SECURITY_CLIENT_ID:?Error: MODEL_SECURITY_CLIENT_ID not set}"
: "${MODEL_SECURITY_CLIENT_SECRET:?Error: MODEL_SECURITY_CLIENT_SECRET not set}"
: "${TSG_ID:?Error: TSG_ID not set}"

API_ENDPOINT="${MODEL_SECURITY_API_ENDPOINT:-https://api.sase.paloaltonetworks.com/aims}"
TOKEN_ENDPOINT="${MODEL_SECURITY_TOKEN_ENDPOINT:-https://auth.apps.paloaltonetworks.com/oauth2/access_token}"

TOKEN_RESPONSE=$(curl -sf -X POST "$TOKEN_ENDPOINT" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -u "$MODEL_SECURITY_CLIENT_ID:$MODEL_SECURITY_CLIENT_SECRET" \
    -d "grant_type=client_credentials&scope=tsg_id:$TSG_ID") || {
    echo "Error: Failed to obtain SCM access token" >&2; exit 1; }

SCM_TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r '.access_token')
[[ -z "$SCM_TOKEN" || "$SCM_TOKEN" == "null" ]] && { echo "Error: Failed to extract access token" >&2; exit 1; }

PYPI_RESPONSE=$(curl -sf -X GET "$API_ENDPOINT/mgmt/v1/pypi/authenticate" \
    -H "Authorization: Bearer $SCM_TOKEN") || {
    echo "Error: Failed to retrieve PyPI URL" >&2; exit 1; }

PYPI_URL=$(echo "$PYPI_RESPONSE" | jq -r '.url')
[[ -z "$PYPI_URL" || "$PYPI_URL" == "null" ]] && { echo "Error: Failed to extract PyPI URL" >&2; exit 1; }

echo "$PYPI_URL"
EOF
chmod +x get-pypi-url.sh
```

**Step 2 — set credentials as environment variables** (from §10.3; do not commit these — same handling as the registry credentials in [doc 08 §8.3](08-useful-commands-reference.md#83-checking-available-gateway-image-versions-in-the-registry)):

```bash
export MODEL_SECURITY_CLIENT_ID='<your-client-id>'
export MODEL_SECURITY_CLIENT_SECRET='<your-client-secret>'
export TSG_ID='<your-tsg-id>'
export MODEL_SECURITY_API_ENDPOINT="https://api.sase.paloaltonetworks.com/aims"
```

**Step 3 — install the package** (inside the activated venv). Pick the storage-provider extras you actually need — `all` pulls every cloud SDK; for this lab, Hugging Face/local scans don't need any extra:

```bash
pip install "model-security-client[all]" --extra-index-url "$(./get-pypi-url.sh)"
```

**Step 4 — verify the CLI is installed and authenticated:**

```bash
model-security --version
```

## 10.6 Run a Lightweight Verification Scan

Use a small public Hugging Face model for the first run — it avoids downloading anything to local disk and keeps this verification cheap on CPU/RAM:

```bash
model-security scan \
  --security-group-uuid "<default-hugging-face-security-group-uuid>" \
  --model-uri "https://huggingface.co/microsoft/DialoGPT-medium" \
  -l env=lab-verification
```

The CLI exits non-zero if the model is flagged unsafe by your Security Group's rules, zero otherwise — suitable for scripting/CI gating later if you want it.

## 10.7 Keeping Scans From Competing With the Gateway/NCC for Resources

This is the actual "won't conflict" guarantee for this module — enforced at the process level, not the network level:

- **Prefer Hugging Face / object-storage scans over local-disk scans** where possible — they avoid the local download + up-to-4GB-RAM footprint that local model scans carry.
- **Cap the scanner's resource usage** with a transient systemd scope so a large scan can't starve the gateway pods or the NCC, instead of running it unbounded in your shell:

```bash
systemd-run --user --scope -p CPUQuota=150% -p MemoryMax=2G \
  model-security scan --security-group-uuid "<uuid>" --model-uri "<model-uri>"
```

Adjust `CPUQuota`/`MemoryMax` to whatever headroom you confirm is actually free (see next point) — the values above are a starting point, not a guarantee.

- **Check headroom before a real (non-verification) scan**, especially a local-model scan:

```bash
free -h
nproc
kubectl top nodes 2>/dev/null   # if metrics-server is installed; informational only if not
```

- **Confirm the gateway and NCC are unaffected** during/after a scan, the same way doc 09 §9.9 verifies NCC didn't disturb the gateway:

```bash
kubectl get pods -n airs-gw -o wide
kubectl get pods -n network-channel-client -o wide
curl -s http://192.168.110.161/v1/health
```

If you ever see gateway latency or pod restarts correlating with scan runs, that's a resource-contention signal — tighten the `systemd-run` limits above or move heavy local-model scanning to a different host entirely (the CLI/SDK has no dependency on this host beyond outbound network access, so it's trivially relocatable).

## 10.8 Notes

- This module writes nothing to `values.yaml`, the `airs-gw` Helm release, HAProxy config, or the `network-channel-client` namespace — it's entirely additive and lives in `~/ai-model-security` on the host, outside Kubernetes.
- The private PyPI index URL from `get-pypi-url.sh` is short-lived (OAuth2-token-backed) — re-run the script (or just re-run `pip install`, which re-invokes it) rather than caching the URL long-term.
- `model-security-client` pins specific versions of `airs-schemas`, `click`, `httpx`, `httpx-retries`, `modelscan-pai`, `orjson`, `packaging`, `pydantic`, and `pyjwt` — another reason the venv in §10.5 matters, since these could otherwise collide with dependencies of unrelated tooling on this host.
- A scan handles at most 1,000 files, and scans cannot be deleted once created — keep that in mind before scanning large repositories just to "try it out."
