# 3. AI Gateway Operations & Testing

**Core lab** — required. Prerequisite: [02-ai-gateway-deployment.md](./02-ai-gateway-deployment.md) completed — gateway reachable through HAProxy on port 80.

Completing this doc finishes the required lab: a working 2-pod gateway, load-balanced, serving real LLM traffic. Docs 04-07 below are independent advanced/optional modules — see the lab map in [00-prerequisites.md §0.3](00-prerequisites.md#03-lab-map).

## 3.1 Functional Testing

### 3.1.1 Health Check (via HAProxy vs Direct Pod)

Through HAProxy (the path real clients use):

```bash
curl -i http://127.0.0.1/v1/health
```

Directly against one pod, bypassing both load-balancer layers, useful when isolating whether a problem is in HAProxy, Kubernetes, or the pod itself:

```bash
kubectl get pods -n airs-gw -o name
kubectl port-forward -n airs-gw <pod-name> 8787:8787
# in another terminal:
curl -i http://127.0.0.1:8787/v1/health
```

### 3.1.2 Real Gateway Request (End-to-End Auth/Routing)

Send an actual request through the gateway using its client auth and a saved provider integration:

```bash
curl -i http://127.0.0.1/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-portkey-api-key: <client-auth-value-from-PORTKEY_CLIENT_AUTH>" \
  -H "x-portkey-provider: @<your-saved-integration-slug>" \
  -d '{
        "model": "<a-model-allowed-for-that-integration>",
        "messages": [{"role": "user", "content": "ping"}]
      }'
```

- `x-portkey-provider` must reference a **saved integration** using the `@` prefix (e.g. `@google-vertex-aug`) — find the slug in **Strata Cloud Manager's AI Gateway → Integrations/Providers** page (tied to `ORGANISATIONS_TO_SYNC` in `values.yaml`).
- `model` must be one of that integration's allowed models, per SCM's configuration for it.

A successful response returns `HTTP 200` with a completion payload and headers like `x-portkey-provider: vertex-ai` and `x-portkey-trace-id: ...`, confirming auth, routing, and the upstream provider integration all work end-to-end. For example:

```json
{"id":"chatcmpl-...","object":"chat.completion","model":"gemini-3.5-flash","provider":"vertex-ai",
 "choices":[{"message":{"role":"assistant","content":"Pong! How can I help you today?"},"finish_reason":"stop"}],
 "usage":{"prompt_tokens":1,"completion_tokens":184,"total_tokens":185}}
```

### 3.1.3 MCP Server Request (Initialize + Tools List)

If you have MCP server integrations configured (see [doc 2 §2.9](./02-ai-gateway-deployment.md#29-mcp-model-context-protocol-support)), verify them through HAProxy:

```bash
curl -s http://127.0.0.1:8788/<mcp-slug>/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "x-portkey-api-key: <client-auth-value-from-PORTKEY_CLIENT_AUTH>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"curl-test","version":"1.0"}}}'
```

- The route is **`/<mcp-slug>/mcp`** — slug first, `mcp` suffix second. Find the slug on Strata Cloud Manager's **AI Gateway → MCP Servers & Tools** page (not the general Integrations/Providers page used for LLM provider slugs).
- A successful response is `HTTP 200`, `Content-Type: text/event-stream`, with a JSON-RPC result in the SSE body:

```
event: message
data: {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-03-26","capabilities":{"completions":{},"prompts":{},"resources":{},"tools":{}},"serverInfo":{"name":"github-mcp-server","version":"..."}}}
```

Once `initialize` succeeds, list the tools the MCP server exposes:

```bash
curl -s http://127.0.0.1:8788/<mcp-slug>/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "x-portkey-api-key: <client-auth-value-from-PORTKEY_CLIENT_AUTH>" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

This returns the full tool catalog for that MCP integration — for a GitHub MCP server, that includes tools like `get_file_contents`, `create_branch`, `create_or_update_file`, `create_pull_request`, `fork_repository`, `issue_read`, `issue_write`, `get_commit`, `get_latest_release`, and dozens more covering repos, branches, commits, issues, PRs, and releases, each with a full JSON Schema `inputSchema`.

No `mcp-session-id` handling was needed for this server — `initialize` and `tools/list` each succeeded as independent, stateless requests.

## 3.2 Verifying Load Distribution Across Both Pods

The point of running 2 replicas is that traffic actually spreads across them. Verify it two ways:

### 3.2.1 Confirm Both Pods Are Live Service Endpoints

`/v1/health` isn't logged by the gateway (by design, to avoid log spam), so watching logs while curling health checks won't show anything — the app's logs are dominated by internal scheduled queue jobs (`reSyncDataQueue`, `syncDataMasterQueue`, etc.) that each pod runs independently regardless of external traffic. Instead, confirm both pods are registered as Service endpoints — this is what actually guarantees kube-proxy routes to both:

```bash
kubectl get endpoints -n airs-gw airs-gw
```

Expect both pod IPs listed on port 8787.

To see real evidence of traffic hitting both pods, generate several **completion** requests (these are logged, via an analytics batch-insert job) and grep both pods' logs for it:

```bash
kubectl get pods -n airs-gw   # note both pod names

for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1/v1/chat/completions \
    -H "Content-Type: application/json" \
    -H "x-portkey-api-key: <client-auth-value-from-PORTKEY_CLIENT_AUTH>" \
    -H "x-portkey-provider: @<your-saved-integration-slug>" \
    -d '{"model": "<a-model-allowed-for-that-integration>", "messages": [{"role": "user", "content": "ping"}]}'
done

kubectl logs -n airs-gw <pod-1-name> --tail=30 | grep generations
kubectl logs -n airs-gw <pod-2-name> --tail=30 | grep generations
```

If `Successfully added batch insert job to queue {"count":1,"table":"generations"}` shows up in **both** pods' output, that's conclusive proof traffic is spreading across both replicas.

### 3.2.2 Simple Load Test

Install a lightweight load generator and run a short burst:

```bash
sudo apt install -y apache2-utils   # provides `ab`
ab -n 200 -c 10 http://127.0.0.1/v1/health
```

Or with [`hey`](https://github.com/rakyll/hey) (Go-based, single static binary — download from its GitHub releases):

```bash
hey -n 200 -c 10 http://127.0.0.1/v1/health
```

Review the summary output (requests/sec, latency distribution, status code breakdown — expect all `200`s). Cross-check against the HAProxy stats page (`/stats`, see §3.3) and `kubectl top pods -n airs-gw` (requires `metrics-server`; not installed by default on k3s — install it if you want live CPU/RAM per pod: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`).

## 3.3 HAProxy Stats Dashboard

```bash
ssh -L 8404:127.0.0.1:8404 <user>@<server>
```

Browse to `http://127.0.0.1:8404/stats` on your workstation. Key things to check:
- `ai_gateway_backend` → server row shows **Status: UP**, session counts increasing as you send traffic
- `Chk` column shows recent health-check results (should be green/passing)
- Total sessions and bytes in/out give a quick sanity check of real usage

## 3.4 Scaling Replicas

Two equivalent ways to change the number of gateway pods — prefer the second for anything you want to persist:

**Ad-hoc (until the next `helm upgrade` overwrites it):**
```bash
kubectl scale deployment/airs-gw -n airs-gw --replicas=3
```

**Persistent (recommended):** edit `replicaCount` in `~/ai-gateway/values.yaml`, then:
```bash
helm upgrade airs-gw airs-gw/airs-gw -f ~/ai-gateway/values.yaml -n airs-gw
```

`values.yaml` is the source of truth — always reconcile it after any ad-hoc `kubectl scale`, or the next Helm upgrade will silently revert the replica count.

If you scale beyond what a single node comfortably handles, revisit the [sizing table in doc 1](./01-ubuntu-server-preparation.md#12-sizing-recommendations).

## 3.5 Log Inspection

```bash
# Gateway pod logs
kubectl logs -n airs-gw deployment/airs-gw --all-containers --tail=200

# Specific pod, following
kubectl logs -n airs-gw <pod-name> -f

# Redis pod logs
kubectl logs -n airs-gw -l app.kubernetes.io/name=redis --tail=100

# HAProxy logs (rsyslog forwards haproxy's local0 facility)
sudo tail -f /var/log/syslog | grep haproxy
# or, if using journald exclusively:
journalctl -u haproxy -f
```

> If HAProxy logs don't appear in `/var/log/syslog`, ensure rsyslog is installed and listening on `/dev/log` (`sudo apt install -y rsyslog && sudo systemctl restart rsyslog haproxy`).

## 3.6 Troubleshooting

| Symptom | Likely Cause | Check |
|---|---|---|
| Pod `ImagePullBackOff` | Registry credentials in `values.yaml` (`imageCredentials`) wrong/expired | `kubectl describe pod -n airs-gw <pod>` → Events; verify `registry`/`username`/`password` |
| Pod `CrashLoopBackOff` | App failing to start (bad env var, Redis unreachable) | `kubectl logs -n airs-gw <pod> --previous` |
| Pod stuck `Running` but `0/1 Ready` | Readiness probe (`/v1/health`) failing | `kubectl describe pod` → probe failure events; `kubectl exec` in and `curl localhost:8787/v1/health` |
| `curl` to NodePort times out | Wrong port (the NodePort is auto-assigned by k8s, not fixed — see [doc 2 §2.4](./02-ai-gateway-deployment.md#24-verify-the-deployment)), or Service/pods not up | `kubectl get svc -n airs-gw` to confirm the real port; `kubectl get endpoints -n airs-gw` (should list 2 pod IPs) |
| HAProxy returns `503 Service Unavailable` | Backend server marked `DOWN` (health check failing), or `haproxy.cfg` still points at a stale NodePort from before a reinstall | HAProxy stats page `/stats`; re-check `kubectl get svc -n airs-gw` and `curl 127.0.0.1:<nodeport>/v1/health` directly |
| HAProxy returns `502 Bad Gateway` | NodePort not accepting connections at all, or `haproxy.cfg` has the wrong port number | `kubectl get svc -n airs-gw` to confirm current NodePort; `sudo ss -tlnp \| grep <nodeport>`; confirm k3s Service exists and node is `Ready` |
| Works via `curl 127.0.0.1` but not from LAN | UFW not allowing 80/443, or listening only on loopback | `sudo ufw status`; confirm HAProxy `bind *:80` (not `127.0.0.1:80`) |
| All traffic seems to hit only one pod | Client reusing a single keep-alive HTTP connection (correct behavior — LB happens per-connection, not per-request, for HTTP/1.1 keep-alive) | Re-test with multiple short-lived connections (`curl` without `-k`/reuse, or `ab`/`hey` which open many connections) |
| MCP requests return `{"status":"not found"}` | Wrong path ordering | Use `/<mcp-slug>/mcp` — slug **first**, then the literal `/mcp` suffix (not `/mcp/<mcp-slug>` or `/mcp`). Find the slug on SCM's **AI Gateway → MCP Servers & Tools** page, not the Integrations/Providers page |

## 3.7 Upgrading the Chart / Config Changes

Whenever `values.yaml` changes (new model routes, secrets rotation, replica count, etc.):

```bash
helm repo update
helm upgrade airs-gw airs-gw/airs-gw -f ~/ai-gateway/values.yaml -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

`kubectl rollout status` blocks until the rolling update completes, giving you a clear pass/fail signal.

## 3.8 Uninstalling

```bash
helm uninstall airs-gw --namespace airs-gw
kubectl delete namespace airs-gw
sudo systemctl stop haproxy
```

To remove k3s entirely from the host (irreversible — only do this if decommissioning the whole node):

```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

## 3.9 Backup Notes

The only truly unrecoverable input in this setup is `~/ai-gateway/values.yaml` (it embeds the SCM registration, registry credentials, and client auth token). Keep a secure, encrypted off-box copy (e.g. a password manager or your organization's secrets vault) — everything else (k3s, Helm chart, pods) can be re-created from it plus these three guides.

## 3.10 Operations Checklist

- [ ] Health check succeeds through HAProxy and directly via port-forward
- [ ] A real gateway request (with auth) returns a valid completion
- [ ] MCP `initialize` and `tools/list` succeed through HAProxy (if MCP servers are configured)
- [ ] Traffic observed hitting both gateway pods under load
- [ ] HAProxy stats page reachable and backend shows `UP`
- [ ] Scaling tested via `values.yaml` + `helm upgrade`, confirmed with `kubectl rollout status`
- [ ] Logs (gateway pods + HAProxy) confirmed accessible
- [ ] `values.yaml` backed up securely off-box
