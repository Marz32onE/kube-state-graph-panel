## 1. Param resolution (TDD)

- [x] 1.1 Add the failing cases to `src/features/node-detail/assembleDashboardParams.test.ts`, inside the existing `cluster resolution` describe, one per new spec scenario: a `netapp-node` whose parent is an `isStorageCluster` container resolves `cluster` from `data.storageCluster`; a `netapp-aggr` resolves the same value by walking up through its `netapp-node` parent; a NetApp node with no cluster-container ancestor falls back to `labels.ontap_cluster`; and a node carrying both `labels.cluster` and `labels.ontap_cluster` resolves to `labels.cluster` — verify `npx jest src/features/node-detail/assembleDashboardParams.test.ts` fails on exactly these four (RED)
- [x] 1.2 Widen `resolveCluster` in `src/features/node-detail/assembleDashboardParams.ts`: in the single existing upward pass, return on the first ancestor that is `isCluster` (taking a non-empty string `data.cluster`) **or** `isStorageCluster` (taking a non-empty string `data.storageCluster`); after the walk, fall back to `labels.cluster` then `labels.ontap_cluster`, still returning `undefined` when neither exists — verify the suite from 1.1 passes (GREEN)
- [x] 1.3 Update the function's doc comment to name the ONTAP source and why one param name serves both cluster namespaces (design D1/D2/D3), keeping the `!('cluster' in params)` own-wins guard at the call site untouched (design D4) — verify `npm run lint` and `npm run typecheck` pass

## 2. Contract-accuracy of the existing NetApp test

- [x] 2.1 Fix the `treats a netapp-aggr as eligible but never sends its storage facts as params` case, whose element carries a `labels.cluster` no `netapp-aggr` can have per the graph-data contract: give it `labels: { ontap_cluster: 'ontap-prod', node: 'ontap-prod-01' }` and expect `{ kind: 'netapp-aggr', name: 'aggr1', cluster: 'ontap-prod' }`, keeping its existing assertions that `health` / `usage` / `usageRatio` are absent — verify the case passes and still asserts `labels.node` is not promoted to a param

## 3. Regression + spec sync

- [x] 3.1 Verify no non-NetApp node's params changed: run `npm run test:ci` and confirm the pre-existing `cluster resolution`, compound-merge, `controller resolution`, and `from_time / to_time` cases all pass unmodified
- [x] 3.2 Run `openspec validate netapp-ontap-cluster-dashboard-param --strict` and confirm the delta's `Dashboard 請求參數組裝` requirement matches the implemented behavior (ancestor first-match, `labels.cluster` before `labels.ontap_cluster`, omission when neither exists)
