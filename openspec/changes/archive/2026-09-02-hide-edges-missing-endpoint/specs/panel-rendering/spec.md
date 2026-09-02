## ADDED Requirements

### Requirement: A drawn edge has both endpoints on the canvas

After kind, edge-type, and ingress filtering, every remaining drawn edge MUST have **both**
endpoints on the canvas. A node is on the canvas only when it is itself visible **and** every
ancestor compound is visible (effective visibility is the AND of a node and its ancestors). An
edge whose source or target fails that test MUST hide — even when that edge's type is still
enabled, and even when the far endpoint is still a visible kind sitting outside the hidden
compound.

The rule is kind-agnostic and edge-type-agnostic. Hiding a `deployment` in controller mode is
the reproducing case (`pod-mounts-pvc` left attached to a still-visible PVC while the pod sits
inside the hidden deployment); hiding a `statefulset`, a K8s `node`, or any other compound
with a descendant that has an edge leaving the compound MUST behave the same way.

When hiding that edge leaves a node with no remaining visible incident edge and no visible
child, the existing orphan cascade MUST reclaim it. Filter hiding stays `visibility: hidden`
and MUST NOT trigger a layout rerun.

#### Scenario: Hiding a deployment drops the pod-to-PVC edge

- **WHEN** the graph is in controller mode with a `deployment` containing a `pod`, that pod
  has a `pod-mounts-pvc` edge to a `pvc`, and the user hides the `deployment` kind
- **THEN** the deployment is hidden, the pod is off-canvas (its ancestor is hidden), and the
  `pod-mounts-pvc` edge is hidden even though the PVC kind is still enabled

#### Scenario: The far-side PVC stays when it still has another visible edge

- **WHEN** that same PVC also has a visible `pvc-to-netapp-aggr` edge to a still-visible
  aggregate
- **THEN** the PVC and the storage edge remain; only the one-sided `pod-mounts-pvc` edge hides

#### Scenario: The far-side PVC hides when the dropped edge was its last

- **WHEN** that PVC has no other visible incident edge and no visible child after the
  `pod-mounts-pvc` edge hides
- **THEN** the PVC is hidden by the orphan cascade

#### Scenario: Any compound kind with a leaving edge hides that edge

- **WHEN** any compound kind is filtered out and a descendant of that compound has an edge to
  a node outside it
- **THEN** that edge hides; it MUST NOT remain drawn against the still-visible far endpoint

#### Scenario: Edges with both sides on the canvas are unchanged

- **WHEN** both endpoints of an edge remain visible and every ancestor of both endpoints
  remains visible
- **THEN** that edge stays visible
