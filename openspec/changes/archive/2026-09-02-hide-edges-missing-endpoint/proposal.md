## Why

Hiding a compound kind (the reproducing case: the legend eye on `deployment` in controller mode) leaves `pod-mounts-pvc` edges drawn against the still-visible PVC, even though the pod endpoint is off-canvas. Cytoscape's effective visibility is the AND of a node and all its ancestors, so the child is gone from the canvas while the filter still treats both endpoints as present. A drawn edge MUST have both sides on the canvas; otherwise it MUST hide.

## What Changes

- **Add** a panel-rendering requirement: after filtering, every remaining drawn edge MUST have both endpoints on the canvas. A node is on the canvas only when it is visible **and** every ancestor compound is visible. An edge that fails that test MUST hide, even when its edge type is still enabled.
- The existing kind-filter, endpoint-kind, and orphan-cascade requirements stay as they are. This change does not rewrite them.
- The rule is kind-agnostic and edge-type-agnostic. Hiding `deployment` / `statefulset` / `node` / any other compound is the same bug; `pod-mounts-pvc` is the reported instance, not a special case.
- No new panel option, no new legend control, no layout rerun. Filter hiding stays `visibility: hidden`.

No **BREAKING** change: every currently-correct graph (both endpoints already on-canvas) is unchanged. Graphs that currently show a one-sided edge start hiding it.

## Capabilities

### New Capabilities

<!-- None. The rule belongs on the existing `panel-rendering` capability as an added requirement. -->

### Modified Capabilities

- `panel-rendering`: **add** a requirement that a drawn edge MUST have both endpoints on the canvas (including when an endpoint is off-canvas only because an ancestor compound was hidden). Do not rewrite the existing kind-filter / endpoint / orphan-cascade requirements.

## Impact

- **Modified**: `src/features/element-filter/computeVisibility.ts` and `computeVisibility.test.ts` (the failing case is controller-mode `deployment > pod` plus `pod-mounts-pvc` to a still-visible PVC, with the deployment kind filtered out).
- **Unaffected**: `useElementFilter` (still applies the sets; it does not decide them), the legend, collapse, ingress toggle, wire contract, fixture, and layout.
