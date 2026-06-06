# homelab

Kubernetes (AKS) homelab infrastructure. All first-party manifests are plain YAML applied with `kubectl apply -f`; Helm is used only for third-party charts (CrowdSec, Authentik).

Public site: [seaze.dev](https://seaze.dev)

## Traffic flow

```
Internet
  └─> Envoy Gateway (TLS terminate, :80/:443)
        └─> CrowdSec SecurityPolicy on the Gateway  ← screens ALL ingress first
              (ext-auth gRPC: IP bouncer + AppSec WAF)
                ├─> Public    (seaze.dev/)          → Anubis (bot PoW) → Homepage
                ├─> Protected (seaze.dev/dashboard) → Entra ID OIDC    → Homepage
                └─> Auth      (auth.seaze.dev)      → Authentik server
```

CrowdSec's `SecurityPolicy` is attached to the **Gateway**, so its IP bouncer + AppSec WAF screen all ingress *before* Envoy routes to any backend — it sits in front of Anubis, not behind it. A second `SecurityPolicy` on the `/dashboard` route enforces Entra ID OIDC via Authentik's embedded outpost. TLS is terminated with cert-manager-issued Let's Encrypt certs.

## Secrets

All secrets come from **Azure Key Vault** (`kv-homelab-matt`) via the Secrets Store CSI driver. Each namespace that needs secrets has a `SecretProviderClass` in [`cluster/secretprovider.yml`](cluster/secretprovider.yml). No secrets are committed to this repo.

## Layout

| Path | Purpose |
|------|---------|
| `gateway/` | GatewayClass, Gateway, ReferenceGrants |
| `cluster/` | HTTPRoutes, SecretProviderClasses, cert-manager Certificates |
| `security/anubis/` | Anubis bot-protection deployment |
| `security/crowdsec/` | CrowdSec Helm values + SecurityPolicy |
| `security/authentik/` | Authentik Helm values |
| `apps/login/` | Public login landing page (Caddy) |
| `apps/homepage/` | Protected dashboard (Caddy) |
| `apps/bible-bot/` | Discord bot |
| `apps/postgres/` | PostgreSQL StatefulSet backing Authentik |

## Applying

```bash
kubectl apply -f <path>        # single file
kubectl apply -f <directory>/  # whole directory
kubectl apply --dry-run=client -f <path>   # validate first
```

> **Note:** `cluster/certificates.yaml` references a `letsencrypt-prod` ClusterIssuer that is provisioned separately (not tracked here).

## Secret scanning

No secrets live in this repo — they all come from Azure Key Vault. To keep it that way, a pre-commit hook scans staged changes with [betterleaks](https://github.com/betterleaks/betterleaks) and blocks any commit containing a detected secret.

Enable it after cloning (one-time, per clone):

```bash
# 1. Install betterleaks (Linux x64 example; see repo for brew/docker/other arches)
VER=1.4.1
curl -fsSL "https://github.com/betterleaks/betterleaks/releases/download/v${VER}/betterleaks_${VER}_linux_x64.tar.gz" \
  | tar -xz -C /tmp betterleaks && install -m0755 /tmp/betterleaks ~/.local/bin/betterleaks

# 2. Point git at the tracked hooks dir
git config core.hooksPath .githooks
```

The hook lives at [`.githooks/pre-commit`](.githooks/pre-commit). It warns (rather than blocking) if betterleaks isn't installed, so commits aren't bricked on a fresh clone — install it to get real protection.

This is the local half of a belt-and-suspenders setup; the remote half is **GitHub secret scanning + push protection** (Settings → Code security), which should also be enabled once the repo is on GitHub.

## Commit conventions

Commits follow [Conventional Commits](https://www.conventionalcommits.org):

```
<type>(<optional scope>): <description>
```

| Type | Use for |
|------|---------|
| `feat` | a new app, manifest, or capability |
| `fix` | a bug fix |
| `docs` | documentation only |
| `refactor` | restructuring with no behavior change |
| `chore` | tooling, deps, housekeeping |
| `ci` | CI / automation changes |

Scope is optional and usually the area touched — e.g. `gateway`, `crowdsec`, `authentik`, `homepage`, `bible-bot`, `postgres`.

Examples:

```
feat(bible-bot): pull OpenAI key via init container
fix(gateway): correct TLS hostname on the auth listener
docs: document betterleaks pre-commit setup
chore(crowdsec): bump appsec collection version
```

See [`CLAUDE.md`](CLAUDE.md) for full architecture notes.
