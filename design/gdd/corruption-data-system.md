# Corruption Data System

> **Status**: Approved
> **Author**: Human + Claude Code
> **Last Updated**: 2026-03-28
> **Implements Pillar**: Pillar 2 (Information is Power, Corrupted), Pillar 4 (Earn the Knife), Pillar 6 (Truth is a Fractured Mirror)

## Overview

The Corruption Data System (C1) is the accounting layer that sits above the Timeline Data Architecture (A1) and answers the question every downstream system needs: *how corrupted is the map, and why?* A1 stores raw corruption tags on individual nodes; C1 interprets them — owning the penalty magnitude tables that define how severe a stealth failure, investigation error, or social misstep is, computing aggregate corruption metrics (per-node, per-mission, and overall), emitting threshold-crossing signals that the Fractured Reality Map and adaptive audio react to, and maintaining source attribution records that distinguish self-inflicted erosion from external NPC counter-deception. The player never directly perceives C1 — they see its output as map degradation, hear it as tonal audio shifts, and feel it as the rising cost of timeline repair. Source attribution remains opaque during play (the paranoid uncertainty is intentional), but is surfaced in the post-mission analysis interface as a breakdown of self-inflicted versus externally-planted corruption. Without C1, seven dependent systems would each independently compute corruption state from A1's raw tag data, producing inconsistency and fragmenting the game's central resource into disconnected local measurements. C1 is the single source of truth for corruption state across the entire game session.

## Player Fantasy

The Corruption Data System is never seen — only felt. Its fantasy is not power or clarity; it is *forensic paranoia*: the experience of being a detective who must investigate their own mind. When the player's map degrades, their first instinct is not despair but interrogation. *Did I cause this? Did someone plant it? Do I trust this node or has it been turned against me?* The corruption system is the mechanism that makes that question unanswerable with certainty — and teaches the player, over many missions, to navigate the uncertainty rather than resolve it.

The secondary fantasy is *earned rarity of clarity*. Because corruption is the default state and clean map nodes are hard-won, a single CONFIRMED node that has never been touched carries disproportionate psychological weight. The player learns to treat clean nodes as precious and suspect clean stretches as suspicious. Clarity that comes easily should feel wrong.

Mastery of C1 does not feel like mastery over the corruption. It feels like *calibrated suspicion*: the ability to read the shape of the uncertainty itself. An expert player looking at a GHOSTED node doesn't think "I need to fix that" — they think "that was a single large hit, probably external, the timestamp says mission two, I know which NPC was active then." They are not certain. They are *informed*. The fantasy ceiling is not a clean map; it is the ability to make confident decisions *in spite* of a corrupted one.

The corruption system serves Pillar 4 (Earn the Knife) through this arc: early missions feel like drowning in noise; late missions feel like reading a crime scene. The player earns the right to surgical precision not by removing uncertainty but by learning its grammar.

## Detailed Design

### Core Rules

**Rule 1 — C1 Data Structures**

C1 maintains three internal stores separate from A1's BeliefGraph and TruthGraph.

**EventRecord** — appended once per processed `report_event()` call; C1's own audit log:
```
timestamp:         float                # session time (matches CorruptionTag.timestamp)
event_type:        CorruptionEventType  # STEALTH_DETECTION | INVESTIGATION_ERROR
                                        # SOCIAL_FAILURE | NPC_DECEPTION
severity:          CorruptionSeverity   # MINOR | MODERATE | MAJOR
source:            CorruptionSource     # SELF_INFLICTED | EXTERNAL_DECEPTION
target_node_id:    StringName           # node this event was applied to
penalty_applied:   float                # resolved magnitude from penalty table
mission_id:        StringName           # mission context
```

**MissionCorruptionState** — one record per active mission; recomputed on read (not cached):
```
mission_id:            StringName
total_penalty_sum:     float        # sum of all penalty_applied for this mission
self_penalty_sum:      float        # penalties where source == SELF_INFLICTED
external_penalty_sum:  float        # penalties where source == EXTERNAL_DECEPTION
event_count:           int
event_timestamps:      Array[float] # all event timestamps; used for rate calculation
```

**ThresholdState** — one entry per value in `CORRUPTION_THRESHOLDS`:
```
threshold_value:   float   # e.g., 0.25, 0.50, 0.75
last_crossed_at:   float   # corruption level at last crossing (-1.0 = never crossed)
is_active:         bool    # true = currently above this threshold; prevents re-fire
```

**CorruptionMetrics** — return type of `get_metrics(mission_id: StringName)`:
```
mission_id:               StringName
mission_corruption_level: float        # 0.0–1.0; cumulative damage-history metric
attribution_split:        Dictionary   # { "self": float, "external": float } raw penalty sums
event_count:              int          # total events this mission
```

**C1 Public API** — methods exposed to downstream systems:
```
report_event(event_type, severity, target_node_id, source_actor, mission_id) -> void
get_metrics(mission_id: StringName) -> CorruptionMetrics
get_node_pressure(node_id: StringName) -> float
get_attribution_split(mission_id: StringName) -> Dictionary  # { "self": float, "external": float }
get_event_log(mission_id: StringName) -> Array[EventRecord]  # optional; for advanced post-mission UI
reset_threshold_states() -> void  # called by D1 on mission transition
```

*Note: `current_session_time()` is an internal helper returning elapsed seconds since session start. Godot implementation: `Time.get_ticks_msec() / 1000.0`. All timestamps in EventRecord and CorruptionTag use this value.*

---

**Rule 2 — `report_event()` Interface**

All gameplay systems route corruption events through this function. No system calls `A1.apply_corruption()` directly for corruption events.

```
report_event(
    event_type:     CorruptionEventType,
    severity:       CorruptionSeverity,
    target_node_id: StringName,
    source_actor:   StringName,   # NPC id or failure-event id
    mission_id:     StringName
) -> void
```

Execution (7 ordered steps):

1. Resolve penalty: `penalty = PENALTY_TABLE[event_type][severity]`
2. Resolve source: SELF_INFLICTED for STEALTH_DETECTION / INVESTIGATION_ERROR / SOCIAL_FAILURE; EXTERNAL_DECEPTION for NPC_DECEPTION
3. Resolve corruption type: `corruption_type = CORRUPTION_TYPE_MAP[event_type][severity]`
4. Build tag: `CorruptionTag { source, corruption_type, source_actor, mission_id, timestamp = current_session_time() }`
5. Write to A1: `A1.apply_corruption(target_node_id, tag, penalty)`
6. Append EventRecord to `_event_log`
7. Recompute aggregate metrics for this mission; evaluate threshold signals

Calling system contracts:

| Calling System | Event Type | Who Sets Severity |
|----------------|-----------|-------------------|
| B1 Stealth | STEALTH_DETECTION | B1 — based on detection tier (compromised vs. full alert) |
| B5 Investigation | INVESTIGATION_ERROR | B5 — based on how wrong the deduction was |
| B8 Social Manipulation | SOCIAL_FAILURE | B8 — based on manipulation attempt quality |
| C5 via B6 | NPC_DECEPTION | C5 — based on NPC psychology deception tier |

Exception — NPC-planted false nodes: when an NPC invents a node with no TruthGraph counterpart, C5 routes through B6 → `A1.create_false_node()` directly. `report_event(NPC_DECEPTION)` is for corruption of a real, existing node the NPC has access to. The C5 GDD must specify which deception actions use which path.

---

**Rule 3 — Penalty Magnitude Table**

```gdscript
## @export — all values are tuning knobs
const PENALTY_TABLE: Dictionary = {
    CorruptionEventType.STEALTH_DETECTION: {
        CorruptionSeverity.MINOR:    0.10,
        CorruptionSeverity.MODERATE: 0.22,
        CorruptionSeverity.MAJOR:    0.40   # == GHOST_THRESHOLD; always ghosts a CONFIRMED node
    },
    CorruptionEventType.INVESTIGATION_ERROR: {
        CorruptionSeverity.MINOR:    0.08,
        CorruptionSeverity.MODERATE: 0.18,
        CorruptionSeverity.MAJOR:    0.38   # just below GHOST_THRESHOLD; two MAJOR errors ghost
    },
    CorruptionEventType.SOCIAL_FAILURE: {
        CorruptionSeverity.MINOR:    0.10,
        CorruptionSeverity.MODERATE: 0.22,
        CorruptionSeverity.MAJOR:    0.40   # == GHOST_THRESHOLD
    },
    CorruptionEventType.NPC_DECEPTION: {
        CorruptionSeverity.MINOR:    0.06,
        CorruptionSeverity.MODERATE: 0.16,
        CorruptionSeverity.MAJOR:    0.36   # below threshold; deception rarely ghosts alone
    },
}
```

NPC_DECEPTION is scaled slightly lower than self-inflicted types across all severities. Rationale: NPC corruption is invisible at the moment it occurs and discovered later. Keeping it below GHOST_THRESHOLD by default preserves the paranoia fantasy — the player doesn't know how bad the deception was at the time. Raise NPC_DECEPTION MAJOR to 0.40 if playtesting shows NPC deception feels toothless.

INVESTIGATION_ERROR penalties are slightly lower than STEALTH_DETECTION. A wrong deduction represents incomplete reasoning (recoverable); a stealth detection is a moment of crisis. Flatten these to match stealth values if investigation feels consequence-free.

---

**Rule 4 — CorruptionType Mapping**

```gdscript
## @export — values are tuning knobs
const CORRUPTION_TYPE_MAP: Dictionary = {
    CorruptionEventType.STEALTH_DETECTION: {
        CorruptionSeverity.MINOR:    CorruptionType.CONFIDENCE_DEGRADATION,
        CorruptionSeverity.MODERATE: CorruptionType.CONFIDENCE_DEGRADATION,
        CorruptionSeverity.MAJOR:    CorruptionType.NODE_GHOST
    },
    CorruptionEventType.INVESTIGATION_ERROR: {
        CorruptionSeverity.MINOR:    CorruptionType.CONFIDENCE_DEGRADATION,
        CorruptionSeverity.MODERATE: CorruptionType.LABEL_ALTERED,
        CorruptionSeverity.MAJOR:    CorruptionType.LABEL_ALTERED
    },
    CorruptionEventType.SOCIAL_FAILURE: {
        CorruptionSeverity.MINOR:    CorruptionType.CONFIDENCE_DEGRADATION,
        CorruptionSeverity.MODERATE: CorruptionType.EDGE_OBSCURED,
        CorruptionSeverity.MAJOR:    CorruptionType.EDGE_OBSCURED
    },
    CorruptionEventType.NPC_DECEPTION: {
        CorruptionSeverity.MINOR:    CorruptionType.LABEL_ALTERED,
        CorruptionSeverity.MODERATE: CorruptionType.LABEL_ALTERED,
        CorruptionSeverity.MAJOR:    CorruptionType.LABEL_ALTERED
    },
}
```

Rationale:
- **STEALTH_DETECTION MAJOR → NODE_GHOST**: A major detection is a traumatic rupture — what was known fragments. NODE_GHOST is the type that can trigger GHOSTED state when penalty ≥ GHOST_THRESHOLD.
- **INVESTIGATION_ERROR MODERATE/MAJOR → LABEL_ALTERED**: A meaningful wrong deduction misidentifies something — the node exists but is labeled wrong. Post-mission reveal: "I thought this was Director Vasquez; it wasn't." Roll back to CONFIDENCE_DEGRADATION for MODERATE if playtesters find mid-mission mislabeling too disorienting.
- **SOCIAL_FAILURE MODERATE/MAJOR → EDGE_OBSCURED**: A failed manipulation exposes that the player misunderstood a relationship. The connection between nodes becomes uncertain.
- **NPC_DECEPTION always → LABEL_ALTERED**: NPC deception corrupts the player's *identification* of real nodes — the NPC feeds false information about a real person, event, or document, causing the player to misidentify it. `CorruptionType.FALSE_NODE` is reserved exclusively for the `create_false_node()` path (wholly invented nodes with no TruthGraph counterpart), which routes through B6 and does not pass through C1.

---

**Rule 5 — Aggregate Metrics**

C1 exposes four metrics via `get_metrics(mission_id: StringName) -> CorruptionMetrics`.

**Mission Corruption Level** (cumulative damage-history model):
```
mission_corruption_level = clamp(total_penalty_sum / NODE_COUNT_NORMALIZER, 0.0, 1.0)
```
This metric only increases. Repair (C4) does not reduce it — repair cleans the visible BeliefGraph, but the record of damage inflicted is permanent. Audio and map visual intensity are driven by what the player has done, not how clean it looks now.

**Attribution Split** (raw penalty sums, not percentages; E5 UI handles presentation):
```
split.self     = self_penalty_sum for mission
split.external = external_penalty_sum for mission
```

**Per-Node Corruption Pressure**:
```
node_corruption_pressure(node_id) =
    sum of tag.penalty_applied
    for each tag in A1.query_belief(node_id).corruption_tags
```
*Requires A1 schema amendment: add `penalty_applied: float` to CorruptionTag. See Open Questions.*

**Corruption Rate** (rolling window, events per minute):
```
corruption_rate =
    count of events in _event_log where timestamp >= (now - RATE_WINDOW_SECONDS)
    / RATE_WINDOW_SECONDS * 60.0
```

---

**Rule 6 — Threshold Signal Logic**

```gdscript
## @export tuning knobs
const CORRUPTION_THRESHOLDS: Array[float] = [0.25, 0.50, 0.75]
const RATE_SPIKE_THRESHOLD:  float = 3.0    # events per minute
const RATE_SPIKE_COOLDOWN:   float = 15.0   # seconds between spike emissions
```

After every `report_event()` call, C1 evaluates both signals:

**`corruption_threshold_crossed(threshold: float)`** — fires once per threshold per session:
```
for each threshold in CORRUPTION_THRESHOLDS:
    if mission_corruption_level >= threshold AND state.is_active == false:
        state.is_active = true
        emit corruption_threshold_crossed(threshold)
```
Because mission_corruption_level is cumulative and can only increase, each threshold fires at most once per session. No downward hysteresis needed.

**`corruption_rate_spiked(rate: float)`** — fires when sustained crisis detected:
```
if corruption_rate >= RATE_SPIKE_THRESHOLD:
    if time_since_last_emission >= RATE_SPIKE_COOLDOWN:
        emit corruption_rate_spiked(current_rate)
```
The signal carries the actual rate value so E6 (Adaptive Audio) can scale its response.

### States and Transitions

C1 is stateless with respect to individual nodes — it does not own BeliefState or confidence; those live in A1's BeliefGraph. C1's own state consists of the `_event_log`, `_threshold_states`, and the `_last_rate_spike_emission_time` scalar.

**C1 internal states:**

| State | Condition | Description |
|-------|-----------|-------------|
| `IDLE` | No `report_event()` in progress | Awaiting gameplay events |
| `PROCESSING` | Inside `report_event()` | Resolving penalty, writing to A1, updating metrics, checking thresholds |

These are implicit (C1 is synchronous and single-threaded); no explicit state machine is required.

**Per-threshold crossing state:**

| State | Condition | Transition |
|-------|-----------|------------|
| `BELOW` | `is_active == false` | → ABOVE when `mission_corruption_level >= threshold_value` |
| `ABOVE` | `is_active == true` | No return (level is cumulative; no downward transition) |

**Corruption rate state (for spike signal):**

| State | Condition | Signal Behavior |
|-------|-----------|----------------|
| `NOMINAL` | `corruption_rate < RATE_SPIKE_THRESHOLD` | No spike signal |
| `SPIKING` | `corruption_rate >= RATE_SPIKE_THRESHOLD` | `corruption_rate_spiked` emitted, subject to cooldown |
| `COOLING` | In SPIKING but within `RATE_SPIKE_COOLDOWN` window | Signal suppressed until cooldown expires |

### Interactions with Other Systems

| System | Direction | Interface | Notes |
|--------|-----------|-----------|-------|
| **A1 Timeline Data Architecture** | C1 writes + reads | `A1.apply_corruption(node_id, tag, penalty)` — write path. `A1.query_belief(node_id)` — standard read for per-node pressure metric. `A1.query_truth(node_id)` — restricted read; authorized but currently unused (reserved for future live divergence computation). | C1 is the only system besides PostMissionAnalysisUI permitted to call `query_truth()`. No active use case exists in the current design — the authorization is held in reserve. |
| **B1 Stealth System** | B1 calls C1 | `C1.report_event(STEALTH_DETECTION, severity, node_id, actor, mission_id)` | B1 classifies severity based on detection tier. B1 does NOT call `A1.apply_corruption()` directly for corruption. |
| **B5 Investigation/Deduction System** | B5 calls C1 | `C1.report_event(INVESTIGATION_ERROR, severity, node_id, actor, mission_id)` | B5 classifies severity based on how committed the player was to the wrong deduction. |
| **B8 Social Manipulation System** | B8 calls C1 | `C1.report_event(SOCIAL_FAILURE, severity, node_id, actor, mission_id)` | B8 also calls `A1.write_belief()` directly for successful manipulation. Only failures route through C1. |
| **C5 NPC Counter-Deception System** | C5 calls C1 (via B6) | `C1.report_event(NPC_DECEPTION, severity, node_id, npc_id, mission_id)` for corruption of real nodes. C5 routes `A1.create_false_node()` through B6 for invented nodes. | C5 does not call C1 directly — all calls route through B6's NPC write path. B6 is the NPC-to-graph gateway. |
| **C2 Fractured Reality Map** | C2 reads C1 | Subscribes to `corruption_threshold_crossed` and `corruption_rate_spiked` signals. Calls `C1.get_metrics(mission_id)` for visual intensity scaling. | C2 also subscribes to A1's `node_corrupted` signal directly for per-node visual updates. C1 provides the aggregate-level signals for ambient visual effects. |
| **C3 Anomaly Detection System** | C3 reads C1 | Calls `C1.get_node_pressure(node_id)` for each node when ranking anomalies post-mission. | Requires `penalty_applied` on CorruptionTag for exact pressure — see Open Questions. |
| **C4 Timeline Repair Economy** | C4 reads C1 | Calls `C1.get_node_pressure(node_id)` to determine repair eligibility and cost. Calls `C1.get_attribution_split(mission_id)` to inform repair prioritization UI. | C4 owns the repair logic; C1 provides the data C4 reads to compute costs. |
| **D3 Save/Load System** | D3 serializes C1 | Full serialization of C1's `_event_log` and `_threshold_states`. | C1's internal records are session-persistent and must survive save/load. MissionCorruptionState is recomputed from EventRecord on load — not separately serialized. |
| **E6 Adaptive Audio System** | E6 subscribes to C1 | Subscribes to `corruption_threshold_crossed(threshold)` and `corruption_rate_spiked(rate)`. | E6 uses threshold crossings for music layer shifts and rate spikes for moment-of-crisis audio escalation. |
| **D1 Mission State System** | D1 triggers C1 | Calls `C1.reset_threshold_states()` when `load_mission()` fires. This resets `_threshold_states.is_active` to false for the new mission so threshold signals fire fresh. | D1 owns the mission lifecycle; C1 reacts to mission transitions via this call. Without this trigger, threshold states from mission 1 suppress signals in mission 2. |

## Formulas

### Formula 1: Penalty Resolution

```
penalty = PENALTY_TABLE[event_type][severity]
```

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `event_type` | CorruptionEventType | — | Calling system | STEALTH_DETECTION, INVESTIGATION_ERROR, SOCIAL_FAILURE, NPC_DECEPTION |
| `severity` | CorruptionSeverity | — | Calling system | MINOR, MODERATE, MAJOR |
| `penalty` | float | 0.06–0.40 | PENALTY_TABLE lookup | Confidence penalty forwarded to `A1.apply_corruption()` |

**Expected output range**: 0.06 (NPC_DECEPTION MINOR) to 0.40 (STEALTH_DETECTION MAJOR, SOCIAL_FAILURE MAJOR). No values exceed A1's `GHOST_THRESHOLD` by default.

---

### Formula 2: Mission Corruption Level

```
mission_corruption_level = clamp(total_penalty_sum(mission_id) / NODE_COUNT_NORMALIZER, 0.0, 1.0)

total_penalty_sum(mission_id) = Σ record.penalty_applied
                                 for each record in _event_log
                                 where record.mission_id == mission_id
```

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `total_penalty_sum` | float | 0.0–∞ | C1 EventRecord log | Accumulated applied penalties this mission |
| `NODE_COUNT_NORMALIZER` | float | 2.0–6.0 | Tuning knob | Divisor mapping penalty sum to 0.0–1.0. Default: 3.0 |
| `mission_corruption_level` | float | 0.0–1.0 | Calculated | Normalized cumulative damage state |

**Expected output at key scenarios** (NODE_COUNT_NORMALIZER = 3.0):
- 1 MAJOR stealth failure (0.40): level = 0.133
- 3 MAJOR stealth failures (3 × 0.40 = 1.20): level = 1.0 (saturated)
- 10 MINOR failures (10 × 0.10 = 1.00): level = 0.333

**Design note**: This metric never decreases. C4 Timeline Repair reduces BeliefGraph node confidence; it does not reduce `total_penalty_sum`. Use `node_corruption_pressure` (Formula 4) if a metric that responds to repair is needed.

---

### Formula 3: Attribution Split

```
attribution_split.self     = Σ record.penalty_applied where source == SELF_INFLICTED
attribution_split.external = Σ record.penalty_applied where source == EXTERNAL_DECEPTION
```
(Both filtered to `record.mission_id == mission_id`)

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `attribution_split.self` | float | 0.0–∞ | C1 EventRecord log | Raw accumulated self-inflicted penalties |
| `attribution_split.external` | float | 0.0–∞ | C1 EventRecord log | Raw accumulated NPC-inflicted penalties |

**Post-mission UI calculation** (owned by E5, not C1):
```
self_pct     = attribution_split.self / (self + external) * 100.0
external_pct = attribution_split.external / (self + external) * 100.0
```

---

### Formula 4: Per-Node Corruption Pressure

```
node_corruption_pressure(node_id) =
    Σ tag.penalty_applied
    for each tag in A1.query_belief(node_id).corruption_tags
```

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `tag.penalty_applied` | float | 0.06–0.40 | CorruptionTag field | Penalty recorded at time of application (requires A1 schema amendment — see Open Questions) |
| `node_corruption_pressure` | float | 0.0–∞ | Calculated | Cumulative penalty sum for this node across all corruption events |

**Expected range**: A node hit by one MAJOR event: 0.40. A node hit by five MODERATE events: ~1.10. No upper bound — the same node can be hit repeatedly.

---

### Formula 5: Corruption Rate

```
corruption_rate(mission_id, window_seconds) =
    count(records where mission_id matches AND timestamp >= now - window_seconds)
    / window_seconds
    * 60.0
```

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `window_seconds` | float | 10.0–120.0 | Tuning knob `RATE_WINDOW_SECONDS` | Rolling window duration. Default: 60.0 |
| `corruption_rate` | float | 0.0–∞ | Calculated | Events per minute in the rolling window |

**Spike threshold**: 3.0 events/minute (roughly one event every 20 seconds sustained for 60 seconds).

**Example**: 5 events in the last 60 seconds = 5.0 events/minute. Spike threshold exceeded; `corruption_rate_spiked(5.0)` emitted (subject to cooldown).

## Edge Cases

| Scenario | Expected Behavior | Rationale |
|----------|------------------|-----------|
| `report_event()` called with a `target_node_id` not in BeliefGraph | Log error and no-op. Do not create a node implicitly. Do not call `A1.apply_corruption()`. Do not append EventRecord. | Nodes must exist before they can be corrupted. Silent creation bypasses fog initialization and produces orphaned corruption records. This is a calling-system bug, not a C1 recovery case. |
| `report_event()` called with `NPC_DECEPTION` on a false-positive node (is_false_positive = true) | Process normally. Append EventRecord, apply penalty, write to A1. | False-positive nodes are valid BeliefNodes. Compounding deception is valid game state: one NPC plants a false node, another corrupts it further. Attribution still captured correctly. |
| Two `report_event()` calls targeting the same node in the same frame | Process sequentially (GDScript is single-threaded). Both EventRecords appended. Both penalties applied to the node via `A1.apply_corruption()`. | A1's GHOSTED trigger only fires on the first call if the first penalty >= GHOST_THRESHOLD. The second call hits an already-GHOSTED node, which A1 handles correctly (appends tag, no re-trigger). From C1's perspective, both events are valid and distinct audit records. |
| Mission transition: new mission loads via D1 | Reset `_threshold_states` (all `is_active = false`) for the new mission. Preserve `_event_log` (cross-mission attribution must survive). `MissionCorruptionState` for the new mission starts at zero. | Each mission is a distinct corruption episode — threshold crossings should fire fresh for each. The EventLog persists so post-campaign analysis can compute overall self/external attribution across all missions. |
| `NODE_COUNT_NORMALIZER` set to 0.0 or negative | Guard at initialization: clamp `NODE_COUNT_NORMALIZER` to minimum 0.1. Log a configuration warning. | Division by zero produces NaN or infinity, which would corrupt the threshold comparison and downstream systems. This is a content configuration error, not a runtime recovery case; the clamp provides a safety floor while the warning surfaces the misconfiguration. |
| Corruption rate calculation after save/load | Event timestamps in `_event_log` are session timestamps. After load, `current_session_time()` resumes from 0.0. Events from the previous session will have timestamps in a prior range and will exit the rolling window immediately. | Rate effectively resets to 0.0 at session start. This is acceptable — `corruption_rate_spiked` is an in-session signal for live audio reactivity, not a persistent state. The first few events in the new session build the rate fresh. |
| All three thresholds crossed before NPC counter-deception is introduced (early-game scenario) | No special handling. Thresholds already crossed; no new threshold signals fire for the rest of the mission. | Audio and map visual intensity are already at their highest bands. NPC deception adding further corruption to an already-saturated mission is valid. C3 (Anomaly Detection) and C4 (Repair Economy) continue operating regardless of threshold state. |
| `report_event()` called with a valid event type during post-mission analysis phase | Process normally. Append EventRecord. The mission is technically still open from C1's perspective until D1 signals a new mission load. | Post-mission analysis may trigger additional corruption (e.g., a deduction made during analysis causes a retroactive investigation error). C1 has no concept of "analysis phase" — that boundary is owned by D1. |

## Dependencies

**Upstream Dependencies (systems C1 requires)**

| System | Dependency Type | What C1 Requires |
|--------|----------------|-----------------|
| **A1 Timeline Data Architecture** | Hard | `apply_corruption()` — write path for all corruption events. `query_truth()` — restricted access for divergence computation. `query_belief()` — read path for per-node pressure metric. `CorruptionTag` schema, `CorruptionSource` and `CorruptionType` enums. |

**Schema amendment pending**: A1's `CorruptionTag` requires the addition of `penalty_applied: float`. C1's per-node pressure formula (Formula 4) and C4's repair cost calculations depend on this field. This must be resolved before C3 or C4 GDDs are written. See Open Questions.

---

**Downstream Dependents (systems that depend on C1)**

| System | Dependency Type | What They Require From C1 |
|--------|----------------|--------------------------|
| **B1 Stealth System** | Hard | `report_event(STEALTH_DETECTION, ...)` — C1 is the write path for stealth corruption |
| **B5 Investigation/Deduction System** | Hard | `report_event(INVESTIGATION_ERROR, ...)` |
| **B8 Social Manipulation System** | Hard | `report_event(SOCIAL_FAILURE, ...)` — only for failure cases; successes go directly to `A1.write_belief()` |
| **C5 NPC Counter-Deception System** | Hard (via B6) | `report_event(NPC_DECEPTION, ...)` for real-node corruption. Does not call C1 directly — routes through B6. |
| **C2 Fractured Reality Map** | Hard | `corruption_threshold_crossed` signal; `corruption_rate_spiked` signal; `C1.get_metrics(mission_id)` for visual intensity |
| **C3 Anomaly Detection System** | Hard | `C1.get_node_pressure(node_id)` — per-node corruption pressure for anomaly ranking |
| **C4 Timeline Repair Economy** | Hard | `C1.get_node_pressure(node_id)` — repair eligibility and cost; `C1.get_attribution_split(mission_id)` — repair prioritization |
| **D3 Save/Load System** | Hard | Full serialization of `_event_log` and `_threshold_states` |
| **E6 Adaptive Audio System** | Hard | `corruption_threshold_crossed` signal; `corruption_rate_spiked` signal |

**Bidirectionality note**: Each downstream system listed above must reference C1 as an upstream hard dependency in its own Dependencies section. C5 references B6; B6 references C1.

## Tuning Knobs

All tuning knobs must be `@export` constants or data-driven config values. No hardcoded gameplay numbers in GDScript.

| Knob | Default | Safe Range | Too High | Too Low | Affects |
|------|---------|-----------|----------|---------|---------|
| `PENALTY_TABLE[STEALTH_DETECTION][MAJOR]` | 0.40 | 0.30–0.50 | >0.50: Exceeds GHOST_THRESHOLD — every MAJOR stealth event ghosts CONFIRMED nodes instantly | <0.30: MAJOR detection too forgiving; stealth feels low-stakes | Primary drama lever for stealth consequences |
| `PENALTY_TABLE[SOCIAL_FAILURE][MAJOR]` | 0.40 | 0.30–0.50 | Same as stealth: always ghosts on MAJOR | <0.30: Social failure too forgiving; dialogue feels low-risk | Primary drama lever for social consequences |
| `PENALTY_TABLE[INVESTIGATION_ERROR][MAJOR]` | 0.38 | 0.28–0.45 | ≥0.40: Equals GHOST_THRESHOLD — a single committed wrong deduction ghosts. May be too punishing for investigation-primary players | <0.28: Investigation errors barely sting; deduction feels consequence-free | Investigation difficulty tuning |
| `PENALTY_TABLE[NPC_DECEPTION][MAJOR]` | 0.36 | 0.25–0.42 | ≥0.40: NPC deception ghosts CONFIRMED nodes — full parity with stealth failures. Appropriate for late-game adversaries | <0.25: NPC deception feels toothless even at MAJOR tier | How dangerous a masterful NPC deceiver feels |
| `PENALTY_TABLE[*][MINOR]` | 0.06–0.10 | ±0.04 from defaults | High MINOR values penalize every small mistake heavily; exploration becomes overly punishing | Low MINOR values make routine errors feel consequence-free | Baseline tension from ordinary play |
| `NODE_COUNT_NORMALIZER` | 3.0 | 2.0–6.0 | ≥6.0: Even catastrophic play stays below 0.50 corruption level; audio/map feel under-reactive | ≤2.0: 2 MAJOR events saturate the meter; level reaches 1.0 too fast, all signals fire in rapid succession | How quickly the corruption level meter fills; threshold signal timing |
| `CORRUPTION_THRESHOLDS` | [0.25, 0.50, 0.75] | Each ±0.10 | High thresholds: audio/map never reach mid/high reactivity bands in ordinary play | Low thresholds: all three bands fire within the first few minutes; progression feels compressed | Pacing of audio tension escalation and map visual intensification across a mission |
| `RATE_SPIKE_THRESHOLD` | 3.0 | 1.5–6.0 | ≥6.0: Only extreme sustained crisis triggers spike; most crisis moments are missed | ≤1.5: Any two events in quick succession triggers spike; E6 escalates too readily | Sensitivity of crisis detection for audio |
| `RATE_WINDOW_SECONDS` | 60.0 | 30.0–120.0 | ≥120.0: Rate metric is very smooth; slow to react to sudden bursts; audio escalation feels sluggish | ≤30.0: Rate spikes on brief bursts; audio escalates and de-escalates rapidly | Responsiveness of rate-based audio signals |
| `RATE_SPIKE_COOLDOWN` | 15.0 | 5.0–60.0 | ≥60.0: During prolonged crisis, audio receives one spike signal per minute maximum; may feel under-responsive | ≤5.0: Spike signals fire almost continuously during crisis; audio spam | How often E6 is notified of sustained crisis |

**Critical interaction — PENALTY_TABLE MAJOR values and GHOST_THRESHOLD**:
`GHOST_THRESHOLD` is owned by A1 (default: 0.40). Any PENALTY_TABLE MAJOR value at or above `GHOST_THRESHOLD` means a single MAJOR event on a CONFIRMED node will always trigger GHOSTED. Values below mean two events or a combined hit are needed. Always tune MAJOR penalty values with awareness of `GHOST_THRESHOLD`. If A1's `GHOST_THRESHOLD` is raised, raise MAJOR penalties proportionally.

**Critical interaction — NODE_COUNT_NORMALIZER and CORRUPTION_THRESHOLDS**:
If `NODE_COUNT_NORMALIZER` is raised (making the level harder to fill), also lower `CORRUPTION_THRESHOLDS` proportionally to keep the first threshold (0.25) reachable within an ordinary mission. If the first threshold never fires, E6 and C2 never receive their lowest-tier signal, and the reactivity progression never starts.

## Visual/Audio Requirements

C1 is a pure logic and accounting layer. It emits signals; it does not render anything or play any audio. All visual and audio responses are owned by downstream systems.

| Signal | Downstream Consumer | Required Response |
|--------|------------------|-------------------|
| `corruption_threshold_crossed(threshold: float)` | C2 Fractured Reality Map | Intensify ambient corruption visual effects to the band corresponding to this threshold |
| `corruption_threshold_crossed(threshold: float)` | E6 Adaptive Audio | Shift music tension layer to the band corresponding to this threshold |
| `corruption_rate_spiked(rate: float)` | C2 Fractured Reality Map | Trigger a brief visual crisis response proportional to `rate` |
| `corruption_rate_spiked(rate: float)` | E6 Adaptive Audio | Escalate moment-of-crisis audio cue proportional to `rate` |

C1 requires no art assets. All visual and audio specifications belong to the C2 and E6 GDDs.

## UI Requirements

C1 has no direct UI surface. All UI is owned by E5 (Post-Mission Analysis UI). C1's data contract with E5:

| Data | C1 Method | E5 Usage |
|------|-----------|---------|
| Self vs. NPC attribution split | `get_attribution_split(mission_id)` → `{self: float, external: float}` | Displayed as percentage breakdown: "X% your mistakes / Y% planted deception" |
| Per-node corruption history | `get_node_pressure(node_id)` → float | Used to rank nodes for repair prioritization display |
| Full EventRecord log | `get_event_log(mission_id)` → `Array[EventRecord]` | Optional: advanced post-mission breakdown showing each corruption event in chronological order (E5 GDD decides whether to expose this level of detail) |

**Contract requirement**: All C1 query methods return typed values — no untyped Dictionary payloads to E5. E5 must never hold cached copies of C1 data that spans multiple calls; always query fresh when the post-mission analysis session opens.

## Acceptance Criteria

All criteria are verifiable via C1's public API and signal monitoring. Tests must use `C1.get_metrics()`, `C1.get_node_pressure()`, and signal observation — no direct inspection of `_event_log` permitted in release builds.

| # | Criterion | Test Method | Pass Condition |
|---|-----------|------------|----------------|
| AC-C1-01 | `report_event()` resolves correct penalty and calls A1 | Call `report_event(STEALTH_DETECTION, MAJOR, node, actor, mission)`. Observe A1 `node_corrupted` signal. | Signal fired with penalty = 0.40. CorruptionTag has source = SELF_INFLICTED, corruption_type = NODE_GHOST. |
| AC-C1-02 | `report_event()` resolves correct penalty for all four event types at MODERATE | Call each event type with MODERATE severity. Observe A1 `node_corrupted` signal for each. | Penalties: STEALTH=0.22, INVESTIGATION=0.18, SOCIAL=0.22, NPC=0.16. CorruptionSource: first three SELF_INFLICTED, NPC is EXTERNAL_DECEPTION. |
| AC-C1-03 | `mission_corruption_level` accumulates correctly | With NODE_COUNT_NORMALIZER=3.0: call `report_event(STEALTH_DETECTION, MAJOR, ...)` once (penalty 0.40). Query `C1.get_metrics(mission_id).mission_corruption_level`. | Returns 0.40/3.0 ≈ 0.133. Repeat three times total. Fourth query returns 1.20/3.0 = 1.0 (clamped). |
| AC-C1-04 | `mission_corruption_level` does not decrease after repair | Raise level via three events. Record level. Simulate C4 repair of one node (A1 confidence restored). Re-query level. | Level unchanged. Repair does not reduce `total_penalty_sum`. |
| AC-C1-05 | `corruption_threshold_crossed` fires at correct breakpoints | With thresholds [0.25, 0.50, 0.75] and NODE_COUNT_NORMALIZER=3.0: generate events totaling 0.75 (fires 0.25 threshold), then 1.50 (fires 0.50), then 2.25 (fires 0.75). | Signal fired exactly once at each threshold value. No duplicate firings if level continues rising above an already-crossed threshold. |
| AC-C1-06 | Threshold states reset on mission transition | Cross threshold 0.25 in mission 1. Signal D1 new mission load; C1 resets threshold states. Add events in mission 2 totaling above 0.25. | `corruption_threshold_crossed(0.25)` fires again in mission 2. Mission 1 threshold crossing does not suppress mission 2 crossing. |
| AC-C1-07 | `corruption_rate_spiked` fires at RATE_SPIKE_THRESHOLD | With RATE_SPIKE_THRESHOLD=3.0 and RATE_WINDOW_SECONDS=60.0: fire 4 events within 60 seconds. | `corruption_rate_spiked(rate)` emitted with rate ≥ 3.0. |
| AC-C1-08 | `corruption_rate_spiked` respects cooldown | After one `corruption_rate_spiked` emission: fire additional events immediately. Observe signal. | No second signal emitted within RATE_SPIKE_COOLDOWN (15.0 seconds). Signal emitted again after cooldown expires if rate still ≥ threshold. |
| AC-C1-09 | Attribution split correctly separates self vs. NPC | Fire 2 STEALTH_DETECTION MODERATE events (2 × 0.22 = 0.44 self) and 1 NPC_DECEPTION MAJOR event (0.36 external). Query `get_metrics(mission_id).attribution_split`. | `split.self ≈ 0.44`, `split.external ≈ 0.36`. |
| AC-C1-10 | Per-node pressure reflects accumulated tag penalties | Fire two corruption events on the same node (penalties 0.22 and 0.10). Query `C1.get_node_pressure(node_id)`. | Returns 0.32 (sum of both penalties). Requires CorruptionTag.penalty_applied field — see Open Questions. |
| AC-C1-11 | `report_event()` on unknown node is a no-op | Call `report_event()` with a node_id not in BeliefGraph. Observe A1 `node_corrupted` signal and C1 EventRecord count. | Signal not fired. EventRecord count unchanged. Error logged. |
| AC-C1-12 | EventLog persists across mission boundaries | Fire events in mission 1. Load mission 2. Query `C1.get_metrics("mission_1")`. | Mission 1 EventRecords still present. `total_penalty_sum` for mission 1 unchanged. |
| AC-C1-13 | Save/load roundtrip preserves C1 state | Fire events, cross two thresholds. Serialize. Deserialize. Re-query metrics and threshold states. | `_event_log` record count preserved. `_threshold_states.is_active` preserved for crossed thresholds. `mission_corruption_level` recomputed from preserved log equals pre-save value. |
| AC-C1-14 | NPC_DECEPTION on a false-positive node processes normally | Create a false-positive node via A1. Call `report_event(NPC_DECEPTION, MINOR, false_node_id, npc, mission)`. | EventRecord appended. A1 `node_corrupted` signal fired. No error logged. |

## Open Questions

| # | Question | Owner | Target Resolution | Notes |
|---|----------|-------|------------------|-------|
| OQ-C1-01 | A1 schema amendment: add `penalty_applied: float` to `CorruptionTag` | Lead Programmer + Systems Designer | Before C3 and C4 GDDs are authored | C1's per-node pressure formula (Formula 4) and C4's repair cost calculations require this field. Without it, both systems must use the `(1.0 - confidence)` approximation, which loses per-tag resolution. The A1 GDD must be updated to reflect this change. |
| OQ-C1-02 | (From A1 OQ-02) Can `write_belief()` raise a false-positive node's confidence? | Systems Designer | Before C5 GDD is authored | Current spec allows it. Risk: player reinforces an NPC-planted lie through their own investigation. This may be intentional (Pillar 6) or an undesired exploit. If reinforcement is blocked, C1 must add a check before calling `A1.write_belief()` on `is_false_positive` nodes (though this is A1's domain — A1 may need a guard). |
| OQ-C1-03 | How does C5 determine which NPC deception events route through `C1.report_event(NPC_DECEPTION)` vs. directly through `A1.create_false_node()`? | C5 GDD + Systems Designer | Before C5 GDD is authored | Current spec: `report_event(NPC_DECEPTION)` = NPC corrupts a real existing node. `create_false_node()` = NPC invents a node. The C5 GDD must enumerate which NPC deception actions use which path and under what conditions. |
| OQ-C1-04 | Should C1 expose a cross-mission aggregate corruption level (total across all 7 missions)? | Systems Designer + UX Designer | Before E5 (Post-Mission Analysis UI) GDD | Currently C1 computes per-mission levels only. A campaign-level "total corruption index" could be used for the final post-campaign analysis or for long-term audio adaptation. Low priority for MVP. |
| OQ-C1-05 | What happens to `_threshold_states` during save/load within a mission (mid-mission save)? | Lead Programmer + D3 GDD | Before D3 GDD is authored | Threshold states must be serialized as part of C1's save data so that thresholds don't re-fire after loading a mid-mission save. D3 GDD must include C1's `_threshold_states` in the serialization spec. |
| OQ-C1-06 | A1 GDD Interactions table must be amended to reflect C1 routing for B1, B5, B8 | Lead Programmer + Systems Designer | Before implementation begins | A1 currently states that B1 calls `A1.apply_corruption()` directly. With C1 as the routing layer, A1's table should read "B1 calls C1.report_event(); C1 calls A1.apply_corruption()." Same correction needed for B5 and B8. This is a GDD consistency fix, not a design change. |
