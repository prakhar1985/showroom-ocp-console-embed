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

## Solution: Dedicated IngressController with Route Sharding

Instead of modifying the default IngressController or patching operator-managed routes, we create a **second IngressController** that strips headers and serve custom routes through it.

### Deployment Flow

```
RHDP Order (ocp-field-asset-cnv)
  → GitOps Repo: github.com/prakhar1985/showroom-ocp-console-embed
  → Revision: main
  → Path: showroom/helm
        ↓
ArgoCD deploys App of Apps
  ├── App: showroom          (lab guide + terminal + OCP Console tab)
  └── App: ocp-console-embed
        ├── IngressController/showroom-embed  (strips headers, serves labeled routes)
        ├── Job: create Route/console-embed   (custom route, label: showroom-embed=true)
        ├── Job: create Route/oauth-embed     (custom route, label: showroom-embed=true)
        └── Job: patch OAuthClient/console    (add embed redirect URI)

Later: push to git → ArgoCD syncs → no re-order needed
```

### How It Works

```
┌─────────────────────────────────────────────────┐
│  IngressController: "default"                   │
│  (UNTOUCHED -- all security headers ON)         │
│  Domain: *.apps.DOMAIN                          │
│  Serves: all routes (no routeSelector)          │
│  Console, OAuth, Prometheus, etc. keep headers  │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  IngressController: "showroom-embed"            │
│  Domain: *.embed.apps.DOMAIN                    │
│  routeSelector:                                 │
│    matchLabels:                                 │
│      showroom-embed: "true"                     │
│  httpHeaders.actions:                           │
│    DELETE X-Frame-Options                       │
│    DELETE Content-Security-Policy               │
│  Serves: ONLY labeled routes                    │
│                                                 │
│  Routes served:                                 │
│    console-embed  → console service             │
│    oauth-embed    → oauth-openshift service     │
└─────────────────────────────────────────────────┘
```

### What the Setup Job Does

| Step | Target | Change |
|------|--------|--------|
| 1 | `IngressController/showroom-embed` | Wait for it to become Available (created declaratively) |
| 2 | `ConfigMap/v4-0-config-system-service-ca` | Read service CA cert for reencrypt TLS |
| 3 | `Route/console-embed` in `openshift-console` | Create custom route with label `showroom-embed=true`, reencrypt TLS |
| 4 | `Route/oauth-embed` in `openshift-authentication` | Create custom route with label `showroom-embed=true`, reencrypt TLS + service CA |
| 5 | `OAuthClient/console` | Add embed redirect URI so OAuth login works via the embed hostname |
| 6 | Verify | Poll embed console URL until responding without `X-Frame-Options` |

### What It Does NOT Do (vs the Current Insecure Role)

- Does NOT modify `IngressController/default` -- default IC stays untouched
- Does NOT modify operator-managed routes -- custom routes are independent
- Does NOT disable the authentication operator -- operator stays running
- Does NOT create ResourceQuotas blocking operator pods -- CVO stays happy
- `oc get clusterversion` stays clean, no degraded operators

### Why This Works (and per-route patching doesn't)

Operators reconcile routes they **own** (`Route/console`, `Route/oauth-openshift`) and wipe any custom `spec.httpHeaders`. But operators do NOT touch:
- Custom routes they didn't create (`Route/console-embed`, `Route/oauth-embed`)
- The IngressController's `spec.httpHeaders` (managed by ingress-operator, which preserves user config)
- Additional entries in OAuthClient `redirectURIs` (needs testing)

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

NEW (this repo -- dedicated IngressController):
┌─────────────────────────────────────────┐
│  IngressController: "default"           │
│  (UNTOUCHED - all security headers ON)  │
│                                         │
│  IngressController: "showroom-embed"    │
│    DELETE X-Frame-Options               │
│    DELETE Content-Security-Policy       │
│    routeSelector: showroom-embed=true   │
│                                         │
│  Route/console-embed → console svc      │
│  Route/oauth-embed   → oauth svc       │
│  (custom routes, not operator-managed)  │
│                                         │
│  auth-operator: RUNNING (untouched)     │
│  Affects: ONLY embed routes             │
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

# Verify embed IngressController is running
oc get ingresscontroller showroom-embed -n openshift-ingress-operator

# Verify embed routes exist
oc get route console-embed -n openshift-console
oc get route oauth-embed -n openshift-authentication

# Verify X-Frame-Options removed on embed route (NOT the original)
curl -skI https://console-openshift-console.embed.apps.DOMAIN | grep -i x-frame

# Original routes should still have headers (untouched)
curl -skI https://console-openshift-console.apps.DOMAIN | grep -i x-frame

# Confirm operators are healthy
oc get clusterversion
oc get pods -n openshift-authentication-operator
```

## Experiment Log

### Attempt 1: Per-Route `spec.httpHeaders` (Failed)

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

**Conclusion:** Per-route httpHeaders on operator-managed routes do not survive reconciliation.

### Attempt 2: Dedicated IngressController (Current Approach)

Instead of modifying operator-managed routes, we create **custom routes** (`console-embed`, `oauth-embed`) on a **dedicated IngressController** (`showroom-embed`). Operators don't know about these routes and won't reconcile them. The IC-level header stripping handles the rest.

**Open question:** Does the console-operator reconcile the `console` OAuthClient's `redirectURIs` and remove our embed entry? Needs testing.

### Comparison of All Approaches

| Approach | Operators intact? | Default IC intact? | OAuth flow works? | Headers always stripped? |
|----------|:-:|:-:|:-:|:-:|
| **Old role** (cluster-wide IC + kill operator) | No | No | Yes | Yes |
| Per-route patch (one-shot Job) | Yes | Yes | Yes | No (reverted) |
| **Dedicated IC + custom routes** (this repo) | Yes | Yes | Needs testing | Yes |
