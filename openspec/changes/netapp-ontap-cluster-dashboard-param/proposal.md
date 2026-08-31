## Why

Every dashboard-eligible node sends a `cluster` query parameter to `/dashboard` so the backend
can scope the dashboard it hands back — except the two NetApp kinds. `netapp-aggr` and
`netapp-node` belong to no Kubernetes cluster, so they carry no `labels.cluster` and have no
`isCluster` ancestor; `resolveCluster` therefore returns `undefined` and the request goes out
unscoped. Their ONTAP cluster name is right there in `labels.ontap_cluster` (and again as the
enclosing `storage-cluster` container's name), and `labels` is wholesale denylisted, so today
the panel throws that name away at exactly the moment the backend needs it.

## What Changes

- `assembleDashboardParams` resolves the `cluster` parameter for NetApp nodes from their ONTAP
  cluster: the nearest **`isStorageCluster`** ancestor's `data.storageCluster`, falling back to
  the node's own **`labels.ontap_cluster`**, falling back to omitting `cluster` as before.
- The parameter **name stays `cluster`** — one field for all node kinds, so the backend keeps a
  single `/dashboard` scoping key. The panel does not introduce an `ontap_cluster` parameter.
- Resolution stays symmetric with the existing rule: ancestor is authoritative over labels, and
  a node's own `data.cluster` still wins over both (own-wins). The K8s branch keeps priority —
  an `isCluster` ancestor is preferred over an `isStorageCluster` one, and `labels.cluster` over
  `labels.ontap_cluster` — so no existing node's parameter changes value.
- **Out of scope**: `netapp-aggr`'s `labels.node` (the owning ONTAP controller) stays unsent;
  `labels` remains in the denylist and no other label is promoted to a parameter.

## Capabilities

### New Capabilities

<!-- None. This narrows an existing parameter-assembly rule. -->

### Modified Capabilities

- `node-dashboard-url`: the **Dashboard 請求參數組裝** requirement's `cluster` bullet gains the
  ONTAP source (storage-cluster ancestor, then `labels.ontap_cluster`) plus scenarios for a
  NetApp node resolving `cluster` and for the K8s sources keeping priority.

## Impact

- **Modified**: `src/features/node-detail/assembleDashboardParams.ts` — `resolveCluster` walks
  for either cluster flag and gains the `labels.ontap_cluster` fallback;
  `src/features/node-detail/assembleDashboardParams.test.ts` — new cases.
- **Unaffected**: the wire contract (`src/shared/types/wire.ts`), the fixture, `normalizeGraph`,
  the denylist (`labels` stays excluded), `isDashboardEligible` / `resolveSelectedNode` (NetApp
  nodes were already eligible), `resolveController`, the pinned tooltip, and every non-NetApp
  node's existing parameter map.
