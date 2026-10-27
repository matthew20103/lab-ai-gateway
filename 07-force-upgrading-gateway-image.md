# 7. Force-Upgrading the Gateway Image Version

**Advanced / optional module** — only needed if you plan to continue to [06-known-limitations.md](06-known-limitations.md) (TypeSafe/decisions, A2A agent routing). MCP (docs 02 §2.9, 04, 05) does not need this upgrade.

This guide documents overriding the `airs-gw-helm` chart's pinned gateway image tag to a newer app version than the chart officially ships — used here to go from chart `1.2.0`'s default app `2.21.0` to app `2.25.1`, in order to pick up TypeSafe/Jev `/v1/decisions` support (see [06-known-limitations.md](06-known-limitations.md) §6.1).

**This is an unsupported/untested chart+app combination.** Chart `1.2.0` was only ever published paired with app `2.21.0`. Treat this as a deliberate risk trade-off, not a routine upgrade, and follow the rollback plan below if anything looks wrong after applying it.

## 7.1 Feasibility Checks (do these before editing anything)

**Check 1 — confirm the chart exposes an image-tag override field:**

```bash
helm show values airs-gw/airs-gw --version 1.2.0 | grep -i -B2 -A5 "image"
```

Look for `images.gatewayImage.tag`. The chart's own comment (`# Minimum supported version is 2.15.0`) confirms this field is intended to be user-overridable within a range.

**Check 2 — confirm the target tag actually exists in the registry and pulls with your existing credentials.** Run a throwaway pod in the **same namespace as the real deployment** (the image-pull secret only exists there) and poll until it reaches a terminal state:

```bash
kubectl run registry-check -n airs-gw \
  --image=registry.portkey.ai/airsgw/gateway_enterprise:2.25.1 \
  --overrides='{"spec":{"imagePullSecrets":[{"name":"airsgatewayregistrycredentials"}]}}' \
  --restart=Never -- sleep 3600

for i in $(seq 1 30); do
  phase=$(kubectl get pod registry-check -n airs-gw -o jsonpath='{.status.phase}' 2>/dev/null)
  reason=$(kubectl get pod registry-check -n airs-gw -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null)
  echo "check $i: phase=$phase reason=$reason"
  [ "$phase" = "Running" ] || [ "$phase" = "Succeeded" ] && break
  [ "$reason" = "ImagePullBackOff" ] || [ "$reason" = "ErrImagePull" ] && break
  sleep 3
done

kubectl describe pod registry-check -n airs-gw | grep -A5 Events
kubectl delete pod registry-check -n airs-gw
```

A `Successfully pulled image` event confirms the tag exists and is pullable. Both checks passed before proceeding here.

## 7.2 Backup Current State (rollback safety net)

```bash
mkdir -p ~/ai-gateway/backups
helm get values airs-gw -n airs-gw -o yaml > ~/ai-gateway/backups/pre-2.25.1-values.yaml
helm history airs-gw -n airs-gw
```

Note the current revision number — that's the rollback target if the upgrade misbehaves.

## 7.3 Edit `values.yaml`

Add a top-level `images` block with just the field being overridden — Helm deep-merges this with the chart's defaults, so `repository`, `pullPolicy`, and every other image (Redis, MinIO, etcd, Milvus) are untouched:

```yaml
images:
  gatewayImage:
    tag: "2.25.1"
```

## 7.4 Apply

```bash
helm repo update airs-gw
helm upgrade airs-gw airs-gw/airs-gw -f ~/ai-gateway/values.yaml -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

## 7.5 Verify

```bash
kubectl get pods -n airs-gw -o wide
kubectl get deployment airs-gw -n airs-gw -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# Expect: registry.portkey.ai/airsgw/gateway_enterprise:2.25.1

kubectl logs -n airs-gw -l app.kubernetes.io/name=airs-gw --tail=100
# Watch for crash loops, startup errors, config-schema complaints

POD=$(kubectl get pod -n airs-gw -l app.kubernetes.io/name=airs-gw -o jsonpath="{.items[0].metadata.name}")
kubectl exec -n airs-gw $POD -- wget -qO- http://localhost:8787/v1/health
# Expect: {"status":"success","message":"Server is healthy","version":"2.25.1"}
```

**Confirmed working** — the upgrade applied cleanly, both pods came up on `2.25.1`, and `/v1/health` reports the new version.

## 7.6 Rollback (if the upgrade misbehaves)

```bash
helm rollback airs-gw <revision-from-7.2> -n airs-gw
kubectl rollout status deployment/airs-gw -n airs-gw
```

## 7.7 Caveat

A successful pull and clean rollout prove the image exists and starts — they do **not** prove full compatibility with this chart's env-var set, Redis version pin (`7.2-alpine`), or config schema. Watch logs closely for the first few minutes after any future re-application of this override (e.g. after a `helm upgrade` that resets to chart defaults), and re-verify functionality (per [06-known-limitations.md](06-known-limitations.md)) rather than assuming it silently continues to work across chart upgrades.
