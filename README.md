# Showroom OCP Console Embed (GitOps)

Embeds the OpenShift Console inside Showroom as an iframe tab -- deployed via ArgoCD, with all cluster operators intact and no security hacks.

## The Problem

Showroom displays lab instructions on the left and an iframe on the right. To show the OCP Console in that iframe, the browser needs the console to allow being embedded. By default, OpenShift blocks this with security headers (`X-Frame-Options`, `Content-Security-Policy`).

The existing solution (`ocp4_workload_showroom_ocp_integration` Ansible role) removes these headers, but it also **kills the authentication operator** and **blocks OpenShift from restarting it**. This leaves the cluster unable to recover from authentication failures during the lab.

## Why This Repo Is More Secure

**In plain terms:** both the old approach and this repo remove the same iframe-blocking headers. The difference is what else the old approach does -- it disables critical cluster safety mechanisms that have nothing to do with iframe embedding.

Think of it like unlocking a door so guests can enter. The old approach unlocks the door **and** disconnects the alarm system, fires the security guard, and tapes over the cameras. This repo just unlocks the door.

### What the old role breaks (and we don't)

| | Old Role | This Repo |
|---|---|---|
| **Login system recovery** | Disabled. If login breaks during the lab, nobody can fix it automatically. Students get locked out. | Intact. OpenShift can detect and fix login issues on its own. |
| **Cluster health status** | Reports "degraded" because it detects a missing operator. Confusing for anyone checking cluster health. | Reports healthy. Everything looks normal. |
| **Cleanup after lab** | None. The next person who uses the cluster inherits a broken setup. | ArgoCD removes resources when the app is deleted. |
| **Headers removed** | Three headers removed, including `referrer-policy` which leaks internal URLs to external sites. | Only two headers removed -- the minimum needed for iframe embedding. |
| **Login route** | Deleted and recreated from scratch. If the recreation fails, login is permanently broken. | Patched in place. If the patch fails, the original route is still there. |

### Security comparison at a glance

```
OLD ROLE:
  [x] Authentication operator killed
  [x] Cluster reports degraded
  [x] Login route deleted and recreated (risky)
  [x] referrer-policy stripped (leaks internal URLs)
  [x] No cleanup after lab ends

THIS REPO:
  [ ] Authentication operator running
  [ ] Cluster reports healthy
  [ ] Login route patched in place (safe)
  [ ] referrer-policy intact
  [ ] GitOps cleanup via ArgoCD
```

### Technical details

The old role scales `authentication-operator` to 0 replicas and creates a `ResourceQuota` with `pods: "0"` to prevent CVO (Cluster Version Operator) from restarting it. This puts CVO in a degraded state and removes the cluster's ability to self-heal authentication. The `remove_workload.yml` is tagged `debug` and never runs in production, so these changes persist after the lab.

Our approach patches `IngressController/default` to strip `X-Frame-Options` and `Content-Security-Policy` (not `referrer-policy`) and patches `Route/oauth-openshift` with reencrypt TLS using the service CA. The authentication operator stays running throughout. CVO reports healthy. All changes are managed by ArgoCD with `prune: true`.

## How It Works

An ArgoCD App of Apps deploys two components:

1. **showroom** -- the lab guide with an embedded OCP Console tab
2. **ocp-console-embed** -- a one-time Job that configures the cluster for iframe embedding

```
RHDP Order
  → ArgoCD syncs this repo
        ↓
  ├── showroom           (lab guide + terminal + OCP Console tab)
  └── ocp-console-embed  (Job patches IngressController + OAuth route)
```

### What the Job does

| Step | What | Why |
|------|------|-----|
| 1 | Patch `IngressController/default` to delete `X-Frame-Options` and `Content-Security-Policy` headers | Allows the console to load inside an iframe |
| 2 | Read the service CA certificate | Needed for secure communication with the OAuth service |
| 3 | Patch `Route/oauth-openshift` with reencrypt TLS | Ensures the login flow works through the iframe |
| 4 | Verify the console responds without `X-Frame-Options` | Confirms the setup worked |

### Why IngressController-level patching is required

OpenShift operators actively manage their own routes. If you try to add `httpHeaders` directly to the `console` or `oauth-openshift` routes, the console-operator and authentication-operator will revert the change within seconds. The only reliable way to strip these headers is at the IngressController level.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  BROWSER                                                        │
│                                                                 │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐ │
│  │ Left Pane            │  │ Right Pane (iframe tabs)         │ │
│  │ Lab instructions     │  │                                  │ │
│  │ (AsciiDoc rendered   │  │  Tab 1: OCP Console              │ │
│  │  by Antora)          │  │  Tab 2: ArgoCD                   │ │
│  │                      │  │  Tab N: ...                       │ │
│  └──────────────────────┘  └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
         │                              │
         │                              │ Browser fetches console directly
         ▼                              ▼
   Showroom Pod                  IngressController (default)
   ├── nginx (proxy)             ├── X-Frame-Options: DELETED
   ├── content (lab guide)       └── Content-Security-Policy: DELETED
   └── terminal (ttyd)
                                 Authentication operator: RUNNING
                                 CVO: HEALTHY
```

## Multi-User Support

Set `numUsers` in `values.yaml` or pass it at deploy time via AgnosticV:

| `numUsers` | What gets created |
|------------|-------------------|
| `0` (default) | Single `showroom-admin` instance for admin/demo use |
| `N` | `showroom-user1` through `showroom-userN`, each in its own namespace |

Each user gets their own Showroom instance with its own namespace, pod, route, and persistent storage. The `ocp-console-embed` Job runs once regardless -- it's a cluster-level operation.

```
numUsers=0:                        numUsers=3:
  └── showroom-admin                 ├── showroom-user1
                                     ├── showroom-user2
                                     └── showroom-user3

ocp-console-embed runs once (cluster-level, shared by all users)
```

## Configurable Tabs

Tabs are defined in `values.yaml`. When set, a ConfigMap overrides the content repo's `ui-config.yml` -- no need to fork or modify the content repo.

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

- `${DOMAIN}` is automatically replaced with the cluster's domain at runtime
- If `tabs: []` (empty), the content repo's own `ui-config.yml` is used
- Add or remove tabs by editing `values.yaml` and pushing -- ArgoCD syncs the change

## User Info Reporting (Babylon Integration)

After deployment, Showroom URLs are automatically reported back to the RHDP portal via the `ocp4_workload_gitops_bootstrap` role. A ConfigMap with the `demo.redhat.com/userinfo` label is created in `openshift-gitops`, and the bootstrap role discovers it and feeds the data to Babylon.

| Mode | What gets reported |
|------|-------------------|
| Admin (`numUsers=0`) | Single `showroom_primary_view_url` for `showroom-admin` |
| Multi-user (`numUsers=N`) | Per-user `showroom_primary_view_url` for each `showroom-userN` via `users_json` |

Both modes also report `openshift_cluster_console_url` and `cluster_domain`.

Users see their Showroom URL in the RHDP portal after ordering -- no manual lookup needed.

## Deployment

Order from RHDP with:

| Field | Value |
|-------|-------|
| Repo URL | `https://github.com/prakhar1985/showroom-ocp-console-embed.git` |
| Revision | `main` |
| Path | `showroom/helm` |

Works on both single-node and multi-node RHDP clusters.

## Configuration

Edit `showroom/helm/values.yaml`:

- **Content repo**: `components.showroom.values.showroom.content.repoUrl`
- **Components**: toggle `components.showroom.enabled` / `components.ocpConsoleEmbed.enabled`
- **Multi-user**: set `numUsers` (0 = admin only, N = N user instances)
- **Tabs**: configure `components.showroom.values.showroom.tabs`
- **Domain**: injected by RHDP at order time (`deployer.domain`)

## Verification

```bash
# Check ArgoCD apps are synced
oc get applications -n openshift-gitops

# Verify iframe headers are stripped
curl -skI https://console-openshift-console.apps.DOMAIN | grep -i x-frame
# Should return nothing

# Confirm operators are healthy (not degraded)
oc get clusterversion
oc get pods -n openshift-authentication-operator

# Check user namespaces (multi-user)
oc get namespaces | grep showroom
```

## Full Comparison

| | Old (`ocp4_workload_showroom_ocp_integration`) | This repo |
|---|---|---|
| **How it's deployed** | Ansible role, runs once at provision time | GitOps -- push to git, ArgoCD syncs automatically |
| **Headers removed** | `X-Frame-Options` + `Content-Security-Policy` + `referrer-policy` | `X-Frame-Options` + `Content-Security-Policy` only |
| **Authentication operator** | Killed (scaled to 0) | Running, untouched |
| **ResourceQuota** | `pods: 0` blocking CVO from recovery | None |
| **OAuth route** | Deleted and recreated | Patched in place |
| **Cluster health (CVO)** | Degraded | Healthy |
| **Cleanup after lab** | None (`remove_workload.yml` is debug-only) | ArgoCD prune removes resources |
| **Multi-user** | Ansible loop | `numUsers=N` creates N instances via App of Apps |
| **Tabs** | Hardcoded in content repo | Configurable from Helm values |
| **Content repo changes needed** | Yes (fork or branch) | No -- tabs override from values |
| **User info in RHDP portal** | Manual or custom Ansible | Automatic via `demo.redhat.com/userinfo` ConfigMap |
