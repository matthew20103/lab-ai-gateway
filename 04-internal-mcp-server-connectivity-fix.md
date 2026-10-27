# 4. Connecting the AI Gateway to an Internal-Only MCP Server

**Advanced / optional module** — do this only after completing the core lab ([01-03](00-prerequisites.md#03-lab-map)). Does not require the chart force-upgrade in [doc 07](07-force-upgrading-gateway-image.md).

This guide documents the fix required to let the AI Gateway reach a **self-hosted MCP server on an internal-only hostname** — in this example, `mcphub.yourdomain.lab`, which resolves only via your internal DNS server (`<internal-dns-ip>`) to a private LAN address (`<mcp-hub-lan-ip>`) and cannot be resolved from the public internet. Substitute your own domain and IPs throughout.

This is a two-part fix. Both parts are required — fixing only one will not restore connectivity:

1. **DNS resolution** — the gateway pods must be able to resolve the internal-only hostname.
2. **SSRF allowlisting** — the gateway's built-in SSRF protection blocks outbound requests to hostnames that resolve to private IP addresses, unless the hostname is explicitly trusted.

## 4.1 Symptom

A self-hosted MCP server (e.g. registered with slug `openserp`, backend `http://mcphub.yourdomain.lab:8080/openserp/mcp`) is registered in the AI Gateway's MCP tool registry, but the MCP server's own Nginx access log shows **no incoming traffic** from the gateway at all — the gateway never even reaches it.

## 4.2 Part 1 — Pod DNS Resolution for Internal-Only Hostnames

### Root cause

k3s's CoreDNS forwards unresolved queries to the host's `/etc/resolv.conf`, which must point at a DNS server that can resolve your internal domain (here, `<internal-dns-ip>`).

Even with the host's DNS correctly pointed at the internal resolver, pod-level lookups can still fail. The gateway image is Alpine Linux (musl libc), and musl's `getaddrinfo()` performs a **combined A + AAAA lookup**. If the upstream DNS server returns `NXDOMAIN` for the AAAA sub-query (instead of the RFC-correct `NOERROR` with an empty answer, for a name that only has an A record), musl fails the **entire** combined lookup — even though the A record resolved successfully. This is exactly the behavior of the internal DNS server at `<internal-dns-ip>` for `mcphub.yourdomain.lab`.

### Fix

**Step 1 — confirm the Ubuntu host resolves the internal domain**, i.e. its netplan DNS configuration includes the internal DNS server:

```bash
resolvectl status | grep -A3 "DNS Servers"
# Expect <internal-dns-ip> listed
cat /etc/resolv.conf
```

If not already set, update netplan to point at `<internal-dns-ip>` and apply with `sudo netplan apply`.

**Step 2 — make CoreDNS answer AAAA queries cleanly instead of relaying the internal DNS server's `NXDOMAIN`.** k3s supports a `coredns-custom` ConfigMap in `kube-system` for exactly this kind of override, without editing the main (`# File managed by k3s. DO NOT EDIT.`) CoreDNS ConfigMap:

```bash
kubectl create configmap coredns-custom -n kube-system \
  --from-literal=aaaa.override="template ANY AAAA . {
    rcode NOERROR
}"

kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system
```

This makes CoreDNS itself answer every AAAA query cluster-wide with an empty `NOERROR` ("no such record type, but the name exists") instead of forwarding it and relaying the internal DNS server's non-compliant `NXDOMAIN`. This is safe cluster-wide since the gateway environment has no IPv6 connectivity to use AAAA records for anyway.

### Verify

```bash
POD=$(kubectl get pod -n airs-gw -l app.kubernetes.io/name=airs-gw -o jsonpath="{.items[0].metadata.name}")
kubectl exec -n airs-gw $POD -- node -e "require('dns').lookup('mcphub.yourdomain.lab',(e,a)=>console.log(e||a))"
```

Expected output: the resolved IP (e.g. `<mcp-hub-lan-ip>`), with no error.

## 4.3 Part 2 — SSRF Allowlist for Private-IP Destinations

### Root cause

Even once DNS resolves correctly, the gateway has a built-in SSRF (Server-Side Request Forgery) protection policy that blocks any outbound request whose target hostname resolves to a private/internal IP address. This is visible directly in the gateway pod logs:

```
warn: SSRF block: DNS resolution rejected {"addresses":"<mcp-hub-lan-ip>","hostname":"mcphub.yourdomain.lab"}
[MCP-Handler] [ERROR] Failed to initialize session ... request to http://mcphub.yourdomain.lab:8080/openserp/mcp failed,
reason: Outbound request rejected by SSRF policy: mcphub.yourdomain.lab resolves to <mcp-hub-lan-ip>, which is blocked
```

This protection cannot be disabled outright, but the gateway supports an explicit hostname allowlist via the `TRUSTED_CUSTOM_HOSTS` environment variable. A hostname listed there is exempted from the private-IP block. (A small, separate, non-configurable blocklist — cloud metadata IPs, `127.x` addresses, etc. — always applies regardless of this allowlist, but does not affect ordinary internal LAN hosts like this one.)

### Fix

Add the internal MCP hub hostname to `TRUSTED_CUSTOM_HOSTS` in `values.yaml`, under `environment.data`:

```yaml
environment:
  create: true
  secret: true
  data:
    PORTKEY_CLIENT_AUTH: "..."
    ORGANISATIONS_TO_SYNC: "..."
    PORT: "8787"
    MCP_GATEWAY_BASE_URL: "http://<gateway-lan-ip>:8788"
    TRUSTED_CUSTOM_HOSTS: "mcphub.yourdomain.lab"
```

`TRUSTED_CUSTOM_HOSTS` accepts a **comma-separated list** of hostnames, and supports a `*.` wildcard prefix to trust an entire subdomain (e.g. `*.yourdomain.lab`). Add further internal hosts as a comma-separated list, e.g. `TRUSTED_CUSTOM_HOSTS: "mcphub.yourdomain.lab,other-internal-host.example.com"`.

Apply with Helm:

```bash
helm repo update airs-gw
helm upgrade airs-gw airs-gw/airs-gw -f ~/ai-gateway/values.yaml -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

### Verify

```bash
POD=$(kubectl get pod -n airs-gw -l app.kubernetes.io/name=airs-gw -o jsonpath="{.items[0].metadata.name}")
kubectl exec -n airs-gw $POD -- printenv TRUSTED_CUSTOM_HOSTS
# Expect: mcphub.yourdomain.lab
```

## 4.4 End-to-End Test

1. Tail gateway logs in one terminal:
   ```bash
   kubectl logs -n airs-gw -l app.kubernetes.io/name=airs-gw -f --prefix
   ```
2. Trigger the MCP call again (e.g. `tools/list` or `initialize`) against the registered MCP route for the internal server.
3. Confirm no `SSRF block` line appears in the gateway logs for that request.
4. Confirm the MCP server's own Nginx access log now shows an incoming request from the gateway.

## 4.5 When You'll Need This Again

Any future MCP server, model provider, or webhook target that is:
- hosted on an internal-only domain (not resolvable from the public internet), **and/or**
- reachable only via a private IP address,

will need the same two-part fix: confirm/extend DNS resolution (Part 1), and add its hostname to `TRUSTED_CUSTOM_HOSTS` (Part 2).
