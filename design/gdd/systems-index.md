# Systems Index: TIME-PUNK

> **Status**: Approved
> **Created**: 2026-03-28
> **Last Updated**: 2026-03-28
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

TIME-PUNK requires 27 systems organized around three equal gameplay verbs (stealth, investigation, social manipulation) and one novel information layer (the Fractured Reality Map). Unlike most games, the central resource is not health or ammo — it is information trust. Every gameplay system either reads from or writes to the timeline data architecture and corruption model, making those two foundation systems the highest-priority design work in the project. The game has no combat system, no economy, and no progression tree — its mechanical depth comes entirely from the interaction between the three verbs and the map's corruption state. All missions are handcrafted; no procedural systems are required. The design order is determined strictly by dependency: data foundations first, novel mechanics second, conventional systems third, presentation last.

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | Timeline Data Architecture | Foundation | MVP | Designed | design/gdd/timeline-data-architecture.md | — |
| 2 | Corruption Data System | Foundation | MVP | Approved | design/gdd/corruption-data-system.md | A1 |
| 3 | Input System | Foundation | MVP | Approved | design/gdd/a4-input-system.md | — |
| 4 | Scene Management | Foundation | MVP | Approved | design/gdd/scene-management.md | — |
| 5 | Character Controller | Foundation | MVP | Not Started | — | A4 |
| 6 | NPC Psychology State Machine | Gameplay: Social | MVP | Not Started | — | A1 |
| 7 | Mission State System | Foundation | MVP | Not Started | — | A1, A4 (Scene Mgmt) |
| 8 | Evidence/Clue Data System | Gameplay: Investigation | MVP | Not Started | — | A1 |
| 9 | Fractured Reality Map (+ Codex) | Information | MVP | Not Started | — | A1, C1, D1 |
| 10 | Stealth System | Gameplay: Stealth | MVP | Not Started | — | A2 |
| 11 | AI Perception System | Gameplay: Stealth | MVP | Not Started | — | A2 |
| 12 | Dialogue System | Gameplay: Social | MVP | Not Started | — | B6 |
| 13 | Investigation/Deduction System | Gameplay: Investigation | MVP | Not Started | — | A1, B4 |
| 14 | Social Manipulation System | Gameplay: Social | MVP | Not Started | — | B6, B7, C1 |
| 15 | Guard Patrol System | Gameplay: Stealth | MVP | Not Started | — | A2, B2 |
| 16 | Anomaly Detection System | Information | MVP | Not Started | — | C1, C2 |
| 17 | NPC Counter-Deception System | Information | Vertical Slice | Not Started | — | B6, C1 |
| 18 | Timeline Repair Economy | Information | Vertical Slice | Not Started | — | C1, C2 |
| 19 | Save/Load System *(inferred)* | Persistence | MVP | Not Started | — | A1, A3 (soft), C1, D1 |
| 20 | Narrative System *(inferred)* | Narrative | Vertical Slice | Not Started | — | A1, D1 |
| 21 | Camera System *(inferred)* | UI | MVP | Not Started | — | A2, B1 |
| 22 | Timeline Map UI | UI | MVP | Not Started | — | C2 |
| 23 | HUD *(inferred)* | UI | MVP | Not Started | — | B1, D1 |
| 24 | Dialogue UI *(inferred)* | UI | MVP | Not Started | — | B6, B7 |
| 25 | Post-Mission Analysis UI | UI | MVP | Not Started | — | C2, C3 |
| 26 | Adaptive Audio System *(inferred)* | Audio | Vertical Slice | Not Started | — | C1, D1 |
| 27 | Settings / Main Menu *(inferred)* | Meta | Alpha | Not Started | — | A3 (Scene), D3 |

*Systems marked (inferred) were not explicitly named in the concept doc but are required by explicit systems.*

---

## Categories

| Category | Description | Systems |
|----------|-------------|---------|
| **Foundation** | Data models and engine abstractions everything else writes to | A1, A2, A3, A4, D1 |
| **Gameplay: Stealth** | Detection, patrol, cover, movement | B1, B2, B3 |
| **Gameplay: Investigation** | Evidence, deduction, clue data | B4, B5 |
| **Gameplay: Social** | Dialogue, NPC psychology, manipulation | B6, B7, B8 |
| **Information** | Fractured Reality Map, corruption, anomaly, repair, counter-deception | C1, C2, C3, C4, C5 |
| **Narrative** | Story progression, mission story beats | D2 |
| **Persistence** | Save/load and state continuity | D3 |
| **UI** | All player-facing information surfaces | E1, E2, E3, E4, E5 |
| **Audio** | Adaptive music and SFX | E6 |
| **Meta** | Settings, menus, session management | E8 |

---

## Priority Tiers

| Tier | Definition | Systems Count |
|------|------------|---------------|
| **MVP** | Required for prologue + 1 timeline mission to be playable and testable | 22 |
| **Vertical Slice** | Adds dual-source corruption, repair economy, adaptive audio, narrative system | 4 |
| **Alpha** | Settings and main menu | 1 |
| **Full Vision** | Content complete (missions 5–7); no new systems required | — |

---

## Dependency Map

### Foundation Layer *(no dependencies — design first)*

1. **A1 Timeline Data Architecture** — the data model every other system reads from and writes to; the wrong schema here cascades into every system
2. **A3 Scene Management** — environment loading/unloading; required before any mission can run
3. **A4 Input System** — action abstraction layer; required before the character can be controlled

### Core Layer *(depends on Foundation)*

4. **A2 Character Controller** — depends on: A4 (Input System)
5. **B4 Evidence/Clue Data System** — depends on: A1
6. **B6 NPC Psychology State Machine** — depends on: A1
7. **C1 Corruption Data System** — depends on: A1 *(second bottleneck; 7 systems depend on this)*
8. **D1 Mission State System** — depends on: A1, A3

### Feature Layer *(depends on Core)*

9. **C2 Fractured Reality Map (+ Codex)** — depends on: A1, C1, D1 *(the game's primary feedback surface and codex)*
10. **B1 Stealth System** — depends on: A2
11. **B2 AI Perception System** — depends on: A2
12. **B5 Investigation/Deduction System** — depends on: A1, B4
13. **B7 Dialogue System** — depends on: B6
14. **B8 Social Manipulation System** — depends on: B6, B7, C1
15. **C5 NPC Counter-Deception System** — depends on: B6, C1
16. **D2 Narrative System** — depends on: A1, D1
17. **D3 Save/Load System** — depends on: A1, C1, D1
18. **E6 Adaptive Audio System** — depends on: C1, D1

### Presentation Layer *(depends on Feature)*

19. **B3 Guard Patrol System** — depends on: A2, B2
20. **B8 Social Manipulation** already in Feature Layer above
21. **C3 Anomaly Detection System** — depends on: C1, C2
22. **C4 Timeline Repair Economy** — depends on: C1, C2
23. **E1 Camera System** — depends on: A2, B1
24. **E2 Timeline Map UI** — depends on: C2
25. **E3 HUD** — depends on: B1, D1
26. **E4 Dialogue UI** — depends on: B6, B7
27. **E8 Settings / Main Menu** — depends on: A3, D3

### Polish Layer *(deepest)*

28. **E5 Post-Mission Analysis UI** — depends on: C2, C3

---

## Recommended Design Order

Systems at the same layer with no cross-dependencies can be designed in parallel. Design order combines dependency layer + MVP priority.

| Order | System | Priority | Layer | Lead Agent(s) | Est. Effort |
|-------|--------|----------|-------|--------------|-------------|
| 1 | **A1 Timeline Data Architecture** | MVP | Foundation | `systems-designer` + `lead-programmer` | L |
| 2 | **C1 Corruption Data System** | MVP | Foundation | `systems-designer` + `lead-programmer` | L |
| 3 | **A4 Input System** | MVP | Foundation | `gameplay-programmer` | S |
| 4 | **A3 Scene Management** | MVP | Foundation | `engine-programmer` | S |
| 5 | **A2 Character Controller** | MVP | Core | `gameplay-programmer` | M |
| 6 | **B6 NPC Psychology State Machine** | MVP | Core | `systems-designer` + `ai-programmer` | L |
| 7 | **D1 Mission State System** | MVP | Core | `systems-designer` | M |
| 8 | **B4 Evidence/Clue Data System** | MVP | Core | `systems-designer` | M |
| 9 | **C2 Fractured Reality Map (+ Codex)** | MVP | Feature | `systems-designer` + `ux-designer` | L |
| 10 | **B1 Stealth System** | MVP | Feature | `game-designer` + `gameplay-programmer` | M |
| 11 | **B2 AI Perception System** | MVP | Feature | `ai-programmer` | L |
| 12 | **B7 Dialogue System** | MVP | Feature | `systems-designer` + `narrative-director` | M |
| 13 | **B5 Investigation/Deduction System** | MVP | Feature | `systems-designer` + `ux-designer` | M |
| 14 | **B8 Social Manipulation System** | MVP | Feature | `systems-designer` + `narrative-director` | L |
| 15 | **B3 Guard Patrol System** | MVP | Presentation | `ai-programmer` | M |
| 16 | **C3 Anomaly Detection System** | MVP | Feature | `systems-designer` | M |
| 17 | **C5 NPC Counter-Deception System** | Vertical Slice | Feature | `systems-designer` + `ai-programmer` | M |
| 18 | **C4 Timeline Repair Economy** | Vertical Slice | Feature | `economy-designer` + `systems-designer` | M |
| 19 | **D3 Save/Load System** | MVP | Feature | `gameplay-programmer` | M |
| 20 | **D2 Narrative System** | Vertical Slice | Feature | `narrative-director` | L |
| 21 | **E1 Camera System** | MVP | Presentation | `engine-programmer` | S |
| 22 | **E2 Timeline Map UI** | MVP | Presentation | `ux-designer` + `ui-programmer` | L |
| 23 | **E3 HUD** | MVP | Presentation | `ui-programmer` | S |
| 24 | **E4 Dialogue UI** | MVP | Presentation | `ux-designer` + `ui-programmer` | M |
| 25 | **E5 Post-Mission Analysis UI** | MVP | Polish | `ux-designer` + `ui-programmer` | M |
| 26 | **E6 Adaptive Audio System** | Vertical Slice | Presentation | `audio-director` + `sound-designer` | M |
| 27 | **E8 Settings / Main Menu** | Alpha | Meta | `ui-programmer` | S |

*Effort: S = 1 session, M = 2–3 sessions, L = 4+ sessions. A session = one focused design conversation producing a complete GDD.*

---

## Circular Dependencies

None detected. The dependency graph is a directed acyclic graph (DAG). All information flows in one direction: Foundation → Core → Feature → Presentation → Polish.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| **A1 Timeline Data Architecture** | Design + Technical | The data schema must accommodate: node states, edge relationships, corruption tags (self vs. external), probability values, ghost references, and timeline branching — all queryable in real time. A wrong model requires rewriting every dependent system. | Design before any other system. Run `/architecture-decision` for the schema. Prototype with static data before implementing live writes. |
| **C1 Corruption Data System** | Design | Dual-source corruption requires a tagging model that can distinguish self-inflicted erosion from NPC counter-deception while remaining opaque to the player initially. If the attribution model is wrong, the core skill loop cannot be taught. | Design immediately after A1. The distinction mechanism must be defined before B6, C2, or C5 can be designed. |
| **C2 Fractured Reality Map** | Design + Technical | Novel UI/UX with no direct template in any reference game. Three distinct states (fog, live ripple, analysis) on one surface. Must feel readable under corruption without revealing too much. Highest UX design risk in the project. | Prototype the visualization before writing the GDD. The open question "how does the player learn to distinguish corruption sources?" must be answered through UX testing, not design documents alone. |
| **B6 NPC Psychology State Machine** | Design + Technical | Bidirectional social interaction (player reads NPCs, NPCs read player back) with deception potential as a design spec is a novel mechanic with no direct Godot template. The state machine must drive dialogue choices, tell visibility, counter-deception triggers, and corruption output simultaneously. | Design the state model in GDD before any dialogue or manipulation system is touched. Prototype one NPC end-to-end before speccing the full ecosystem. |
| **B2 AI Perception System** | Technical | Stealth AI in Godot 4.6 using Jolt Physics and NavigationAgent3D requires bespoke perception cones, hearing radius, and alert state escalation. No existing Godot 4.6 template covers this configuration. | Prototype early and independently of GDD completion. Block the Stealth System GDD until perception prototype validates the approach. |
| **E2 Timeline Map UI** | Technical | Custom Godot UI system with dynamic fog rendering, real-time node ripple animation, and dual-source corruption visualization. No direct Godot Control/CanvasItem template exists. | Prototype the visual rendering before writing the full GDD. Treat this as a shader + custom Control system design problem. |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 27 |
| Design docs started | 0 |
| Design docs in review | 0 |
| Design docs approved | 4 |
| Design docs designed (pending review) | 0 |
| MVP systems designed | 4 / 22 |
| Vertical Slice systems designed | 0 / 4 |
| Alpha systems designed | 0 / 1 |

---

## Next Steps

- [ ] Design **A1 Timeline Data Architecture** first — run `/design-system timeline-data-architecture`
- [ ] Design **C1 Corruption Data System** second — run `/design-system corruption-data-system`
- [ ] Run `/architecture-decision` for the timeline node schema before any code is written
- [ ] Prototype **C2 Fractured Reality Map** visualization early — run `/prototype fractured-reality-map` to answer the open UX question before writing the GDD
- [ ] Prototype **B2 AI Perception System** early — run `/prototype stealth-ai-perception` to validate the Godot 4.6 approach
- [ ] After 5+ MVP GDDs are complete, run `/gate-check pre-production` to assess build readiness
- [ ] When ready to implement, run `/sprint-plan new` to plan the first development sprint
