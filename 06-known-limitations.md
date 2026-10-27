# 6. Known Limitations of the Self-Hosted Gateway

**Advanced / optional module.** Requires the chart force-upgrade in [doc 07](07-force-upgrading-gateway-image.md) first — this doc's `/v1/decisions` and `/v1/agent/*` features don't exist on the chart's default app version. MCP (docs 02 §2.9, 04, 05) does not need that upgrade.

This doc tracks features documented on Portkey's public docs site that are **not available** in the self-hosted `airs-gw-helm` chart, as confirmed by direct inspection of the deployed build. Examples below use placeholder hostnames/IPs and integration slugs — substitute your own.

## 6.1 TypeSafe (Jev) Decision Models / `/v1/decisions`

**Status as of gateway app `2.21.0`: not supported at all (route and provider type absent from the bundle).**

**Status as of gateway app `2.25.1` (force-upgraded per [07-force-upgrading-gateway-image.md](07-force-upgrading-gateway-image.md)): working, including against a self-hosted [Contrastive-LM (CLM)](https://github.com/Contrastive-LM/CLM) server.** A `TypeSafe`-type integration with a custom host pointing at CLM routes `/v1/decisions` to it — confirmed end-to-end in the original deployment. See §6.1.3.

### 6.1.1 On `2.21.0` — feature completely absent

The [TypeSafe integration docs](https://portkey.ai/docs/integrations/llms/typesafe) describe a `/v1/decisions` endpoint and a `TypeSafe`/`Jev` provider type for typed judgments (`choice`, `score`, `noul` question types). Testing against the self-hosted gateway on `2.21.0` failed at request-validation time, before reaching any decisions-specific logic:

```
{"status":"failure","message":"Invalid provider passed"}
```

Confirmed by direct inspection of the bundled server code (`/app/build/start-server.js` inside the gateway pod): the literal strings `"typesafe"`, `"jev"`, `"noul"`, and `"decisions"` did not appear anywhere in the bundle on `2.21.0`.

### 6.1.2 On `2.25.1` — route exists, provider-type check now passes

After upgrading to app `2.25.1`, the same string search finds `typesafe`, `jev`, and `/v1/decisions` all present, and `typesafe` is now included in the hardcoded provider-type allowlist. A saved Model Catalog integration whose provider type is literally `TypeSafe` will now pass the `/v1/decisions` route's checks.

### 6.1.3 Using `/v1/decisions` with a self-hosted CLM server

CLM's `clm-serve` exposes a TypeSafe-compatible API (`POST /v1/systemone`, `Authorization: Bearer <key>`, body `state`/`model`/`questions`, response `answers`), which is exactly what the gateway's bundled `typesafe` provider sends and expects:

```js
typesafe: {
  decisions: { state:{required}, model:{required, default:"jev-latest"}, questions:{required} },
  api: {
    getBaseURL: () => "https://api.typesafe.ai",
    headers: ({providerOptions:e}) => ({ Authorization: `Bearer ${e.apiKey}` }),
    getEndpoint: ({fn:e}) => "decisions"===e ? "/v1/systemone" : ""
  },
  ...
}
```

`getBaseURL` is hardcoded, but a custom host set on the integration overrides it (the override is applied generically, above the provider-specific code). So the gateway calls `<custom host>/v1/systemone`.

**Setup:**

1. `clm.yourdomain.lab` must be in `TRUSTED_CUSTOM_HOSTS` (done — see [05-adding-trusted-custom-hosts.md](05-adding-trusted-custom-hosts.md)).
2. In SCM Model Catalog, create an integration with:
   - **Provider type:** `TypeSafe` (any other type fails with `decisions is not supported by <type>`)
   - **Custom host:** the CLM server base URL, e.g. `http://clm.yourdomain.lab:8700` — **without** `/v1`, since the gateway appends `/v1/systemone`
   - **API key:** CLM's `CLM_API_KEY` (any placeholder if CLM runs without one)
3. Call it with an API key that has **no default Config attached** (see §6.1.4):

```bash
curl http://<gateway-lan-ip>/v1/decisions \
  -H "Content-Type: application/json" \
  -H "x-portkey-api-key: <gateway API key>" \
  -H "x-portkey-provider: @your-clm-integration" \
  -d '{
    "model": "clm-latest",
    "state": "User prompt: Write a Python function that parses a CSV file and sums a column",
    "questions": {
      "intent": {
        "type": "choice",
        "instructions": "What is the purpose of the user request?",
        "criteria": {
          "coding": "The user wants to write, debug, or understand code",
          "chatting": "The user wants casual conversation or general knowledge"
        }
      }
    }
  }'
```

Confirmed response (yours will show your own CLM server's values):

```json
{"model":"clm-latest","answers":{"intent":{"type":"choice","choice":"coding","confidence":0.918,"probabilities":{"coding":0.959,"chatting":0.041}}},"usage":{"billing_units":1,"input_tokens":45,"output_tokens":0},"provider":"typesafe"}
```

`"provider":"typesafe"` is the gateway's provider type label; `"model":"clm-latest"` confirms CLM answered (TypeSafe's cloud has no such model).

**Notes:**
- **Always send `"model"`.** If omitted, the gateway fills in `jev-latest`, which CLM rejects with `422`.
- **Only `state`, `model` and `questions` are forwarded.** CLM's optional `temperature` is dropped by the gateway's parameter map.
- **`/v1/rank`** (CLM's other endpoint) has no gateway route; call CLM directly if you need it.

### 6.1.4 Gotchas

**A default Config on the API key overrides `x-portkey-provider`.** If an API key has a default routing Config bound to it in SCM (e.g. one set up for LLM routing to Vertex AI/Gemini), that Config takes priority and silently overrides an explicit `x-portkey-provider: @slug` header — the request is routed to the Config's target with no error. **Always test with an API key that has no default Config attached** when validating a specific provider integration.

**Earlier `Method Not Allowed` error (cause unconfirmed).** Before the working setup above, a `TypeSafe`-type integration returned:

```json
{"error":{"message":"typesafe error: Method Not Allowed","type":null,"param":null,"code":null},"provider":"typesafe"}
```

This was originally attributed to the gateway sending GET instead of POST, but the bundle contradicts that: the `/v1/decisions` route handler passes `"POST"` explicitly (`$L(e, n, body, r, "decisions", "POST", "config")`), and the CLM setup in §6.1.3 works. The actual cause was not identified. If it recurs, check the integration's custom host value and the target server's access log for the method and path it received.

### Using TypeSafe.ai's hosted Jev models

Same as §6.1.3, but with a real TypeSafe API key, **no** custom host, and `"model": "jev-latest"`. Not tested end-to-end on this deployment.

## 6.2 A2A agent cards via `/v1/agent/<slug>/.well-known/agent-card.json`

**Status on app `2.25.1`: the gateway returns a near-empty card pointing at Portkey's cloud.**

```bash
curl -s -H "x-portkey-api-key: <gateway API key>" \
  http://<gateway-lan-ip>/v1/agent/<your-subagent-slug>/.well-known/agent-card.json
# {"url":"https://aigw.portkey.ai/v1/agent/<your-subagent-slug>"}
```

The sub-agent's own card (`http://db-agent.yourdomain.lab:8080/.well-known/agent-card.json`) has the full name, skills and security schemes.

**Cause (confirmed in `/app/build/start-server.js`):** requests for `/.well-known/agent-card.json` or `/.well-known/agent.json` are answered from the control plane (`controlPlane.getAgentCard(agentServerId)`) and are never forwarded to the sub-agent. All other paths, including JSON-RPC `message/send`, are forwarded to the sub-agent's real endpoint. The `aigw.portkey.ai` host matches the chart's default `CONTROL_PLANE_BASEPATH`.

**Do not override `CONTROL_PLANE_BASEPATH`** to fix the URL: the gateway uses it to push metrics (`/dp/metrics`) and logs (`/dp/logs`) to the control plane.

**Workaround:** A2A clients that need a card should use a local copy of the sub-agent's real card with `url` and every `supportedInterfaces[].url` set to `http://<gateway-lan-ip>/v1/agent/<slug>`, and send `x-portkey-api-key` on each request. Clients that call `message/send` directly don't need the card.
