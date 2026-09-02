## 1. Failing tests

- [x] 1.1 Add a `describe('hidden-ancestor endpoints')` in `src/features/element-filter/computeVisibility.test.ts` covering the spec scenarios: hide `deployment` with nested `pod` → `pod-mounts-pvc` → `pvc` (edge and pod drop; PVC stays iff it still has `pvc-to-netapp-aggr`); hide `deployment` when that PVC has no other edge (PVC orphans); hide a generic compound with a leaving edge to an outside node (that edge drops); both-sides-visible control (edge stays). Verify the new tests fail on current `computeVisibility` (`npx jest src/features/element-filter/computeVisibility.test.ts`).

## 2. Descendant strip

- [x] 2.1 After the kind + ingress pass and before the edge pass in `computeVisibility`, seed `collectDescendantIds` with every node id in `elements` that is not in `visibleNodeIds`, then delete those descendants from `visibleNodeIds`. Reuse the existing `childrenByParent` index. Verify the new tests in 1.1 pass and the existing `computeVisibility.test.ts` suite stays green.

## 3. Check

- [x] 3.1 Run `npm run typecheck && npm run lint && npm run test:ci` and verify all three exit 0.
