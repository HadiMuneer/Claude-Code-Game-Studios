# Scene Management (A3)

> **Status**: Approved
> **Author**: Human + Claude Code
> **Last Updated**: 2026-03-30
> **Implements Pillar**: Infrastructure (enables all pillars)

## Overview

The Scene Management System (A3) is the engine-level loading and transition utility that every system uses to change the active scene in TIME-PUNK. It wraps Godot's `SceneTree.change_scene_to_file()` and `ResourceLoader` in a typed GDScript autoload singleton (`SceneManager`) that provides: asynchronous scene loading with progress tracking, configurable transition animations (fade, cut), a scene registry mapping logical scene identifiers to file paths, and loading-error handling. A3 has no knowledge of mission phase logic, game state, or narrative context — it does exactly one thing: given a logical scene identifier, replace the current scene with the target scene.

TIME-PUNK requires A3 because every session-phase transition (pre-mission map → gameplay scene → post-mission analysis) and every meta-navigation (main menu → scene, game over → main menu) must go through a single controlled loading path. Without A3, scene changes would be scattered across multiple systems with inconsistent transition behavior, no centralized error handling, and no way to guarantee loading state coherence. A3 is invisible to the player — its success condition is transitions that feel intentional and load times that never break the noir atmosphere.

## Player Fantasy

The Scene Management System has no player fantasy of its own — it is infrastructure that the player never consciously perceives. Its success condition is invisibility: scene transitions that feel deliberate, weighted, and tonally consistent without exposing the mechanical act of loading. A jarring white flash or instant cut between the post-mission analysis and the main menu would break Pillar 5 (The Weight Stays). A3's job is to make the seams disappear.

For the developer, A3's fantasy is **reliability and centralization**: one function to call, one place to debug, one place to define transitions. Adding a new scene to TIME-PUNK should require one line in the scene registry — not hunting through five scripts to find where scene loading lives.

## Detailed Design

### Core Rules

**Rule 1 — Scene Registry**

All scenes are registered as `StringName` constants in a dictionary in `SceneManager.gd`. No system passes file paths directly to `SceneManager` — they pass scene IDs.

```gdscript
const SCENES: Dictionary = {
    &"main_menu":             "res://src/ui/scenes/main_menu.tscn",
    &"mission_prologue":      "res://scenes/missions/prologue.tscn",
    &"mission_01":            "res://scenes/missions/mission_01.tscn",
    &"mission_02":            "res://scenes/missions/mission_02.tscn",
    &"mission_03":            "res://scenes/missions/mission_03.tscn",
    &"mission_04":            "res://scenes/missions/mission_04.tscn",
    &"mission_05":            "res://scenes/missions/mission_05.tscn",
    &"mission_06":            "res://scenes/missions/mission_06.tscn",
    &"mission_07":            "res://scenes/missions/mission_07.tscn",
    &"pre_mission_map":       "res://src/ui/scenes/pre_mission_map.tscn",
    &"post_mission_analysis": "res://src/ui/scenes/post_mission_analysis.tscn",
}
```

Adding a new scene requires one dictionary entry. No other system changes.

---

**Rule 2 — Loading Strategy: Async Threaded**

All scene loads use `ResourceLoader.load_threaded_request()`. Loading and fade-out begin **concurrently** — the load request is issued at the moment `load_scene()` is called, not after fade-out completes. This hides as much load time as possible behind the fade animation.

Full sequence:

1. `load_scene(scene_id, transition)` is called — A3 enters TRANSITIONING state
2. A3 simultaneously: (a) plays the **fade-out** phase AND (b) calls `ResourceLoader.load_threaded_request(path)` — both begin on the same frame
3. A3 polls `ResourceLoader.load_threaded_get_status()` each `_process()` frame
4. A3 waits until BOTH conditions are true: fade-out is complete AND `THREAD_LOAD_LOADED` status returned
5. A3 calls `get_tree().change_scene_to_packed(packed_scene)` to swap the scene
6. A3 plays the **fade-in** phase
7. A3 emits `scene_loaded(scene_id)` — A3 returns to IDLE state

The scene swap always occurs while the screen is fully faded — never visible to the player. If the scene loads faster than the fade-out, SceneManager simply holds the opaque screen until fade-out completes. If the scene loads slower, the screen stays opaque after fade-out until loading finishes.

*Implementation note: The fade overlay is a `ColorRect` child of a permanent `CanvasLayer` autoload node. `fade_out_duration` and `fade_in_duration` are applied via `Tween`. The `CanvasLayer` persists across scene changes — it is not owned by the loaded scene.*

---

**Rule 3 — Transition Types**

```gdscript
enum TransitionType {
    FADE_BLACK,   ## default — slow fade to black; out before load, in after swap
    FADE_WHITE,   ## time-travel effect — overexposed flash; same timing as FADE_BLACK
    CUT,          ## instant — no animation; scene swaps in one frame
}
```

Callers specify the transition as a parameter. Default is `FADE_BLACK`.

| Transition | Use Case | Duration |
|---|---|---|
| `FADE_BLACK` | All mission-to-mission, mission-to-analysis, mission-to-menu transitions | `fade_duration` (tuning knob, default 0.4s per direction) |
| `FADE_WHITE` | Timeline entry points — the flash of arriving in a new era | Same as `FADE_BLACK` duration |
| `CUT` | Fast meta navigation where a fade would feel padded | 0s (instant) |

---

**Rule 4 — `SceneManager` Singleton API**

```gdscript
## SceneManager — autoload singleton (res://src/core/scene_manager.gd)
## The only valid way to change scenes in TIME-PUNK.
## Never call get_tree().change_scene_to_file() directly in gameplay code.

## Load a scene by registered ID
func load_scene(scene_id: StringName, transition: TransitionType = TransitionType.FADE_BLACK) -> void

## Query current state
func get_current_scene_id() -> StringName
func is_loading() -> bool   # true while TRANSITIONING

## Signals
signal scene_load_started(scene_id: StringName)
signal scene_load_progress(progress: float)   # 0.0–1.0; useful for loading screen if added later
signal scene_loaded(scene_id: StringName)
signal scene_load_failed(scene_id: StringName, error: String)
```

### States and Transitions

| State | Entry Condition | Exit Condition | Behavior |
|-------|----------------|----------------|----------|
| `IDLE` | Game start; scene load complete | `load_scene()` called | Accepts calls; `is_loading()` returns `false` |
| `TRANSITIONING` | `load_scene()` called | Scene swap complete + fade-in done | Rejects new `load_scene()` calls (logs error). Plays transition, loads async, swaps, fades in. |

**Invalid call: `load_scene()` while already TRANSITIONING** — logged as error, ignored. The caller must wait for `scene_loaded` signal before requesting another load.

### Interactions with Other Systems

| System | Direction | Interface | Notes |
|--------|-----------|-----------|-------|
| **D1 Mission State System** | Calls SceneManager | `load_scene()` to trigger mission start, post-mission, menu return. Listens to `scene_loaded` to know when the new scene is ready before initializing mission state. | D1 is the primary orchestrator of which scene loads when. A3 does not know why D1 is loading a scene. |
| **E8 Settings / Main Menu** | Calls SceneManager | `load_scene("mission_prologue")` for New Game. `load_scene("main_menu")` for quit-to-menu. | E8 drives the top-level game navigation from the menu. |
| **A4 Input System** | Passive | No direct calls. Input context stack is not modified by scene loads. | On scene load, the previous context stack persists. D1 is responsible for calling `InputSystem.pop_context()` if an overlay was active when a scene change was triggered. |
| **D3 Save/Load System** | Listens to SceneManager | Subscribes to `scene_loaded` to trigger post-load state restoration (e.g., restore mission progress after loading a save). | D3 needs to know when the new scene's nodes are ready before it can hydrate them with saved state. |

## Formulas

The Scene Management System contains no gameplay math. Its only quantified behavior is transition timing.

### Formula 1: Transition Duration

For `FADE_BLACK` and `FADE_WHITE` (concurrent fade + load):

```
total_transition_time = max(fade_out_duration, load_time) + fade_in_duration
```

| Variable | Type | Range | Description |
|---|---|---|---|
| `fade_out_duration` | float | 0.2s–1.0s | Time to fade screen to black/white. Tuning knob. Default: 0.4s |
| `load_time` | float | 0.0s–∞ | Async load time; hardware-dependent. Runs concurrently with fade-out. |
| `fade_in_duration` | float | 0.2s–1.0s | Time to fade screen back to gameplay. Tuning knob. Default: 0.4s |

**Expected output examples:**
- Fast hardware, load_time = 0.1s: `max(0.4, 0.1) + 0.4 = 0.8s` (fade fully governs)
- Slow hardware, load_time = 2.0s: `max(0.4, 2.0) + 0.4 = 2.4s` (load governs; held black screen after fade)

`fade_out_duration` and `fade_in_duration` are controlled. `load_time` is hardware-dependent. No timeout is applied — SceneManager waits indefinitely for the load to finish.

### Formula 2: CUT Transition Duration

```
total_transition_time (CUT) = load_time   (animation overhead = 0)
```

`CUT` eliminates fade animation entirely but **still uses async threaded loading** (`ResourceLoader.load_threaded_request()`). It is never a synchronous blocking call. The scene swap occurs as soon as loading completes — on fast hardware this may appear instantaneous; on slow hardware the player sees a brief single-frame flash or held frame during the load.

*Note: If a truly zero-wait scene change is ever needed (e.g., an already-preloaded scene), that is a future API extension, not the current behavior of `CUT`.*

## Edge Cases

| Scenario | Expected Behavior | Rationale |
|----------|------------------|-----------|
| `get_current_scene_id()` called before any `load_scene()` has completed | Returns `&""` (empty StringName). | At game startup, no scene has been loaded via SceneManager — the boot scene was loaded directly by Godot, not through the registry. Callers that check scene identity at startup must handle the empty-string case. |
| `load_scene()` called while already TRANSITIONING | Log error; ignore the call. Current transition continues uninterrupted. | Double-loading is a caller bug. The caller should await the `scene_loaded` signal before queuing the next load. |
| `load_scene()` called with an unregistered scene ID | Log error with the unknown ID; do not attempt to load. Emit `scene_load_failed(scene_id, "Unregistered scene ID")`. | An unregistered ID is a code error, not a runtime condition. Fail loudly so it is caught in development. |
| `ResourceLoader.load_threaded_get_status()` returns `THREAD_LOAD_FAILED` | Emit `scene_load_failed(scene_id, error_string)`. SceneManager returns to IDLE. Screen fades back in to display the previous scene. | A failed load must not leave the game on a black screen. The previous scene remains active as fallback. |
| Load completes before fade-out animation finishes | SceneManager waits for fade-out to complete before swapping the scene, even if the load is already done. | Never swap mid-fade — the player must see a fully opaque screen before the scene changes. |
| `load_scene()` called during scene initialization (from `_ready()` of a node in the newly loaded scene) | Log a warning; defer the call one frame (via `call_deferred`). | Calling `change_scene_to_packed()` during `_ready()` can crash the SceneTree. Deferred call prevents this. |
| `FADE_WHITE` transition with HDR/bloom active on the 3D scene | No special handling — the fade overlay is a `CanvasLayer` drawn on top of the 3D scene. Bloom/glow effects on the 3D scene do not affect the flat overlay. | `CanvasLayer` renders after post-processing; no interaction. |
| Minimum-spec hardware: load time exceeds 10 seconds | No timeout. SceneManager waits indefinitely for the load to complete. | A timeout requires a fallback scene decision. Waiting is simpler and correct — player sees a held black screen, not a crash. Add a timeout in a future revision if performance budgets identify this as a UX problem. |

## Dependencies

**Upstream Dependencies**

None. A3 depends only on Godot engine systems: `ResourceLoader`, `SceneTree`, and `CanvasLayer` — all always available. No game-authored system is a prerequisite.

---

**Downstream Dependents**

| System | Dependency Type | What They Require From A3 |
|--------|----------------|--------------------------|
| **D1 Mission State System** | Hard | `load_scene()` to transition between mission phases. `scene_loaded` signal to know when the new scene is ready. `get_current_scene_id()` to verify current scene. |
| **E8 Settings / Main Menu** | Hard | `load_scene()` for New Game and quit-to-menu navigation. |
| **D3 Save/Load System** | Soft | `scene_loaded` signal — subscribes to trigger post-load state restoration. D3 does not call `load_scene()` directly; it reacts to scene changes initiated by D1/E8. |

*Note: D3's soft dependency on A3 is not currently reflected in the systems index (index lists D3 depends on A1, C1, D1 only). The systems index should be updated to add A3 as a soft dependency for D3.*

**Bidirectionality obligation**: When D1 (Mission State System) and E8 (Settings / Main Menu) GDDs are authored, their Dependencies sections must list A3 as a hard dependency. When D3 (Save/Load System) GDD is authored, it must list A3 as a soft dependency. Verify at design-review time for each of those systems.

## Tuning Knobs

| Knob | Default | Safe Range | Too High | Too Low | Affects |
|------|---------|-----------|----------|---------|---------|
| `fade_out_duration` | 0.4s | 0.2s–1.0s | Transitions feel sluggish; player waits too long on a black screen | Transition feels abrupt; jarring despite the fade being present | Tonal weight of scene exit; noir gravity |
| `fade_in_duration` | 0.4s | 0.2s–1.0s | New scene reveal is too slow; momentum loss | Scene pops in too fast; player hasn't mentally reset | Tonal weight of scene entry; first impression of new scene |

*`fade_out_duration` and `fade_in_duration` are independent knobs and may be set asymmetrically. A slow fade-out (0.6s) with a faster fade-in (0.3s) creates a "heavy exit, decisive arrival" feel appropriate for timeline mission entry.*

*No other tuning knobs. Scene count, file paths, and transition type assignments are design decisions expressed in code, not live-tunable values.*

## Visual/Audio Requirements

N/A — A3 is infrastructure with no player-facing presentation. The fade overlay (`ColorRect` on a `CanvasLayer`) is an implementation detail, not an art asset. Transition color (black vs. white) is determined by the caller's `TransitionType` parameter. No audio events are emitted by A3.

## UI Requirements

N/A — A3 has no in-game UI. The only user-facing surface is the transition itself (a full-screen fade), which has no interactive elements and requires no HUD or screen text. Transition timing is configurable via tuning knobs by a designer, not through a settings screen.

## Acceptance Criteria

All criteria verifiable via `SceneManager`'s public API and signal monitoring. No raw `change_scene_to_file()` calls in tests.

| # | Criterion | Test Method | Pass Condition |
|---|-----------|------------|----------------|
| AC-A3-01 | Default state at startup is IDLE | Call `SceneManager.is_loading()` immediately after game start. | Returns `false`. |
| AC-A3-02 | `load_scene()` with valid ID triggers TRANSITIONING state | Call `load_scene("main_menu")`. Immediately call `is_loading()`. | Returns `true`. Signal `scene_load_started("main_menu")` emitted. |
| AC-A3-03 | Scene swap completes and `scene_loaded` emitted | Call `load_scene("main_menu")`. Await `scene_loaded`. Call `get_current_scene_id()`. | Returns `&"main_menu"`. `is_loading()` returns `false`. |
| AC-A3-04 | Double `load_scene()` call rejected | Call `load_scene("mission_01")`. Immediately call `load_scene("mission_02")`. Await first `scene_loaded`. Call `get_current_scene_id()`. | `mission_01` loaded. `mission_02` rejected (error logged). `get_current_scene_id()` returns `&"mission_01"`. |
| AC-A3-05 | Unregistered scene ID emits failure signal | Call `load_scene(&"nonexistent_scene")`. | `scene_load_failed("nonexistent_scene", ...)` emitted. `is_loading()` returns `false`. No crash. |
| AC-A3-06 | `FADE_BLACK` transition: screen fully opaque before scene swap | Instrument SceneManager to record the frame when the scene swap occurs. Run `load_scene("mission_01", FADE_BLACK)`. | Scene swap does not occur until fade overlay alpha = 1.0. |
| AC-A3-07 | `CUT` transition: scene swap completes in one frame | Call `load_scene("main_menu", CUT)`. Count frames until `scene_loaded`. | `scene_loaded` emitted within 1–2 frames (load time only; no animation delay). |
| AC-A3-08 | `FADE_WHITE` transition uses white overlay | Call `load_scene("mission_01", FADE_WHITE)`. Monitor transition overlay color. | Overlay modulate is `Color(1,1,1,1)` at peak, not `Color(0,0,0,1)`. |
| AC-A3-09 | `scene_load_progress` emits valid range | Call `load_scene("mission_01", FADE_BLACK)`. Collect all `scene_load_progress` emissions. | All values in [0.0, 1.0]. At least one value between 0 and 1 (progress tracked, not just 0 then 1). |
| AC-A3-10 | `get_current_scene_id()` tracks each transition | Load `"mission_01"`, then `"post_mission_analysis"`, then `"main_menu"`. Query after each `scene_loaded`. | Returns the expected ID after each transition. |
| AC-A3-11 | No raw scene-change calls outside SceneManager | Grep `src/` and `scenes/` for `change_scene_to_file` and `change_scene_to_packed`. | Zero matches outside of `scene_manager.gd`. |
| AC-A3-12 | All registered scene IDs resolve to valid file paths | Add a debug validation pass in `SceneManager._ready()` checking `FileAccess.file_exists()` for each SCENES entry (stripped in release builds). | Zero missing paths reported on project load. |

## Open Questions

| # | Question | Owner | Target Resolution | Notes |
|---|----------|-------|------------------|-------|
| OQ-A3-01 | Should A3 support a loading screen scene for very long loads (e.g., minimum-spec hardware)? | Engine Programmer + D1 GDD author | Before D1 GDD is authored | Current spec: screen stays black during load with no feedback. If mission load times exceed ~3 seconds on minimum-spec PC, a loading screen (even a simple animated logo) would prevent player confusion. Requires A3 to either (a) embed a loading screen overlay as a CanvasLayer, or (b) load a lightweight loading-screen scene before the heavy mission scene. D1's mission flow design should drive this decision. |
| OQ-A3-02 | Are `pre_mission_map` and `post_mission_analysis` standalone scenes or overlays within the mission scene? | D1 GDD author + E2 (Timeline Map UI) GDD author | Before D1 GDD is authored | Current spec assumes they are separate scenes that A3 loads/unloads. If they are overlay CanvasLayers within the mission scene (so the 3D world stays loaded in the background), A3 does not need to manage them — they would be UI show/hide operations owned by D1 or C2. This decision has significant implications for A3's scene list and for memory usage. |
| OQ-A3-03 | Should mission scenes be broken into sub-scenes loaded additively (e.g., a base level + NPC layer + events layer)? | Engine Programmer | Before mission scene prototyping begins | Additive scene loading (`load_scene` with `add_to_current_scene`) could reduce load times and allow hot-swapping NPC/event layers. Current spec assumes each mission is a single monolithic .tscn. If additive loading is needed, A3's API and state machine would require extension. Defer until first mission scene prototype reveals actual load time data. |
