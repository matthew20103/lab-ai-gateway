# 0. Prerequisites: Account Setup & Lab Map

**Core lab** — required. Complete this before [01-ubuntu-server-preparation.md](01-ubuntu-server-preparation.md).

## 0.1 Create Your Own Strata Cloud Manager Account and AI Gateway Resource

Every student needs their **own** Strata Cloud Manager (SCM) account and their own AI Gateway resource — never share credentials or a `values.yaml` file with another student, and never reuse the instructor's.

1. Sign up / log in at Strata Cloud Manager and create an AI Runtime Security → AI Gateway resource. Follow the official onboarding guide: https://docs.paloaltonetworks.com/ai-runtime-security/administration/configure-ai-gateway
2. Once the resource exists, SCM generates a `values.yaml` file for it. Download this file — it's the one artifact every later doc in this lab refers to as `~/ai-gateway/values.yaml`.

## 0.2 What's Inside `values.yaml`

You don't need to understand every field yet (doc 02 covers the fields you'll edit), but know its shape before you see a real one. It looks roughly like this — **the values below are placeholders, not real credentials**:

```yaml
imageCredentials:
  - name: airsgatewayregistrycredentials
    create: true
    registry: "https://registry.portkey.ai"
    username: "<your-registry-username>"
    password: "<your-registry-password>"

environment:
  create: true
  secret: true
  data:
    PORTKEY_CLIENT_AUTH: "<your-client-auth-token>"
    ORGANISATIONS_TO_SYNC: "<your-org-id>"
    PORT: "8787"

replicaCount: 1

service:
  type: LoadBalancer
  port: 80
  containerPort: 8787
```

**The moment SCM generates your real file, it contains live secrets** — the registry password and `PORTKEY_CLIENT_AUTH` token are credentials, not configuration. Treat this file exactly like a password:

- `chmod 600` it as soon as it lands on your server (doc 01 §1.12 does this)
- Never commit it to git, paste it into a chat tool, attach it to an email, or post it anywhere
- Never hand it to another student or use a classmate's

## 0.3 Lab Map

| Doc | Status | Depends on |
|---|---|---|
| [01-ubuntu-server-preparation.md](01-ubuntu-server-preparation.md) | **Core — required** | This doc |
| [02-ai-gateway-deployment.md](02-ai-gateway-deployment.md) | **Core — required** | Doc 01 |
| [03-operations-and-testing.md](03-operations-and-testing.md) | **Core — required** | Doc 02 |
| [04-internal-mcp-server-connectivity-fix.md](04-internal-mcp-server-connectivity-fix.md) | Advanced / optional | Doc 03 |
| [05-adding-trusted-custom-hosts.md](05-adding-trusted-custom-hosts.md) | Advanced / optional | Doc 03 (doc 04 helpful but not required) |
| [06-known-limitations.md](06-known-limitations.md) | Advanced / optional | **Doc 07 first**, not doc 04/05 |
| [07-force-upgrading-gateway-image.md](07-force-upgrading-gateway-image.md) | Advanced / optional | Doc 03 |

Complete 01→02→03 in order — that's the whole required lab: a working 2-pod gateway behind a load balancer, serving real LLM traffic. Docs 04-07 are independent deep dives you can do in any order, except doc 06 (TypeSafe/decisions, A2A agent routing), which needs the chart force-upgrade from doc 07 first. MCP (docs 02 §2.9, 04, 05) does **not** need that upgrade.

Examples in the advanced docs originally came from one real deployment and use placeholder hostnames/IPs (e.g. `mcphub.yourdomain.lab`, `<gateway-lan-ip>`) — substitute your own throughout.
