# Grafana on OpenShift — Installation Guide

Deploys Grafana with authentication routed through the OpenShift OAuth server
(backed by an LDAP identity provider configured at the cluster level). Users log in
with their LDAP credentials via the standard OpenShift login page. Grafana never
communicates with LDAP directly.

**Architecture summary:**
```
Browser → Route (TLS edge) → oauth-proxy sidecar (port 4180)
                                  ↓  validates token via OpenShift OAuth
                              Grafana (port 3000, loopback-only)
                              [auth.proxy] trusts X-Forwarded-User header
```

**SCC:** `nonroot` — Grafana runs as UID 472, oauth-proxy as UID 65532. No `anyuid`
is used anywhere.

---

## Prerequisites

### Tools

| Tool | Minimum version | Purpose |
|---|---|---|
| `oc` | 4.10 | Apply manifests, inspect resources |
| `kubectl` / `kustomize` | 1.24 / 5.0 | Alternative to `oc apply -k` |
| `git` | any | Clone the repository |
| `openssl` | any | Generate secrets |
| `python3` | 3.6+ | Generate cookie secret |

Verify:

```bash
oc version
oc whoami          # must be logged in
oc whoami --show-server
```

### Cluster-admin prerequisites

The following must be in place **before** running the installation. They require
`cluster-admin` or equivalent privileges and are outside the scope of these manifests.

**1. LDAP identity provider configured on the cluster**

```bash
# Confirm an LDAP IDP exists (cluster-admin required)
oc get oauth cluster -o jsonpath='{.spec.identityProviders[*].type}{"\n"}'
# Expected output includes: LDAP
```

If no LDAP IDP is present, ask your cluster admin to add one:

```bash
# Example — cluster admin edits the cluster OAuth config
oc edit oauth cluster
```

Minimal LDAP IDP entry:

```yaml
identityProviders:
- name: ldap
  type: LDAP
  mappingMethod: claim
  ldap:
    attributes:
      id:         ["dn"]
      email:      ["mail"]
      name:       ["cn"]
      preferredUsername: ["uid"]
    bindDN: "cn=service-account,dc=example,dc=com"
    bindPassword:
      name: ldap-bind-password   # Secret in openshift-config namespace
    insecure: false
    url: "ldaps://ldap.example.com/dc=example,dc=com?uid?sub"
```

**2. Default StorageClass available**

```bash
oc get storageclass
# At least one StorageClass must be marked (default)
```

If no default exists, set one or specify `storageClassName` explicitly in
`grafana/overlays/<env>/kustomization.yaml` before applying.

**3. Wildcard DNS for the cluster**

```bash
# Confirm the wildcard domain — new Routes auto-assign hostnames under this domain
oc get ingresses.config.openshift.io cluster \
  -o jsonpath='{.spec.domain}{"\n"}'
```

**4. `system:openshift:scc:nonroot` ClusterRole exists**

This is present by default on all OCP 4.x clusters. Verify:

```bash
oc get clusterrole system:openshift:scc:nonroot
```

### Permissions required for the installer

The user or service account running `oc apply` needs the following:

```bash
# Minimum namespace-scoped permissions (all in the grafana namespace)
oc policy can-i create namespace            # to create the Namespace
oc policy can-i create serviceaccount -n grafana
oc policy can-i create rolebinding -n grafana
oc policy can-i create configmap -n grafana
oc policy can-i create secret -n grafana
oc policy can-i create deployment -n grafana
oc policy can-i create service -n grafana
oc policy can-i create route -n grafana
oc policy can-i create persistentvolumeclaim -n grafana

# Cluster-scoped permissions (requires cluster-admin or custom ClusterRole)
oc policy can-i create clusterrolebinding   # for auth-delegator-crb.yaml
```

If you do not have `ClusterRoleBinding` create rights, ask your cluster admin to apply
`grafana/base/rbac/auth-delegator-crb.yaml` separately.

---

## Step 1 — Clone the repository

```bash
git clone <repository-url>
cd Grafana
```

Directory structure relevant to this installation:

```
grafana/
├── base/                    # Environment-agnostic manifests
└── overlays/
    ├── dev/                 # Development environment settings
    └── prod/                # Production environment settings
docs/
├── installation.md          # This document
└── troubleshooting.md
```

---

## Step 2 — Determine the Route hostname

The Route hostname must be set before generating secrets, because the oauth-proxy
embed it in the OAuth callback URL.

```bash
# Get the cluster wildcard domain
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster \
  -o jsonpath='{.spec.domain}')
echo $CLUSTER_DOMAIN
# example: apps.cluster.example.com

# Your Grafana hostname will be:
echo "grafana.${CLUSTER_DOMAIN}"
```

You can use the auto-assigned hostname (`grafana.<cluster-domain>`) or a custom
hostname. If using a custom hostname, ensure DNS resolves it to the OCP router before
applying.

---

## Step 3 — Configure the overlay

Edit the overlay for your target environment. Use `overlays/dev/` for development and
`overlays/prod/` for production.

```bash
OVERLAY=grafana/overlays/dev    # or overlays/prod
HOSTNAME="grafana.${CLUSTER_DOMAIN}"
```

Open `${OVERLAY}/kustomization.yaml` and replace every occurrence of
`CLUSTER_DOMAIN` with your actual cluster domain:

```bash
sed -i "s/CLUSTER_DOMAIN/${CLUSTER_DOMAIN}/g" ${OVERLAY}/kustomization.yaml
```

Verify the result:

```bash
grep -n "grafana.apps" ${OVERLAY}/kustomization.yaml
# Should show two lines:
#   value: grafana.apps.<your-cluster-domain>
```

For production, also set the storage class:

```bash
# Find the correct storage class name
oc get storageclass

# Edit overlays/prod/kustomization.yaml and replace:
#   value: "REPLACE_WITH_PROD_STORAGE_CLASS"
# with the actual class name (e.g. gp3-csi, managed-csi, ocs-storagecluster-ceph-rbd)
```

---

## Step 4 — Generate and apply secrets

Secrets must be created **outside** of git. The `grafana-secrets.yaml` file in the
repository contains only placeholder values and must not be applied directly.

### Option A — `oc create secret` (recommended for initial install)

```bash
oc new-project grafana 2>/dev/null || true   # create namespace if it doesn't exist

oc create secret generic grafana-secrets \
  -n grafana \
  --from-literal=GF_SECURITY_ADMIN_PASSWORD="$(openssl rand -base64 32)" \
  --from-literal=OAUTH_COOKIE_SECRET="$(python3 -c 'import os,base64; print(base64.b64encode(os.urandom(24)).decode())')" \
  --from-literal=OAUTH_CLIENT_SECRET="$(openssl rand -hex 32)"
```

Save the admin password somewhere secure before continuing:

```bash
oc get secret grafana-secrets -n grafana \
  -o jsonpath='{.data.GF_SECURITY_ADMIN_PASSWORD}' | base64 -d
```

> **Note:** `OAUTH_CLIENT_SECRET` is generated here for completeness but is only
> used if you enable the explicit `oauthclient.yaml` manifest. In the default SA
> token mode it is not consumed.

### Option B — Kustomize secretGenerator (CI/CD pipelines)

Create a `.env` file that is **not committed to git** (add it to `.gitignore`):

```bash
cat > grafana/.secrets.env <<EOF
GF_SECURITY_ADMIN_PASSWORD=$(openssl rand -base64 32)
OAUTH_COOKIE_SECRET=$(python3 -c 'import os,base64; print(base64.b64encode(os.urandom(24)).decode())')
OAUTH_CLIENT_SECRET=$(openssl rand -hex 32)
EOF
chmod 600 grafana/.secrets.env
echo "grafana/.secrets.env" >> .gitignore
```

Add a `secretGenerator` to your overlay's `kustomization.yaml`:

```yaml
secretGenerator:
- name: grafana-secrets
  envs:
  - ../../.secrets.env
  options:
    disableNameSuffixHash: true
```

Then remove the `secrets/grafana-secrets.yaml` line from the `resources` list in
`base/kustomization.yaml` to avoid a conflict.

### Option C — External secrets operator (production)

Replace `base/secrets/grafana-secrets.yaml` with an `ExternalSecret` CR pointing to
your vault (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, etc.). Consult
your secrets operator documentation for the exact syntax. The required secret keys are:

- `GF_SECURITY_ADMIN_PASSWORD`
- `OAUTH_COOKIE_SECRET`
- `OAUTH_CLIENT_SECRET` (only needed for explicit OAuthClient mode)

---

## Step 5 — Apply the manifests

### Dry run first

Preview all resources that will be created without applying them:

```bash
oc apply -k ${OVERLAY} --dry-run=client
```

Review the output. Confirm:
- Namespace `grafana` will be created (or already exists)
- `ClusterRoleBinding grafana-auth-delegator` is included
- The Route hostname matches what you set in Step 3

### Apply

```bash
oc apply -k ${OVERLAY}
```

Expected output:

```
namespace/grafana created (or configured)
serviceaccount/grafana created
rolebinding.rbac.authorization.k8s.io/grafana-nonroot-scc created
clusterrolebinding.rbac.authorization.k8s.io/grafana-auth-delegator created
configmap/grafana-config created
persistentvolumeclaim/grafana-data created
deployment.apps/grafana created
service/grafana created
route.openshift.io/grafana created
```

> If you see `grafana-secrets created` in the output, the placeholder Secret was
> applied. Delete it and re-create it with real values:
> ```bash
> oc delete secret grafana-secrets -n grafana
> # Re-run the oc create secret command from Step 4
> ```

---

## Step 6 — Verify the deployment

### 6a. Pod status

```bash
oc get pods -n grafana -w
```

Wait until the pod shows `2/2 Running`. Both containers (`grafana` and `oauth-proxy`)
must be ready. This typically takes 30–60 seconds on the first start (PVC provisioning
+ fsGroup chown + Grafana database initialisation).

If the pod does not reach `Running` within 3 minutes, check events:

```bash
oc get events -n grafana --sort-by='.lastTimestamp' | tail -20
oc logs -n grafana deployment/grafana -c grafana     --tail=30
oc logs -n grafana deployment/grafana -c oauth-proxy --tail=30
```

### 6b. SCC assigned

```bash
oc get pod -n grafana -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}{"\n"}'
# Must return: nonroot
```

### 6c. OAuthClient auto-created

```bash
oc get oauthclient system:serviceaccount:grafana:grafana -o yaml
```

Confirm:
- `redirectURIs` contains `https://<your-hostname>/oauth/callback`
- The entry exists (may take a few seconds after pod start)

### 6d. Route accessible

```bash
GRAFANA_URL="https://$(oc get route grafana -n grafana \
  -o jsonpath='{.spec.host}')"
echo "Grafana URL: ${GRAFANA_URL}"

# Health check — bypasses OAuth (--skip-auth-regex in proxy args)
curl -sk "${GRAFANA_URL}/api/health"
# Expected: {"commit":"...","database":"ok","version":"10.4.3"}
```

### 6e. Grafana internal loopback binding

```bash
oc exec -n grafana deployment/grafana -c grafana -- \
  wget -qO- http://localhost:3000/api/health
# Expected: {"commit":"...","database":"ok","version":"10.4.3"}
```

Confirm port 3000 is bound only to loopback:

```bash
oc exec -n grafana deployment/grafana -c grafana -- \
  sh -c 'ss -tlnp 2>/dev/null || netstat -tlnp 2>/dev/null' | grep 3000
# Must show 127.0.0.1:3000
```

---

## Step 7 — First login

1. Open `${GRAFANA_URL}` in a browser.
2. You will be redirected to the OpenShift OAuth login page.
3. Log in with your LDAP credentials (the same credentials used to log in to the
   OpenShift console).
4. On first login, Grafana creates a user record and assigns the `Viewer` role.
5. You will land on the Grafana home dashboard.

> **Admin login:** The Grafana `admin` user can log in via the API using the password
> stored in `GF_SECURITY_ADMIN_PASSWORD`. The login form is disabled in the UI, so
> admin access from a browser requires temporarily re-enabling it or using the API:
> ```bash
> ADMIN_PASS=$(oc get secret grafana-secrets -n grafana \
>   -o jsonpath='{.data.GF_SECURITY_ADMIN_PASSWORD}' | base64 -d)
> curl -sk -u "admin:${ADMIN_PASS}" "${GRAFANA_URL}/api/org" | python3 -m json.tool
> ```

---

## Step 8 — Post-install configuration

### Assign roles to LDAP users or groups

New users created via OAuth proxy are assigned the `Viewer` role by default
(`auto_assign_org_role = Viewer` in grafana.ini). To change a user's role:

**Via the Grafana UI** (as admin):
Administration → Users → select user → change role

**Via the Grafana API:**

```bash
ADMIN_PASS=$(oc get secret grafana-secrets -n grafana \
  -o jsonpath='{.data.GF_SECURITY_ADMIN_PASSWORD}' | base64 -d)

# List users
curl -sk -u "admin:${ADMIN_PASS}" "${GRAFANA_URL}/api/org/users" \
  | python3 -m json.tool

# Promote a user to Editor (replace <userId> with the numeric ID from above)
curl -sk -u "admin:${ADMIN_PASS}" \
  -X PATCH "${GRAFANA_URL}/api/org/users/<userId>" \
  -H "Content-Type: application/json" \
  -d '{"role":"Editor"}'
```

To change the default role for all new users, edit `grafana-config.yaml`:

```bash
oc edit configmap grafana-config -n grafana
# Change: auto_assign_org_role = Viewer → Editor (or Admin)
oc rollout restart deployment/grafana -n grafana
```

### Grant access to Grafana via OpenShift RBAC

The `--openshift-sar` flag on the proxy requires users to have `get services`
permission in the `grafana` namespace. Grant this to a specific user or LDAP group
(synced to an OCP Group):

```bash
# Individual user
oc policy add-role-to-user view <ocp-username> -n grafana

# OCP Group (use the group name from `oc get groups`)
oc policy add-role-to-group view <ocp-group-name> -n grafana
```

### Provision data sources and dashboards via ConfigMaps

Add data sources or dashboards to the provisioning directory by mounting additional
ConfigMaps. Example for a Prometheus data source:

```yaml
# grafana/base/configmaps/grafana-datasources.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: grafana
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus-k8s.openshift-monitoring.svc:9090
      isDefault: true
```

Mount it in `deployment.yaml` (add to volumes and volumeMounts):

```yaml
# Under volumes:
- name: grafana-datasources
  configMap:
    name: grafana-datasources
    defaultMode: 0440

# Under containers[grafana].volumeMounts:
- name: grafana-datasources
  mountPath: /etc/grafana/provisioning/datasources
  readOnly: true
```

---

## Updating the deployment

### Update Grafana version

Edit the image tag in the overlay's `kustomization.yaml`:

```yaml
images:
- name: grafana/grafana
  newTag: "10.5.0"   # new version
```

Apply and monitor the rollout:

```bash
oc apply -k ${OVERLAY}
oc rollout status deployment/grafana -n grafana
```

Because the strategy is `Recreate`, there will be brief downtime during the update.
The SQLite database is automatically migrated by Grafana on startup.

### Update grafana.ini configuration

Edit `grafana/base/configmaps/grafana-config.yaml`, then:

```bash
oc apply -k ${OVERLAY}
oc rollout restart deployment/grafana -n grafana
```

### Rotate secrets

```bash
# Generate new values
NEW_ADMIN_PASS=$(openssl rand -base64 32)
NEW_COOKIE_SECRET=$(python3 -c 'import os,base64; print(base64.b64encode(os.urandom(24)).decode())')

# Update the Secret
oc set data secret/grafana-secrets -n grafana \
  "GF_SECURITY_ADMIN_PASSWORD=${NEW_ADMIN_PASS}" \
  "OAUTH_COOKIE_SECRET=${NEW_COOKIE_SECRET}"

# Restart to pick up new values (env vars are never hot-reloaded)
oc rollout restart deployment/grafana -n grafana
```

> Rotating `OAUTH_COOKIE_SECRET` invalidates all active browser sessions. All users
> will be required to log in again via OpenShift OAuth.

---

## Uninstall

```bash
# Delete all namespace-scoped resources
oc delete -k ${OVERLAY}

# Delete the ClusterRoleBinding (cluster-scoped, not removed by namespace delete)
oc delete clusterrolebinding grafana-auth-delegator

# Delete the auto-created OAuthClient
oc delete oauthclient system:serviceaccount:grafana:grafana 2>/dev/null || true

# Delete the namespace (also removes the PVC and all data)
oc delete namespace grafana
```

> To preserve Grafana data (dashboards, users, alert rules stored in SQLite), back up
> the PVC before deleting the namespace:
> ```bash
> oc exec -n grafana deployment/grafana -c grafana -- \
>   tar czf - /var/lib/grafana/grafana.db | gzip > grafana-backup-$(date +%F).tar.gz
> ```

---

## Quick reference

| Resource | Name | Namespace |
|---|---|---|
| Namespace | `grafana` | — |
| ServiceAccount | `grafana` | `grafana` |
| RoleBinding (SCC) | `grafana-nonroot-scc` | `grafana` |
| ClusterRoleBinding | `grafana-auth-delegator` | — |
| ConfigMap | `grafana-config` | `grafana` |
| Secret | `grafana-secrets` | `grafana` |
| PVC | `grafana-data` | `grafana` |
| Deployment | `grafana` | `grafana` |
| Service | `grafana` (port 4180) | `grafana` |
| Route | `grafana` | `grafana` |
| OAuthClient (auto) | `system:serviceaccount:grafana:grafana` | — |

| Port | Container | Purpose |
|---|---|---|
| 4180 | `oauth-proxy` | Public entry point — Service and Route target |
| 3000 | `grafana` | Internal only — bound to 127.0.0.1, no Service |

| File | Purpose |
|---|---|
| `grafana/base/configmaps/grafana-config.yaml` | `grafana.ini` — all Grafana settings |
| `grafana/base/secrets/grafana-secrets.yaml` | Placeholder only — use `oc create secret` |
| `grafana/overlays/dev/kustomization.yaml` | Dev environment patches |
| `grafana/overlays/prod/kustomization.yaml` | Prod environment patches |
| `docs/troubleshooting.md` | Problem diagnosis and fixes |
