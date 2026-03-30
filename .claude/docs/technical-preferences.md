# Technical Preferences

<!-- Populated by /setup-engine. Updated as the user makes decisions throughout development. -->
<!-- All agents reference this file for project-specific standards and conventions. -->

## Engine & Language

- **Engine**: Godot 4.6.1
- **Language**: GDScript (primary), C++ via GDExtension (performance-critical systems only)
- **Rendering**: Forward+ (3D default; D3D12 backend on Windows as of 4.6)
- **Physics**: Jolt Physics 3D (default as of Godot 4.6 — replaces GodotPhysics3D as default)

## Naming Conventions

- **Classes**: PascalCase (e.g., `PlayerController`, `FracturedRealityMap`)
- **Variables/functions**: snake_case (e.g., `move_speed`, `take_damage()`)
- **Signals/Events**: snake_case past tense (e.g., `health_changed`, `node_corrupted`, `timeline_updated`)
- **Files**: snake_case matching class name (e.g., `player_controller.gd`, `fractured_reality_map.gd`)
- **Scenes/Prefabs**: PascalCase matching root node (e.g., `PlayerController.tscn`, `TimelineMap.tscn`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_CORRUPTION`, `TIMELINE_NODE_COUNT`)
- **Enums**: PascalCase name, UPPER_SNAKE_CASE values (e.g., `enum CorruptionSource { SELF_INFLICTED, EXTERNAL_DECEPTION }`)
- **Private members**: Prefix with `_` (e.g., `_corruption_level`, `_npc_psychology_state`)

## Performance Budgets

- **Target Framerate**: [TO BE CONFIGURED — recommend 60fps for stealth/investigation genre]
- **Frame Budget**: [TO BE CONFIGURED — 16.6ms at 60fps]
- **Draw Calls**: [TO BE CONFIGURED — establish after first scene prototype]
- **Memory Ceiling**: [TO BE CONFIGURED — establish after asset pipeline is scoped]

## Testing

- **Framework**: [TO BE CONFIGURED — GUT (Godot Unit Testing) recommended for GDScript]
- **Minimum Coverage**: [TO BE CONFIGURED]
- **Required Tests**: Timeline map data integrity, corruption source attribution, NPC psychology state transitions, navigation mesh validity

## Forbidden Patterns

<!-- Add patterns that should never appear in this project's codebase -->
- `$"Long/Node/Path"` in `_process()` or `_physics_process()` — cache with `@onready var` instead
- Untyped `Array` or `Dictionary` in hot paths — use `Array[Type]` and typed vars
- String-based `connect("signal_name", ...)` — use typed callable connections
- `duplicate()` for nested resources — use `duplicate_deep()` explicitly
- `Texture2D` as shader uniform hint — use `Texture` base type (changed in 4.4)
- Hardcoded gameplay values in GDScript — all tuning knobs must be `@export` or data-driven

## Allowed Libraries / Addons

<!-- Add approved third-party dependencies here -->
- [None configured yet — add as dependencies are approved]

## Architecture Decisions Log

<!-- Quick reference linking to full ADRs in docs/architecture/ -->
- [No ADRs yet — use /architecture-decision to create one]
- Priority ADR: Timeline map data architecture (highest technical risk for TIME-PUNK)
