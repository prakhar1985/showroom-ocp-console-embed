# Showroom OCP Console Embed (GitOps)

Deploys Showroom with an embedded OpenShift Console tab using a **per-route** approach that strips `X-Frame-Options` and `Content-Security-Policy` headers from the console and OAuth routes only -- without modifying the IngressController or disabling operators.

## Architecture

ArgoCD App of Apps deployed from `showroom/helm`:

```
ArgoCD
├── App: showroom          (lab guide + terminal + OCP Console tab)
└── App: ocp-console-embed (Job patches routes per-route)
```

### What the embed Job does

| Step | Target | Change |
|------|--------|--------|
| 1 | `Route/console` in `openshift-console` | Delete `X-Frame-Options` + `Content-Security-Policy` via `spec.httpHeaders` |
| 2 | `ConfigMap/v4-0-config-system-service-ca` in `openshift-authentication` | Read service CA cert |
| 3 | `Route/oauth-openshift` in `openshift-authentication` | Add reencrypt TLS with service CA + delete security headers |
| 4 | Verify | Poll console URL until `X-Frame-Options` is gone |

### What it does NOT do

- Does NOT modify `IngressController/default`
- Does NOT disable the authentication operator
- Does NOT create ResourceQuotas blocking operator pods
- Only touches the two specific routes that need iframe embedding

## Deployment

Order from RHDP (`ocp-field-asset-cnv`) with:

| Field | Value |
|-------|-------|
| Repo URL | `https://github.com/prakhar-srivastav/showroom-ocp-console-embed.git` |
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

## Experiment Notes

If OCP operators revert the `httpHeaders` on the routes, switch the Job to a CronJob (every 5 min) or fall back to a dedicated IngressController for the console routes.
