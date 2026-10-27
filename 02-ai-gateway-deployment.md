# 2. AI Gateway Deployment (2 Pods + HAProxy Load Balancer)

**Core lab** — required. Prerequisite: [01-ubuntu-server-preparation.md](./01-ubuntu-server-preparation.md) completed and all checklist items ticked.

This guide deploys the AIRS Gateway Helm chart with **2 gateway replicas**, then configures **HAProxy** in front of the cluster to distribute inbound traffic.

References:
- Helm chart configuration reference: https://github.com/Portkey-AI/airs-gw-helm/blob/main/charts/airs-gw/docs/Configuration.md
- Helm chart README / deployment guide: https://github.com/Portkey-AI/airs-gw-helm/blob/main/charts/airs-gw/README.md
- Official Palo Alto AI Runtime Security docs: https://docs.paloaltonetworks.com/ai-runtime-security/administration/configure-ai-gateway

## 2.1 Why Two Load-Balancer Layers

- **HAProxy** (host-level, port 80/443): the single stable entry point for clients. It health-checks and forwards to the Kubernetes NodePort.
- **Kubernetes Service (NodePort) + kube-proxy**: once traffic reaches the NodePort, Kubernetes itself round-robins across the 2 gateway pod replicas. This is inherent to how a k8s Service works — it's already a load balancer across the pods behind it.

This means: HAProxy gives you a fixed, host-level entry point that also survives if you later add more Ubuntu nodes (just add each node's IP:NodePort as another HAProxy backend server), while Kubernetes handles the pod-to-pod distribution automatically as you scale replicas up/down.

## 2.2 Edit `values.yaml`

Open `~/ai-gateway/values.yaml` and make one change: set `replicaCount: 2` (not present in the SCM-downloaded file, so add it — the chart defaults to `1`).

```yaml
replicaCount: 2

service:
  type: NodePort
  port: 80
  containerPort: 8787
```

> Why not keep `service.type: LoadBalancer`? That type expects a cloud load-balancer controller (AWS/Azure/GCP) or an in-cluster provisioner like MetalLB to assign an external IP. We disabled k3s's own `servicelb` in the prep guide (to avoid it fighting with HAProxy), so a `LoadBalancer`-type Service would sit in `Pending` forever. `NodePort` gives HAProxy a reachable `127.0.0.1:<port>` target instead.

> **No `nodePort:` field here** — this chart's Service template (`templates/gateway/service.yaml`) doesn't expose a way to pin a specific NodePort number via `values.yaml`; only `service.type` and `service.port` are wired up. Kubernetes will auto-assign a random port from its NodePort range (30000–32767) the first time the Service is created. §2.4 below shows how to discover it. That auto-assigned port is stable for the life of this Service — it only changes if the Service is deleted and recreated (e.g. a full `helm uninstall` + reinstall), not on ordinary `helm upgrade`.

Everything else in the file (image credentials, `environment.data`, etc.) stays as SCM generated it — do not remove the secrets.

## 2.3 Add the Helm Repo and Deploy

```bash
helm repo add airs-gw https://portkey-ai.github.io/airs-gw-helm
helm repo update

helm upgrade --install airs-gw airs-gw/airs-gw \
  -f ~/ai-gateway/values.yaml \
  -n airs-gw \
  --create-namespace
```

> **Reproducibility note (recommended for a classroom/lab setting):** without a `--version`, this installs whatever chart release is newest the day you run it. For a cohort doing this on different days, pin an exact chart version instead — check available versions with `helm search repo airs-gw/airs-gw --versions` and add `--version <x.y.z>` to the command above. Note that the chart version is independent of the gateway app version (see [doc 07](07-force-upgrading-gateway-image.md)) — pinning one doesn't pin the other.

## 2.4 Verify the Deployment

```bash
kubectl get pods -n airs-gw -o wide
```

Expect two gateway pods (e.g. `airs-gw-xxxxxxxxxx-aaaaa`, `airs-gw-xxxxxxxxxx-bbbbb`) showing `Running` and `1/1 Ready`, plus a Redis pod.

```bash
kubectl get deployment -n airs-gw
kubectl get svc -n airs-gw
```

You'll see something like:

```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                       AGE
airs-gw         NodePort    10.43.169.109   <none>        80:32057/TCP,8788:32295/TCP   36s
airs-gw-redis   ClusterIP   10.43.122.13    <none>        6379/TCP                      36s
```

The `80:XXXXX/TCP` entry is the **auto-assigned NodePort** for the gateway (in this example, `32057`; yours will differ). The second port pair (`8788:YYYYY/TCP`) is the **MCP NodePort** — only relevant if you have MCP server integrations configured in SCM (see §2.9). Capture both into variables so the rest of this guide and doc 3 can reuse them:

```bash
export GATEWAY_NODE_PORT=$(kubectl get svc airs-gw -n airs-gw -o=jsonpath='{.spec.ports[?(@.name=="gateway")].nodePort}')
export MCP_NODE_PORT=$(kubectl get svc airs-gw -n airs-gw -o=jsonpath='{.spec.ports[?(@.name=="mcp")].nodePort}')
echo $GATEWAY_NODE_PORT $MCP_NODE_PORT
```

> This value is stable across `helm upgrade` runs. Only re-run the command above (and update your HAProxy config in §2.6) if you ever fully `helm uninstall` and reinstall the release.

If a pod is not `Running`, inspect it before continuing:

```bash
kubectl describe pod -n airs-gw <pod-name>   # check Events section
kubectl logs -n airs-gw <pod-name>
```

Common early issues: `ImagePullBackOff` (registry credentials in `values.yaml` wrong/expired) or probe failures against `/v1/health` (pod still starting — the readiness probe has a 5s initial delay and 60s period, so allow a minute).

## 2.5 Sanity-Check the NodePort Directly (Before Involving HAProxy)

```bash
curl -i http://127.0.0.1:$GATEWAY_NODE_PORT/v1/health
```

Expect an HTTP `200` response. Run this a few times — you're hitting the Service, so Kubernetes may route you to either pod. Don't proceed to HAProxy configuration until this succeeds.

## 2.6 Configure HAProxy

Ubuntu's `apt`-installed HAProxy ships with a stock `/etc/haproxy/haproxy.cfg` containing only `global` and `defaults` sections (chroot, stats socket, SSL cipher defaults, error pages) — **no** `frontend`/`backend` yet. **Don't overwrite this file** — those stock settings are fine as-is; just back it up and **append** the AI Gateway frontend/backend/stats blocks to the end of it.

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
```

Confirm your shell still has the right port (re-export if you've started a new session — see §2.4):

```bash
echo $GATEWAY_NODE_PORT
```

Append the new sections:

```bash
sudo tee -a /etc/haproxy/haproxy.cfg > /dev/null <<EOF

frontend ai_gateway_http
    bind *:80
    default_backend ai_gateway_backend

# --- Optional TLS termination (uncomment once you have a cert) ---
# frontend ai_gateway_https
#     bind *:443 ssl crt /etc/haproxy/certs/ai-gateway.pem
#     default_backend ai_gateway_backend

backend ai_gateway_backend
    balance roundrobin
    option httpchk
    http-check send meth GET uri /v1/health hdr Host localhost
    http-check expect status 200
    default-server inter 5s fall 3 rise 2
    # Single node today: k8s NodePort already fans out across the 2 gateway pods.
    server k3s-node1 127.0.0.1:${GATEWAY_NODE_PORT} check

listen stats
    bind 127.0.0.1:8404
    stats enable
    stats uri /stats
    stats refresh 10s
EOF
```

The `EOF` here is **unquoted** so `${GATEWAY_NODE_PORT}` gets substituted with the actual port number (e.g. `32057`) at the time you run the command — check the result:

```bash
tail -n 20 /etc/haproxy/haproxy.cfg
```

Your full file should now look like this (global/defaults exactly as Ubuntu shipped it, our sections appended below):

```haproxy
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend ai_gateway_http
    bind *:80
    default_backend ai_gateway_backend

# --- Optional TLS termination (uncomment once you have a cert) ---
# frontend ai_gateway_https
#     bind *:443 ssl crt /etc/haproxy/certs/ai-gateway.pem
#     default_backend ai_gateway_backend

backend ai_gateway_backend
    balance roundrobin
    option httpchk
    http-check send meth GET uri /v1/health hdr Host localhost
    http-check expect status 200
    default-server inter 5s fall 3 rise 2
    # Single node today: k8s NodePort already fans out across the 2 gateway pods.
    server k3s-node1 127.0.0.1:32057 check

listen stats
    bind 127.0.0.1:8404
    stats enable
    stats uri /stats
    stats refresh 10s
```

(`mode http`, logging, and timeouts are already inherited from the stock `defaults` block above — no need to redeclare them.)

> **Scaling to multiple nodes later:** if you ever add more Ubuntu servers each running a k3s agent + the same NodePort service, add one `server` line per node (e.g. `server k3s-node2 <node2-ip>:<same-gateway-nodeport> check` — NodePorts are consistent across all nodes in a cluster for a given Service) so HAProxy load-balances across nodes too, in addition to Kubernetes balancing across pods on each node.

> **Optional TLS:** for production use, obtain a certificate (e.g. via `certbot` if you have a public domain, or a self-signed/internal CA cert for LAN-only use), concatenate the cert+key into a single `.pem` at `/etc/haproxy/certs/ai-gateway.pem`, and uncomment the HTTPS frontend block above. Since this HAProxy install uses `chroot /var/lib/haproxy`, double-check certificate paths resolve correctly against `crt-base /etc/ssl/private` at config-parse time (before the chroot applies) — this guide keeps the default path on plain HTTP for simplicity.

Validate the config syntax, then reload:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl status haproxy --no-pager
```

## 2.7 First End-to-End Test Through HAProxy

```bash
curl -i http://127.0.0.1/v1/health
curl -i http://<server-lan-ip>/v1/health    # from another machine on the network
```

Expect HTTP `200` in both cases. Also open the HAProxy stats page (local machine or via SSH tunnel, since it's bound to `127.0.0.1:8404`):

```bash
ssh -L 8404:127.0.0.1:8404 <user>@<server>
# then browse to http://127.0.0.1:8404/stats on your workstation
```

Confirm the `ai_gateway_backend` server shows **UP** (green).

> **Deployment fully confirmed working end-to-end** once you can also complete a real authenticated request through the whole stack (client → HAProxy → NodePort → gateway pod → upstream provider) — not just the health check. See [03-operations-and-testing.md §3.1.2](./03-operations-and-testing.md#312-real-gateway-request-end-to-end-authrouting) for the exact headers required (`x-portkey-api-key` + `x-portkey-provider: @<saved-integration-slug>`) and a worked example. A successful response looks like `HTTP 200` with a completion body and headers such as `x-portkey-provider: vertex-ai` and `x-portkey-trace-id: ...`.

## 2.8 Deployment Checklist

- [ ] `values.yaml` updated: `replicaCount: 2`, `service.type: NodePort`
- [ ] `helm upgrade --install` completed without errors
- [ ] `kubectl get pods -n airs-gw` shows 2 gateway pods `Running`/`Ready` + Redis pod
- [ ] Actual gateway NodePort discovered (`$GATEWAY_NODE_PORT`) and noted for HAProxy config
- [ ] Direct NodePort curl to `/v1/health` returns `200`
- [ ] HAProxy config validated (`haproxy -c`) and service restarted
- [ ] HAProxy stats page shows backend `UP`
- [ ] `curl http://<server-ip>/v1/health` returns `200` from outside the box

## 2.9 MCP (Model Context Protocol) Support

If you have MCP server integrations configured in SCM (e.g. a GitHub MCP registration), the gateway exposes them on the same Service's `mcp` port, using the path pattern **`/<mcp-slug>/mcp`** — the slug comes first, then the literal `/mcp` suffix. Find the slug on Strata Cloud Manager's **AI Gateway → MCP Servers & Tools** page (a different page from the general Integrations/Providers page used for LLM provider slugs) — it's the first path segment of the URL shown there, e.g. `https://mcp-aigw.portkey.ai/<mcp-slug>/mcp`.

### 2.9.1 Add `MCP_GATEWAY_BASE_URL`

Add this under `environment.data` in `~/ai-gateway/values.yaml`, pointing at the address HAProxy will expose for MCP (configured in §2.9.3):

```yaml
environment:
  data:
    MCP_GATEWAY_BASE_URL: "http://<server-lan-ip>:8788"
```

Apply it:

```bash
helm upgrade airs-gw airs-gw/airs-gw -f ~/ai-gateway/values.yaml -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

### 2.9.2 Sanity-Check the MCP NodePort Directly

```bash
echo $MCP_NODE_PORT   # re-export from §2.4 if needed

curl -sv http://127.0.0.1:$MCP_NODE_PORT/<mcp-slug>/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "x-portkey-api-key: <client-auth-value-from-PORTKEY_CLIENT_AUTH>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"curl-test","version":"1.0"}}}'
```

Expect `HTTP/1.1 200 OK`, `Content-Type: text/event-stream`, with an SSE body:

```
event: message
data: {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-03-26","capabilities":{"completions":{},"prompts":{},"resources":{},"tools":{}},"serverInfo":{"name":"<mcp-server-name>","version":"..."}}}
```

### 2.9.3 Configure HAProxy for MCP

Append a dedicated frontend/backend for MCP — separate from the gateway's port 80, since MCP has no `/v1/health`-style endpoint to health-check against:

```bash
echo $MCP_NODE_PORT   # confirm still set

sudo tee -a /etc/haproxy/haproxy.cfg > /dev/null <<EOF

frontend ai_gateway_mcp
    bind *:8788
    default_backend ai_gateway_mcp_backend

backend ai_gateway_mcp_backend
    balance roundrobin
    option tcp-check
    server k3s-node1-mcp 127.0.0.1:${MCP_NODE_PORT} check
EOF

sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
sudo ufw allow 8788/tcp
```

`option tcp-check` is used instead of `httpchk` because MCP has no documented health-check path — every probed route returns either a valid JSON-RPC response or a generic 404, neither reliable as a liveness signal.

### 2.9.4 Test Through HAProxy

```bash
curl -sv http://<server-lan-ip>:8788/<mcp-slug>/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "x-portkey-api-key: <client-auth-value-from-PORTKEY_CLIENT_AUTH>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"curl-test","version":"1.0"}}}'
```

The same `HTTP 200` SSE response as §2.9.2 confirms MCP is reachable end-to-end (client → HAProxy → NodePort → gateway pod). See [03-operations-and-testing.md §3.1.3](./03-operations-and-testing.md#313-mcp-server-request-initialize--tools-list) to also confirm `tools/list`.

### 2.9.5 MCP Checklist

- [ ] `MCP_GATEWAY_BASE_URL` added to `values.yaml`, `helm upgrade` applied
- [ ] MCP NodePort discovered (`$MCP_NODE_PORT`)
- [ ] Direct NodePort curl to `/<mcp-slug>/mcp` returns a valid `initialize` response
- [ ] HAProxy MCP frontend/backend added, config validated, port 8788 open in UFW
- [ ] `curl http://<server-ip>:8788/<mcp-slug>/mcp` returns a valid response from outside the box

Continue to [03-operations-and-testing.md](./03-operations-and-testing.md) to verify load distribution across both pods and run functional tests.
