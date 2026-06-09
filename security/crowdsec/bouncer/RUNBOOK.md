# CrowdSec Envoy bouncer — vendor-into-ACR cutover

Replaces the third-party `kdwils/envoy-proxy-bouncer` Helm release with a vendored
image in `acrhomelabmatt.azurecr.io` + the self-rendered manifests in this dir.
This touches the **live ext_authz / WAF path** (`securitypolicy.yml` is `failOpen:false`),
so the cutover briefly denies ingress — **do it in a low-traffic maintenance window**.

All 5 build-time unknowns are resolved (verified against upstream at tag `v0.6.1`):
image tag `v0.6.1`, Go 1.26, build `.`, entrypoint `serve --config /app/config/config.yaml`,
gRPC health services `liveness`/`readiness`, env `ENVOY_BOUNCER_{BOUNCER,WAF}_APIKEY`.

## 1. Build the vendored image into ACR (no cluster impact)
Builds from the upstream repo at the pinned tag using its own (distroless, nonroot) Dockerfile:
```bash
az acr build -r acrhomelabmatt -t envoy-proxy-bouncer:v0.6.1 \
  "https://github.com/kdwils/envoy-proxy-crowdsec-bouncer.git#v0.6.1"

# Capture the digest and PIN it in deployment.yaml:
az acr repository show -n acrhomelabmatt \
  --image envoy-proxy-bouncer:v0.6.1 --query digest -o tsv
# -> set image: acrhomelabmatt.azurecr.io/envoy-proxy-bouncer@sha256:<digest>
```

## 2. Pre-validate the new pod WITHOUT touching the live bouncer (no impact)
Prove the image + config + gRPC probes actually produce a Ready, enforcing pod
before the cutover, so the live swap is just a quick replace of a known-good build:
```bash
# Render the manifests under TEMP names so they don't collide with the live ones:
kustomize build security/crowdsec/bouncer \
  | sed 's/crowdsec-envoy-bouncer/cs-bouncer-canary/g' \
  | kubectl apply -f -
kubectl -n envoy-gateway-system rollout status deploy/cs-bouncer-canary --timeout=60s
kubectl -n envoy-gateway-system logs deploy/cs-bouncer-canary | tail
# Confirm it reached Ready (gRPC health OK) and logs show LAPI+AppSec connected, then clean up:
kubectl -n envoy-gateway-system delete deploy/cs-bouncer-canary svc/cs-bouncer-canary \
  cm -l app.kubernetes.io/name=cs-bouncer-canary 2>/dev/null
```
If the canary did NOT go Ready, STOP — do not cut over. Fix and re-validate.

## 3. Wire into Flux (reviewed via PR) with reconcile paused
Suspend so the merge doesn't race the live Helm release, then merge:
```bash
flux suspend kustomization infra-security        # pause; live bouncer keeps running
# add `crowdsec/bouncer` to security/kustomization.yaml, commit (signed), PR, CI, merge
```

## 4. Cutover (maintenance window — brief ingress denial)
```bash
helm uninstall crowdsec-envoy-bouncer -n envoy-gateway-system   # removes the old chart's Deploy/Svc
flux resume kustomization infra-security
flux reconcile kustomization infra-security --with-source       # Flux applies the vendored set now
kubectl -n envoy-gateway-system rollout status deploy/crowdsec-envoy-bouncer --timeout=90s
```

## 5. Verify enforcement (do NOT skip — failOpen:false hides a dead bouncer as "allow")
```bash
kubectl -n envoy-gateway-system logs deploy/crowdsec-envoy-bouncer | tail
# IP bouncer: add a test decision in LAPI, confirm a 403 from that IP.
# WAF: send a known AppSec-tripping request, confirm a 403.
# Confirm normal traffic still flows (no false self-DoS).
```

## 6. Rollback (fast path back to the known-good chart)
```bash
flux suspend kustomization infra-security
kubectl -n envoy-gateway-system delete deploy/crowdsec-envoy-bouncer svc/crowdsec-envoy-bouncer
helm install crowdsec-envoy-bouncer oci://ghcr.io/kdwils/charts/envoy-proxy-bouncer \
  --version 0.6.1 -n envoy-gateway-system -f security/crowdsec/bouncer-values.yml
# revert the security/kustomization.yaml reference on main, then:
flux resume kustomization infra-security
```

## Notes
- api-key stays in secret `crowdsec-envoy-bouncer-secrets` (Key Vault). Never in Git.
- Keep `bouncer-values.yml` until the cutover is proven — it's the rollback input.
- The digest-pinned image makes the running version immutable; re-run step 1 to bump.
