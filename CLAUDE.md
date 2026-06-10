# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Kubernetes (AKS) homelab infrastructure repo. All manifests are plain YAML applied with `kubectl apply -f`. There is no Helm for first-party manifests; Helm is used only for third-party charts (CrowdSec, Authentik) with values files in this repo.

## Applying manifests

```bash
# Apply a single file
kubectl apply -f <path>

# Apply an entire directory
kubectl apply -f <directory>/

# Dry-run before applying
kubectl apply --dry-run=client -f <path>
```

## Architecture

### Traffic flow

```
Internet
  └─> Envoy Gateway (envoy-gateway-system, ports 80/443, TLS terminate)
        └─> CrowdSec SecurityPolicy on the Gateway  ← screens ALL ingress first
              (ext-auth gRPC: IP bouncer + AppSec WAF)
                ├─> Public route    (seaze.dev/)           → Anubis (bot PoW) → Homepage
                ├─> Blog route      (blog.seaze.dev)       → Anubis (bot PoW) → Blog
                ├─> Protected route (dashboard.seaze.dev)  → Authentik OIDC   → Homepage
                └─> Auth route      (auth.seaze.dev)       → Authentik server
```

The gateway terminates TLS with cert-manager-issued Let's Encrypt certs. CrowdSec's `SecurityPolicy` is attached to the **Gateway**, so its ext-auth (IP bouncer + AppSec WAF) runs against all ingress *before* Envoy routes to any backend — i.e. it sits in front of Anubis, not after it. A second `SecurityPolicy` on the protected HTTPRoute enforces Entra ID OIDC (via the Authentik embedded outpost) for `dashboard.seaze.dev`. The old `seaze.dev/dashboard` path 301-redirects to the subdomain.

> Note: when a Gateway-scoped and a Route-scoped `SecurityPolicy` overlap on the same route, Envoy Gateway's policy-attachment precedence may apply the more specific (route) policy rather than chaining both. Confirm on-cluster that CrowdSec still applies to `dashboard.seaze.dev` and isn't shadowed by the Authentik policy.

### Secrets

All secrets come from **Azure Key Vault** (`kv-homelab-matt`) via the Secrets Store CSI driver. Each namespace that needs secrets has a `SecretProviderClass` in `cluster/secretprovider.yml` that mounts vault secrets as Kubernetes `Secret` objects using a user-assigned managed identity (`b5d4d0bb-da13-4f5f-85e8-a0c6dc2f221e`).

Never hardcode secrets; always add a new entry to `cluster/secretprovider.yml`.

### Directory layout

| Path | Purpose |
|------|---------|
| `gateway/` | GatewayClass, Gateway, ReferenceGrants |
| `cluster/` | HTTPRoutes, SecretProviderClasses, cert-manager Certificates |
| `security/anubis/` | Anubis bot-protection deployment (raw manifests) |
| `security/crowdsec/` | CrowdSec Helm values + SecurityPolicy |
| `security/authentik/` | Authentik Helm values (external Postgres, Redis bundled) |
| `apps/login/` | Public login landing page (Caddy serving inline HTML) |
| `apps/homepage/` | Protected dashboard (Caddy serving inline HTML) |
| `apps/bible-bot/` | Discord bot pulling secrets via init container |
| `apps/postgres/` | PostgreSQL StatefulSet backing Authentik |

### Static HTML pattern

Both the login page and homepage embed their full HTML inside a `ConfigMap` within the same YAML file as the Deployment. The Caddy container mounts the ConfigMap as files. Editing UI means editing the `index.html` key in the respective ConfigMap.

### Namespaces

Each app runs in its own namespace (`anubis`, `crowdsec`, `authentik`, `homepage`, `login-page`, `bible-bot`, `postgres`). Cross-namespace service references require a `ReferenceGrant` in `gateway/reference_grant.yml`.

### Hostnames

Primary domain is **`seaze.dev`** (public site) with **`auth.seaze.dev`** for the Authentik instance. Both resolve to the Azure public IP in Central US. The original `homelab-matt.centralus.cloudapp.azure.com` cloudapp name is kept as a legacy alias (still served by `homelab-route-public-legacy`).

### Container registry

Custom images are pushed to `acrhomelabmatt.azurecr.io` (e.g., `bible-bot:latest`).
