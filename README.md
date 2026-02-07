# Showroom OCP Console Embed (GitOps)

Deploys Showroom with an embedded OpenShift Console tab using a **per-route** approach that strips `X-Frame-Options` and `Content-Security-Policy` headers from the console and OAuth routes only -- without modifying the IngressController or disabling operators.

## Problem: The Current Approach Is Dangerous

Today, teams use the `ocp4_workload_showroom_ocp_integration` Ansible role to embed the OCP console in Showroom. Here's what it actually does:

### Step 1 -- Strip Security Headers Cluster-Wide

Patches `IngressController/default` in `openshift-ingress-operator` to **delete** three response headers on **every route on the cluster**:
- `X-Frame-Options` -- clickjacking protection
- `Content-Security-Policy` -- resource loading control
- `referrer-policy` -- referrer information leakage

### Step 2 -- Kill the Authentication Operator

- Scales `authentication-operator` to 0 replicas
- Creates a `ResourceQuota` with `pods: "0"` to prevent CVO from restarting it

This is done because the auth operator would revert the OAuth route changes. Disabling it means the cluster can no longer self-heal its authentication configuration.

### Step 3 -- Reconfigure OAuth Routing

- Reads service CA from `ConfigMap/v4-0-config-system-service-ca`
- Deletes and recreates `oauth-openshift` route with `reencrypt` TLS

### Why This Is Unacceptable

| Problem | Detail | Impact |
|---------|--------|--------|
| **Cluster-wide header stripping** | `IngressController/default` applies to ALL routes | Every application on the cluster loses clickjacking protection. Prometheus, Grafana, AlertManager, internal APIs -- all lose CSP. |
| **Auth operator killed** | Scaled to 0 with ResourceQuota preventing restart | If OAuth breaks during the lab, there's no self-healing. CVO reports the cluster as degraded. |
| **No rollback on destroy** | `remove_workload.yml` is debug-only | After the lab ends, the cluster still has headers stripped and auth operator dead. Next user gets a weakened cluster. |
| **CVO conflict** | CVO tries to restore auth operator, gets blocked by ResourceQuota | Cluster reports degraded operator status, confusing for customers. |

```
┌─────────────────────────────────────────────────────────────────┐
│  BROWSER                                                        │
│                                                                 │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐ │
│  │ Left Pane            │  │ Right Pane (iframe tabs)         │ │
│  │ AsciiDoc lab content │  │                                  │ │
│  │ rendered by Antora   │  │ <iframe src="https://console-   │ │
│  │                      │  │  openshift-console.DOMAIN">     │ │
│  │                      │  │                                  │ │
│  │                      │  │ (direct browser fetch, NOT       │ │
│  │                      │  │  proxied through Showroom)       │ │
│  └──────────────────────┘  └───────────────┬──────────────────┘ │
└────────────────────────────────────────────┼─────────────────────┘
                                             │
              ┌──────────────────────────────┘
              │  Browser fetches console directly
              ▼
┌─────────────────────────────────────────────────────────────────┐
│  OpenShift IngressController (default)                          │
│                                                                 │
│  CURRENT HACK: httpHeaders.actions DELETE on ALL routes:        │
│    - X-Frame-Options     (clickjacking protection)              │
│    - Content-Security-Policy  (resource loading control)        │
│    - referrer-policy     (referrer leakage)                     │
│                                                                 │
│  ALSO: authentication-operator scaled to 0 + ResourceQuota      │
│        to prevent operator from reverting OAuth route changes    │
└─────────────────────────────────────────────────────────────────┘
```

## Solution: Per-Route Header Patching via GitOps

Instead of modifying the IngressController cluster-wide and killing operators, we patch **only the two routes that need it** using `spec.httpHeaders` on the route objects themselves.

### Deployment Flow

```
RHDP Order (ocp-field-asset-cnv)
  → GitOps Repo: github.com/prakhar1985/showroom-ocp-console-embed
  → Revision: main
  → Path: showroom/helm
        ↓
ArgoCD deploys App of Apps
  ├── App: showroom          (lab guide + terminal + OCP Console tab)
  └── App: ocp-console-embed (Job patches routes per-route)

Later: push to git → ArgoCD syncs → no re-order needed
```

### What the Embed Job Does

| Step | Target | Change |
|------|--------|--------|
| 1 | `Route/console` in `openshift-console` | Delete `X-Frame-Options` + `Content-Security-Policy` via `spec.httpHeaders` |
| 2 | `ConfigMap/v4-0-config-system-service-ca` in `openshift-authentication` | Read service CA cert |
| 3 | `Route/oauth-openshift` in `openshift-authentication` | Add reencrypt TLS with service CA + delete security headers |
| 4 | Verify | Poll console URL until `X-Frame-Options` is gone |

### What It Does NOT Do (vs the Current Insecure Role)

- Does NOT modify `IngressController/default` -- only the two specific routes
- Does NOT disable the authentication operator -- operator stays running and healthy
- Does NOT create ResourceQuotas blocking operator pods -- CVO stays happy
- `oc get clusterversion` stays clean, no degraded operators

### Old vs New Comparison

```
OLD (ocp4_workload_showroom_ocp_integration):
┌─────────────────────────────────────────┐
│  IngressController: "default"           │
│  httpHeaders.actions DELETE on ALL:     │
│    - X-Frame-Options                    │
│    - Content-Security-Policy            │
│    - referrer-policy                    │
│  auth-operator: KILLED (scale 0)        │
│  ResourceQuota: pods=0 (block restart)  │
│  Affects: EVERY route on the cluster    │
└─────────────────────────────────────────┘

NEW (this repo -- per-route patching):
┌─────────────────────────────────────────┐
│  IngressController: "default"           │
│  (UNTOUCHED - all security headers ON)  │
│                                         │
│  Route/console:                         │
│    spec.httpHeaders → DELETE X-Frame,   │
│                       DELETE CSP        │
│  Route/oauth-openshift:                 │
│    spec.httpHeaders → DELETE X-Frame,   │
│                       DELETE CSP        │
│    spec.tls → reencrypt with service CA │
│                                         │
│  auth-operator: RUNNING (untouched)     │
│  Affects: ONLY 2 routes                 │
└─────────────────────────────────────────┘
```

### Showroom Pod Internals

```
Pod: showroom
├── Init: git-cloner      → clones content repo to /showroom/repo
├── Init: antora-builder   → renders AsciiDoc to /showroom/www
│
├── Container: nginx       → port 8080 (reverse proxy)
│   ├── / → proxy_pass http://127.0.0.1:8000 (content)
│   └── /terminal/ → proxy_pass http://127.0.0.1:7681 (terminal, WebSocket)
│
├── Container: content     → port 8000 (showroom-content app)
│   ├── Serves rendered HTML with two-pane layout
│   ├── Reads ui-config.yml for tab definitions
│   ├── Substitutes ${DOMAIN} in tab URLs
│   └── Renders <iframe src="..."> for each tab
│
└── Container: terminal    → port 7681 (optional, ttyd-based)
```

## Deployment

Order from RHDP (`ocp-field-asset-cnv`) with:

| Field | Value |
|-------|-------|
| Repo URL | `https://github.com/prakhar1985/showroom-ocp-console-embed.git` |
| Revision | `main` |
| Path | `showroom/helm` |

## Configuration

Edit `showroom/helm/values.yaml` to:

- Change the Showroom content repo (`components.showroom.values.showroom.content.repoUrl`)
- Toggle components on/off (`components.showroom.enabled`, `components.ocpConsoleEmbed.enabled`)
- Set deployer domain (injected by RHDP at order time)

## Local Verification

```bash
helm template showroom/helm \
  --set deployer.domain=apps.cluster.example.com \
  --set demo.namespace=demo \
  --set demo.appName=myapp
```

## Post-Deploy Verification

```bash
# Check ArgoCD apps
oc get applications -n openshift-gitops

# Verify X-Frame-Options removed
curl -sI https://console-openshift-console.apps.DOMAIN | grep -i x-frame

# Confirm operators are healthy
oc get clusterversion
oc get pods -n openshift-authentication-operator
```

## Experiment Results

### Finding: Operators Reconcile Per-Route `spec.httpHeaders`

**Tested on:** OCP 4.x cluster via RHDP (`cluster-gpzh2.dynamic.redhatworkshops.io`)

We applied `spec.httpHeaders` patches directly to the `console` and `oauth-openshift` routes. The patches applied successfully, but within seconds the **console-operator** and **authentication-operator** reconciled their routes and **wiped the `httpHeaders` field**.

```
$ oc patch route console -n openshift-console --type=merge -p '{...httpHeaders...}'
route.route.openshift.io/console patched

$ oc get route console -n openshift-console -o jsonpath='{.spec.httpHeaders}'
  (empty -- operator reverted it)

$ curl -skI https://console-openshift-console.apps.DOMAIN | grep -i x-frame
x-frame-options: DENY    ← still there
```

### Why Dedicated IngressController Won't Work Either

A dedicated IngressController with route sharding (our first choice) has an **OAuth redirect problem**: the console's OAuthClient is hardcoded to redirect to `console-openshift-console.apps.DOMAIN`. Custom routes on a different IC domain (`*.embed.apps.DOMAIN`) would break the login flow. Modifying the OAuthClient gets reverted by the console-operator.

### Solution: CronJob

The ocp-console-embed component now uses a **CronJob** (every 2 minutes) that:

1. Checks if `spec.httpHeaders` is still present on both routes
2. Only re-patches if the operator has wiped it
3. Keeps the auth operator **running and healthy** (no kill, no ResourceQuota)

```
Operator reconciles route    →  httpHeaders wiped
CronJob runs (≤2 min later)  →  httpHeaders re-applied
Router reloads               →  X-Frame-Options gone from response
```

**Trade-off:** There's a brief window (up to 2 min) after operator reconciliation where headers are present and the iframe won't render. For a lab/demo environment this is acceptable. Adjust `schedule` in `values.yaml` to run more frequently if needed.

### Comparison of All Approaches

| Approach | Operators intact? | IngressController intact? | OAuth flow works? | Headers always stripped? |
|----------|:-:|:-:|:-:|:-:|
| **Old role** (cluster-wide IC + kill operator) | No | No | Yes | Yes |
| Per-route patch (one-shot Job) | Yes | Yes | Yes | No (reverted) |
| Dedicated IngressController + route sharding | Yes | Yes | **No** (OAuth redirect breaks) | Yes |
| **CronJob** (this repo) | Yes | Yes | Yes | ~Yes (≤2 min gap) |
