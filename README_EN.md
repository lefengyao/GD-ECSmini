# ECSmini

[中文](README.md) | **English**

**A lightweight, pure-GDScript entity-component-system (ECS) runtime designed for Godot 4.**

ECSmini splits game logic into data (components) plus behavior (systems), organizing entities
through composition instead of inheritance. The world (`EcsWorld`) is a pure logic object fully
decoupled from the scene tree — it can tick independently, be unit-tested in isolation, or run on
headless servers. When visuals are needed, an optional presentation layer binds entities to
`Node2D` instances.

- Current version: `0.1.0` (matches `ECSmini.VERSION`)
- Requirements: Godot 4.x (uses typed arrays, `StringName`, etc.; 4.2+ recommended)
- Language: GDScript (no C# / GDExtension needed)

---

## Features

- 🧩 **Minimal mental model**: an entity is just an `int`; components are arbitrary values attached under `StringName` keys; systems are plain GDScript classes.
- ⚡ **Efficient queries**: sparse-set component storage, condition-cached queries, and "driver set" pruning.
- 🛡️ **Iteration safety**: structural changes made during ticks or query callbacks are queued automatically and applied at safe boundaries; value writes take effect immediately.
- 🔌 **Decoupled from the scene tree**: `EcsWorld` extends `RefCounted` and knows nothing about nodes; the `EcsRunner` node hooks the world into `_process` / `_physics_process` with one line.
- 🎨 **Optional 2D presentation layer**: `EcsNodeSync` maintains a bidirectional entity↔node binding table and `EcsTransformSyncSystem` mirrors transforms onto bound nodes. Skip the whole layer when there is nothing to render.
- 📦 **Zero-intrusion install**: single plugin folder, no editor panels or custom inspectors.

## When to use it

| Good fit | Poor fit |
| --- | --- |
| 2D games with hundreds to thousands of active entities | Massive simulations (100k+ entities) — prefer a native C++ ECS |
| Projects separating logic from presentation, or needing deterministic simulation/replay | Out-of-the-box 3D projects (you can extend an integration layer modeled on `godot_2d`) |
| Logic that should be unit-testable outside the scene tree | |

---

## Installation

1. Copy the whole `addons/ecsmini` folder into your project's `addons/` directory:
   ```
   your_project/
   └── addons/
       └── ecsmini/
   ```
2. In the editor, open **Project → Project Settings → Plugins** and enable **ECSmini**.

> **Note**: the current plugin entry script performs no editor-side registration. All runtime
> classes are globally available through their `class_name` declarations — scripts work even
> without enabling the plugin. Enabling it simply follows convention and prepares for future
> editor-side features.

## Quick start in 30 seconds

The example below does not touch the scene tree at all — run it from any `_ready()`:

```gdscript
var world := ECSmini.create_world()          # equivalent to EcsWorld.new()

# An entity = an int; a component = any value under a StringName key
var player := world.create_entity()
world.add_component(player, &"position", Vector2.ZERO)
world.add_component(player, &"velocity", Vector2.RIGHT * 120.0)

# Register a built-in system: position += velocity * delta
world.add_system(EcsMovementSystem.new())

for i in 60:
    world.tick(1.0 / 60.0)                   # advance 60 frames

print(world.get_component(player, &"position"))   # about (120, 0)
```

## Hooking into the Godot scene tree (recommended)

In a real game, host the world with an `EcsRunner` node so Godot drives the simulation every frame:

```gdscript
extends Node2D   # any script on a scene root


func _ready() -> void:
    var runner := EcsRunner.new()
    runner.use_physics_process = true        # fixed physics step; set BEFORE add_child!
    add_child(runner)                        # entering the tree calls world.start()

    runner.add_system(EcsMovementSystem.new())          # priority=100

    # Optional: render ECS state
    var sync := EcsNodeSync.new(runner.world)
    runner.add_system(EcsTransformSyncSystem.new(sync))  # priority=1000

    var entity := runner.world.create_entity()
    runner.world.add_component(entity, &"position", Vector2(100, 100))
    runner.world.add_component(entity, &"velocity", Vector2(50, 0))
```

For a complete step-by-step walkthrough see **[the 2D demo tutorial](docs/en/tutorial-2d-demo.md)**.

---

## Core concepts

| Concept | Class | One-liner |
| --- | --- | --- |
| Entity | `int` | An integer ID carrying a generation stamp; holds no data itself |
| Component | any value | Data tagged by a `StringName` key (e.g. `&"health"`); no predefined schema |
| System | subclass of `EcsSystem` | Behavior invoked once per tick; finds targets via queries |
| World | `EcsWorld` | Owns entities, component stores, query cache, and system scheduling |
| Query | `EcsQuery` | Filters entities by `all / any / none` criteria; results are cached |
| Runner | `EcsRunner` (node) | Translates the frame loop into `world.start() / tick() / stop()` |
| Node bridge | `EcsNodeSync` | Bidirectional entity↔`Node2D` binding table (optional) |

### What happens in one frame

1. `EcsRunner` receives `_process` / `_physics_process` and calls `world.tick(delta)`;
2. If the world has not started yet, every system's `on_start()` runs in schedule order;
3. Each system's `on_update(world, delta)` runs in ascending `priority`;
4. Inside systems, `query({...})` iterates target entities and reads/writes components;
5. Structural changes (`destroy_entity` / `add_component` / `remove_component`) made along the way are not applied immediately — they enter a command queue;
6. After all systems finish updating, the queue is flushed and signals such as `entity_destroyed` fire.

> Value writes via `set_component` are **immediate** and iteration-safe — input, movement, and other high-frequency logic should use it.

## Built-in contents

| Class | Path | Purpose | Default priority |
| --- | --- | --- | --- |
| `ECSmini` | `runtime/ecsmini.gd` | Static entry: `create_world()`, `VERSION` | — |
| `EcsWorld` | `runtime/ecs_world.gd` | World core: entities/components/queries/scheduling | — |
| `EcsQuery` | `runtime/ecs_query.gd` | Cached queries and safe iterators | — |
| `EcsSystem` | `runtime/ecs_system.gd` | System base class (four lifecycle virtuals) | 0 |
| `EcsRunner` | `runtime/ecs_runner.gd` | Scene-tree bridge node | — |
| `EcsComponentStore` | `runtime/component_store.gd` | Per-type sparse-set store (internal) | — |
| `EcsMovementSystem` | `systems/2d/movement_system.gd` | Built-in movement: `position += velocity * delta` | 100 |
| `EcsNodeSync` | `integrations/godot_2d/ecs_node_sync.gd` | entity↔`Node2D` binding table | — |
| `EcsTransformSyncSystem` | `integrations/godot_2d/transform_sync_system.gd` | Copies ECS transforms to bound nodes | 1000 |

Lower priority values run first; ties break by registration order.

## Repository layout

```
ECSmini/
├── README.md                        ← overview (Chinese)
├── README_EN.md                     ← this file
├── docs/
│   ├── api-reference.md             ← full API reference (Chinese)
│   ├── tutorial-2d-demo.md          ← hands-on tutorial (Chinese)
│   ├── architecture.md              ← internals explained (Chinese)
│   └── en/                          ← English editions
│       ├── api-reference.md
│       ├── tutorial-2d-demo.md
│       └── architecture.md
└── addons/
    └── ecsmini/
        ├── plugin.cfg / plugin.gd   ← plugin manifest & entry
        ├── runtime/                 ← core runtime
        │   ├── ecsmini.gd           ECSmini           static entry
        │   ├── ecs_world.gd         EcsWorld          world core
        │   ├── ecs_query.gd         EcsQuery          cached query
        │   ├── ecs_system.gd        EcsSystem         system base class
        │   ├── ecs_runner.gd        EcsRunner         scene-tree bridge node
        │   └── component_store.gd   EcsComponentStore sparse-set storage (internal)
        ├── systems/
        │   └── 2d/
        │       └── movement_system.gd            EcsMovementSystem built-in movement
        └── integrations/
            └── godot_2d/
                ├── ecs_node_sync.gd              EcsNodeSync entity↔node bindings
                └── transform_sync_system.gd      EcsTransformSyncSystem transform sync
```

## Documentation

| Document | Contents |
| --- | --- |
| [2D demo tutorial](docs/en/tutorial-2d-demo.md) | Build a runnable demo step by step: WASD movement, click-to-spawn enemies, lifetime despawn |
| [API reference](docs/en/api-reference.md) | Full signatures, semantics, return values, and examples for every public class |
| [Architecture notes](docs/en/architecture.md) | Entity ID encoding, sparse sets, query caching, command buffering |

## FAQ (selected)

**Q: Why can't I see a component I added inside `for_each` when querying right after?**
A: Structural changes during iteration are deferred until the traversal ends (or the tick ends). This is deliberate, so one traversal sees a consistent snapshot.

**Q: `set_component` returned `false`?**
A: It only overwrites **existing** components. Use `add_component` for the first write, or check with `has_component` first.

**Q: Changing `runner.use_physics_process` has no effect?**
A: That property decides which callback the node enables — set it **before** `add_child(runner)`.

**Q: Can I store entity IDs across frames?**
A: Yes. IDs carry a generation stamp, so stale IDs become invalid automatically once their slot is reused. Validate with `is_entity_alive(id)` before use; calling APIs on invalid IDs only returns failure values and never harms new entities.

**Q: What performance should I expect?**
A: Pure GDScript implementation: hundreds to a few thousand active entities with around a dozen systems per frame is a comfortable range. Beyond that, consider splitting worlds or moving to a native extension.

More answers live in the [API reference appendices](docs/en/api-reference.md) and the [architecture notes](docs/en/architecture.md).
