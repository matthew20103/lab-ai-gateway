# 8. Useful Command Reference

**Running notes** — a cheat-sheet of commands used repeatedly across this lab, grouped by task. This doc doesn't re-explain the *why* behind each command; see the referenced docs for that context. Add new sections here as new recurring tasks come up.

## 8.1 Applying a `values.yaml` Change (Helm Upgrade Workflow)

General pattern any time `values.yaml` is edited (new `TRUSTED_CUSTOM_HOSTS` entry, image tag override, etc.) — see [doc 05](05-adding-trusted-custom-hosts.md) and [doc 07](07-force-upgrading-gateway-image.md) for specific examples.

```bash
helm repo update airs-gw
helm upgrade airs-gw airs-gw/airs-gw -f ~/ai-gateway/values.yaml -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

Verify a specific env var landed on both pods:

```bash
for POD in $(kubectl get pod -n airs-gw -l app.kubernetes.io/name=airs-gw -o jsonpath="{.items[*].metadata.name}"); do
  kubectl exec -n airs-gw $POD -- printenv <VAR_NAME>
done
```

If `printenv` still shows the old value after `helm upgrade`, the pods weren't restarted:

```bash
kubectl rollout restart deployment/airs-gw -n airs-gw
```

**Before any `helm upgrade`:** confirm `values.yaml` still has `images.gatewayImage.tag` set if the [doc 07](07-force-upgrading-gateway-image.md) force-upgrade has been applied — omitting it reverts the gateway to the chart's default image.

## 8.2 Checking the A2A Agent Gateway URL

Pattern (see [doc 06 §6.2](06-known-limitations.md)):

```
http://<gateway-lan-ip>/v1/agent/<slug>
```

In this deployment, `<gateway-lan-ip>` is `192.168.110.161`, reached on **port 80** via HAProxy (same port as the rest of the main gateway API — `/v1/chat/completions`, `/v1/decisions`). It is **not** the separate `8788` MCP port. `<slug>` comes from the Agent integration registered in SCM Model Catalog.

Don't rely on the agent-card endpoint to confirm the URL — it's answered by the control plane, not forwarded, and returns a near-empty stub:

```bash
curl -s -H "x-portkey-api-key: <gateway API key>" \
  http://192.168.110.161/v1/agent/<your-subagent-slug>/.well-known/agent-card.json
```

Verify the real routing instead with a JSON-RPC call (this path **is** forwarded to the sub-agent):

```bash
curl -s http://192.168.110.161/v1/agent/<your-subagent-slug> \
  -H "Content-Type: application/json" \
  -H "x-portkey-api-key: <gateway API key>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"message/send","params":{...}}'
```

Prerequisite: the sub-agent's real hostname must be in `TRUSTED_CUSTOM_HOSTS` (§8.1 workflow) and the gateway must be on the force-upgraded image ([doc 07](07-force-upgrading-gateway-image.md)) for `/v1/agent/*` to exist at all.

## 8.3 Checking Available Gateway Image Versions in the Registry

Not a documented/supported Portkey procedure — assembled from general Docker Registry V2 API knowledge, since `registry.portkey.ai` exposes no documented tag-listing endpoint.

Credentials come from `imageCredentials` in `values.yaml` (`registry: "https://registry.portkey.ai"`, `username`, `password`). Export them as shell variables rather than typing them inline, to keep them out of shell history — and never commit these exported values or paste them into chat tools:

```bash
export REG_USER='<username from imageCredentials>'
export REG_PASS='<password from imageCredentials>'
export REG=registry.portkey.ai
export REPO=airsgw/gateway_enterprise
```

**Option A — `skopeo` (simplest).** Ubuntu 22.04/24.04 ship it in the `universe` repo:

```bash
sudo apt update
sudo apt install -y skopeo jq
# if the package isn't found:
sudo add-apt-repository universe && sudo apt update && sudo apt install -y skopeo jq

skopeo list-tags "docker://$REG/$REPO" --creds "$REG_USER:$REG_PASS"
```

**Option B — raw Docker Registry V2 API via `curl`.** Probe the auth scheme first:

```bash
curl -sI -u "$REG_USER:$REG_PASS" "https://$REG/v2/$REPO/tags/list"
```

If `200` directly, re-run without `-I`:

```bash
curl -s -u "$REG_USER:$REG_PASS" "https://$REG/v2/$REPO/tags/list" | jq .
```

If `401` with a `Www-Authenticate: Bearer realm="...",service="...",scope="..."` header, fetch a token first:

```bash
TOKEN=$(curl -s -u "$REG_USER:$REG_PASS" "https://<realm>?service=<service>&scope=<scope>" | jq -r .token)
curl -s -H "Authorization: Bearer $TOKEN" "https://$REG/v2/$REPO/tags/list" | jq .
```

Watch for a `Link` response header — some registries paginate and only return a partial tag list per call (`?last=<tag>&n=<count>`).

**Caveat:** a `403`/empty result doesn't prove no other versions exist — some registries disable tag discovery for an account while still letting it pull specific named tags. If discovery is blocked, check Portkey's release notes instead, then validate any candidate tag individually with the throwaway `registry-check` pod from [doc 07 §7.1 Check 2](07-force-upgrading-gateway-image.md#71-feasibility-checks-do-these-before-editing-anything) before putting it in `values.yaml`.

**Confirmed on this deployment (2026-11-04):** `skopeo list-tags` worked directly with basic `--creds` — no token-exchange dance needed (Option B's token flow is kept above as a fallback for registries that do require it). Full output had ~290 tags; the highest non-prerelease, non-fips tag was `2.32.0`, with a `2.33.0-prerelease1` already published ahead of it. Tag-name sort order isn't reliable for finding the newest version — the history has inconsistent prerelease naming (`1.11.9.prereleas1`, `.prerelease1` vs `-prerelease1`) and, starting around `2.32.0`, architecture-suffixed variants (`2.32.0-amd64`, `2.32.0-arm64`, `2.32.0-fips-amd64`, `2.32.0-fips-arm64`) alongside the plain multi-arch tag. Re-run `skopeo list-tags` and eyeball the real list rather than assuming this stays current.

**`latest` exists** (`latest` and `latest-fips` are both real tags) — but a tag existing doesn't tell you *which* version it currently points to, since it's a floating pointer. Check with a digest comparison:

```bash
skopeo inspect --creds "$REG_USER:$REG_PASS" "docker://$REG/$REPO:latest" | jq -r '.Digest'
skopeo inspect --creds "$REG_USER:$REG_PASS" "docker://$REG/$REPO:2.32.0" | jq -r '.Digest'
```
Matching digests confirm what `latest` resolves to right now.

**Avoid pinning the literal `latest` tag in `values.yaml` anyway** — the chart's default `imagePullPolicy: IfNotPresent` means a pod won't re-pull a newer image behind an already-pulled `latest` tag on subsequent `helm upgrade`s, and it makes the running version impossible to identify from `kubectl get deployment ... -o jsonpath='{.spec.template.spec.containers[0].image}'`. Resolve `latest` to its concrete version (above) and pin that explicit tag instead, per [doc 07](07-force-upgrading-gateway-image.md) — remembering any jump beyond the already-tested `2.21.0`/`2.25.1` combo is a fresh "unsupported chart+app combination" risk, not an assumed-safe bump.

**What `-fips` tags mean:** built against FIPS 140-2/140-3 validated cryptographic modules (e.g. a certified OpenSSL/BoringCrypto build) instead of the standard crypto library — needed only for environments with a U.S. federal/regulated compliance requirement to use FIPS-certified crypto. Functionally equivalent otherwise. For this lab (no compliance requirement), use the plain tag, not the `-fips` variant.

## 8.4 Upgrading the Gateway Image Version (Feasibility Check → Apply → Verify)

Full procedure in [doc 07](07-force-upgrading-gateway-image.md); this is the condensed command sequence for repeat use. Treat **every** tag jump as an unsupported chart+app combination to re-validate, not an assumed-safe bump — the chart (`1.2.0`) was only ever officially paired with app `2.21.0`.

**1. Edit `values.yaml`:**
```yaml
images:
  gatewayImage:
    tag: "<new-tag>"   # explicit version from §8.3 — never "latest"
```

**2. Confirm the tag pulls with the existing image-pull secret:**
```bash
kubectl run registry-check -n airs-gw \
  --image=registry.portkey.ai/airsgw/gateway_enterprise:<new-tag> \
  --overrides='{"spec":{"imagePullSecrets":[{"name":"airsgatewayregistrycredentials"}]}}' \
  --restart=Never -- sleep 3600

kubectl describe pod registry-check -n airs-gw | grep -A5 Events
kubectl delete pod registry-check -n airs-gw
```
Look for `Successfully pulled image`.

**3. Back up current state (rollback safety net):**
```bash
mkdir -p ~/ai-gateway/backups
helm get values airs-gw -n airs-gw -o yaml > ~/ai-gateway/backups/pre-<new-tag>-values.yaml
helm history airs-gw -n airs-gw
```
Note the current revision number as the rollback target.

**4. Apply** (same as §8.1's general workflow):
```bash
helm repo update airs-gw
helm upgrade airs-gw airs-gw/airs-gw -f ~/ai-gateway/values.yaml -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

**5. Verify:**
```bash
kubectl get deployment airs-gw -n airs-gw -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# Expect: registry.portkey.ai/airsgw/gateway_enterprise:<new-tag>

kubectl logs -n airs-gw -l app.kubernetes.io/name=airs-gw --tail=100
# Watch for crash loops, startup errors, config-schema complaints

POD=$(kubectl get pod -n airs-gw -l app.kubernetes.io/name=airs-gw -o jsonpath="{.items[0].metadata.name}")
kubectl exec -n airs-gw $POD -- wget -qO- http://localhost:8787/v1/health
# Expect the new version in the response
```

**6. Rollback if anything misbehaves:**
```bash
helm rollback airs-gw <revision-from-step-3> -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

**After any jump:** re-verify behavior documented in [doc 06](06-known-limitations.md) (`/v1/decisions`, `/v1/agent/*`) — it was only confirmed against `2.25.1`; a larger version jump can change it.

**Log:**
| Date | From → To | Result |
|---|---|---|
| 2026-11-04 | `2.25.1` → `2.32.0` | pending — `values.yaml` updated, not yet applied to cluster |

## 8.5 Changing HAProxy's `timeout client` / `timeout server`

These live purely at the HAProxy layer in front of the gateway ([doc 02 §2.6](02-ai-gateway-deployment.md#26-installing-and-configuring-haproxy)), not in `values.yaml` — no Helm upgrade needed. `timeout server` is what governs how long HAProxy waits for a response (including a slow LLM call) before giving up and serving the `504.http` error page; `timeout client` should generally match or exceed it, or HAProxy may drop the client side first.

**1. Edit the `defaults` block in `/etc/haproxy/haproxy.cfg`:**
```bash
sudo nano /etc/haproxy/haproxy.cfg
```
```haproxy
        timeout connect 5000
        timeout client  90s
        timeout server  90s
```
Values are milliseconds by default, but `s`/`m` suffixes (e.g. `90s`, `2m`) are accepted and more readable.

**2. Validate syntax, then reload:**
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy   # graceful, no dropped connections
sudo systemctl status haproxy --no-pager
```

**Before picking a value:** check how long your slowest real LLM calls through this gateway actually take (long completions, bigger models) — if any provider can legitimately run past the current timeout, raise it past that, or you'll get spurious 504s on valid-but-slow requests.
