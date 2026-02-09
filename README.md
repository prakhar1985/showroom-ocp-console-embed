# Showroom OCP Console Embed (GitOps)

Embeds the OpenShift Console inside Showroom as an iframe tab -- deployed via ArgoCD, with all cluster operators fully Managed and no security compromises.

## TL;DR

We need the OpenShift Console to appear inside a Showroom iframe so lab participants can follow instructions on the left and work in the console on the right. OpenShift blocks this by default for security. The old solution disables critical cluster components to make it work. **This repo makes it work without disabling anything.**

## What Changed from the Old Approach

| | Old Way | This Repo |
|---|---|---|
| **How it works** | Kills the login system operator, hopes nothing breaks | Intercepts and adjusts operator writes without disabling it |
| **Cluster health** | Degraded -- alarms go off | Healthy -- everything looks normal |
| **If login breaks during lab** | Nobody can fix it automatically | Operator self-heals as usual |
| **After the lab ends** | Cluster stays broken | Cluster automatically returns to normal |
| **Long-running clusters** | Dangerous | Safe |
| **Clickjacking protection** | None -- any site on the internet can frame the console | Only Showroom routes can frame the console |

## How It Works (The Short Version)

OpenShift has an operator that manages the login (OAuth) route. This operator insists on a configuration that prevents iframe embedding. Every few seconds, it checks the route and fixes any changes we make.

**Old approach:** Kill the operator so it can't revert our changes.

**Our approach:** Let the operator run normally, but install a "filter" (webhook) between the operator and the database. When the operator writes its configuration, our filter adjusts one setting before it gets saved. The operator thinks it wrote what it wanted. The database has what we need. Everyone is happy.

```
Operator writes:    "use passthrough"
                          |
                    [ our filter ]
                          |
Database stores:    "use reencrypt"    <-- this allows iframe embedding
```

The filter is removed automatically when the lab ends. The operator's next write goes through unfiltered, and the cluster returns to its original state.

## Why This Matters

### For lab authors
- Console appears in Showroom iframe -- participants see instructions and console side by side
- No need to fork or modify your content repo -- tabs are configurable from values
- Push a change to git, ArgoCD syncs it -- no re-ordering needed

### For platform/security teams
- Authentication operator stays **fully Managed** -- cert rotation, pod management, self-healing all work
- CVO reports **healthy** -- no degraded operators
- No ResourceQuota hacks, no operator killing, no permanent cluster modifications
- CSP `frame-ancestors` restricted to **only Showroom routes** -- other apps on the cluster cannot frame the console (clickjacking mitigation)
- Automatic cleanup via ArgoCD prune

### For RHDP operations
- Works on ephemeral and long-running shared clusters
- Tested on both **SNO** and **multi-node** clusters (multi-node takes a bit longer as the router config rolls out across all pods)
- GUID-suffixed resources prevent naming collisions
- Supports single-user (admin demo) and multi-user (up to 50 participants)
- Showroom URLs auto-reported to RHDP portal

---

## Technical Details

### The Problem (Deep Dive)

Two routes need iframe embedding:

1. **Console route** (`console-openshift-console.*`) -- Uses `reencrypt` TLS. The HAProxy router can modify HTTP headers on reencrypt routes. We patch the IngressController to strip `X-Frame-Options` and set a `Content-Security-Policy` that only allows the specific Showroom routes to frame the console (not all `*.DOMAIN` routes -- this prevents clickjacking by other apps on the cluster). This works immediately.

2. **OAuth route** (`oauth-openshift.*`) -- Uses `passthrough` TLS. HAProxy acts as a TCP proxy and cannot touch HTTP headers. The route must be converted to `reencrypt` for header stripping to work. But the authentication operator watches this route and reverts any TLS change back to `passthrough` within 3 seconds via watch-based reconciliation.

### The Solution (Deep Dive)

We deploy a **MutatingAdmissionWebhook** -- a Kubernetes-native mechanism for intercepting API writes before they reach etcd.

When the auth operator issues an UPDATE to set `Route/oauth-openshift` to `passthrough`:

1. The API server sends the request to our webhook
2. The webhook returns a JSON patch: change `passthrough` to `reencrypt`, add the service CA cert as `destinationCACertificate`
3. The API server applies the patch and stores `reencrypt` in etcd
4. The operator reads back the route, sees `reencrypt` (not what it expects), and writes `passthrough` again
5. The webhook intercepts again -- loop continues at ~1 cycle per minute

This loop is harmless. The operator continues to manage everything else (cert rotation, pod health, config changes). It just never wins the TLS termination argument.

### Components

```
ocp-console-embed-{guid}/
  ├── Deployment (webhook)     Python HTTPS server, intercepts Route writes
  ├── Service                  OCP auto-provisions TLS cert (serving-cert annotation)
  ├── MutatingWebhookConfig    Registered with API server (CA auto-injected)
  ├── ConfigMap (service-ca)   Service CA cert for route destinationCACertificate
  ├── ConfigMap (script)       Python webhook code
  ├── Job (PostSync)           Patches IngressController + OAuth route, verifies
  └── RBAC                     ServiceAccount, ClusterRole, ClusterRoleBinding
```

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  BROWSER                                                        │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐ │
│  │ Lab instructions     │  │ OCP Console (iframe)             │ │
│  └──────────────────────┘  └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
         │                              │
         ▼                              ▼
   Showroom Pod                  IngressController (default)
   ├── nginx                     ├── X-Frame-Options: DELETED
   ├── content                   └── CSP: frame-ancestors showroom-*.DOMAIN only
   └── terminal
                                        │
                            ┌───────────┴───────────┐
                            │                       │
                    Console Route            OAuth Route
                    (reencrypt)            (reencrypt via webhook)
                    headers stripped        headers stripped
                                                    ▲
                                         ┌──────────┴──────────┐
                                         │  Mutating Webhook    │
                                         │  passthrough →       │
                                         │  reencrypt            │
                                         └─────────────────────┘

   Auth operator: RUNNING (Managed)    CVO: HEALTHY
```

### What the Job Does

| Step | Action | Purpose |
|------|--------|---------|
| 1 | Patch `IngressController/default` | Strip `X-Frame-Options`, set `Content-Security-Policy` to `frame-ancestors 'self'` + only the specific Showroom route URLs (not all `*.DOMAIN`) |
| 1b | Wait for router rollout | Polls `IngressController` status until `Progressing=False`. On SNO (1 router pod) this is instant. On multi-node (3 router pods) this waits for all pods to pick up the new config. |
| 2 | Read service CA, patch `Route/oauth-openshift` to reencrypt | Immediate conversion so iframe works without waiting for webhook timing |
| 3 | Verify console headers | Confirm `X-Frame-Options` is gone |
| 4 | Verify OAuth route | Confirm route is `reencrypt` and headers are stripped |

### Failure Modes

| Scenario | What happens |
|----------|-------------|
| Webhook pod crashes | `failurePolicy: Ignore` -- operator writes passthrough normally. OAuth still works, just not in iframe. Pod restarts via Deployment. |
| ArgoCD app deleted | MWC removed. Operator writes passthrough on next cycle. Cluster self-heals. |
| Auth operator reconciles | Writes passthrough → webhook converts to reencrypt. Normal loop. |
| Service CA rotates | Webhook pod restarts (cert secret changes), picks up new CA on startup. |

### RBAC (Minimal)

The Job's ServiceAccount permissions:
- `ingresscontrollers` get/patch (IngressController header config)
- `routes` get/patch on `oauth-openshift` only (initial reencrypt patch + verification)
- `routes/custom-host` create/update (required when setting `destinationCACertificate`)
- `configmaps` get on `v4-0-config-system-service-ca` (read service CA cert)

The webhook itself needs **no RBAC** -- it receives AdmissionReview objects from the API server and doesn't make any API calls.

---

## Multi-User Support

| `numUsers` | What gets created |
|------------|-------------------|
| `0` (default) | Single `showroom-{guid}-admin` instance |
| `N` | `showroom-{guid}-user1` through `showroom-{guid}-userN` |

Each user gets their own namespace, pod, route, and storage. The webhook and Job run once per cluster.

## Configurable Tabs

Tabs can be defined in **either** of two places:

1. **Helm values** (this repo's `values.yaml` or AgnosticV `ocp4_workload_gitops_bootstrap_helm_values`) -- takes precedence
2. **Content repo** (`ui-config.yml` in the Showroom content repository) -- used when no tabs are defined in Helm values

Most Showroom content repos define tabs in their own `ui-config.yml`. When you define tabs in the Helm values here, they **override** the content repo's `ui-config.yml` via a ConfigMap mount. If you leave `tabs: []` empty, the content repo's own tab config is used instead.

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
```

`${DOMAIN}` is replaced at runtime.

## Deployment

Order from RHDP with:

| Field | Value |
|-------|-------|
| Repo URL | `https://github.com/prakhar1985/showroom-ocp-console-embed.git` |
| Revision | `main` |
| Path | `showroom/helm` |

## Verification

```bash
# ArgoCD apps synced
oc get applications -n openshift-gitops

# No X-Frame-Options on console or OAuth
curl -skI https://console-openshift-console.apps.DOMAIN | grep -i x-frame
curl -skI https://oauth-openshift.apps.DOMAIN | grep -i x-frame

# Auth operator is Managed and healthy
oc get authentication.operator.openshift.io cluster -o jsonpath='{.spec.managementState}'
oc get clusteroperator authentication

# Webhook running
oc get pods -n ocp-console-embed-GUID -l app=ocp-console-embed-GUID-webhook

# OAuth route is reencrypt
oc get route oauth-openshift -n openshift-authentication -o jsonpath='{.spec.tls.termination}'
```
