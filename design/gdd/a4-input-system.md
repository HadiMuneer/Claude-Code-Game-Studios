# Input System (A4)

> **Status**: Approved
> **Author**: Human + Claude Code
> **Last Updated**: 2026-03-29
> **Implements Pillar**: Pillar 3 (Three Paths, One Truth), Pillar 4 (Earn the Knife)

## Overview

The Input System (A4) is the action abstraction layer between raw hardware input and all gameplay systems in TIME-PUNK. It translates keyboard, mouse, and (in future milestones) gamepad events into named game actions (`move_forward`, `interact`, `enter_dialogue`, `examine`, `crouch`), and exposes a single query API that every gameplay system calls. This decoupling means keybindings are remappable without touching gameplay code, input sources are transparent to callers, and the abstraction can be extended to support controller at a later milestone without modifying downstream systems. The player never interacts with the Input System directly — they interact with it only through the Settings menu (keybinding remapping). It operates as a silent prerequisite for every physical action in the game.

TIME-PUNK requires three distinct input contexts that map the same physical keys to different semantic actions depending on game state: `EXPLORATION` (movement, stealth, investigation interactions), `DIALOGUE` (social manipulation, conversation navigation), and `MAP` (Fractured Reality Map review and node inspection). The Input System maintains an explicit context stack. Only actions registered for the active context are forwarded to gameplay systems — pressing "interact" during a dialogue sequence does not accidentally trigger evidence pickup. Contexts push and pop as gameplay state changes; the system beneath a pushed context remains paused.

Implemented in Godot 4.6 using the engine's `InputMap` for binding storage and `Input` class for polling and event-driven queries. The Input System wraps these in a typed GDScript autoload singleton (`InputSystem`) that exposes `is_action_pressed()`, `is_action_just_pressed()`, and `get_movement_vector()` methods with context validation built in.

## Player Fantasy

The Input System has no player fantasy of its own — it is infrastructure that serves other systems' fantasies. Its success condition is invisibility: controls that feel so immediate and consistent that the player never thinks about the mapping between intent and action. A player planning a stealth approach thinks about cover angles and patrol timing, not about which key to press. A player navigating a tense dialogue reads behavioral cues and chooses manipulation tactics, not which button advances the line. The Input System earns its place by never calling attention to itself.

The one moment the Input System becomes player-facing is keybinding remapping in the Settings menu — and even here the fantasy is ergonomic comfort, not engagement. The player should be able to configure controls once and forget the system exists. Inaccessible defaults (e.g., hardcoded bindings that cannot be changed) would violate this contract and break accessibility for players with non-standard setups.

For the developer and designer, the Input System's "fantasy" is **zero maintenance burden**: adding a new gameplay action should require adding one entry to the action registry and one line of context registration — not modifying five different input-handling files. The abstraction should feel correct and permanent at design time, not something revisited every sprint.

## Detailed Design

### Core Rules

**Rule 1 — Action Taxonomy**

All input actions are named string constants registered in Godot's `InputMap`. No gameplay system reads raw `InputEvent` data — they query `InputSystem` by action name.

**EXPLORATION context actions** (default context; movement, stealth, investigation):

| Action Name | Default Binding | Type | Description |
|---|---|---|---|
| `move_forward` | W | Axis | Forward movement component |
| `move_back` | S | Axis | Backward movement component |
| `move_left` | A | Axis | Strafe left component |
| `move_right` | D | Axis | Strafe right component |
| `look` | Mouse delta | Axis | Camera look direction |
| `sprint` | Left Shift (hold) | Held | Increases movement speed; raises noise output |
| `crouch` | C | Toggle | Reduces silhouette and noise; reduces movement speed |
| `lean_left` | Q (hold) | Held | Lateral lean for peeking around cover |
| `lean_right` | E (hold) | Held | Lateral lean for peeking around cover |
| `interact` | F | Just-pressed | Context-sensitive: examine object, open door, pick up item |
| `inspect_detail` | Hold F / RMB | Held | Focus/zoom on an object for detailed examination |
| `open_map` | Tab | Just-pressed | Pushes MAP context; opens Fractured Reality Map |
| `enter_dialogue` | F (near NPC) | Just-pressed | Pushes DIALOGUE context; shares binding with `interact` — resolved by proximity check in A2 |
| `pause` | Escape | Just-pressed | Open/close pause menu; fires only in EXPLORATION — Escape in an overlay fires `map_close` or `dialogue_cancel` instead |

*Escape priority: if MAP or DIALOGUE context is active, Escape fires the context-cancel action first (`map_close` / `dialogue_cancel`). `pause` fires only when EXPLORATION is the active context (no overlay). There are no always-on actions.*

**DIALOGUE context actions** (social manipulation, conversation):

| Action Name | Default Binding | Type | Description |
|---|---|---|---|
| `dialogue_advance` | Space or LMB | Just-pressed | Advance dialogue line or confirm selection |
| `dialogue_choice_1` | 1 | Just-pressed | Select first dialogue option |
| `dialogue_choice_2` | 2 | Just-pressed | Select second dialogue option |
| `dialogue_choice_3` | 3 | Just-pressed | Select third dialogue option |
| `dialogue_choice_4` | 4 | Just-pressed | Select fourth dialogue option |
| `dialogue_cancel` | Escape | Just-pressed | Exit dialogue; pops DIALOGUE context |

**MAP context actions** (Fractured Reality Map review):

| Action Name | Default Binding | Type | Description |
|---|---|---|---|
| `map_pan` | Mouse drag (LMB hold) / WASD | Axis | Pan the map view |
| `map_zoom_in` | Scroll up / = | Just-pressed | Zoom in on map |
| `map_zoom_out` | Scroll down / - | Just-pressed | Zoom out on map |
| `map_select` | LMB click | Just-pressed | Select / inspect a map node |
| `map_node_inspect` | RMB / Hold LMB | Held | Open detailed node view |
| `map_close` | Tab or Escape | Just-pressed | Close map; pop MAP context |

---

**Rule 2 — Context Stack**

```
InputContext enum: EXPLORATION | DIALOGUE | MAP
```

- `EXPLORATION` is the base context. It is always present and cannot be popped.
- `DIALOGUE` and `MAP` are overlay contexts. Only one overlay may be active at a time.
- Context transitions are push/pop: `push_context(ctx)` / `pop_context()`.
- When an overlay is active, EXPLORATION actions are suspended. `pause` is an EXPLORATION action and therefore does not fire while an overlay is active.
- Direct overlay-to-overlay transitions are not permitted — must return to EXPLORATION first.

**Context transitions:**

| Event | From | To | Operation |
|-------|------|----|-----------|
| Player presses `open_map` | EXPLORATION | MAP | push(MAP) |
| Player presses `map_close` or Escape | MAP | EXPLORATION | pop() |
| Player triggers `enter_dialogue` | EXPLORATION | DIALOGUE | push(DIALOGUE) |
| Player presses `dialogue_cancel` or dialogue ends | DIALOGUE | EXPLORATION | pop() |
| `pause` pressed | EXPLORATION only | (pause menu, not a context) | separate system |

---

**Rule 3 — `InputSystem` Singleton API**

```gdscript
## InputSystem — autoload singleton (res://src/core/input_system.gd)
## All gameplay systems query input through this interface only.
## Never read Input.* directly in gameplay code.

func push_context(ctx: InputContext) -> void
func pop_context() -> void
func get_active_context() -> InputContext

## Action queries — return false if action not valid in active context
func is_action_pressed(action: StringName) -> bool
func is_action_just_pressed(action: StringName) -> bool
func is_action_just_released(action: StringName) -> bool

## Movement axis — only valid in EXPLORATION context; returns Vector2.ZERO otherwise
func get_movement_vector() -> Vector2   # normalized WASD input

## Look delta — only valid in EXPLORATION context; returns Vector2.ZERO in overlay contexts
## Raw unscaled mouse delta for this frame. E1 (Camera System) applies sensitivity scaling.
func get_look_delta() -> Vector2

## Binding remapping
func remap_action(action: StringName, new_event: InputEvent) -> void
func reset_action_to_default(action: StringName) -> void
func save_bindings() -> void   # persists to user://input_bindings.cfg
func load_bindings() -> void   # called at game startup

## Signals
signal context_pushed(context: InputContext)
signal context_popped(context: InputContext)
signal binding_changed(action: StringName, new_event: InputEvent)
```

---

**Rule 4 — Controller Readiness**

All action names are hardware-agnostic string constants. When controller support is added (Vertical Slice milestone), only two changes are required: (1) add gamepad `InputEvent` entries to each action in `InputMap`; (2) extend `get_movement_vector()` to blend keyboard and analog stick input. No gameplay system code changes required.

### States and Transitions

The Input System's state is the context stack. The stack has a maximum depth of 2: the base `EXPLORATION` context plus at most one overlay.

| Context | Entry Condition | Exit Condition | Actions Active |
|---------|----------------|----------------|----------------|
| `EXPLORATION` | Game start; always present at stack base | Cannot exit (base; always present) | All EXPLORATION actions (including `pause`) |
| `DIALOGUE` | `push_context(DIALOGUE)` called by Dialogue System | `pop_context()` called when dialogue ends or `dialogue_cancel` fired | All DIALOGUE actions only; EXPLORATION (including `pause`) suspended |
| `MAP` | `push_context(MAP)` called by Map System on `open_map` | `pop_context()` called when `map_close` or Escape fired | All MAP actions only; EXPLORATION (including `pause`) suspended |

**Invalid state: double overlay**

If `push_context()` is called while an overlay is already active, the call is rejected and an error is logged. The caller must pop the current overlay before pushing a new one. This prevents undefined states like DIALOGUE + MAP being simultaneously active.

**Stack diagram:**

```
Normal play:        [EXPLORATION]
During dialogue:    [EXPLORATION | DIALOGUE]   ← DIALOGUE is top; EXPLORATION suspended
During map review:  [EXPLORATION | MAP]         ← MAP is top; EXPLORATION suspended
```

### Interactions with Other Systems

| System | Direction | Interface | Notes |
|--------|-----------|-----------|-------|
| **A2 Character Controller** | Reads InputSystem | `InputSystem.get_movement_vector()`, `is_action_pressed("sprint")`, `is_action_pressed("crouch")`, `is_action_pressed("lean_left")`, `is_action_pressed("lean_right")` | A2 polls these every `_physics_process()`. A2 owns the `interact` / `enter_dialogue` disambiguation: checks proximity to NPC before deciding which action to fire. |
| **E1 Camera System** | Reads InputSystem | `InputSystem.get_look_delta()` | E1 polls `get_look_delta()` every `_process()` and applies its own sensitivity/inversion scaling. `get_look_delta()` returns `Vector2.ZERO` when an overlay context is active, which naturally freezes camera look during dialogue and map review. |
| **B7 Dialogue System** | Pushes/pops context; reads InputSystem | Calls `InputSystem.push_context(DIALOGUE)` on dialogue start. Reads `dialogue_advance`, `dialogue_choice_1–4`, `dialogue_cancel`. Calls `InputSystem.pop_context()` on dialogue end. | Dialogue System owns the lifecycle; InputSystem just gates the actions. |
| **C2 Fractured Reality Map** | Pushes/pops context; reads InputSystem | Calls `InputSystem.push_context(MAP)` when map opens. Reads all MAP context actions. Calls `InputSystem.pop_context()` on map close. | Map System subscribes to `context_pushed` and `context_popped` signals to know when it has input focus. |
| **E8 Settings / Main Menu** | Reads InputSystem; calls remapping API | Calls `remap_action()`, `reset_action_to_default()`, `save_bindings()` for the keybinding UI. `pause` is an EXPLORATION action; the pause menu consumer queries it via `is_action_just_pressed("pause")` (returns false when an overlay is active). | Settings owns the remapping UI; InputSystem owns the data. |
| **D3 Save/Load System** | Triggers `load_bindings()` | Calls `InputSystem.load_bindings()` at game startup before any scene is playable. | Binding persistence is InputSystem's responsibility; D3 triggers the load. |

## Formulas

The Input System contains no gameplay math. Its only formula is movement vector normalization for diagonal input correction.

### Formula 1: Movement Vector

```
raw_vector = Vector2(
    Input.get_axis("move_left", "move_right"),
    Input.get_axis("move_forward", "move_back")
)

movement_vector = raw_vector.normalized()   if raw_vector.length() > 0.0
                  Vector2.ZERO              otherwise
```

| Variable | Type | Range | Description |
|---|---|---|---|
| `raw_vector.x` | float | -1.0–1.0 | Left/right axis from `get_axis()` |
| `raw_vector.y` | float | -1.0–1.0 | Forward/back axis from `get_axis()` |
| `movement_vector` | Vector2 | magnitude 0.0–1.0 | Normalized output; diagonal input has magnitude 1.0, not √2 |

**Why normalize**: Without normalization, diagonal movement (W+D) produces a vector of magnitude √2 ≈ 1.41, giving diagonal movement ~41% more speed than cardinal movement. Normalization ensures consistent speed in all directions. A2 (Character Controller) multiplies `movement_vector` by its own speed constant — it never receives raw magnitude.

**Expected output at key inputs**:
- No input: `Vector2(0.0, 0.0)`
- W only (forward): `Vector2(0.0, -1.0)` (Godot convention: -Y = forward in 2D input space)
- W + D (forward-right diagonal): `Vector2(0.707, -0.707)` (normalized)

*Note: A2 applies camera-relative orientation to this vector. InputSystem returns input-space direction; world-space direction is A2's concern.*

## Edge Cases

| Scenario | Expected Behavior | Rationale |
|----------|------------------|-----------|
| `push_context()` called while an overlay is already active | Log error; reject the push. Active context unchanged. | Double-overlay is an invalid state. The calling system must pop the current overlay first. This prevents DIALOGUE + MAP being simultaneously active. |
| `pop_context()` called when only EXPLORATION is in the stack | Log error; no-op. EXPLORATION cannot be popped. | The base context is permanent; attempting to pop it is a calling-system bug. |
| Action queried for a context that is not currently active | Return `false` for `is_action_just_pressed()` / `is_action_pressed()`. Return `Vector2.ZERO` for `get_movement_vector()`. | Callers should only query actions in their context. Returning false (not error) prevents crashes in systems that query defensively before checking context. |
| `interact` and `enter_dialogue` share the F key near an NPC | A2 Character Controller resolves disambiguation: if an interactable NPC is within dialogue range, `enter_dialogue` fires; otherwise `interact` fires. InputSystem makes both actions available; A2 owns the priority rule. | The Input System does not know about world geometry or NPC proximity — A2 does. Disambiguation belongs to the consumer, not the router. |
| Escape pressed while MAP is active | `map_close` fires; MAP context is popped. `pause` does NOT fire. | Escape priority rule: context-cancel takes precedence when an overlay is active. |
| Escape pressed while DIALOGUE is active | `dialogue_cancel` fires; DIALOGUE context is popped. `pause` does NOT fire. | Same Escape priority rule. |
| Escape pressed while EXPLORATION is active | `pause` fires (always-on). No context change. | No overlay to cancel; Escape falls through to pause. |
| Key remapped to a binding already in use by another action | `remap_action()` warns but allows the conflict. Duplicate bindings are resolved by Godot's `InputMap`: the first matching action in registry order fires. Log the conflict for Settings UI to display. | Preventing all conflicts would restrict valid remapping combinations. Warn, don't block. |
| `load_bindings()` called and `user://input_bindings.cfg` does not exist | Load silently fails; fall back to default `InputMap` bindings defined in Project Settings. No error logged. | First launch. Missing file is expected, not exceptional. |
| Input received during a scene transition (no active gameplay systems) | InputSystem accepts input normally. Context stack is unchanged. Systems that are not loaded cannot receive events. | InputSystem is an autoload; it is always active. Scene transitions are scene management's concern, not InputSystem's. |
| Controller connected mid-session (future milestone) | No change to current session behavior. Controller readiness is implemented at the binding level; hotplug handling deferred to controller milestone. | Out of scope for MVP. Document as known future work. |

## Dependencies

**Upstream Dependencies (systems A4 requires)**

None. The Input System is a Foundation layer system. It depends only on Godot's engine-level `InputMap` and `Input` APIs, which are always available.

---

**Downstream Dependents (systems that depend on A4)**

| System | Dependency Type | What They Require From A4 |
|--------|----------------|--------------------------|
| **A2 Character Controller** | Hard | `get_movement_vector()`, `is_action_pressed("sprint")`, `is_action_pressed("crouch")`, `is_action_pressed("lean_left")`, `is_action_pressed("lean_right")`. Owns `interact` / `enter_dialogue` disambiguation. |
| **E1 Camera System** | Hard | `get_look_delta()` every frame. Returns `Vector2.ZERO` in overlay contexts; E1 applies sensitivity/inversion. |
| **B7 Dialogue System** | Hard | `push_context(DIALOGUE)` / `pop_context()` for lifecycle. `dialogue_advance`, `dialogue_choice_1–4`, `dialogue_cancel` queries. |
| **C2 Fractured Reality Map** | Hard | `push_context(MAP)` / `pop_context()` for lifecycle. All MAP context action queries. `context_pushed` / `context_popped` signals. |
| **E8 Settings / Main Menu** | Hard | `remap_action()`, `reset_action_to_default()`, `save_bindings()` for keybinding UI. `pause` consumer queries via `is_action_just_pressed("pause")` (EXPLORATION-only). |
| **D3 Save/Load System** | Soft | Calls `load_bindings()` at startup. No ongoing dependency after loading. |

**Note**: Every system that reads player input is a downstream dependent of A4. The systems listed above are the direct consumers; B1 (Stealth), B5 (Investigation), and B8 (Social Manipulation) will also read input via A2 or directly from InputSystem once those systems are designed. Their Dependencies sections must reference A4.

## Tuning Knobs

The Input System has minimal tuning surface — it is a routing layer, not a gameplay system. Its only tuning knob at the GDD level is the default keybinding table. All other "feel" tuning (mouse sensitivity, deadzone, movement speed) belongs to downstream systems (A2, Camera System).

| Knob | Default | Safe Range | Too High | Too Low | Affects |
|------|---------|-----------|----------|---------|---------|
| Default keybinding layout | WASD + mouse + QECF + Shift/C/Tab/Space | Any valid `InputEvent` | N/A — no "too high" for bindings | N/A | Starting ergonomics; player can remap. Should follow PC conventions. |

**Mouse sensitivity and input scaling are owned by A2 (Character Controller) and E1 (Camera System)** — not the Input System. InputSystem returns raw mouse delta; scaling is a downstream concern.

**Controller analog deadzone** — deferred to controller milestone. Will be added as a tuning knob in `InputSystem` when gamepad support is implemented. Safe range: 0.05–0.25. Default: 0.15 (standard controller deadzone).

**No tuning knobs required for MVP** beyond the default binding table.

## Visual/Audio Requirements

The Input System has no visual or audio requirements. It emits signals; it does not render anything or play audio.

| Signal | Downstream Consumer | Required Response |
|--------|------------------|-------------------|
| `context_pushed(context)` | C2 Fractured Reality Map | Know when MAP context is active / inactive |
| `context_pushed(context)` | B7 Dialogue System | Know when DIALOGUE context is active / inactive |
| `context_popped(context)` | C2, B7 (same consumers) | Resume or suspend accordingly |
| `binding_changed(action, event)` | E8 Settings / Main Menu | Update keybinding display in UI |

No art or audio assets required.

## UI Requirements

The Input System has no in-game UI. Its only UI surface is the keybinding remapping screen in E8 (Settings / Main Menu).

**Data contract with E8:**
- `InputSystem.remap_action(action, event)` → E8 calls this when player confirms a new binding
- `InputSystem.reset_action_to_default(action)` → E8 calls this on "Reset to default" button
- `InputSystem.save_bindings()` → E8 calls this on Settings close
- E8 is responsible for displaying all bindable actions, their current bindings, and conflict warnings
- InputSystem provides the data; E8 owns the presentation

## Acceptance Criteria

All criteria verifiable via `InputSystem`'s public API and signal monitoring. No raw `Input.*` calls in tests.

| # | Criterion | Test Method | Pass Condition |
|---|-----------|------------|----------------|
| AC-A4-01 | Default context is EXPLORATION at startup | Call `InputSystem.get_active_context()` immediately after scene load. | Returns `InputContext.EXPLORATION`. |
| AC-A4-02 | `push_context(DIALOGUE)` transitions correctly | From EXPLORATION, call `push_context(DIALOGUE)`. Query `get_active_context()`. | Returns `InputContext.DIALOGUE`. Signal `context_pushed(DIALOGUE)` emitted. |
| AC-A4-03 | `pop_context()` returns to EXPLORATION | Push DIALOGUE, then call `pop_context()`. Query `get_active_context()`. | Returns `InputContext.EXPLORATION`. Signal `context_popped(DIALOGUE)` emitted. |
| AC-A4-04 | EXPLORATION actions suspended during overlay | Push MAP. Simulate W keypress. Call `is_action_pressed("move_forward")`. | Returns `false`. Action suppressed in non-EXPLORATION context. |
| AC-A4-05 | MAP context actions active during MAP | Push MAP. Simulate scroll-up input. Call `is_action_just_pressed("map_zoom_in")`. | Returns `true`. |
| AC-A4-06 | `pause` fires in EXPLORATION; does not fire when overlay is active | From EXPLORATION (no overlay), simulate Escape. Call `is_action_just_pressed("pause")`. Then push DIALOGUE, simulate Escape again, call `is_action_just_pressed("pause")`. | First call returns `true`. Second call returns `false` (EXPLORATION suspended by overlay). |
| AC-A4-07 | Escape fires context-cancel, not pause, during overlay | Push MAP. Simulate Escape. Call `is_action_just_pressed("map_close")` and `is_action_just_pressed("pause")`. | `map_close` returns `true`. `pause` returns `false`. Only context-cancel fires. |
| AC-A4-08 | Double push rejected | Push DIALOGUE. Attempt `push_context(MAP)`. Query `get_active_context()`. | Remains `InputContext.DIALOGUE`. Error logged. |
| AC-A4-09 | Pop of base context rejected | From EXPLORATION (no overlay), call `pop_context()`. Query `get_active_context()`. | Remains `InputContext.EXPLORATION`. Error logged. |
| AC-A4-10 | `get_movement_vector()` normalizes diagonal input | Simulate simultaneous W + D keypress. Call `get_movement_vector()`. | Returns `Vector2` with `length() ≈ 1.0` (within float tolerance). Not `Vector2(1.0, -1.0)`. |
| AC-A4-11 | `get_movement_vector()` returns zero outside EXPLORATION | Push MAP. Simulate W keypress. Call `get_movement_vector()`. | Returns `Vector2.ZERO`. |
| AC-A4-12 | Keybinding remapped and persisted | Call `remap_action("interact", new_event)`. Call `save_bindings()`. Reload `InputSystem`. Call `load_bindings()`. Simulate new binding. | `is_action_just_pressed("interact")` returns `true` for new binding. Original binding no longer triggers it. |
| AC-A4-13 | `reset_action_to_default()` restores original binding | Remap `crouch`. Call `reset_action_to_default("crouch")`. Simulate original C key. | `is_action_just_pressed("crouch")` returns `true` for C key. |
| AC-A4-14 | Missing bindings file falls back to defaults | Delete `user://input_bindings.cfg`. Call `load_bindings()`. Simulate default WASD. | No error. `get_movement_vector()` returns expected values for WASD. |

## Open Questions

| # | Question | Owner | Target Resolution | Notes |
|---|----------|-------|------------------|-------|
| OQ-A4-01 | Should `inspect_detail` (Hold F) and `enter_dialogue` (F near NPC) conflict? | Systems Designer + A2 GDD | Before A2 GDD is authored | Current spec: A2 owns disambiguation via NPC proximity check. But if the player is both near an NPC AND near an examinable object, what takes priority? Dialogue or investigation? Needs a priority rule in A2 GDD. |
| OQ-A4-02 | How does the Sprint action integrate with noise generation in B1 (Stealth)? | B1 GDD + Gameplay Programmer | Before B1 GDD is authored | Current spec: `is_action_pressed("sprint")` is readable by A2. The stealth noise cost of sprinting is B1's design concern. B1 GDD must define the noise multiplier when sprint is held and connect it to C1's `report_event(STEALTH_DETECTION)` path. |
| OQ-A4-03 | Controller support milestone: is this Vertical Slice or post-launch? | Producer | Sprint planning | Systems index says controller is a "later milestone." This needs a concrete target before E8 (Settings) is designed, as controller bindings affect the remapping UI scope. |
| OQ-A4-04 | Should dialogue choice selection support mouse hover + click in addition to number keys? | UX Designer + E4 (Dialogue UI) GDD | Before E4 GDD is authored | Number key selection (1–4) is defined. Mouse-based selection is a UX decision owned by E4. InputSystem currently has no `dialogue_hover` action. If mouse selection is added, InputSystem needs a new action or E4 handles mouse input directly via Godot UI nodes. |
| OQ-A4-05 | `map_pan` (LMB hold) and `map_node_inspect` (RMB / Hold LMB) both claim LMB hold in MAP context — how are they distinguished? | C2 Fractured Reality Map GDD author | Before C2 GDD is authored | Both actions are bound to LMB hold in the same context. C2 GDD must define the disambiguation protocol: does drag distance resolve the ambiguity (small drag = inspect, large drag = pan)? Does clicking on an empty area vs. a node determine intent? Or does `map_node_inspect` only apply to RMB (removing the LMB-hold binding)? InputSystem will expose both actions; C2 owns the resolution logic. |
