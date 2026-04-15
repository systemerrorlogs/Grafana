# Grafana on OpenShift — Troubleshooting Guide

This guide covers problems specific to the deployment architecture: Grafana running
under the `nonroot` SCC with an `openshift/oauth-proxy` sidecar authenticating users
against the OpenShift OAuth server (which delegates to an LDAP identity provider
configured at the cluster level).

---

## Quick-reference diagnostics

Run these first to get an overall picture before diving into specific sections.

```bash
# Pod and container status
oc get pods -n grafana -o wide

# Both containers must be Running. If either is not, check its logs:
oc logs -n grafana deployment/grafana -c grafana      --tail=50
oc logs -n grafana deployment/grafana -c oauth-proxy  --tail=50

# Events — surface SCC rejections, image pull failures, volume mount errors
oc get events -n grafana --sort-by='.lastTimestamp' | tail -30

# Route and Service
oc get route,svc -n grafana

# RBAC applied to the SA
oc get rolebindings,clusterrolebindings -n grafana \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.roleRef.name}{"\n"}{end}'

# Effective SCC for the running pod
oc get pod -n grafana -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}{"\n"}'

# OAuthClient auto-created by SA token mode
oc get oauthclient system:serviceaccount:grafana:grafana -o yaml
```

---

## 1. Pod fails to start — SCC / security context issues

### Symptom
Pod stays in `Pending` or is immediately evicted. Events contain messages like:

```
unable to validate against any security context constraint: ...
pods "grafana-..." is forbidden: unable to validate against any SCC
```

### Causes and fixes

**A. `nonroot` SCC not granted to the `grafana` ServiceAccount**

The `scc-rolebinding.yaml` RoleBinding must exist and reference the correct SA.

```bash
# Verify the RoleBinding exists
oc get rolebinding grafana-nonroot-scc -n grafana -o yaml

# If missing, re-apply
oc apply -f grafana/base/rbac/scc-rolebinding.yaml

# Confirm the SA can use the SCC
oc adm policy who-can use scc nonroot
```

**B. Wrong UID in securityContext**

The pod spec sets `runAsUser: 472`. If the cluster's admission webhook is overriding
this (e.g., a custom OPA/Gatekeeper policy that forces the namespace UID range), the
container will be rejected or start as the wrong user.

```bash
# Check if a Gatekeeper/Kyverno policy is overriding the UID
oc describe pod -n grafana -l app.kubernetes.io/name=grafana | grep -A5 "Security Context"

# The grafana container must show runAsUser: 472
# The oauth-proxy container must show runAsUser: 65532
```

If the cluster enforces namespace UID ranges and `472` falls outside the assigned
range, switch to omitting `runAsUser` from the pod spec and rely on the namespace's
UID range — but note that the PVC ownership fix (fsGroup) must still apply so
`/var/lib/grafana` is writable.

**C. `seccompProfile: RuntimeDefault` not supported**

Clusters older than OCP 4.11 may reject `seccompProfile`. Remove the field from
`deployment.yaml` if your cluster is on OCP 4.10 or earlier.

```bash
oc version  # confirm cluster version
```

---

## 2. Pod stuck in `ContainerCreating` — volume / PVC issues

### Symptom
```
Events:
  Warning  FailedMount  Unable to attach or mount volumes: ...
  Warning  FailedMount  timed out waiting for the condition
```

### Causes and fixes

**A. PVC not bound**

```bash
oc get pvc -n grafana
# STATUS must be Bound. If Pending:
oc describe pvc grafana-data -n grafana
```

- `no persistent volumes available for this claim` — no StorageClass can satisfy the
  request. Set `storageClassName` explicitly in the overlay to a class that exists on
  the cluster.
- `waiting for a volume to be created` — the StorageClass is dynamic but the
  provisioner is not running or has no capacity.

```bash
# List available StorageClasses
oc get storageclass

# Patch the PVC to use a specific class (re-create required if already Pending)
oc patch pvc grafana-data -n grafana \
  -p '{"spec":{"storageClassName":"your-storage-class"}}'
```

**B. Previous pod still holds the RWO volume**

The Deployment strategy is `Recreate`, which terminates the old pod before the new
one starts. If the old pod is stuck in `Terminating`, the PVC detach is blocked.

```bash
oc get pods -n grafana
# Force-delete a stuck Terminating pod only after confirming it is truly gone
# from the node (check node logs if unsure)
oc delete pod -n grafana <pod-name> --force --grace-period=0
```

**C. `/var/lib/grafana` not writable — fsGroup not applied**

The pod relies on `fsGroup: 472` + `fsGroupChangePolicy: Always` to make the PVC
mount writable as UID 472 without a UID-0 initContainer. If the storage driver does
not honour fsGroup (e.g., some NFS provisioners), Grafana will log:

```
GF_PATHS_DATA='/var/lib/grafana' is not writable.
```

```bash
# Test from inside the container
oc exec -n grafana deployment/grafana -c grafana -- ls -la /var/lib/grafana
# Expected: drwxrwsr-x ownership with GID 472
```

If the driver ignores fsGroup, the workaround is a privileged initContainer — but that
requires `anyuid`, which is explicitly disallowed here. Instead:

1. Use a storage class that supports fsGroup (ODF CephRBD, AWS gp3-csi, Azure disk).
2. Or pre-provision the PV with GID 472 ownership using a storage admin job.
3. Or annotate the PV with `pv.beta.kubernetes.io/gid: "472"` if the provisioner
   supports the GID annotation.

---

## 3. oauth-proxy fails to start or CrashLoopBackOff

### Symptom
`oc logs deployment/grafana -c oauth-proxy` shows errors at startup.

### A. Cookie secret wrong length

```
error: invalid cookie secret, must be 16, 24 or 32 bytes, got N
```

The `OAUTH_COOKIE_SECRET` value in the Secret must base64-decode to exactly 16, 24,
or 32 bytes. Generate a valid value:

```bash
# Generates a 24-byte secret (32 base64 chars)
python3 -c "import os,base64; print(base64.b64encode(os.urandom(24)).decode())"

# Update the Secret
oc set data secret/grafana-secrets -n grafana \
  OAUTH_COOKIE_SECRET="<new-value>"

# Restart the pod to pick up the new Secret value
oc rollout restart deployment/grafana -n grafana
```

### B. Cookie secret file not found

```
open /etc/oauth/secrets/cookie-secret: no such file or directory
```

The Secret volume in `deployment.yaml` projects only the `OAUTH_COOKIE_SECRET` key to
the path `cookie-secret`. Verify the Secret contains that key:

```bash
oc get secret grafana-secrets -n grafana \
  -o jsonpath='{.data}' | python3 -m json.tool
# OAUTH_COOKIE_SECRET must appear in the output
```

If the key is missing, add it:

```bash
oc set data secret/grafana-secrets -n grafana \
  OAUTH_COOKIE_SECRET="$(python3 -c 'import os,base64; print(base64.b64encode(os.urandom(24)).decode())')"
```

### C. `system:auth-delegator` ClusterRoleBinding missing

oauth-proxy calls the API server's TokenReview endpoint on every request. Without
`system:auth-delegator`, every token review returns 403 and users see a login loop.

```bash
oc get clusterrolebinding grafana-auth-delegator -o yaml

# If missing
oc apply -f grafana/base/rbac/auth-delegator-crb.yaml
```

Proxy logs will show:
```
error: failed to make token review: tokenreviews.authentication.k8s.io is forbidden
```

### D. oauth-proxy image version mismatch

The image is pinned to `quay.io/openshift/origin-oauth-proxy:4.15`. On an OCP 4.12
cluster, the 4.15 proxy may use API features not present in the older OAuth server.

```bash
oc version  # note cluster version
# Pin the proxy image to match the cluster minor version in the overlay:
# quay.io/openshift/origin-oauth-proxy:4.12
```

---

## 4. Browser redirects to OpenShift login, but login fails or loops

### A. OAuthClient redirect URI mismatch

After a successful LDAP login, OCP's OAuth server redirects to the `redirect_uri`
the proxy requested. If that URI does not match any `redirectURIs` in the OAuthClient,
OCP returns:

```
redirect_uri is not allowed
```

In SA token mode, OCP derives redirect URIs from the `oauth-redirectreference`
annotation on the ServiceAccount. If the Route hostname changed after the SA was
created, the OAuthClient may have stale URIs.

```bash
# Check the auto-created OAuthClient
oc get oauthclient system:serviceaccount:grafana:grafana -o yaml
# redirectURIs must contain: https://<current-route-hostname>/oauth/callback

# Check the actual Route hostname
oc get route grafana -n grafana -o jsonpath='{.spec.host}{"\n"}'

# If stale, delete the OAuthClient so OCP recreates it from the current annotation
oc delete oauthclient system:serviceaccount:grafana:grafana
# It will be recreated automatically on the next proxy startup — restart the pod:
oc rollout restart deployment/grafana -n grafana
```

If using the explicit `oauthclient.yaml` instead of SA token mode, update the
`redirectURIs` field to match the current Route hostname and re-apply.

### B. `root_url` in grafana.ini does not match the Route hostname

Grafana uses `root_url` to generate redirect links. A mismatch causes the sign-out
redirect to send the browser to the wrong host.

```bash
# Get the current Route hostname
oc get route grafana -n grafana -o jsonpath='{.spec.host}{"\n"}'

# Check what Grafana thinks root_url is
oc exec -n grafana deployment/grafana -c grafana -- \
  env | grep GF_SERVER_ROOT_URL
```

Override `root_url` via the `GF_SERVER_ROOT_URL` environment variable in the overlay
patch (see `overlays/dev/kustomization.yaml`) rather than editing the ConfigMap
directly. After updating the overlay, apply and restart:

```bash
oc apply -k grafana/overlays/dev/
oc rollout restart deployment/grafana -n grafana
```

### C. LDAP identity provider not configured at cluster level

The OpenShift OAuth server must have an LDAP identity provider configured before any
LDAP-based login will work. This is a cluster-admin concern and is outside the scope
of these manifests.

```bash
# Check the cluster OAuth configuration (requires cluster-admin)
oc get oauth cluster -o yaml

# identityProviders must contain an entry of type LDAP
```

If no LDAP IDP is present, work with the cluster admin to add one via:

```bash
oc edit oauth cluster
# Add an identityProvider entry of type LDAP
```

### D. `--openshift-sar` blocking users

The oauth-proxy is configured with:

```
--openshift-sar={"namespace":"grafana","resource":"services","verb":"get"}
```

A user who can authenticate to OCP but has no RBAC `get services` permission in the
`grafana` namespace will be denied with a 403 after login. This is intentional as an
authorization gate, but can be surprising.

```bash
# Test whether a specific user passes the SAR check
oc policy can-i get services -n grafana --as=<username>

# Grant access to a user or group (adjust as needed)
oc policy add-role-to-user view <username> -n grafana
# or for an LDAP group mapped to an OCP group:
oc policy add-role-to-group view <ocp-group-name> -n grafana
```

To remove the authorization gate entirely and allow any authenticated OCP user,
delete the `--openshift-sar` and `--openshift-delegate-urls` args from the Deployment.

---

## 5. Logged in but Grafana shows wrong role or "User not found"

### A. New user not auto-provisioned

`auto_sign_up = true` in `[auth.proxy]` should create a Grafana user record on first
login. If this is not happening, check that `X-Forwarded-User` is actually being sent:

```bash
# Tail Grafana logs during a login attempt
oc logs -n grafana deployment/grafana -c grafana -f | grep -i "proxy\|auth\|user"
```

Look for `"auth.proxy"` entries. If absent, the request is not reaching Grafana with
the header set — the issue is in oauth-proxy (see section 3) or in the whitelist check.

### B. `whitelist = 127.0.0.1` rejecting the header

If `X-Forwarded-User` arrives from any source other than `127.0.0.1`, Grafana strips
it silently and the user is not authenticated. This should never happen in the normal
flow (proxy is a sidecar on localhost), but can occur during local development or if
the pod networking is unusual.

```bash
oc logs -n grafana deployment/grafana -c grafana | grep "auth.proxy"
# Look for: "Request has X-Forwarded-User header but IP ... is not whitelisted"
```

### C. All users land as Viewer, not expected role

`auto_assign_org_role = Viewer` is the configured default. Role assignment must be
done manually inside Grafana (Configuration → Users) or by an admin after first login.

To change the default role for new users:

```bash
oc edit configmap grafana-config -n grafana
# Change auto_assign_org_role = Editor  (or Admin)
oc rollout restart deployment/grafana -n grafana
```

This only affects new user records created after the change. Existing users keep
their current role.

---

## 6. Grafana is accessible but shows a blank page or 502/504

### A. oauth-proxy cannot reach Grafana on localhost:3000

The proxy connects to `http://localhost:3000`. If the `grafana` container is not yet
ready (probe failures, slow start), the proxy returns a 502.

```bash
# Check the grafana container readiness
oc describe pod -n grafana -l app.kubernetes.io/name=grafana \
  | grep -A10 "Readiness"

# Check health directly from within the pod
oc exec -n grafana deployment/grafana -c grafana -- \
  wget -qO- http://localhost:3000/api/health
```

Expected output: `{"commit":"...","database":"ok","version":"10.4.3"}`

If the command hangs, Grafana is not listening. Check logs for startup errors:

```bash
oc logs -n grafana deployment/grafana -c grafana | tail -30
```

### B. `http_addr = 127.0.0.1` not being applied

If `GF_SERVER_HTTP_ADDR` is set elsewhere (e.g., another env var, a different
ConfigMap key), it can override the `grafana.ini` setting and cause Grafana to bind
to `0.0.0.0`, which changes behaviour but does not cause a 502.

Confirm the active binding:

```bash
oc exec -n grafana deployment/grafana -c grafana -- \
  sh -c 'ss -tlnp 2>/dev/null || netstat -tlnp 2>/dev/null' | grep 3000
# Must show 127.0.0.1:3000, not 0.0.0.0:3000
```

### C. Route timeout for long-running queries

The Route annotation `haproxy.router.openshift.io/timeout: 300s` sets a 5-minute
timeout. Long Grafana queries that exceed this will receive a 504 from HAProxy.

Increase the timeout if needed:

```bash
oc annotate route grafana -n grafana \
  haproxy.router.openshift.io/timeout=600s --overwrite
```

---

## 7. Sign-out does not fully log out

### Symptom
After clicking Sign Out in Grafana, the user is immediately logged back in without
being prompted for credentials.

### Cause
`signout_redirect_url = /oauth/sign_out` in grafana.ini sends the browser to the
oauth-proxy sign-out endpoint, which clears the proxy's session cookie. However, the
user's OpenShift OAuth session (held in a separate OCP cookie) is still valid. The
proxy immediately re-authenticates the user using that session.

### Fix

To force a full logout including the OCP session, the sign-out URL must also hit the
OCP OAuth server's logout endpoint:

```
signout_redirect_url = https://<oauth-server-host>/logout?redirect=https://<grafana-route-hostname>/oauth/sign_out
```

Find the OCP OAuth server hostname:

```bash
oc get route oauth-openshift -n openshift-authentication \
  -o jsonpath='{.spec.host}{"\n"}'
```

Update `grafana-config.yaml` with the full sign-out chain and restart:

```bash
oc rollout restart deployment/grafana -n grafana
```

---

## 8. ConfigMap or Secret changes not picked up

### Symptom
You edited `grafana-config` ConfigMap or `grafana-secrets` Secret but the pod still
uses the old values.

### Cause
Pods do not restart automatically when ConfigMaps or Secrets change (unless you use
a reloader like `stakater/reloader`). Volume-mounted files are eventually updated by
kubelet (within the `syncPeriod`), but environment variables injected from Secrets
are **never** updated in a running pod.

### Fix
Always restart the Deployment after any config or secret change:

```bash
oc rollout restart deployment/grafana -n grafana

# Watch the rollout
oc rollout status deployment/grafana -n grafana
```

For `Recreate` strategy, the old pod is terminated first. There will be a brief
downtime period while the new pod starts. This is expected with a `ReadWriteOnce` PVC.

---

## 9. Image pull failures (air-gapped clusters)

### Symptom
```
Failed to pull image "grafana/grafana:10.4.3": ... connection refused
Failed to pull image "quay.io/openshift/origin-oauth-proxy:4.15": ...
```

### Fix

Mirror both images to the internal OpenShift registry or a private registry:

```bash
# Mirror Grafana
skopeo copy \
  docker://grafana/grafana:10.4.3 \
  docker://registry.internal.example.com/grafana/grafana:10.4.3

# Mirror oauth-proxy
skopeo copy \
  docker://quay.io/openshift/origin-oauth-proxy:4.15 \
  docker://registry.internal.example.com/openshift/origin-oauth-proxy:4.15
```

Update the image references in the prod overlay's `kustomization.yaml`:

```yaml
images:
- name: grafana/grafana
  newName: registry.internal.example.com/grafana/grafana
  newTag: "10.4.3"
- name: quay.io/openshift/origin-oauth-proxy
  newName: registry.internal.example.com/openshift/origin-oauth-proxy
  newTag: "4.15"
```

---

## 10. Checking effective configuration at runtime

Verify what grafana.ini settings Grafana actually loaded (env var overrides included):

```bash
# View the mounted grafana.ini inside the container
oc exec -n grafana deployment/grafana -c grafana -- \
  cat /etc/grafana/grafana.ini

# View all active GF_* environment variable overrides
oc exec -n grafana deployment/grafana -c grafana -- \
  env | grep '^GF_' | sort

# Call Grafana's settings API (requires admin credentials)
GRAFANA_HOST=$(oc get route grafana -n grafana -o jsonpath='{.spec.host}')
curl -sk -u admin:"${GRAFANA_ADMIN_PASSWORD}" \
  https://${GRAFANA_HOST}/api/admin/settings | python3 -m json.tool | grep -A2 '"auth"'
```

View oauth-proxy startup flags as seen by the running process:

```bash
oc exec -n grafana deployment/grafana -c oauth-proxy -- \
  cat /proc/1/cmdline | tr '\0' '\n'
```

---

## 11. Useful log filters

```bash
# Grafana — all authentication events
oc logs -n grafana deployment/grafana -c grafana \
  | python3 -c "import sys,json; [print(json.dumps(json.loads(l), indent=2))
    for l in sys.stdin if 'auth' in l.lower()]" 2>/dev/null

# oauth-proxy — token validation errors
oc logs -n grafana deployment/grafana -c oauth-proxy \
  | grep -iE "error|forbidden|invalid|failed|denied"

# oauth-proxy — successful authentications
oc logs -n grafana deployment/grafana -c oauth-proxy \
  | grep "AuthSuccess\|authenticated"

# API server audit log (cluster-admin required) — TokenReview calls from grafana SA
oc adm node-logs --role=master --path=kube-apiserver/audit.log \
  | grep '"user":{"username":"system:serviceaccount:grafana:grafana"'
```

---

## 12. Full redeploy checklist

If the deployment is broken and you want a clean slate:

```bash
# 1. Delete the Deployment (keeps PVC data intact)
oc delete deployment grafana -n grafana

# 2. Verify the PVC is Released (if re-creating with a new PVC)
oc get pvc -n grafana

# 3. Delete the auto-created OAuthClient so it is recreated cleanly
oc delete oauthclient system:serviceaccount:grafana:grafana 2>/dev/null || true

# 4. Re-apply the full stack
oc apply -k grafana/overlays/dev/     # or prod

# 5. Watch startup
oc get pods -n grafana -w

# 6. Tail both container logs simultaneously
oc logs -n grafana deployment/grafana -c grafana     -f &
oc logs -n grafana deployment/grafana -c oauth-proxy -f &
```
