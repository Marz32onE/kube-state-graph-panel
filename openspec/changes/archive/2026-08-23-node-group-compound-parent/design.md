## Context

See `proposal.md` — Why. The panel's element pipeline already ends in a chain of pure view
transforms applied in `KsgPanel`:

```
normalizeGraph(payload)  →  applyPodParentMode(elements, mode)  →  wrapSwitchFabric(elements)
```

`wrapSwitchFabric` is the exact precedent for this change: a panel-owned compound wrapper
(`network > switch`) synthesized after normalization, absent from the wire contract, backing
off when the data already provides the grouping. This change adds a sibling pass with the
same shape.

Constraints that shape the approach:

- `applyPodParentMode`'s `node` mode walks the **original** parent chain to find each
  element's cluster ancestor. Any pass that inserts a tier into that chain must therefore run
  **after** it, never before.
- Cytoscape aliases the `data` object handed to `cy.add`, and expand-collapse mutates it in
  place. Every element leaving a view transform must be a fresh object (the rule
  `applyPodParentMode`'s `cloneElement` already enforces).
- The expand-collapse `+` / `−` cue is drawn only for a node that is **selected** and is a
  parent (or is already collapsed). A `selectable: false` container — the treatment clusters
  get — can never show the cue.

## Goals / Non-Goals

**Goals:**

- One more pure pass in the existing chain, testable without a live cytoscape instance.
- Zero change to the wire contract, the fixture, and `normalizeGraph`.
- The group reads as part of its cluster without inventing new colour vocabulary.

**Non-Goals:**

- No backend or wire representation of the group, now or later — it is a rendering
  convenience, not a resource.
- No legend surface (per the request), and no new swatch section.
- No status roll-up on the group (see decision D5).
- No change to how `applyPodParentMode` reshapes the hierarchy.

## Decisions

### D1 — A separate pass, run after `applyPodParentMode`, beside `wrapSwitchFabric`

`wrapNodeGroup(elements)` in `features/graph-data/`, composed in `KsgPanel`'s `elements` memo:

```ts
wrapNodeGroup(wrapSwitchFabric(applyPodParentMode(baseElements, podParentMode)));
```

_Why not inside `normalizeGraph`?_ The group is a view concern, and normalize is the
anti-corruption layer for the wire contract. Putting a panel-invented compound there would
make it indistinguishable from backend structure and would leak into every consumer that reads
pre-view-transform elements (the variable exports, `collectIngressNodeIds`).

_Why not inside `applyPodParentMode`?_ That function would then have to synthesize the group in
one branch and preserve it through the other's teardown pass, and its cluster-ancestor walk
would have to learn to skip the tier it just inserted. Keeping the two passes disjoint keeps
both simple. `wrapSwitchFabric` and `wrapNodeGroup` are order-independent (disjoint element
sets), so the composition order between them is arbitrary.

### D2 — Group by current parent, always, with no size threshold

Bucket every `node`-kind element by the `parent` it carries at that point in the pipeline
(including an `undefined` bucket for top-level nodes), one synthesized group per bucket, the
group inheriting the bucket's parent.

_Alternative considered:_ only wrap when a parent holds ≥2 nodes, to avoid a box around a
single machine. Rejected: the graph's shape would then depend on cluster size, and a cluster
would silently change structure — and lose the user's collapse state for that box — the moment
its second node appears. A predictable structure is worth one redundant box in the demo's `dr`
cluster.

_Why key on the current parent rather than on `data.cluster`?_ The parent chain is what
cytoscape actually renders, and it is the only thing correct in both modes. A cluster-name key
would need a fallback for every element the backend nests differently.

### D3 — Identity: `isNodeGroup`, kind-less, **selectable**

The group is declared via `isNodeGroup?: boolean` on `NodeDataDefinition` (declaration merging
in `shared/types/cytoscape.d.ts`), the same shape `isNamespace` / `isApplication` use. It is
kind-less so it drops out of the legend, the kind filter and the icon mapper for free.

It is **selectable** — unlike `isCluster` / `isStorageCluster`, which are `selectable: false`.
This is forced by the requirement to collapse via the `+` cue (see Context). The consequence is
that tapping the group opens the detail panel for a kind-less element, exactly as tapping a
namespace or application box already does; no new handling is needed.

_Alternative considered:_ keep it non-selectable and give it the cluster's double-tap gesture
(`clusterCollapseToggle`). Rejected — the request names the `+` button specifically, and
double-tap is an undiscoverable affordance for a box the user is expected to fold routinely.

Ids are `node-group/<parentId>` (and `node-group/` for the parent-less bucket) — namespaced so
a backend id cannot realistically collide, and deterministic so the collapsed-id set survives a
data refresh.

### D4 — No new accent colour; inherit the cluster's

The existing `node:parent` rule already tints a compound with `resolveParentClusterColor`,
which walks ancestors to the first `clusterColor`. A group sitting directly under a cluster
therefore picks up that cluster's accent with no new code and no new constant.

_Alternative considered:_ a `NODE_GROUP_COLOR` constant mirroring `NAMESPACE_COLOR` /
`APPLICATION_COLOR`. Rejected: those exist because their legend swatch must match the canvas
backplate, and this group has no legend. A fifth fixed accent would also compete with the
constraint set those palettes are tested against (must differ from every status colour, every
edge colour, and each other).

The collapsed-folder-glyph selector list in `getStylesheet` keys each entry on a `data(*Color)`
field; the node-group entry instead resolves the colour through the same
`resolveParentClusterColor` walk, since it has no colour field of its own.

### D5 — No `worstStatus` roll-up

A collapsed node group will not border in the worst status it hides.

_Why:_ `worstStatus` is written by `normalizeGraph` — for a K8s node from its `pod-to-node`
reachable pods, for a controller from its child pods. The purely decorative groups
(`cluster` / `namespace` / `application` / `storage-cluster`) deliberately have none, so a
collapsed cluster box already shows no status border. Adding a roll-up only here would make
the node group the one decorative box that signals health, which is a bigger behavioural
statement than this change is making. Revisit as its own change if collapsed-group health
turns out to matter — for all decorative groups, not just this one.

### D6 — Back off wholesale on collision

If any element already carries a generated id, or already carries `isNodeGroup`, return the
input unchanged. Mirrors `wrapSwitchFabric`'s "the data owns the grouping" back-off, and
guarantees the pass can never produce a hierarchy that is half synthesized.

## Risks / Trade-offs

- **One more nesting level for the layout engine** → fcose already handles
  `cluster > namespace > application > controller > pod` (five levels); one more under the
  cluster is within what it does today. Verify visually on the demo before merging.
- **A single-node cluster gets a redundant box** (D2) → accepted deliberately for structural
  predictability; visible in the demo's `dr` cluster, which has exactly one node.
- **Tapping the group opens a detail panel with almost nothing in it** (D3) → identical to
  today's namespace / application boxes; not a regression this change introduces.
- **A saved `visibleKinds` list can never hide the group** → intentional and consistent with
  the `network` wrapper: hiding a wrapper would hide everything nested inside it, since
  cytoscape's effective visibility is the AND over ancestors. The orphan cascade removes the
  box when its nodes are filtered out.

## Migration Plan

None — additive rendering behaviour, no persisted state, no panel option. Rollback is
reverting the single composition line in `KsgPanel`.
