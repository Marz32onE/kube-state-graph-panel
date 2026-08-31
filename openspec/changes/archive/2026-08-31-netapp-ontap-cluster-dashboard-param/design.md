## Context

See `proposal.md` — Why. `assembleDashboardParams` builds the `/dashboard` query params from the
selected node's `data`. Two constraints already in that file shape this change:

- **`labels` is denylisted wholesale.** No label reaches the wire as a param on its own; every
  label-sourced param exists because a named rule promotes it. `cluster` is that rule's only
  current instance (`resolveCluster`: ancestor walk first, `labels.cluster` fallback).
- **NetApp nodes are already dashboard-eligible.** `isDashboardEligible` excludes only the four
  decorative groups, and `storage-cluster` is one of them — so `netapp-node` and `netapp-aggr`
  reach param assembly today and simply come out without a `cluster`.

The graph-data contract fixes the shape this has to read (see `graph-data-integration`):
`netapp-node`'s labels are exactly `{ontap_cluster}` and `netapp-aggr`'s exactly
`{ontap_cluster, node}`; neither ever carries `cluster`, and neither ONTAP name enters the
top-level `clusters[]`. Their nesting is `storage-cluster > netapp-node > netapp-aggr`, and the
`storage-cluster` container's `data.storageCluster` holds the same ONTAP name as the leaves'
`labels.ontap_cluster`.

## Goals / Non-Goals

**Goals:**

- A NetApp node's `/dashboard` request carries its ONTAP cluster name, resolved by the same
  ancestor-then-labels shape the K8s branch uses.
- No non-NetApp node's param map changes value.

**Non-Goals:**

- Promoting any other label to a param (`netapp-aggr`'s `labels.node` stays unsent).
- Touching the wire contract, the fixture, `normalizeGraph`, or the pinned tooltip — the ONTAP
  cluster name is already in the element data; only the param projection changes.
- Teaching the backend anything. This is a panel-side widening of what one existing param
  can be sourced from.

## Decisions

**D1 — One param name `cluster`, not a new `ontap_cluster`.** The value goes out under the
existing `cluster` key rather than a NetApp-specific one. Rationale: the backend already
receives `kind` on every request, so `kind=netapp-aggr` plus `cluster=ontap-prod` is
self-disambiguating, and a dashboard resolver keyed on one scoping field stays simpler than one
that must check two mutually exclusive fields. *Alternative considered:* emit `ontap_cluster`
as its own param, keeping the Harvest label name and the two cluster namespaces separate. It
reads more precisely, but it forces every consumer of `/dashboard` to know both keys, and this
panel's own K8s-vs-ONTAP distinction is already carried by `kind`. *Also considered and
rejected:* sending both keys with the same value — redundant, and it makes the "which one is
authoritative" question permanent.

**D2 — One ancestor walk that accepts either cluster flag, first match wins.** `resolveCluster`
keeps its single upward pass and returns on the first ancestor that is `isCluster` (taking
`data.cluster`) **or** `isStorageCluster` (taking `data.storageCluster`). *Alternative
considered:* two separate walks, K8s first then ONTAP. It would cost a second full pass to
answer a question the hierarchy makes unambiguous — a K8s cluster and an ONTAP cluster are both
top-level containers and never nest inside one another, so no node has both kinds of ancestor
and "first match" cannot be ambiguous in practice. Making first-match normative (it is in the
spec) also means the rule stays deterministic if that ever stops holding.

**D3 — Label fallback ordered `labels.cluster` then `labels.ontap_cluster`.** Strict ordering
rather than "whichever is present", so a node that somehow carried both resolves the same way
every time, and the K8s meaning keeps priority. Per the contract no node carries both; the
order is there to make the degenerate case defined rather than incidental.

**D4 — Own `data.cluster` still wins over everything (unchanged).** The `!('cluster' in params)`
guard around the resolution stays exactly where it is. NetApp nodes have no `data.cluster`, so
this is a no-op for them, but keeping the guard means the widening cannot disturb the compound
merge or any node that carries its own value.

## Risks / Trade-offs

- **A backend that scopes on `cluster` by matching it against the K8s `clusters[]` list would
  now receive a name that is not in it.** → The request also carries `kind`, and the two NetApp
  kinds are exactly the case where the name is an ONTAP one; a resolver that ignores `kind` was
  already returning nothing useful for these nodes (they sent no `cluster` at all before), so
  the change cannot regress a working lookup — at worst it leaves an already-empty result empty.
- **Two namespaces share one param name (D1's cost).** → Recorded here and in the spec bullet;
  `kind` is the discriminator, and the spec states the panel MUST NOT add an `ontap_cluster`
  param, so the two cannot drift apart later by accident.
- **`storage-cluster` is a decorative group and could be dropped upstream.** → The
  `labels.ontap_cluster` fallback covers exactly that case, and it is specified as a scenario
  rather than left to the ancestor walk.
