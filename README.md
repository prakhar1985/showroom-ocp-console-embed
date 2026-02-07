# Showroom OCP Console Embed (GitOps)

Deploys Showroom with an embedded OpenShift Console tab by stripping `X-Frame-Options` and `Content-Security-Policy` headers from the default IngressController and patching the OAuth route with reencrypt TLS -- all operators stay running, no ResourceQuotas, no CVO degradation.

## Why This Approach Is More Secure

Both approaches strip `X-Frame-Options` and `Content-Security-Policy` from the default IngressController -- that is required for iframe embedding. The security improvement is everything else the old role (`ocp4_workload_showroom_ocp_integration`) does on top of that.

### 1. Authentication Operator Stays Running

The old role **kills the authentication operator** (scales to 0) and blocks CVO from restarting it with a `ResourceQuota`. This means:

- If OAuth breaks during the lab, the cluster **cannot self-heal**. Users are locked out with no recovery path.
- If someone accidentally deletes the OAuth route, no operator exists to recreate it. The cluster's authentication is permanently broken until manual intervention.
- An attacker who gains access during this window faces no self-healing defenses on the authentication layer.

Our approach **never touches the auth operator**. It stays running, continues reconciling, and can recover from failures. The cluster's authentication self-healing is fully intact.

### 2. No ResourceQuota Blocking CVO

The old role creates `ResourceQuota` with `pods: "0"` in `openshift-authentication-operator` to prevent CVO from restarting the auth operator. This:

- Puts CVO in a **degraded state** -- it detects a missing operator and reports the cluster as unhealthy
- Means CVO cannot remediate **any** authentication-related issue
- Leaves a footprint that persists after the lab ends -- the next user inherits a degraded cluster

Our approach creates **no ResourceQuotas**. CVO reports healthy. All operators can be managed normally.

### 3. OAuth Route Patched, Not Destroyed

The old role **deletes** the `oauth-openshift` route and recreates it from scratch. If anything goes wrong during recreation (network issue, partial apply, timing race), the OAuth route is gone and there's no operator to bring it back (because it was killed in step 2).

Our approach **patches the existing route** in place with `--type=merge`. If the patch fails, the original route is untouched. The auth operator can still reconcile it.

### 4. Fewer Headers Stripped

The old role strips three headers: `X-Frame-Options`, `Content-Security-Policy`, **and `referrer-policy`**. Stripping `referrer-policy` leaks the full referrer URL (including paths and query parameters) to external sites for every route on the cluster. This can expose:

- Internal dashboard URLs and paths
- OAuth tokens in URL parameters
- Internal hostnames and namespace names

Our approach strips **only** `X-Frame-Options` and `Content-Security-Policy` -- the minimum needed for iframe embedding. `referrer-policy` stays intact.

### 5. Clean Rollback via GitOps

The old role's `remove_workload.yml` is tagged `debug` -- it never runs in production. After the lab ends, the cluster still has:
- Headers stripped cluster-wide
- Auth operator dead
- ResourceQuota blocking recovery

Our approach uses ArgoCD with `prune: true`. Delete the ArgoCD Application and all resources are removed. The IngressController patch is the only thing that needs manual revert (or a cleanup Job).

### Security Comparison

```
OLD ROLE -- attack surface during lab:
  [x] Auth operator dead -- no self-healing on OAuth
  [x] CVO degraded -- cannot remediate auth issues
  [x] OAuth route deleted/recreated -- risk of broken auth
  [x] referrer-policy stripped -- URL leakage to external sites
  [x] No rollback -- cluster stays weakened after lab

NEW APPROACH -- attack surface during lab:
  [ ] Auth operator running -- self-heals OAuth issues
  [ ] CVO healthy -- full remediation capability
  [ ] OAuth route patched in place -- fallback to original
  [ ] referrer-policy intact -- no URL leakage
  [ ] GitOps rollback -- delete app, resources pruned
```

## What the Old Role Does (and Why It's Dangerous)

Today, teams use the `ocp4_workload_showroom_ocp_integration` Ansible role. Here's what it actually does:

| Step | Action | Security Impact |
|------|--------|-----------------|
| 1 | Patch `IngressController/default` to delete `X-Frame-Options`, `Content-Security-Policy`, `referrer-policy` on **all routes** | Every app on the cluster loses clickjacking/CSP/referrer protection |
| 2 | Scale `authentication-operator` to 0 + `ResourceQuota` `pods: "0"` | Cluster cannot self-heal authentication. CVO reports degraded. |
| 3 | Delete and recreate `oauth-openshift` route with reencrypt TLS | Risky -- if recreation fails, OAuth is broken with no operator to fix it |

```
┌─────────────────────────────────────────────────────────────────┐
│  OpenShift IngressController (default)                          │
│                                                                 │
│  OLD ROLE: httpHeaders.actions DELETE on ALL routes:             │
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

### Attempt 2: Dedicated IngressController with Route Sharding (Failed on All RHDP Clusters)

Created a second IngressController (`showroom-embed`) with domain `*.embed.apps.DOMAIN` and `routeSelector.matchLabels.showroom-embed: "true"`. Custom routes (`console-embed`, `oauth-embed`) were created with the label so only the embed IC would serve them.

**Tested on two clusters:**

**Single-node** (`cluster-gpzh2`): Router pod stuck Pending -- ports 80/443 already bound by default router:

```
Events:
  Warning  FailedScheduling  0/1 nodes are available:
    1 node(s) didn't have free ports for the requested pod ports.
```

**Multi-node** (`cluster-9mggj`, 3 control-plane + 2 workers): Tried three sub-approaches:

1. **HostNetwork on workers** -- separated default IC (control-planes) and embed IC (workers). Embed IC started, but RHDP's external load balancer routes to ALL nodes. Moving default IC off workers caused 503s for original routes when LB hit a worker.

2. **NodePortService** -- embed IC started on non-standard port (32557). Works technically, but browsers won't access `https://host:32557` in iframes. Non-standard ports are not viable for end users.

3. **HostNetwork on all nodes** -- impossible. Two ICs can't share port 80/443 on the same node.

**Root cause:** RHDP clusters use a single external LB with wildcard DNS (`*.apps.DOMAIN`) pointing to it. Both `*.apps.DOMAIN` and `*.embed.apps.DOMAIN` resolve to the same LB IP. There's no way to route traffic to different ICs based on subdomain without a separate LB or DNS entry, which RHDP doesn't support.

**Conclusion:** Dedicated IngressController is not viable on any RHDP cluster topology -- the LB/DNS architecture doesn't support routing to a second IC.

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

| Approach | Operators intact? | Default IC intact? | OAuth works? | Headers stripped? | RHDP compatible? |
|----------|:-:|:-:|:-:|:-:|:-:|
| **Old role** (kill operator + IC patch) | No | No | Yes | Yes | Yes |
| Per-route `spec.httpHeaders` patch | Yes | Yes | Yes | No (reverted) | Yes |
| Dedicated IC + route sharding | Yes | Yes | Yes | Yes | **No** (LB/DNS) |
| **Default IC patch + OAuth reencrypt** (this repo) | Yes | No | Yes | Yes | Yes |

### Key Finding: Why Dedicated IngressController Doesn't Work on RHDP

The dedicated IC approach is architecturally sound but incompatible with RHDP's infrastructure:

1. **Port conflict**: Two HostNetwork ICs can't share the same node (both need port 80/443)
2. **LB routing**: RHDP's external LB routes to all nodes. Separating ICs by node breaks default route traffic when the LB hits a node without the default IC
3. **DNS**: Wildcard DNS `*.apps.DOMAIN` catches `*.embed.apps.DOMAIN` and resolves to the same LB IP. No way to route embed traffic to a different IC
4. **NodePort**: Works but uses non-standard ports (e.g., :32557) that browsers won't use in iframe URLs

### Key Finding: Why Operators Are the Core Challenge

The OpenShift console-operator and authentication-operator actively reconcile the routes they own (`Route/console`, `Route/oauth-openshift`). Any `spec.httpHeaders` added to these routes is wiped within seconds. The only reliable way to strip headers is at the **IngressController level**, which applies to all routes served by that IC.

The critical improvement over the old approach is that **operators stay running** -- no ResourceQuotas, no CVO degradation, no killed authentication operator.
