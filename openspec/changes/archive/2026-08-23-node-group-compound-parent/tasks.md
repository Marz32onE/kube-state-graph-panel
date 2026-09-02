## 1. Element identity

- [x] 1.1 Add `isNodeGroup?: boolean` to `NodeDataDefinition` in `src/shared/types/cytoscape.d.ts` via declaration merging, documented as panel-owned (sibling of `isNamespace` / `isApplication`, but selectable) — verify `npm run typecheck` passes

## 2. The synthesis pass

- [x] 2.1 Write `src/features/graph-data/wrapNodeGroup.test.ts` FIRST, covering every scenario in `specs/node-group-compound/spec.md` — nodes of one cluster boxed together, one group per distinct parent, single-node cluster still grouped, parent-less nodes in a top-level group, input never mutated, no-node-kind no-op, id-collision back-off, `isNodeGroup` back-off — verify the suite fails (RED)
- [x] 2.2 Implement `src/features/graph-data/wrapNodeGroup.ts` as a pure function `(elements: readonly ElementDefinition[]) => ElementDefinition[]`: bucket `node`-kind elements by current `parent` (with an `undefined` bucket), synthesize one kind-less container per bucket with id `node-group/<parentId>` (`node-group/` for the parent-less bucket), label `nodes`, `isNodeGroup: true`, and the bucket's parent; re-parent the bucket's nodes under it — verify `npx jest src/features/graph-data/wrapNodeGroup.test.ts` passes (GREEN)
- [x] 2.3 Mirror `wrapSwitchFabric`'s node/edge discrimination (`group` field, falling back to "an edge always carries source + target") so an element set without explicit `group` is handled — verify the test file's mixed-input case passes
- [x] 2.4 Export `wrapNodeGroup` from `src/features/graph-data/index.ts` — verify `npm run lint` reports no barrel/import-boundary error

## 3. Pipeline composition

- [x] 3.1 In `src/panels/KsgPanel/KsgPanel.tsx`, wrap the existing `elements` memo as `wrapNodeGroup(wrapSwitchFabric(applyPodParentMode(baseElements, podParentMode)))`, updating the surrounding comment to name the third pass and why it runs after `applyPodParentMode` (design D1) — verify `npm run typecheck` and `npm run test:ci` pass
- [x] 3.2 Add a KsgPanel-level (or `wrapNodeGroup`-level, given fixture elements) test asserting the group is present in BOTH pod-parent modes and that `node` mode keeps pods under their node (`cluster > node group > node > pod`) — verify the new test passes

## 4. Rendering

- [x] 4.1 In `src/features/graph-canvas/styles/getStylesheet.ts`, add a `node[?isNodeGroup]` header rule reusing the title-cased render-only label mapper (as `node[kind='network']` does) at the group-header font size/weight, declared after `node:parent` so it wins — verify the `getStylesheet` test asserts the selector is present and ordered after `node:parent`
- [x] 4.2 Add the collapsed folder-glyph selector for `node[?isNodeGroup].cy-expand-collapse-collapsed-node`, resolving its tint through `resolveParentClusterColor` rather than a `data(*Color)` field (design D4) — verify a `getStylesheet` test covers it
- [x] 4.3 Confirm no `selectable: false` is set anywhere for the group (design D3: the `+` cue requires selectability) — verify by asserting the synthesized element carries no `selectable` key in `wrapNodeGroup.test.ts`

## 5. Interaction and filter behaviour

- [x] 5.1 Verify the group defaults to EXPANDED: assert the controller-mode default-collapse effect in `KsgPanel` seeds only `isController` ids, so no `isNodeGroup` id enters `collapsedIds` on load — cover with a test
- [x] 5.2 Verify the group is kind-less end-to-end: add assertions that `deriveLegendKindSets` contributes no row for it and `deriveContainers` (both modes) excludes it from the container swatch section — verify those tests pass
- [x] 5.3 Verify filter behaviour in `computeVisibility`: with `node` hidden, the group has no visible child and is removed by the orphan cascade; with `node` visible, the group is always visible and untogglable — add the two cases to `computeVisibility.test.ts`

## 6. Gates and visual check

- [x] 6.1 Run the full pre-push gate — `npm run lint && npm run typecheck && npm run fixture:check && npm run test:ci` — and verify all four pass (the fixture is untouched, so `fixture:check` must still be clean)
- [x] 6.2 Build and run the demo (`npm run build && docker compose up -d`), open `http://localhost:3000/d/ksg-switch-demo`, and verify by eye in BOTH layout modes: the `prod` group boxes `worker-0` + `worker-1`, the `dr` group boxes `worker-2`, both inherit their cluster's accent, selecting a group draws the `+` / `−` cue, collapsing folds the nodes into one folder-glyph box, and the layout does not degrade with the extra nesting level (design Risks)
