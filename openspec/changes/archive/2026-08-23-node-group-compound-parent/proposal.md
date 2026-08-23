## Why

A cluster's K8s `node` boxes sit as direct children of the cluster container, scattered among
the workload / storage / service boxes that share that level. There is no way to fold them
away as a unit, and no visual statement that "these are the machines". The panel already
synthesizes exactly this kind of purely-visual wrapper for the switch fabric
(`wrapSwitchFabric` → `network > switch`), so the mechanism is proven — it just was never
applied to the nodes.

## What Changes

- Add a panel-synthesized **node-group** compound container between a cluster and its K8s
  `node` children, so the rendered hierarchy becomes `cluster > node group > node` (in
  `node` layout mode, `cluster > node group > node > pod`).
- One node group **per parent**: every `node`-kind element is re-parented under a group
  keyed on the parent it currently has. Nodes that are top-level (no parent) get their own
  top-level group.
- The group is **UI-only**: panel-owned, kind-less, carries no backend identity, and is NOT
  part of the wire contract. `WireGraph`, the fixture, and `normalizeGraph` are untouched.
- Applies to **both** pod-parent modes — the pass runs after `applyPodParentMode`, next to
  `wrapSwitchFabric`, so the group survives a mode flip.
- The group is **selectable**, so the expand-collapse extension draws its `+` / `−` cue and
  the user can fold every node of a cluster into one box. It defaults to expanded.
- **No legend entry.** The group is kind-less, so it contributes nothing to the icon
  "Node kinds" legend, and no new swatch section is added — the existing `Nodes` section
  keeps listing the K8s `node` containers themselves.
- No new accent colour: the box inherits its enclosing cluster's accent through the existing
  `node:parent` tint rule, so it reads as part of that cluster's family.

## Capabilities

### New Capabilities

- `node-group-compound`: the synthesized `cluster > node group > node` container — when it is
  created, when synthesis backs off, how it renders, and how it collapses.

### Modified Capabilities

<!-- None. `pod-parent-mode` describes what `applyPodParentMode` returns; the node-group is a
     later, independent view-transform pass — the same relationship `wrapSwitchFabric`
     already has with that spec. -->

## Impact

- **New**: `src/features/graph-data/wrapNodeGroup.ts` (+ test), exported from the
  `graph-data` barrel.
- **Modified**: `src/panels/KsgPanel/KsgPanel.tsx` — one more pass in the `elements` memo
  chain; `src/features/graph-canvas/styles/getStylesheet.ts` — group header + collapsed
  folder glyph; `src/shared/types/cytoscape.d.ts` — `isNodeGroup` declaration merge.
- **Unaffected**: the wire contract (`src/shared/types/wire.ts`), the fixture, `normalizeGraph`,
  `applyPodParentMode`, the legend feature, `computeVisibility` (a kind-less node is always
  visible and is reclaimed by the existing orphan cascade), and `buildSwitchConstraints`.
