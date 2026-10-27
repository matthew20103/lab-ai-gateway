# 5. Adding a New Trusted Custom Host

**Advanced / optional module** — do this only after completing the core lab ([01-03](00-prerequisites.md#03-lab-map)). Does not require the chart force-upgrade in [doc 07](07-force-upgrading-gateway-image.md).

This guide documents the procedure for allowlisting an additional internal/private hostname so the AI Gateway is permitted to make outbound requests to it (MCP servers, custom model endpoints, webhooks, etc. hosted on a private IP). Background on why this is needed is covered in [04-internal-mcp-server-connectivity-fix.md](04-internal-mcp-server-connectivity-fix.md). Examples below use placeholder hostnames/IPs — substitute your own.

## 5.1 Procedure

**Step 1 — edit `TRUSTED_CUSTOM_HOSTS` in `values.yaml`.** It is a comma-separated list — append the new hostname to the existing value under `environment.data`:

```yaml
environment:
  data:
    TRUSTED_CUSTOM_HOSTS: "mcphub.yourdomain.lab,clm.yourdomain.lab"
```

**Step 2 — apply the change:**

```bash
helm repo update airs-gw
helm upgrade airs-gw airs-gw/airs-gw -f ~/ai-gateway/values.yaml -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

**Step 3 — verify the env var landed:**

```bash
POD=$(kubectl get pod -n airs-gw -l app.kubernetes.io/name=airs-gw -o jsonpath="{.items[0].metadata.name}")
kubectl exec -n airs-gw $POD -- printenv TRUSTED_CUSTOM_HOSTS
# Expect: mcphub.yourdomain.lab,clm.yourdomain.lab
```

**Step 4 — confirm the pod can resolve the new hostname.** The `coredns-custom` AAAA-override fix from doc 04 applies cluster-wide, so any internal-only hostname served by the same internal DNS server should already resolve correctly — but confirm directly:

```bash
kubectl exec -n airs-gw $POD -- node -e "require('dns').lookup('clm.yourdomain.lab',(e,a)=>console.log(e||a))"
```

Expected: the resolved IP, with no error. If this fails, the hostname may be served by a different/uncooperative DNS path — revisit doc 04 §4.2.

**Step 5 — end-to-end test.** Tail gateway logs and trigger a real request to the new host:

```bash
kubectl logs -n airs-gw -l app.kubernetes.io/name=airs-gw -f --prefix
```

Confirm no `SSRF block` line appears for requests to the new hostname, and that the target server's own logs show the incoming request from the gateway.

## 5.2 Notes

- `TRUSTED_CUSTOM_HOSTS` also supports a `*.` wildcard prefix to trust an entire subdomain at once, e.g. `*.yourdomain.lab` instead of listing each subdomain individually. Wildcard entries cannot target raw IP literals.
- A small, fixed, non-configurable blocklist (cloud metadata IPs, loopback addresses, etc.) always applies regardless of this allowlist — it does not affect ordinary internal LAN/DNS hosts.
- This same procedure applies any time a new self-hosted MCP server, custom model endpoint, or other private-IP destination needs to be reachable from the gateway.

## 5.3 Example: Adding `clm.yourdomain.lab`

Example, applied on 2026-09-30 in the original deployment — your date and `TRUSTED_CUSTOM_HOSTS` value will differ. Following the exact steps above produces a final value like:

```
mcphub.yourdomain.lab,clm.yourdomain.lab
```

## 5.4 Example: Adding `db-agent.yourdomain.lab` (A2A sub-agent)

Symptom before the fix, calling the sub-agent through the gateway's agent route:

```json
{"status":"failure","message":"Outbound request rejected by SSRF policy: db-agent.yourdomain.lab resolves to <subagent-lan-ip>, which is blocked"}
```

DNS already resolved, so only the allowlist was needed. Before running `helm upgrade`, confirm `values.yaml` still contains `images.gatewayImage.tag: "2.25.1"` if you've applied [doc 07](07-force-upgrading-gateway-image.md); without it the upgrade reverts the gateway to the chart default. Example final value, confirmed on both gateway pods in the original deployment:

```
mcphub.yourdomain.lab,clm.yourdomain.lab,db-agent.yourdomain.lab
```

If `printenv` still shows the old value after `helm upgrade`, the pods were not restarted: run `kubectl rollout restart deployment/airs-gw -n airs-gw`.
