## Context

See proposal.md for motivation. Visibility is computed once in `computeVisibility` (kind pass
→ edge pass → `hideOrphans`) and applied by `useElementFilter` as `style('visibility')`. The
edge pass already requires `visibleNodeIds.has(source) && visibleNodeIds.has(target)`, which
is why the existing "hide the edge when either endpoint's kind is filtered" tests pass.

That check does not match the canvas. Cytoscape's effective visibility is the AND of a node
and all its ancestors, so a pod nested under a kind-filtered `deployment` is off-canvas while
still sitting in `visibleNodeIds`. The `pod-mounts-pvc` edge therefore survives the edge pass
and is drawn against the still-visible PVC.

The ingress path already folds descendants of its hide-seed into the hidden set for this
reason (`collectDescendantIds` via `buildChildrenByParent`). Kind filtering never did.

## Goals / Non-Goals

**Goals:**

- Make `visibleNodeIds` agree with canvas effective visibility before the edge pass, so the
  existing both-endpoints check hides one-sided edges without a second edge rule.
- Leave `hideOrphans` to reclaim a far endpoint that lost its last visible incident edge.
- Reuse the existing descendant index; do not grow a parallel walk.

**Non-Goals:**

- Re-home a leaving edge onto a remaining ancestor (do not rewrite `pod-mounts-pvc` onto the
  cluster when the deployment hides).
- New panel options, legend controls, layout reruns, or changes to `useElementFilter`.
- Rewriting the existing kind-filter / endpoint-kind / orphan-cascade requirements.

## Decisions

1. **Strip descendants of hidden nodes from `visibleNodeIds` after the kind + ingress pass,
   before the edge pass.** Seed = every node id present in `elements` that is not in
   `visibleNodeIds`. Run the existing `collectDescendantIds(childrenByParent, seed)` and
   delete those ids from the visible set. Then the current edge pass (`both endpoints in
   visibleNodeIds`) and `hideOrphans` do the rest.

   Alternatives considered:

   - *Ancestor walk only inside the edge pass.* Hides the dangling edge, but leaves the
     nested pod in `visibleNodeIds`. Search, the detail panel, and fade already treat that
     set as "on the canvas"; disagreeing is the same class of bug the ingress fold exists to
     prevent.
   - *Walk ancestors per edge endpoint without stripping nodes.* Same disagreement, plus a
     second visibility predicate next to the one the ingress path already uses.

2. **Kind-less compounds (`cluster`, node-group, `network`) never seed the strip.** They are
   not kind-filtered, so they remain in `visibleNodeIds` unless a later `hideOrphans` pass
   reclaims them. Only kind-filtered and ingress-hidden nodes seed the descendant deletion.
   That is automatic if the seed is "ids not in `visibleNodeIds`" after those two passes.

3. **No live-graph special case for collapse.** `computeVisibility` reads the original
   `elements` (pod → PVC), not the expand-collapse `edge.move()` that re-points a boundary
   edge onto a collapsed container. Stripping the nested pod from `visibleNodeIds` drops the
   original edge id from `visibleEdgeIds`; `useElementFilter` then hides that same id on the
   live graph, whether or not collapse has moved it.

## Risks / Trade-offs

- [Hiding a K8s `node` in node mode also drops nested pods from `visibleNodeIds`] → intended:
  those pods are already off-canvas via ancestor AND; today only their leaving edges remain.
  Controller mode is unaffected (pods nest under the controller, not the node).
- [A PVC that still joins an aggregate stays after its `pod-mounts-pvc` edge drops] →
  intended: the storage chain is still a two-sided graph. The orphan cascade only reclaims
  the claim when that dropped edge was its last.

## Migration Plan

None. Filter output changes only for graphs that currently draw a one-sided edge. Default
`visibleKinds` (all kinds on) is unchanged. Roll back by reverting the descendant-strip in
`computeVisibility`.

## Open Questions

None.
