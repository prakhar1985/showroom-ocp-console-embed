# Showroom OCP Console Embed (GitOps)

Deploys Showroom with an embedded OpenShift Console tab by stripping `X-Frame-Options` and `Content-Security-Policy` headers from the default IngressController and patching the OAuth route with reencrypt TLS -- all operators stay running, no ResourceQuotas, no CVO degradation.

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
│  OLD HACK: httpHeaders.actions DELETE on ALL routes:            │
│    - X-Frame-Options     (clickjacking protection)              │
│    - Content-Security-Policy  (resource loading control)        │
│    - referrer-policy     (referrer leakage)                     │
│                                                                 │
│  ALSO: authentication-operator scaled to 0 + ResourceQuota      │
│        to prevent operator from reverting OAuth route changes    │
└─────────────────────────────────────────────────────────────────┘
```

## Solution: Patch Default IngressController + OAuth Route

On single-node RHDP clusters (which is the standard deployment), we patch the **default IngressController** to strip iframe-blocking headers and patch the **OAuth route** with reencrypt TLS. This is the same header stripping as the old approach, but **without killing the auth operator or creating ResourceQuotas**.

### Deployment Flow

```
RHDP Order (ocp-field-asset-cnv)
  → GitOps Repo: github.com/prakhar1985/showroom-ocp-console-embed
  → Revision: main
  → Path: showroom/helm
        ↓
ArgoCD deploys App of Apps
  ├── App: showroom              (lab guide + terminal + OCP Console tab)
  └── App: ocp-console-embed
        ├── RBAC                 (SA + ClusterRole + ClusterRoleBinding)
        └── Job (PostSync):
              ├── Patch IngressController/default (strip X-Frame-Options + CSP)
              ├── Read service CA cert
              ├── Patch Route/oauth-openshift (reencrypt TLS + service CA)
              └── Verify headers stripped

Later: push to git → ArgoCD syncs → no re-order needed
```

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  IngressController: "default"                                    │
│                                                                  │
│  httpHeaders.actions:                                            │
│    DELETE X-Frame-Options                                        │
│    DELETE Content-Security-Policy                                │
│                                                                  │
│  Route/oauth-openshift patched with reencrypt TLS + service CA   │
│                                                                  │
│  auth-operator: RUNNING (untouched)                              │
│  CVO: healthy, no degraded operators                             │
│  No ResourceQuotas blocking anything                             │
└─────────────────────────────────────────────────────────────────┘
```

### What the Setup Job Does

| Step | Target | Change |
|------|--------|--------|
| 1 | `IngressController/default` | Patch `spec.httpHeaders.actions` to delete `X-Frame-Options` + `Content-Security-Policy` |
| 2 | `ConfigMap/v4-0-config-system-service-ca` | Read service CA cert for reencrypt TLS |
| 3 | `Route/oauth-openshift` | Patch with `reencrypt` TLS termination + service CA certificate |
| 4 | Verify | Poll console URL until responding without `X-Frame-Options` |

### What It Does NOT Do (vs the Old Insecure Role)

- Does NOT disable the authentication operator -- operator stays running
- Does NOT create ResourceQuotas blocking operator pods -- CVO stays happy
- Does NOT delete/recreate the OAuth route -- patches in place
- Does NOT strip `referrer-policy` -- only removes what's needed for iframe embedding
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
│  OAuth route: DELETED and recreated     │
│  Affects: EVERY route on the cluster    │
│  CVO: DEGRADED                          │
└─────────────────────────────────────────┘

NEW (this repo):
┌─────────────────────────────────────────┐
│  IngressController: "default"           │
│  httpHeaders.actions DELETE:            │
│    - X-Frame-Options                    │
│    - Content-Security-Policy            │
│  auth-operator: RUNNING (untouched)     │
│  OAuth route: PATCHED (not recreated)   │
│  CVO: HEALTHY                           │
│  No ResourceQuotas                      │
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

## Multi-User Support

Set `numUsers` in `values.yaml` or pass it at deploy time:

| `numUsers` | Result |
|------------|--------|
| `0` (default) | Single `showroom-admin` instance |
| `N` | `showroom-user1` through `showroom-userN`, each in its own namespace |

In AgnosticV, `num_users` is passed as a parameter at order time (e.g., `num_users: 5` creates 5 Showroom instances). Each user gets their own namespace, pod, route, and PVC.

```
numUsers=0:
  └── showroom-admin  (namespace: showroom-admin)

numUsers=3:
  ├── showroom-user1  (namespace: showroom-user1)
  ├── showroom-user2  (namespace: showroom-user2)
  └── showroom-user3  (namespace: showroom-user3)

ocp-console-embed always runs once (cluster-level)
```

## Configurable Tabs

Tabs are defined in `values.yaml` under `showroom.tabs`. When set, a ConfigMap is generated and mounted over the content repo's `ui-config.yml`, so you can add/remove tabs without forking the content repo.

```yaml
components:
  showroom:
    values:
      showroom:
        tabs:
          - name: OCP Console
            url: "https://console-openshift-console.${DOMAIN}"
          - name: ArgoCD
            url: "https://openshift-gitops-server-openshift-gitops.${DOMAIN}"
          - name: Grafana
            url: "https://grafana-open-cluster-management-observability.${DOMAIN}"
```

- `${DOMAIN}` is substituted at runtime by the Showroom content container
- If `tabs: []` (empty), falls back to whatever the content repo has in its `ui-config.yml`
- Tabs appear as iframe panels in the right pane of the Showroom UI

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
- Set `numUsers` for multi-user workshops
- Configure `tabs` for custom iframe panels

## Local Verification

```bash
# Single admin
helm template showroom/helm \
  --set deployer.domain=apps.cluster.example.com \
  --set demo.namespace=demo \
  --set demo.appName=myapp

# Multi-user (3 users)
helm template showroom/helm \
  --set deployer.domain=apps.cluster.example.com \
  --set demo.namespace=demo \
  --set demo.appName=myapp \
  --set numUsers=3
```

## Post-Deploy Verification

```bash
# Check ArgoCD apps
oc get applications -n openshift-gitops

# Verify X-Frame-Options removed
curl -skI https://console-openshift-console.apps.DOMAIN | grep -i x-frame
# Should return nothing (header stripped)

# Confirm operators are healthy
oc get clusterversion
oc get pods -n openshift-authentication-operator

# Check multi-user namespaces
oc get namespaces | grep showroom
```

## Old vs New -- Full Comparison

| Feature | Old (`ocp4_workload_showroom_ocp_integration`) | New (this repo) |
|---------|-------|-----|
| **Delivery** | Ansible role, runs once at provision | GitOps -- push to git, ArgoCD syncs |
| **Header stripping** | Cluster-wide IC + referrer-policy | Cluster-wide IC (X-Frame-Options + CSP only) |
| **Auth operator** | Killed (scale 0) | Running, untouched |
| **ResourceQuota** | `pods: 0` blocking CVO | None |
| **OAuth route** | Deleted and recreated | Patched in place (reencrypt TLS) |
| **CVO status** | Degraded | Healthy |
| **Rollback** | None (`remove_workload.yml` debug-only) | ArgoCD prune -- delete app, resources removed |
| **Multi-user** | Ansible loop | `numUsers=N` generates N instances via App of Apps |
| **Tabs** | Hardcoded in content repo | Configurable from Helm values |
| **Content repo changes** | Required (fork/branch) | Optional -- tabs override from values |

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

**Conclusion:** Per-route httpHeaders on operator-managed routes do not survive reconciliation. Operators own these routes and revert any changes to `spec.httpHeaders`.

### Attempt 2: Dedicated IngressController with Route Sharding (Failed on Single-Node)

Created a second IngressController (`showroom-embed`) with domain `*.embed.apps.DOMAIN` and `routeSelector.matchLabels.showroom-embed: "true"`. Custom routes (`console-embed`, `oauth-embed`) were created with the label so only the embed IC would serve them.

**Problem:** On single-node RHDP clusters, the new router pod could not start because ports 80 and 443 were already bound by the default router:

```
$ oc get pods -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=showroom-embed
NAME                              READY   STATUS    RESTARTS   AGE
router-showroom-embed-...        0/1     Pending   0          5m

Events:
  Warning  FailedScheduling  0/1 nodes are available:
    1 node(s) didn't have free ports for the requested pod ports.
```

**Conclusion:** Dedicated IngressController requires additional nodes or `hostNetwork: false` configuration. Not viable for standard single-node RHDP deployments.

### Attempt 3: Patch Default IngressController + OAuth Route (Working)

Patched `IngressController/default` directly to strip X-Frame-Options and CSP headers. Patched `Route/oauth-openshift` with reencrypt TLS termination using the service CA cert. Auth operator stays running. CVO reports healthy.

```
$ oc patch ingresscontroller default -n openshift-ingress-operator --type=merge \
  -p '{"spec":{"httpHeaders":{"actions":{"response":[
    {"name":"X-Frame-Options","action":{"type":"Delete"}},
    {"name":"Content-Security-Policy","action":{"type":"Delete"}}
  ]}}}}'
ingresscontroller.operator.openshift.io/default patched

$ curl -skI https://console-openshift-console.apps.DOMAIN | grep -i x-frame
  (empty -- header stripped)

$ oc get clusterversion
  (healthy, no degraded operators)
```

**Result:** OCP Console renders in Showroom iframe. OAuth login flow works through the embed. All operators running.

### Comparison of All Approaches

| Approach | Operators intact? | Default IC intact? | OAuth works? | Headers stripped? | Single-node? |
|----------|:-:|:-:|:-:|:-:|:-:|
| **Old role** (kill operator + IC patch) | No | No | Yes | Yes | Yes |
| Per-route `spec.httpHeaders` patch | Yes | Yes | Yes | No (reverted) | Yes |
| Dedicated IC + route sharding | Yes | Yes | Untested | Yes | **No** (port conflict) |
| **Default IC patch + OAuth reencrypt** (this repo) | Yes | No | Yes | Yes | Yes |

### Key Finding: Why Operators Are the Core Challenge

The OpenShift console-operator and authentication-operator actively reconcile the routes they own (`Route/console`, `Route/oauth-openshift`). Any `spec.httpHeaders` added to these routes is wiped within seconds. The only reliable way to strip headers is at the **IngressController level**, which applies to all routes served by that IC.

On multi-node clusters, a dedicated IC with route sharding would be the ideal solution (headers stripped only on embed routes). On single-node RHDP clusters, the default IC must be patched directly. The critical improvement over the old approach is that **operators stay running** -- no ResourceQuotas, no CVO degradation, no killed authentication operator.
