# Timeline Data Architecture

> **Status**: Designed
> **Author**: Human + Claude Code
> **Last Updated**: 2026-03-28
> **Implements Pillar**: Pillar 1 (Every Action Echoes), Pillar 2 (Information is Power, Corrupted), Pillar 6 (Truth is a Fractured Mirror)

## Overview

The Timeline Data Architecture is the foundational data layer of TIME-PUNK — a directed graph representing the criminal conspiracy across all fractured timelines. It is implemented as two parallel graphs: a `TruthGraph` (the immutable ground reality, authored by designers and loaded at mission start, never modified during play) and a `BeliefGraph` (the player's working model of reality, initialized as a fogged copy of TruthGraph at mission start and mutated by every gameplay action). All player-facing systems — the Fractured Reality Map, the post-mission analysis interface, the investigation deduction system, and the social manipulation system — read exclusively from `BeliefGraph`. The `TruthGraph` is only accessed by the Corruption Data System (to calculate divergence between truth and belief) and by the Post-Mission Analysis system (to diff the two graphs and surface anomalies). A node represents any conspiracy element: a person, an event, a document, a location, or a decision point. An edge represents a directional relationship between nodes: causal, temporal, social, or evidential. Every gameplay action in TIME-PUNK — a stealth decision, an evidence discovery, a social manipulation — ultimately resolves as a write to `BeliefGraph`. The architecture has no procedural elements; all nodes, edges, and truth states are handcrafted data loaded per mission.

## Player Fantasy

The player never directly sees the data architecture — they see the Fractured Reality Map. But the architecture is what makes the map feel the way it does. The fantasy is: *I am building a map of a truth the world is trying to hide from me.* Every time the player connects a piece of evidence, turns an NPC, or moves through a space undetected, they are adding to their mental model of the conspiracy. When that model diverges from reality — because they failed, or because someone lied to them — the map shows the fracture. The architecture must make this feel like *discovery corrupted*, not *data updated*. The player should feel the weight of every write: each new node confirmed is a small victory; each node that ghosts is a small loss of certainty. The data architecture serves Pillar 2 directly — *Information is Power, Corrupted* — by making the player's knowledge state a tangible, fragile resource that can be built, damaged, and partially repaired, but never perfectly restored once broken.

## Detailed Design

### Core Rules

**1. Data Model**

The system maintains two parallel directed graphs: `TruthGraph` and `BeliefGraph`. Both are loaded per-mission but `BeliefGraph` persists across missions; `TruthGraph` is discarded and reloaded fresh each mission.

**TimelineNode — TruthGraph record:**
```
id:                 StringName       # unique, stable identifier (e.g., &"node_vasquez_01")
node_type:          NodeType         # PERSON | EVENT | DOCUMENT | LOCATION | DECISION_POINT
timeline_id:        StringName       # mission/timeline of origin (e.g., &"timeline_03")
temporal_position:  float            # ordering on the timeline axis (0.0 = earliest known point)
label:              String           # canonical name ("Director Vasquez", "Port Araya Meeting")
truth_state:        TruthState       # ACTIVE | ELIMINATED | DORMANT
severity_override:  float            # -1.0 = use NodeType default; 0.0–1.0 = authored override
                                     # Use for narratively critical nodes that need elevated anomaly priority
```

**BeliefNode — BeliefGraph record:**
```
id:                 StringName       # matches TruthGraph id if real; unique if false positive
node_type:          NodeType
timeline_id:        StringName
temporal_position:  float            # may deviate from truth if corruption has altered position data
display_label:      String           # what the player sees; may be altered by NPC deception
confidence:         float            # 0.0–1.0 (underlying driver; not shown raw to player)
belief_state:       BeliefState      # UNKNOWN | SUSPECTED | PROBABLE | CONFIRMED | GHOSTED
is_false_positive:  bool             # hidden; true = planted by NPC, no TruthGraph counterpart
corruption_tags:    Array[CorruptionTag]  # ordered history of all corruption events on this node
```

**TimelineEdge — TruthGraph record:**
```
id:                 StringName
from_node:          StringName       # source node id
to_node:            StringName       # target node id
edge_type:          EdgeType         # CAUSAL | TEMPORAL | SOCIAL | EVIDENTIAL
truth_strength:     float            # 0.0–1.0, authored relationship strength
```

**BeliefEdge — BeliefGraph record:**
```
id:                 StringName
from_node:          StringName
to_node:            StringName
edge_type:          EdgeType
is_visible:         bool             # false until player discovers this connection
believed_strength:  float            # 0.0–1.0, player's perceived relationship strength
corruption_tags:    Array[CorruptionTag]
```

**CorruptionTag — appended to nodes and edges on corruption events:**
```
source:             CorruptionSource  # SELF_INFLICTED | EXTERNAL_DECEPTION
corruption_type:    CorruptionType    # CONFIDENCE_DEGRADATION | NODE_GHOST | FALSE_NODE
                                      # EDGE_OBSCURED | LABEL_ALTERED
source_actor:       StringName        # NPC id or failure-event id that caused this tag
mission_id:         StringName        # mission in which the corruption occurred
timestamp:          float             # session time of application (for audit/repair ordering)
```

**2. Confidence–BeliefState Mapping**

Confidence is a float (0.0–1.0). `BeliefState` is the discrete player-facing display state derived from confidence. The mapping is one-directional: confidence drives state, not the reverse.

```
0.0              → UNKNOWN
0.01 – 0.35      → SUSPECTED
0.36 – 0.69      → PROBABLE
0.70 – 1.00      → CONFIRMED
(any state)      → GHOSTED  if a single corruption event applies a penalty ≥ 0.40
```

GHOSTED is a trauma state, not a confidence band. A ghosted node retains its confidence value but renders as fragmented. Confidence can continue degrading while ghosted. A ghosted node exits GHOSTED and re-enters the confidence-band system only when a timeline repair action explicitly restores it.

**3. Access Control**

`query_truth(node_id)` is a restricted operation. Only two systems may call it:
- `CorruptionDataSystem` — to compute divergence between truth and belief
- `PostMissionAnalysisUI` — to diff the graphs and surface anomalies

All other systems call `query_belief(node_id)` only. This is enforced by convention in GDScript; truth access should be documented with a `## RESTRICTED` comment at every call site.

**4. Core Operations**

```
load_mission(mission_id: StringName) -> void
    Load TruthGraph nodes/edges for this mission from data file.
    For each new TruthNode: add a fogged BeliefNode (confidence = 0.0,
    belief_state = UNKNOWN, is_false_positive = false).
    Existing BeliefNodes from prior missions are untouched.
    Emit: mission_loaded(mission_id)

write_belief(node_id: StringName, delta: float) -> void
    Apply delta to BeliefNode.confidence (clamped 0.0–1.0).
    Recompute belief_state from confidence-band table.
    Emit: node_updated(node_id, old_state, new_state)

apply_corruption(node_id: StringName, tag: CorruptionTag,
                 confidence_penalty: float) -> void
    Append tag to BeliefNode.corruption_tags.
    Apply -confidence_penalty to BeliefNode.confidence (floor: 0.0).
    If confidence_penalty >= 0.40 AND prior belief_state == CONFIRMED:
        Set belief_state = GHOSTED regardless of resulting confidence value.
    Emit: node_corrupted(node_id, tag)

create_false_node(belief_node: BeliefNode) -> void
    Add a node to BeliefGraph with is_false_positive = true.
    Node has no TruthGraph counterpart.
    Called exclusively by NPC Counter-Deception System via NPC Psychology layer.
    Emit: node_updated(node_id, UNKNOWN, belief_node.belief_state)

reveal_edge(edge_id: StringName) -> void
    Set BeliefEdge.is_visible = true.
    Emit: edge_revealed(edge_id)

diff_graphs(mission_id: StringName) -> Array[GraphDivergence]
    For each node in TruthGraph scoped to mission_id:
        Compare TruthNode.truth_state with BeliefNode.belief_state and confidence.
        Build a GraphDivergence record for each meaningful mismatch.
    Include false-positive nodes in BeliefGraph with no TruthGraph counterpart.
    Return the divergence array. Does not modify either graph.
```

---

### States and Transitions

**BeliefNode states:**

| State | Confidence | Display | Entry | Exit |
|-------|-----------|---------|-------|------|
| `UNKNOWN` | 0.0 | Hidden / fully fogged | Initial state at mission load | `write_belief()` with any positive delta |
| `SUSPECTED` | 0.01–0.35 | Dim, label blurred | Confidence crosses 0.01 | Confidence crosses 0.36, or GHOSTED event |
| `PROBABLE` | 0.36–0.69 | Visible, label partially legible | Confidence crosses 0.36 | Confidence crosses 0.70, drops below 0.36, or GHOSTED event |
| `CONFIRMED` | 0.70–1.0 | Fully visible, clear label | Confidence crosses 0.70 | Corruption penalty ≥ 0.40 → GHOSTED; gradual decay drops below 0.70 |
| `GHOSTED` | Any (post-trauma) | Fragmented, position uncertain | Single corruption penalty ≥ 0.40 on a CONFIRMED node | Timeline repair action explicitly clears GHOSTED flag; confidence re-evaluated against bands |

**BeliefEdge states:**

| State | Display | Entry | Exit |
|-------|---------|-------|------|
| `HIDDEN` | Not rendered | Initial | `reveal_edge()` called |
| `VISIBLE` | Rendered at `believed_strength` opacity | `reveal_edge()` | Corruption tag `EDGE_OBSCURED` applied |
| `OBSCURED` | Rendered as dashed/uncertain | `EDGE_OBSCURED` tag | Timeline repair clears the tag |

---

### Interactions with Other Systems

| System | Direction | Interface | Notes |
|--------|-----------|-----------|-------|
| **C1 Corruption Data System** | Reads both graphs; writes corruption | Calls `query_truth()`, `query_belief()`, `apply_corruption()` | Only system besides Post-Mission Analysis permitted to call `query_truth()` |
| **B4 Evidence/Clue Data System** | Writes to BeliefGraph | Calls `write_belief()` on node discovery; `reveal_edge()` on connection discovery | Delta magnitude determined by evidence quality (defined in B4 GDD) |
| **B5 Investigation/Deduction System** | Writes to BeliefGraph | Calls `write_belief()` with higher deltas on confirmed deductions; `reveal_edge()` | Higher-confidence writes than B4 — deduction outweighs raw clue discovery |
| **B6 NPC Psychology State Machine** | Writes to BeliefGraph | Calls `write_belief()` when NPC provides information (truthful or false). Routes `create_false_node()` calls from C5 | NPC psychology state determines delta magnitude and whether `is_false_positive` is set |
| **C5 NPC Counter-Deception System** | Writes false data via B6 | Does NOT call A1 directly — all writes route through B6 | Architectural rule: B6 owns the NPC-to-graph write path |
| **B1 Stealth System** | Writes corruption | Calls `apply_corruption()` with `SELF_INFLICTED` tag on detection events | Penalty magnitude scales with detection severity (defined in B1 GDD) |
| **B8 Social Manipulation System** | Writes belief and corruption | `write_belief()` on successful manipulation; `apply_corruption()` on failed manipulation (player's deception was seen through) | Bidirectional: success builds belief, failure corrupts it |
| **D1 Mission State System** | Triggers load/diff | Calls `load_mission()` at mission start; triggers `diff_graphs()` at mission end | Passes divergence array to C3 Anomaly Detection |
| **C2 Fractured Reality Map** | Reads BeliefGraph only | Subscribes to `node_updated` and `edge_revealed` signals; calls `query_belief()` | Never writes. The map is a read-only view of BeliefGraph. |
| **C3 Anomaly Detection System** | Reads divergence output | Receives `Array[GraphDivergence]` from `diff_graphs()` via D1 | Processes divergences into player-facing anomaly discoveries |
| **D3 Save/Load System** | Serializes BeliefGraph | Full serialization of all BeliefNodes, BeliefEdges, CorruptionTags to JSON | TruthGraph is NOT saved — always reloaded fresh from mission data files |

## Formulas

### Confidence Band Evaluation

Called after every `write_belief()` and `apply_corruption()`.

```
new_confidence = clamp(old_confidence + delta, 0.0, 1.0)

belief_state =
    GHOSTED   if ghosted_flag == true                        (trauma overrides bands)
    UNKNOWN   if new_confidence == 0.0
    SUSPECTED if 0.01 <= new_confidence <= 0.35
    PROBABLE  if 0.36 <= new_confidence <= 0.69
    CONFIRMED if 0.70 <= new_confidence <= 1.00
```

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `old_confidence` | float | 0.0–1.0 | BeliefNode | Current confidence before this operation |
| `delta` | float | -1.0–1.0 | Calling system | Positive = belief gain; negative = decay |
| `new_confidence` | float | 0.0–1.0 | Calculated | Result, clamped |

**Expected output**: BeliefState correctly reflects new_confidence. No state transition fires if the band has not changed.

---

### GHOSTED Trigger Evaluation

Called inside `apply_corruption()` after confidence is updated.

```
triggers_ghost =
    (confidence_penalty >= GHOST_THRESHOLD)
    AND (prior_belief_state == CONFIRMED)

if triggers_ghost:
    belief_state = GHOSTED
    ghosted_flag = true
    # confidence value is retained — it continues decaying normally while ghosted
```

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `confidence_penalty` | float | 0.0–1.0 | Calling system (B1, B8, C1) | Magnitude of this corruption event |
| `GHOST_THRESHOLD` | float | 0.30–0.60 | Tuning knob | Minimum single-event penalty to trigger GHOSTED |
| `prior_belief_state` | BeliefState | — | BeliefNode | State before this operation |

**Expected output**: GHOSTED triggers only on CONFIRMED nodes hit by a large single event. Gradual decay through PROBABLE → SUSPECTED does not trigger GHOSTED.

**Design note**: `GHOST_THRESHOLD` defaults to 0.40 and is a primary tuning knob — see Tuning Knobs section.

---

### Graph Divergence Score

Produced by `diff_graphs()`. Each `GraphDivergence` record contains a severity score used by C3 (Anomaly Detection) to rank anomalies by narrative importance.

```
divergence_severity =
    base_severity(node)
    * state_mismatch_multiplier(truth_state, belief_state)
    * (1.0 + false_positive_bonus)

base_severity(node) =
    node.severity_override   if node.severity_override >= 0.0
    TYPE_SEVERITY[node.node_type]   otherwise

false_positive_bonus = 0.5 if BeliefNode.is_false_positive else 0.0
```

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `node.severity_override` | float | -1.0 or 0.0–1.0 | TruthNode authored field | Per-node override; -1.0 means "use type default" |
| `TYPE_SEVERITY` | Dictionary | — | Data constant | PERSON=1.0, DECISION_POINT=0.9, EVENT=0.7, DOCUMENT=0.4, LOCATION=0.3 |
| `state_mismatch_multiplier` | float | 0.0–2.0 | Lookup table | Magnitude of truth/belief discrepancy (see table below) |
| `false_positive_bonus` | float | 0.0 or 0.5 | BeliefNode flag | False nodes are always high-priority anomalies |
| `divergence_severity` | float | 0.0–3.0 | Calculated | Final score; C3 sorts by this descending |

**State mismatch multiplier lookup:**

| Truth State | Belief State | Multiplier | Meaning |
|-------------|-------------|-----------|---------|
| ACTIVE | CONFIRMED | 0.0 | No divergence — correct belief |
| ACTIVE | PROBABLE | 0.3 | Minor divergence |
| ACTIVE | SUSPECTED | 0.7 | Significant divergence |
| ACTIVE | UNKNOWN | 1.0 | Node entirely missed |
| ACTIVE | GHOSTED | 1.5 | Node known but traumatically corrupted |
| ELIMINATED | CONFIRMED | 2.0 | Player believes active node is real; truth says eliminated |
| DORMANT | CONFIRMED | 1.2 | Moderate divergence |
| *(no TruthNode)* | any | — | false_positive_bonus handles these |

**Expected output range**: 0.0 (perfect knowledge) to 3.0 (PERSON with severity_override=1.0, fully wrong, false positive). C3 surfaces the top N anomalies by severity; N is a C3 tuning knob.

**Note for C3 GDD**: `severity_override` is authored on TruthNode. Narratively critical LOCATION nodes (and any other node type that demands elevated anomaly priority) should have `severity_override` set above the type default. C3 reads this field; it does not need to define its own override mechanism.

## Edge Cases

| Scenario | Expected Behavior | Rationale |
|----------|------------------|-----------|
| `write_belief()` called on a node not in BeliefGraph | Log error, no-op. Do not create node implicitly. | Nodes must be created via `load_mission()` or `create_false_node()`. Silent creation bypasses fog initialization. |
| `write_belief()` called with `delta = 0.0` | No-op. No signal emitted. | Prevents signal churn when calling systems compute a zero delta. |
| `apply_corruption()` on an UNKNOWN node | Apply tag and penalty normally. Node stays UNKNOWN (confidence already at 0.0 floor). Tag recorded for audit. | UNKNOWN nodes can be pre-corrupted by NPC deception before discovery — they surface already wrong. Intentional; serves Pillar 6. |
| `apply_corruption()` on a GHOSTED node | Apply tag and penalty. GHOSTED flag not re-triggered (already set). Confidence continues decaying. | A ghosted node can worsen but not be re-ghosted. Multiple corruption events accumulate naturally. |
| Confidence would drop below 0.0 | Clamp to 0.0. BeliefState → UNKNOWN. | No negative confidence. UNKNOWN is the floor. |
| Confidence would exceed 1.0 | Clamp to 1.0. BeliefState → CONFIRMED. | No over-confirmed nodes. Certainty has a ceiling. |
| `diff_graphs()` called mid-mission | Returns snapshot of current divergence. Does not modify either graph. | Allows C1 to query divergence during play for live corruption calculations without waiting for mission end. |
| TruthNode exists with no BeliefNode counterpart | Assert and log. Should never occur after `load_mission()`. BeliefGraph must mirror all TruthNodes at minimum as UNKNOWN. | A missing BeliefNode means fog initialization failed for this mission — a content pipeline error. |
| BeliefNode exists with no TruthNode counterpart | Valid — this is a false positive (`is_false_positive = true`). `diff_graphs()` includes it in divergence output without a truth comparison. | False positive nodes are a first-class game state, not an error. |
| `load_mission()` called for a mission already loaded | No-op for nodes already in BeliefGraph. Add only new nodes. | Prevents re-fogging already-discovered nodes on save/load. |
| `load_mission()` called with a missing data file | Fail loudly. Do not load an empty graph silently. | A mission with no truth data is a content pipeline error, not a recoverable runtime state. |
| Two `apply_corruption()` calls in the same frame exceed GHOST_THRESHOLD on the same CONFIRMED node | First call triggers GHOSTED. Second call appends its tag and applies penalty — flag already set, no re-trigger. Both tags recorded. | Simultaneous corruption events are valid. Both are auditable for later analysis. |
| Timeline repair clears GHOSTED flag; confidence is now in PROBABLE band (e.g., 0.40) | Node exits GHOSTED, re-enters PROBABLE. Confidence band evaluation runs immediately after flag is cleared. | Repair restores the trustworthiness of the node's state, not the confidence level. Player rebuilds confidence through continued play. |
| `query_truth()` called by an unauthorized system | Log a warning in debug builds. Allow the call in release builds. Do not hard-error. | GDScript has no access modifiers. Enforcement is by convention and `## RESTRICTED` documentation. Warnings surface violations during development. |

## Dependencies

**Upstream Dependencies (systems A1 requires)**

None. Timeline Data Architecture is a Foundation layer system with no external system dependencies. It loads from authored mission data files only.

**Downstream Dependents (systems that depend on A1)**

| System | Dependency Type | What They Require From A1 |
|--------|----------------|--------------------------|
| **C1 Corruption Data System** | Hard | `query_truth()` (restricted) and `query_belief()` for divergence computation; `apply_corruption()` write path |
| **B4 Evidence/Clue Data System** | Hard | `write_belief()` and `reveal_edge()` on evidence discovery |
| **B5 Investigation/Deduction System** | Hard | `write_belief()` and `reveal_edge()` on confirmed deductions |
| **B6 NPC Psychology State Machine** | Hard | `write_belief()` on NPC information delivery; routes `create_false_node()` calls from C5 |
| **B8 Social Manipulation System** | Hard | `write_belief()` on successful manipulation; `apply_corruption()` on failed manipulation |
| **B1 Stealth System** | Hard | `apply_corruption()` with `SELF_INFLICTED` tag on detection events |
| **D1 Mission State System** | Hard | `load_mission()` at mission start; `diff_graphs()` at mission end |
| **C2 Fractured Reality Map** | Hard | `query_belief()` for all display data; subscribes to `node_updated` and `edge_revealed` signals |
| **C3 Anomaly Detection System** | Hard (via D1) | Receives `Array[GraphDivergence]` from `diff_graphs()` via D1; `divergence_severity` scores use `severity_override` from TruthNode |
| **C5 NPC Counter-Deception System** | Soft (indirect via B6) | Does not call A1 directly — all writes route through B6's write path |
| **D3 Save/Load System** | Hard | Full serialization of BeliefGraph: all `BeliefNode`, `BeliefEdge`, and `CorruptionTag` records |

**Bidirectionality note**: Each downstream system listed above must reference A1 as an upstream hard dependency in its own Dependencies section. C5 references B6 as its dependency; B6 references A1.

## Tuning Knobs

All tuning knobs must be `@export` constants or data-driven config values. No hardcoded gameplay numbers in GDScript.

| Knob | Default | Safe Range | Too High | Too Low | Affects |
|------|---------|-----------|----------|---------|---------|
| `GHOST_THRESHOLD` | 0.40 | 0.30–0.60 | ≥0.60: GHOSTED almost never fires; trauma system loses weight; map feels safe | ≤0.30: Any mid-sized hit on a CONFIRMED node ghosts it; map fragments constantly; becomes unreadable | Primary trauma lever — how punishing a large single-event corruption feels |
| `PROBABLE_CONFIRMED_BOUNDARY` | 0.70 | 0.60–0.80 | ≥0.80: CONFIRMED is hard to reach; player rarely sees clear nodes; increases difficulty significantly | ≤0.60: Nodes confirm quickly; map clarity comes easy; reduces tension in belief-building | Primary difficulty lever for the belief-building arc |
| `SUSPECTED_PROBABLE_BOUNDARY` | 0.36 | 0.25–0.45 | ≥0.45: SUSPECTED band is wide; nodes stay faint and blurred longer; slower early feedback | ≤0.25: Nodes jump to PROBABLE quickly; discovery feels fast but less weighty | Width of the SUSPECTED band; pacing of early node revelation |
| `TYPE_SEVERITY[PERSON]` | 1.0 | 0.7–1.0 | — | ≤0.7: Person nodes ranked below DECISION_POINT anomalies; counter-intuitive | C3 anomaly ranking weight for person nodes |
| `TYPE_SEVERITY[DECISION_POINT]` | 0.9 | 0.6–1.0 | ≥1.0: Equals PERSON weight — fine if intended | ≤0.6: Decision point anomalies under-surfaced | C3 anomaly ranking weight for decision point nodes |
| `TYPE_SEVERITY[EVENT]` | 0.7 | 0.4–0.9 | ≥0.9: Events compete with persons for top anomaly slots | ≤0.4: Event anomalies rarely surface | C3 anomaly ranking weight for event nodes |
| `TYPE_SEVERITY[DOCUMENT]` | 0.4 | 0.2–0.6 | ≥0.6: Documents compete with events for anomaly slots | ≤0.2: Document anomalies buried | C3 anomaly ranking weight for document nodes |
| `TYPE_SEVERITY[LOCATION]` | 0.3 | 0.1–0.5 | ≥0.5: Locations compete with documents | ≤0.1: Location anomalies effectively invisible | C3 anomaly ranking weight for location nodes |
| `FALSE_POSITIVE_BONUS` | 0.5 | 0.2–0.8 | ≥0.8: False nodes dominate post-mission output; planted deceptions always surface first | ≤0.2: False nodes may not appear in post-mission analysis; NPC deception invisible to player | Urgency of false positive surfacing in C3 / post-mission analysis |

**Interaction warning**: `GHOST_THRESHOLD` and `PROBABLE_CONFIRMED_BOUNDARY` interact. If CONFIRMED requires 0.70 and GHOST_THRESHOLD is 0.40, any node above 0.70 confidence can be ghosted in a single hit. Raising the CONFIRMED threshold to 0.80 reduces the CONFIRMED population and therefore GHOSTED frequency. Always tune these two knobs together during playtesting.

## Visual/Audio Requirements

The Timeline Data Architecture is a pure data layer. It emits signals; it does not render anything or play any audio. All visual and audio responses are owned by downstream systems.

| Signal | Downstream Consumer | Required Response |
|--------|------------------|-------------------|
| `node_updated(node_id, old_state, new_state)` | C2 Fractured Reality Map | Update node visual state on the map |
| `node_corrupted(node_id, tag)` | C2 (visual) + E6 Adaptive Audio (audio) | Trigger corruption visual effect + audio sting |
| `edge_revealed(edge_id)` | C2 Fractured Reality Map | Animate edge appearing on the map |
| `mission_loaded(mission_id)` | D1 Mission State System | Gate-signal for mission start flow |

A1 requires no art or audio assets. All visual and audio specifications belong to the C2 and E6 GDDs.

## UI Requirements

The Timeline Data Architecture has no direct UI. All UI is owned by C2 (Fractured Reality Map), E5 (Post-Mission Analysis UI), and E4 (Dialogue UI).

**Contract requirement**: All signals emitted by A1 must use typed parameters — no untyped `Dictionary` payloads. C2 subscribes to these signals at runtime and must receive strongly typed data to update the map without parsing.

**Data access contract**: C2 calls `query_belief(node_id)` and receives a `BeliefNode` resource. The `belief_state` enum value is the canonical display-state input for the map. C2 must never hold a cached copy of BeliefNode — always query fresh on signal receipt.

## Acceptance Criteria

All criteria are verifiable via the public API and signal monitoring. No direct state inspection permitted — tests must use `query_belief()` and `query_truth()` (where authorized).

| # | Criterion | Test Method | Pass Condition |
|---|-----------|------------|----------------|
| AC-01 | `load_mission()` initializes all TruthNodes as UNKNOWN BeliefNodes | Load a test mission with 5 known TruthNodes. Call `query_belief()` on each. | All 5 return `confidence = 0.0`, `belief_state = UNKNOWN`, `is_false_positive = false`, `corruption_tags = []` |
| AC-02 | `write_belief()` transitions state correctly through all three bands | From UNKNOWN: call `write_belief(+0.20)` → `write_belief(+0.20)` → `write_belief(+0.30)`. | States in order: SUSPECTED → PROBABLE → CONFIRMED. Three `node_updated` signals emitted with correct old/new states. |
| AC-03 | GHOSTED triggers on CONFIRMED node with penalty ≥ GHOST_THRESHOLD | Raise node to CONFIRMED (confidence ≥ 0.70). Call `apply_corruption(node, tag, 0.40)`. | `belief_state = GHOSTED`. `node_corrupted` signal emitted. Confidence decremented but node stays GHOSTED. |
| AC-04 | GHOSTED does NOT trigger on PROBABLE node with same penalty | Raise node to PROBABLE (confidence 0.55). Call `apply_corruption(node, tag, 0.40)`. | `belief_state` follows confidence-band result (SUSPECTED or UNKNOWN). NOT GHOSTED. |
| AC-05 | Gradual decay through CONFIRMED does not trigger GHOSTED | Raise node to CONFIRMED (confidence 0.80). Apply `apply_corruption(node, tag, 0.10)` four times. | Node passes through PROBABLE → SUSPECTED normally. `belief_state` is never GHOSTED. |
| AC-06 | `apply_corruption()` on a GHOSTED node appends tags without re-triggering | GHOST a node (AC-03). Call `apply_corruption(node, tag2, 0.50)`. | `corruption_tags` has 2 entries. `belief_state` remains GHOSTED. No second GHOSTED event fired. |
| AC-07 | Timeline repair exits GHOSTED and re-evaluates confidence band | GHOST a node (post-penalty confidence = 0.45). Trigger timeline repair (clear ghosted_flag externally as C4 would). | `belief_state` becomes PROBABLE. `node_updated` signal emitted with new_state = PROBABLE. |
| AC-08 | `create_false_node()` inserts a node with `is_false_positive = true` | Call `create_false_node()` with a BeliefNode with no TruthGraph counterpart. | `query_belief()` returns the node. `is_false_positive = true`. `diff_graphs()` includes it with no truth-state comparison. |
| AC-09 | `diff_graphs()` is non-destructive | Call `diff_graphs()` during active play. Compare BeliefGraph state before and after call. | Zero BeliefNodes or edges modified. Zero signals emitted during the call. Return value is `Array[GraphDivergence]`. |
| AC-10 | `load_mission()` idempotency — repeated call does not re-fog existing nodes | Load mission A. Raise 2 nodes to CONFIRMED. Call `load_mission()` for the same mission again. | The 2 CONFIRMED nodes remain CONFIRMED with their confidence values unchanged. |
| AC-11 | BeliefGraph persists across mission boundaries | Load mission 1, raise 3 nodes to CONFIRMED. Load mission 2 (different mission_id). Call `query_belief()` on mission 1 nodes. | Mission 1 nodes still present with their confidence and belief_state values. Mission 2 nodes added as UNKNOWN alongside them. |
| AC-12 | Save/Load roundtrip preserves full BeliefGraph state | Confirm 3 nodes, GHOST 1, create 1 false positive, obscure 2 edges. Serialize. Deserialize. | All BeliefNodes present with identical `confidence`, `belief_state`, `is_false_positive`, and `corruption_tags` (count and field values). GHOSTED flag preserved. Edge OBSCURED states preserved. |
| AC-13 | `write_belief(node, 0.0)` is a no-op | Monitor `node_updated` signal. Call `write_belief(node, 0.0)`. | Signal not emitted. BeliefNode state unchanged. |
| AC-14 | `query_truth()` from unauthorized caller logs a debug warning | In debug build: call `query_truth()` from a system other than C1 or PostMissionAnalysis. | Warning logged to output. Call completes without error. No warning logged in release build. |
| AC-15 | Graph Divergence Score formula produces correct output | Case 1: PERSON node (severity_override = -1.0), truth = ACTIVE, belief = GHOSTED, is_false_positive = false. Expected: 1.0 × 1.5 × 1.0 = 1.5. Case 2: PERSON node, truth = ELIMINATED, belief = CONFIRMED, is_false_positive = false. Expected: 1.0 × 2.0 × 1.0 = 2.0. | `diff_graphs()` returns divergence records with `divergence_severity` matching both calculated values. |

## Open Questions

| # | Question | Owner | Target Resolution | Notes |
|---|----------|-------|------------------|-------|
| OQ-01 | Should `BeliefState` label names (SUSPECTED, PROBABLE, CONFIRMED) appear directly in the UI, or be abstracted into visual metaphor only? | UX Designer + Creative Director | Before C2 GDD is authored | Current spec makes `BeliefState` the display contract; C2 GDD decides whether label text is shown |
| OQ-02 | Can `write_belief()` raise a false positive node's confidence? | Systems Designer | Before C5 GDD is authored | Current spec allows it — no false-positive check in `write_belief()`. Risk: player reinforces NPC-planted lies through their own investigation. May be intentional (Pillar 6) or an undesired exploit. |
| OQ-03 | Does timeline repair (C4) remove all CorruptionTags from a node, or only clear the GHOSTED flag? | Systems Designer + C4 GDD | Before C4 GDD is authored | Current spec: repair clears the GHOSTED flag only; CorruptionTags are a permanent audit trail. If repair also strips tags, the post-mission analysis loses attribution history. |
| OQ-04 | Exact serialization schema for `CorruptionTag` arrays and `StringName` → `String` conversion | Lead Programmer + D3 GDD | Before D3 GDD is authored | Godot's `StringName` serializes to `String` in JSON; must confirm round-trip fidelity and float precision guarantees across platforms |
