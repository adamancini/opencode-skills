---
topic: replicated-sdk
source: https://docs.replicated.com/vendor/replicated-sdk-customizing#customize-rbac-for-the-sdk
created: 2026-03-18
updated: 2026-03-18
tags:
  - replicated
  - replicated-sdk
  - rbac
  - kubernetes
  - security
  - license
  - operations
---

# Replicated SDK: Minimal RBAC for License-Enforcement Only

## Summary

When a customer has strict RBAC constraints (no `create`, `list`, `patch`, or `watch`; only
scoped `get` where possible), the SDK's built-in `minimalRBAC: true` mode is still too broad.
By setting `statusInformers: []` and providing a custom ServiceAccount bound to the Role
below, the SDK can be reduced to **`get`-only permissions on five named resources** with no
`create`, `list`, `watch`, `patch`, or `update` at all.

This was derived from source-code analysis of `replicatedhq/replicated-sdk` at v1.18.0 and
verified by a live test against a CMX k3s 1.32 cluster.

## Prerequisites / Helm Values

```yaml
minimalRBAC: true         # suppress the default broad role
statusInformers: []       # MUST be explicit empty slice, not null/omitted
serviceAccountName: replicated  # use the pre-created SA below
```

`statusInformers` must be an explicit empty list (`[]`), not null. When null, the SDK
auto-discovers informers from the Helm release secret, requiring `list` on secrets.

## What the SDK Does at Runtime (Online, Non-Airgap, Production License)

### Startup bootstrap (`pkg/apiserver/bootstrap.go`)

| Operation | K8s call | Notes |
|---|---|---|
| Get replicatedID/appID | `get configmaps/replicated-sdk` | Gracefully handles `Forbidden` — falls back to deployment UID |
| Fallback for IDs | `get deployments/replicated` | Named; also used every heartbeat |
| Integration mode check | `get secrets/replicated` | **Only runs for `dev` licenses.** Returns `false` immediately for production. |
| Distribution detection | `nodes.list` (via discovery) | Fails gracefully — just skips distribution tagging |

### Heartbeat `canReport` check (`pkg/report/util.go`, every 4h)

| Operation | K8s call | Scopeable by resourceName? |
|---|---|---|
| Check deployment revision | `get deployments/replicated` | Yes — named |
| Check pod revision | `get pods/<REPLICATED_POD_NAME>` | No — name is dynamic (downward API) |
| Check replicaset revision | `get replicasets/<owner-ref-name>` | No — name is dynamic (hash suffix) |

### Online instance reporting (`pkg/report/instance.go`)

For non-airgap deployments, `SendOnlineInstanceData` posts directly to `replicated.app` via
HTTP. No K8s secrets are read or written.

### What is NOT needed for license-only, online, non-airgap

- `create` secrets — only used in the **airgap** path (`SendAirgapInstanceData`)
- `update` secrets — only used in the airgap path
- `list` / `watch` anything — only needed for status informers
- `patch` anything — never needed
- `get`/`list` secrets for Helm release storage — only needed when `statusInformers` is null

## Minimal RBAC Manifests

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: replicated
  namespace: <your-namespace>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: replicated-minimal
  namespace: <your-namespace>
rules:

# canReport: verify this pod is the active deployment revision (heartbeat, every 4h)
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get"]
  resourceNames: ["replicated"]   # fully scoped by name

# canReport: look up the ReplicaSet this pod belongs to
# Name is dynamic (e.g., "replicated-5d78b4f9b8") — cannot use resourceNames
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get"]

# canReport: look up this pod's own revision
# Name comes from REPLICATED_POD_NAME env var (downward API) — cannot use resourceNames
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]

# SDK secret: holds integration-mode flag (checked for dev licenses only).
# replicated-meta-data: instance tag data (read-only, best-effort).
# replicated-support-metadata: support bundle metadata (only if that API endpoint is used).
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames:
  - "replicated"
  - "replicated-meta-data"
  - "replicated-support-metadata"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: replicated-minimal
  namespace: <your-namespace>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: replicated-minimal
subjects:
- kind: ServiceAccount
  name: replicated
  namespace: <your-namespace>
```

## Helm Install Command

```bash
helm install replicated oci://registry.replicated.com/library/replicated \
  --namespace <your-namespace> \
  --set-file license=/path/to/license.yaml \
  --set minimalRBAC=true \
  --set serviceAccountName=replicated \
  --set-json 'statusInformers=[]'
```

Note: pass the license as `--set-file`, not `--set` with a base64-encoded value. The chart
handles encoding internally; passing pre-encoded content causes double-encoding and a parse
error at startup.

## Comparison to Default `minimalRBAC: true`

The built-in `minimalRBAC: true` template still includes:

| Permission | Default minimalRBAC | This config |
|---|---|---|
| `secrets: create` | ✅ (unconditional) | ❌ removed |
| `secrets: update` (4 named) | ✅ | ❌ removed |
| `secrets: get/list` (Helm release discovery) | ✅ when statusInformers unset | ❌ removed |
| `pods: get/list/watch` | ✅ | `get` only |
| `deployments/replicasets: get` (all workloads) | ✅ | named `get` only |
| `statefulsets/daemonsets/services/ingresses/pvcs: get/list/watch` | ✅ | ❌ removed |
| `configmaps/replicated-sdk: get` | ✅ | ❌ removed (Forbidden handled gracefully) |

## Airgap Caveat

For **airgap** deployments, `create` on secrets cannot be avoided. `SendAirgapInstanceData`
creates `replicated-instance-report` and `replicated-custom-app-metrics-report` secrets when
they don't exist (code: `pkg/report/report.go:AppendReport`). There is no configuration flag
to disable this — it would require a code change.

The additional permissions needed for airgap:

```yaml
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["update"]
  resourceNames:
  - "replicated-instance-report"
  - "replicated-custom-app-metrics-report"
```

## Dev License Note

For **dev licenses**, `integration.IsEnabled` (`pkg/integration/integration.go`) calls
`get secrets/replicated` during every bootstrap. For **production licenses** this function
returns `false` immediately with no K8s API call. The `replicated` secret is included in the
Role above because `dev` licenses are common in testing; remove it safely for production-only
deployments if desired.

## Operational Concerns

### SDK upgrades

The `canReport` revision check is designed to handle rolling updates: it compares the pod's
ReplicaSet revision to the current Deployment revision and suppresses reporting from
terminating pods. This works correctly under minimal RBAC — only `get` is needed, no
additional permissions are required during an upgrade.

The forward-compatibility risk: **if a future SDK version adds a new K8s API call not covered
by the Role, it will fail at that call.** Because the RBAC grants are not managed by the SDK
chart when `serviceAccountName` is externally provided, the customer must manually audit
`chart/templates/replicated-role.yaml` and the SDK changelog on every upgrade.

### Endpoints broken under minimal RBAC

Not all SDK endpoints work with get-only permissions. The following will fail:

#### `GET /api/v1/app/info` (partial) and `GET /api/v1/app/history` (full failure)

Both call `helm.GetRelease()` / `helm.GetReleaseHistory()` when `IS_HELM_MANAGED=true`. The
Helm secrets storage driver needs `list` on secrets to locate the release record.

- `/api/v1/app/info`: returns `500` for Helm revision fields (`helmReleaseName`,
  `helmReleaseRevision`, `helmReleaseNamespace`, `deployedAt`). Core fields (version label,
  channel info, release notes) come from the in-memory store and work fine.
- `/api/v1/app/history`: returns `500` entirely.

Fix: add `list` on secrets (scoped to the release namespace), or set `IS_HELM_MANAGED=false`.

#### `POST/PATCH /api/v1/app/custom-metrics` and `DELETE /api/v1/app/custom-metrics/{key}`

Calls `meta.SyncCustomAppMetrics` → `get` + `create`/`update` on `replicated-meta-data`.
The `get` is allowed; the write is not. Returns `500`.

Fix — add to the Role:
```yaml
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["update"]
  resourceNames: ["replicated-meta-data"]
```

#### `POST /api/v1/app/instance-tags`

Same code path as custom metrics — `get` + `create`/`update` on `replicated-meta-data`.
The additional rules above cover this as well.

#### `POST/PATCH /api/v1/supportbundle/metadata`

Calls `get` + `update` on `replicated-support-metadata`. The secret is pre-created by the
Helm chart so `create` is not needed, but `update` is. Returns `500`.

Fix — add to the Role:
```yaml
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["update"]
  resourceNames: ["replicated-support-metadata"]
```

### TLS (`tlsCertSecretName`)

If `tlsCertSecretName` is set, `loadTLSConfig` does `get secrets/<tlsCertSecretName>` at
startup. This is a **fatal error path** — a Forbidden response causes the process to exit.
Add the TLS cert secret name to the `resourceNames` list in the secrets `get` rule.

### HA mode (`replicaCount > 1`)

Works correctly under minimal RBAC. Each pod independently runs `canReport` against the same
named Deployment — only the pod matching the current Deployment revision reports. No `list`
is needed; each pod only inspects itself and its own owner ReplicaSet.

### Endpoint / behavior summary

| Endpoint / behavior | Works? | Notes |
|---|---|---|
| `GET /api/v1/license/info` | ✅ | Fully functional |
| `GET /api/v1/license/fields` | ✅ | Fully functional |
| `GET /api/v1/license/fields/{name}` | ✅ | Fully functional |
| `GET /api/v1/app/info` | ⚠️ Partial | Helm revision fields empty; core info works |
| `GET /api/v1/app/status` | ✅ | Reports `ready` with empty informers |
| `GET /api/v1/app/updates` | ✅ | HTTP to replicated.app only |
| `GET /api/v1/app/history` | ❌ | Requires `list` on secrets |
| `POST/PATCH /api/v1/app/custom-metrics` | ❌ | Requires `create`/`update` on `replicated-meta-data` |
| `DELETE /api/v1/app/custom-metrics/{key}` | ❌ | Same as above |
| `POST /api/v1/app/instance-tags` | ❌ | Same as custom metrics |
| `POST/PATCH /api/v1/supportbundle/metadata` | ❌ | Requires `update` on `replicated-support-metadata` |
| Heartbeat / instance reporting | ✅ | Online path is HTTP only; `canReport` is `get`-only |
| Rolling upgrades | ✅ | Revision check is `get`-only, works as designed |
| HA (`replicaCount > 1`) | ✅ | No additional permissions needed |
| TLS (`tlsCertSecretName` set) | ⚠️ | Add TLS secret name to `resourceNames` or startup is fatal |
| Future SDK upgrades | ⚠️ | Must manually audit role requirements on each upgrade |

## Test Verification

Tested on CMX k3s 1.32 cluster, SDK v1.18.0, wg-easy app, dev license:

- `GET /api/v1/license/info` — returned full license JSON ✅
- `GET /api/v1/license/fields` — returned entitlements ✅
- `GET /healthz` — `{"version":"1.18.0"}` ✅
- SDK logs — zero `forbidden`, `error`, or `failed` entries after clean startup ✅
- App state — transitioned to `ready` immediately (empty statusInformers) ✅
