# Showroom OCP Console Embed (GitOps)

Embeds the OpenShift Console inside Showroom as an iframe tab -- deployed via ArgoCD, with all cluster operators fully Managed.

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

## Terminal Configuration

The Showroom pod supports three terminal modes, configured via `terminal.type`:

| Type | Container | Description |
|------|-----------|-------------|
| `ocp` | ttyd (`quay.io/rhpds/openshift-showroom-terminal-ocp`) | In-pod shell with `oc` pre-authenticated via ServiceAccount. No bastion needed. Accessible at `/terminal/`. |
| `wetty` | Wetty (`quay.io/rhpds/wetty:v2.5`) | SSH terminal to a bastion host via Wetty. Accessible at `/wetty`. Requires bastion host, user, and password. |
| `tmux` | Wetty (`quay.io/rhpds/wetty:v2.5`) | SSH + tmux session on bastion via Wetty. Accessible at `/wetty`. Requires bastion host, user, and password. Bastion must have tmux configured. |

When `terminal.enabled` is `false`, no terminal container or PVC is created.

## View Switcher (Panel Toggle)

The default Showroom split layout can feel cramped when reading dense instructions or when the console needs full width. A toolbar is injected automatically via nginx `sub_filter` -- no changes needed in the content repo.

A dark toolbar appears in the **top-right corner** with three buttons:

| Button | Mode | What it does |
|--------|------|-------------|
| **Instructions** | `instructions` | Lab guide gets full width, tabs/console hidden |
| **Split** | `split` | Both panes side by side (default Split.js layout) |
| **Tabs** | `tabs` | Tabs/console gets full width, instructions hidden |

The active mode is highlighted in red.

**Keyboard shortcuts:**

| Shortcut | Mode |
|----------|------|
| `Ctrl+1` | Instructions (full-width) |
| `Ctrl+2` | Split (side by side) |
| `Ctrl+3` | Tabs (full-width) |

Keyboard shortcuts only work when the parent page has focus. After clicking inside a cross-origin iframe (e.g. the OCP Console), the parent page loses focus. Hover over the toolbar to refocus the parent page.

**State persistence:**

- View mode is stored in the **URL hash** (`#view=instructions`) so you can share or bookmark a specific view
- Also saved to **localStorage** (`sr-panel-mode`) so the mode persists across page reloads
- Priority on load: URL hash > localStorage > default mode (configurable via `showroom.panelMode` in values.yaml)

**First-visit hint:** On first visit, a tooltip appears below the toolbar explaining the controls. It auto-dismisses after 8 seconds and won't show again (tracked via `sr-hint-dismissed` in localStorage).

**Lab workflow:** Read the step (`Ctrl+1`) -- do the task (`Ctrl+3`) -- read next step (`Ctrl+1`) -- repeat.

## Using with AgnosticV

This repo is designed to be deployed via the `ocp4_workload_gitops_bootstrap` workload. Below are examples of how to configure different features in your AgnosticV catalog's `common.yaml`.

### Minimal Example (Console Embed Only)

```yaml
ocp4_workload_gitops_bootstrap_repo_url: https://github.com/prakhar1985/showroom-ocp-console-embed.git
ocp4_workload_gitops_bootstrap_repo_revision: main
ocp4_workload_gitops_bootstrap_repo_path: showroom/helm

ocp4_workload_gitops_bootstrap_helm_values:
  components:
    showroom:
      values:
        showroom:
          content:
            repoUrl: "https://github.com/your-org/your-showroom-content.git"
            repoRef: "main"
```

This deploys Showroom with OCP Console embed. Tabs come from the content repo's `ui-config.yml`.

### Custom Tabs (Override Content Repo)

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  components:
    showroom:
      values:
        showroom:
          content:
            repoUrl: "https://github.com/your-org/your-showroom-content.git"
            repoRef: "main"
          # Tabs here OVERRIDE the content repo's ui-config.yml.
          # ${DOMAIN} is replaced at runtime with the cluster domain.
          tabs:
          - name: OCP Console
            url: "https://console-openshift-console.${DOMAIN}"
          - name: ArgoCD
            url: "https://openshift-gitops-server-openshift-gitops.${DOMAIN}"
          - name: AAP
            url: "https://aap-aap.${DOMAIN}"
```

If you omit `tabs` (or set `tabs: []`), the content repo's own `ui-config.yml` is used instead.

### Terminal: SSH via Wetty

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  components:
    showroom:
      values:
        showroom:
          content:
            repoUrl: "https://github.com/your-org/your-showroom-content.git"
            repoRef: "main"
          terminal:
            enabled: true
            type: "wetty"
            bastion:
              host: "{{ bastion_ansible_host }}"
              user: "{{ bastion_ansible_user }}"
              password: "{{ bastion_ansible_ssh_pass }}"
              port: "{{ bastion_ansible_port | default('22') }}"
```

Bastion credentials are typically propagated from the OCP cluster component via `propagate_provision_data`.

### Terminal: In-Pod OCP Shell

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  components:
    showroom:
      values:
        showroom:
          content:
            repoUrl: "https://github.com/your-org/your-showroom-content.git"
            repoRef: "main"
          terminal:
            enabled: true
            type: "ocp"
```

No bastion config needed -- the terminal runs inside the pod with `oc` pre-authenticated via the ServiceAccount.

### Multi-User with All Features

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  numUsers: "{{ num_users | int }}"
  userPassword: "{{ common_password }}"
  components:
    showroom:
      values:
        showroom:
          content:
            repoUrl: "{{ showroom_content_repo }}"
            repoRef: "{{ showroom_content_ref }}"
          tabs:
          - name: OCP Console
            url: "https://console-openshift-console.${DOMAIN}"
          - name: ArgoCD
            url: "https://openshift-gitops-server-openshift-gitops.${DOMAIN}"
          - name: AAP
            url: "https://aap-aap.${DOMAIN}"
          - name: DevSpaces
            url: "https://devspaces.${DOMAIN}"
          - name: OpenShift AI
            url: "https://data-science-gateway.${DOMAIN}"
          terminal:
            enabled: "{{ terminal_type | default('none') != 'none' }}"
            type: "{{ terminal_type | default('none') }}"
            bastion:
              host: "{{ bastion_ansible_host | default('') }}"
              user: "{{ bastion_ansible_user | default('') }}"
              password: "{{ bastion_ansible_ssh_pass | default('') }}"
              port: "{{ bastion_ansible_port | default('22') }}"
          userdata: >-
            {{ lookup('agnosticd_user_data', '*')
               | dict2items
               | rejectattr('key', 'eq', 'users')
               | items2dict }}
```

### Common Route URLs

| Service | Route URL |
|---------|-----------|
| OCP Console | `https://console-openshift-console.${DOMAIN}` |
| ArgoCD | `https://openshift-gitops-server-openshift-gitops.${DOMAIN}` |
| AAP | `https://aap-aap.${DOMAIN}` |
| DevSpaces | `https://devspaces.${DOMAIN}` |
| OpenShift AI 3 | `https://data-science-gateway.${DOMAIN}` |
| Grafana (ACM) | `https://grafana-open-cluster-management-observability.${DOMAIN}` |

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

---

## Developer Guide

### Repository Layout

```
showroom/helm/
├── Chart.yaml                          # App of Apps chart
├── values.yaml                         # Top-level values (numUsers, deployer, component toggles)
├── templates/
│   ├── applications.yaml               # ArgoCD Application resources (one per user + webhook)
│   └── userinfo.yaml                   # ConfigMap for RHDP portal (lab_ui_url, users_json)
└── components/
    ├── showroom/                        # Showroom lab guide + toggle script
    │   ├── Chart.yaml
    │   ├── values.yaml                  # Showroom defaults (content, tabs, terminal, panelMode)
    │   └── templates/
    │       └── showroom.yaml            # All Showroom resources (ConfigMaps, Deployment, Route, etc.)
    └── ocp-console-embed/               # Webhook + Job for iframe embedding
        ├── Chart.yaml
        ├── values.yaml                  # Webhook defaults (namespace, image)
        └── templates/
            ├── webhook-script.yaml      # Python webhook code (passthrough → reencrypt)
            ├── webhook-deployment.yaml  # Webhook pod
            ├── webhook-service.yaml     # Service with serving-cert annotation
            ├── webhook-config.yaml      # MutatingWebhookConfiguration
            ├── webhook-cabundle.yaml    # ConfigMap for service CA cert
            ├── job.yaml                 # PostSync Job (patches IngressController, verifies)
            ├── serviceaccount.yaml      # ServiceAccount for Job + webhook
            ├── clusterrole.yaml         # RBAC permissions
            └── clusterrolebinding.yaml  # ClusterRoleBinding
```

### How Values Flow

```
AgnosticV common.yaml
  └─ ocp4_workload_gitops_bootstrap_helm_values
       └─ showroom/helm/values.yaml          (top-level: numUsers, deployer, components)
            ├─ templates/applications.yaml    (reads components, creates ArgoCD apps)
            └─ components/showroom/values.yaml (per-app: content, tabs, terminal, panelMode)
```

The `ocp4_workload_gitops_bootstrap` role injects `deployer.domain`, `deployer.guid`, `demo.namespace`, and `demo.appName` automatically. You don't need to set these in your catalog.

### Modifying the View Switcher (Toggle Script)

The toggle script lives in:

```
components/showroom/templates/showroom.yaml → ConfigMap "showroom-toggle-script"
```

It's a self-contained JavaScript IIFE injected into every Showroom HTML page via nginx `sub_filter`. The script:

1. Waits for Split.js to render the gutter element
2. Tags panes with CSS marker classes (`sr-left`, `sr-right`, `sr-gutter`)
3. Builds the toolbar and appends it to `<body>`
4. Switches modes by adding/removing CSS classes on `<body>`

#### Key design decision: CSS classes, not inline styles

Split.js manages pane widths via inline `style` attributes. **Never modify these directly** -- it breaks Split.js's internal state and causes bugs when switching back to split mode.

Instead, the toggle uses CSS class overrides on `<body>`:

```
Instructions mode:  body.sr-mode-instructions
  → hides .sr-right and .sr-gutter (display:none!important)
  → expands .sr-left to 100% (width:100%!important)

Tabs mode:          body.sr-mode-tabs
  → hides .sr-left and .sr-gutter (display:none!important)
  → expands .sr-right to 100% (width:100%!important)

Split mode:         (no body class)
  → CSS overrides removed, Split.js inline styles take effect naturally
```

#### Common changes

**Change button labels or icons:**

In the toggle script, find the `btnI`, `btnS`, `btnC` variables. Each has an `innerHTML` with an SVG icon and a `<span>` label. Change the `<span>` text.

```javascript
// Example: rename "Tabs" to "Console"
btnC.innerHTML = icoTabs + '<span>Console</span>';
```

**Change default mode:**

In `components/showroom/values.yaml`, change `panelMode`:

```yaml
showroom:
  panelMode: "split"    # "instructions", "split", or "tabs"
```

**Change toolbar position:**

In the CSS array, find `.sr-toolbar{` and change `top` and `right`:

```css
.sr-toolbar{
  position:fixed;top:8px;right:12px;   /* ← change these */
```

**Change active button color:**

Find `.sr-mode-btn.sr-active{` and change the `background` and `box-shadow`:

```css
.sr-mode-btn.sr-active{
  color:#fff;background:#ee0000;          /* ← button color */
  box-shadow:0 2px 8px rgba(238,0,0,.35); /* ← glow color */
}
```

**Add a new mode:**

1. Add a CSS rule: `body.sr-mode-NEWMODE .sr-left { ... }`
2. Add a button in the toolbar section
3. Add the mode to the `classList.remove(...)` call in `apply()`
4. Add a keyboard shortcut in the `keydown` handler

#### Testing changes

After editing `showroom.yaml`:

```bash
# Push to git
git add -A && git commit -m "Update toggle" && git push

# On the cluster: force ArgoCD to refresh and pick up the new commit
oc annotate application showroom-GUID-user1 -n openshift-gitops \
  argocd.argoproj.io/refresh=hard --overwrite

# Restart pod to pick up the new ConfigMap
oc rollout restart deployment/showroom -n showroom-GUID-user1

# Verify the new code is deployed
oc get configmap showroom-toggle-script -n showroom-GUID-user1 \
  -o jsonpath='{.data.showroom-toggle\.js}' | head -5
```

Then hard-refresh the browser (`Ctrl+Shift+R`) to bypass cache.

### Modifying the Webhook

The webhook Python script is in `components/ocp-console-embed/templates/webhook-script.yaml`.

It only targets `Route/oauth-openshift` in `openshift-authentication`. All other routes pass through unchanged.

The JSON patch it applies:

```json
[
  {"op": "replace", "path": "/spec/tls/termination", "value": "reencrypt"},
  {"op": "add", "path": "/spec/tls/insecureEdgeTerminationPolicy", "value": "Redirect"},
  {"op": "add", "path": "/spec/tls/destinationCACertificate", "value": "<service-ca-cert>"}
]
```

The webhook has `failurePolicy: Ignore`, so if the pod crashes, the auth operator writes `passthrough` normally and OAuth still works -- just not inside an iframe until the pod recovers.

### Modifying the CSP (Content-Security-Policy)

The Job (`components/ocp-console-embed/templates/job.yaml`) builds the CSP `frame-ancestors` directive. It only allows the specific Showroom route URLs -- not all `*.DOMAIN` routes.

```bash
# For 3 users, the CSP looks like:
frame-ancestors 'self' https://showroom-showroom-GUID-user1.DOMAIN https://showroom-showroom-GUID-user2.DOMAIN https://showroom-showroom-GUID-user3.DOMAIN
```

To allow additional origins (e.g. a custom dashboard), edit the `ALLOWED_ORIGINS` variable in `job.yaml`.

### Modifying the RHDP Portal Info

The userinfo ConfigMap in `templates/userinfo.yaml` controls what appears in the RHDP portal (demo.redhat.com). It sets:

- `openshift_cluster_console_url` -- console link
- `lab_ui_url` -- Showroom link (shown to the user)
- `users_json` -- per-user URLs for multi-user mode

The `ocp4_workload_gitops_bootstrap` role discovers ConfigMaps with the `demo.redhat.com/userinfo` label and reports the data to Babylon.

### Adding a New Tab

Tabs can be added in two places:

1. **Helm values** (this repo or AgnosticV catalog) -- takes precedence:

```yaml
showroom:
  tabs:
    - name: My Service
      url: "https://my-service.${DOMAIN}"
```

2. **Content repo** (`ui-config.yml`) -- used when no tabs in Helm values

`${DOMAIN}` is replaced at runtime by the Showroom content container with the actual cluster domain.

### Adding a New Component

To add a new ArgoCD-managed component (e.g. a custom operator):

1. Create `components/my-component/` with `Chart.yaml`, `values.yaml`, and `templates/`
2. Add the component toggle in `showroom/helm/values.yaml`:

```yaml
components:
  myComponent:
    enabled: true
    path: showroom/helm/components/my-component
```

3. Add an ArgoCD Application in `templates/applications.yaml` (copy the `ocpConsoleEmbed` block as a template)

### Clearing User State

If the toggle gets stuck in a bad state, clear localStorage in the browser console:

```javascript
localStorage.removeItem('sr-panel-mode');
localStorage.removeItem('sr-hint-dismissed');
```

Or use an incognito window.
