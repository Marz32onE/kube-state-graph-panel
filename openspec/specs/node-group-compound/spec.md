# node-group-compound Specification

## Purpose

A panel-owned compound container that boxes a cluster's K8s `node` elements into one group
(`cluster > node group > node`), so the machines of a cluster read as one tier and can be
folded away as a unit. It is a rendering convenience only — it carries no backend identity
and never appears in the wire contract.

## Requirements

### Requirement: Node-group synthesis

The system SHALL insert a synthesized compound container between each K8s `node` element and
the parent that node currently has, producing the rendered hierarchy `cluster > node group >
node`. Grouping is keyed on the node's **current parent**: all `node`-kind elements sharing a
parent land in the same group, and each distinct parent gets its own group. A `node` element
with no parent is grouped into a top-level (parent-less) group, so the rule needs no special
case for a graph that ships nodes outside any cluster. The synthesized container SHALL carry
a deterministic id derived from the parent it sits under, a `nodes` label, the marker
`isNodeGroup`, and the parent the grouped nodes had; it SHALL NOT carry a `kind`, a `status`,
a `worstStatus`, or any alert. The pass SHALL be pure: it never mutates its input, and every
element it returns is a fresh object.

#### Scenario: Nodes of one cluster are boxed together

- **WHEN** a cluster contains two or more `node`-kind elements
- **THEN** one synthesized group is inserted under that cluster, both nodes are re-parented
  under it, and the cluster's other children (services, PVCs, workload groups, the storage
  chain) are left where they were

#### Scenario: Every distinct parent gets its own group

- **WHEN** two clusters each contain `node` elements
- **THEN** each cluster gets its own group; no group ever spans two clusters

#### Scenario: A single node is still grouped

- **WHEN** a cluster contains exactly one `node` element
- **THEN** that node is still wrapped in a group — the structure does not depend on how many
  machines a cluster happens to have, so a cluster does not change shape when its second node
  appears

#### Scenario: Parent-less nodes get a top-level group

- **WHEN** a `node` element carries no parent
- **THEN** it is re-parented under a synthesized group that itself has no parent

#### Scenario: The input is never mutated

- **WHEN** the pass runs
- **THEN** no element object of the input array is mutated, and the caller's array is
  unchanged

### Requirement: Synthesis back-off

The system SHALL return the element set unchanged when there is nothing to group — no
`node`-kind element is present — and when synthesis would collide with an existing element:
if any element already carries an id the pass would generate, or already carries the
`isNodeGroup` marker, the pass SHALL back off entirely rather than produce a partial or
conflicting hierarchy.

#### Scenario: No node kinds present

- **WHEN** the graph holds no `node`-kind element
- **THEN** the element set comes back with no group added and no parent rewritten

#### Scenario: An id collision backs the whole pass off

- **WHEN** an element already occupies an id the pass would synthesize
- **THEN** no group is inserted at all and no `node` element is re-parented

### Requirement: Both pod-parent modes carry the group

The node group SHALL be present in **both** pod-parent modes. In `controller` mode the K8s
node is a leaf and the rendered chain is `cluster > node group > node`; in `node` mode the
node boxes its pods and the chain is `cluster > node group > node > pod`. Switching modes
SHALL NOT drop the group, and the grouping SHALL be applied to the elements a mode transform
already produced, so a mode's own re-parenting rules are never disturbed by it.

#### Scenario: Group survives a mode flip

- **WHEN** the user switches between `Controller` and `Node` layout
- **THEN** the node group box is present in both, and the K8s nodes stay inside it

#### Scenario: Node mode keeps pods under their node

- **WHEN** `node` mode has re-parented pods under their K8s node
- **THEN** the group wraps the node, not the pods: the chain reads `cluster > node group >
node > pod` and no pod becomes a direct child of the group

### Requirement: Node-group rendering

The node group SHALL render as a labelled compound backplate consistent with the other group
boxes: no resource icon, tinted with the accent of its enclosing cluster (so it reads as part
of that cluster's family), and headed by its title-cased `Nodes` label in the same enlarged,
semibold treatment the other group headers use. When collapsed, it SHALL show the folder
glyph the other kind-less decorative groups show, tinted the same way, rather than a blank
coloured box. The label transformation SHALL be render-only: the element's own `label` stays
the bare `nodes` string, so anything reading identity is unaffected.

#### Scenario: Expanded group is a tinted labelled box

- **WHEN** a node group is expanded
- **THEN** it renders as a round-rectangle backplate with no icon, tinted with its cluster's
  accent, headed `Nodes`

#### Scenario: Collapsed group shows a folder glyph

- **WHEN** a node group is collapsed
- **THEN** it renders the folder glyph in its cluster's accent — never a blank box — and its
  border thickens like every other collapsed compound

### Requirement: Node-group collapse

The node group SHALL be selectable so the expand-collapse `+` / `−` cue is drawn on it when
selected, giving the user a one-click fold of every K8s node in that cluster. It SHALL default
to **expanded** on load, and its collapse state SHALL flow through the same collapsed-id state
every other compound uses — so it is preserved across a data refresh, reconciled when it is
absent from the graph, and expanded automatically when a search result is located inside it.

#### Scenario: The cue folds the whole group

- **WHEN** the user selects a node group and clicks its `−` cue
- **THEN** every K8s node in that cluster folds into the single group box, and clicking `+`
  restores them

#### Scenario: Expanded by default

- **WHEN** the panel loads, in either pod-parent mode
- **THEN** the node group is expanded — the machines are visible without any interaction

#### Scenario: Locating a hit inside a collapsed group expands it

- **WHEN** a search result resolves to an element nested inside a collapsed node group
- **THEN** the group is expanded so the hit can be selected, exactly as for any other
  collapsed ancestor

### Requirement: The node group is panel-owned, not a wire concept

The node group SHALL NOT appear in the wire contract, in the demo fixture, or in the
normalize boundary: it is synthesized from the already-normalized elements. Because it is
kind-less it SHALL NOT appear in the icon "Node kinds" legend, and no new legend swatch
section SHALL be added for it — the existing container swatch section keeps listing the K8s
`node` containers themselves. It SHALL NOT be togglable by the kind filter, and it SHALL
disappear through the existing orphan cascade when every node it holds has been filtered out.

#### Scenario: Absent from the wire contract

- **WHEN** the backend graph payload is parsed
- **THEN** no node group exists in the parsed elements; it is added only by the later view
  transform

#### Scenario: Contributes nothing to the legend

- **WHEN** the legend is derived
- **THEN** the node group adds no icon row and no swatch row; the container swatch section
  still lists the K8s `node` containers

#### Scenario: Emptied by the filter, the group disappears

- **WHEN** the user hides the `node` kind
- **THEN** the group has no visible child and no visible incident edge, so the orphan cascade
  removes the group box too — no empty box is left on the canvas
