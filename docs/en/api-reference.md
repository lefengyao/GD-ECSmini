# ECSmini API Reference

> 🌐 中文版：[../api-reference.md](../api-reference.md)

Applies to version `0.1.0` (matching `ECSmini.VERSION`).

## Conventions

- **Component keys**: component types are identified by `StringName`, written as literals like
  `&"health"`. Key names are entirely application-defined; the plugin ships no predefined schema.
- **Immediate / deferred**: operations marked "deferred" do not take effect right away when called
  during a `tick`, a query callback, or a flush — they enter a command queue and are applied at a
  safe boundary. Operations marked "immediate" always take effect on the spot.
- **Error handling**: no API raises errors. Failures quietly return `false` / default values /
  `-1` sentinels, so you can call first and check the return value afterwards.
- **Threading model**: the whole runtime is designed for single-threaded (main thread) use. Do not
  touch world APIs concurrently from a `Thread`.

## Class overview

| Class | Base | Module | Purpose |
| --- | --- | --- | --- |
| `ECSmini` | `RefCounted` | runtime | Stateless static entry point |
| `EcsWorld` | `RefCounted` | runtime | World core |
| `EcsQuery` | `RefCounted` | runtime | Cached query |
| `EcsSystem` | `RefCounted` | runtime | System base class |
| `EcsRunner` | `Node` | runtime | Scene-tree bridge node |
| `EcsComponentStore` | `RefCounted` | runtime | Internal sparse-set store (rarely used directly) |
| `EcsMovementSystem` | `EcsSystem` | systems/2d | Built-in movement system |
| `EcsNodeSync` | `RefCounted` | integrations/godot_2d | entity↔node binding table |
| `EcsTransformSyncSystem` | `EcsSystem` | integrations/godot_2d | Transform sync system |

---

## ECSmini

Stateless static entry point providing a factory method and the version string.

```gdscript
var world := ECSmini.create_world()
```

### Constants

| Name | Type | Value | Description |
| --- | --- | --- | --- |
| `VERSION` | `String` | `"0.1.0"` | Runtime version, useful for compatibility checks |

### Static methods

#### `static func create_world() -> EcsWorld`

Creates an empty `EcsWorld`. The world is attached to no node: drive it manually with `tick()`,
or hand it to an `EcsRunner`.

---

## EcsWorld

The root object of the ECS runtime. Owns entity lifetime, component stores, query cache, and
system scheduling.

```gdscript
var world := EcsWorld.new()
var entity := world.create_entity()
world.add_component(entity, &"position", Vector2.ZERO)
world.tick(1.0 / 60.0)
```

### Signals

#### `signal entity_destroyed(entity_id: int)`

Emitted **after** an entity has been destroyed and all of its components removed. Commonly used
to clean up presentation-side resources:

```gdscript
world.entity_destroyed.connect(func(entity_id: int) -> void:
    print("entity destroyed: ", entity_id)
)
```

Note: the signal may fire in bursts while the command queue is being flushed (destroying N
entities in one frame emits it N times in a row).

### Entity management

#### `func create_entity() -> int`

Creates and returns a new entity ID.

- The ID packs two numbers into one 64-bit integer: generation (high 32 bits) and storage slot
  (low 32 bits). Slots are reused and each reuse bumps the generation, so **old IDs become
  invalid automatically once their slot is recycled**.
- Every valid entity ID is a large integer (≥ `4294967297`) and can never collide with sentinel
  values like `0` or `-1`.
- This operation is always **immediate** — even during iteration you get a valid ID at once.

```gdscript
var enemy := world.create_entity()
```

#### `func destroy_entity(entity_id: int) -> bool`

Destroys the entity and removes all of its components.

- Returns `false` when the entity does not exist.
- **Deferred**: during a tick / iteration / flush the request is queued; the returned `true`
  means "accepted", not "already destroyed". Actual destruction happens at the safe boundary,
  followed by the `entity_destroyed` signal.

```gdscript
if world.is_entity_alive(target):
    world.destroy_entity(target)
```

#### `func is_entity_alive(entity_id: int) -> bool`

Returns `true` only if `entity_id` is a currently living entity of this world. Validate stored IDs
before reusing them across frames.

#### `func entity_count() -> int`

Number of live entities (regardless of components).

### Component operations

#### `func add_component(entity_id: int, type_key: StringName, value: Variant = null) -> bool`

Adds a component to a live entity.

- Returns `false` if the component already exists (**no overwriting**), the entity is invalid, or
  the key is empty.
- **Deferred**: queued during iteration; the returned `true` means the request was accepted.
- `value` may be any `Variant`: numbers, `Vector2`, `Dictionary`, custom objects, etc.

```gdscript
world.add_component(player, &"health", 100)
world.add_component(enemy, &"position", Vector2.ZERO)
world.add_component(projectile, &"owner_id", shooter_id)
```

#### `func remove_component(entity_id: int, type_key: StringName) -> bool`

Removes a component. Returns `false` if it is absent or the entity is invalid. **Deferred**.

```gdscript
world.remove_component(player, &"stunned")
```

#### `func has_component(entity_id: int, type_key: StringName) -> bool`

Whether the entity currently owns the component. Returns `false` for invalid entities.

#### `func get_component(entity_id: int, type_key: StringName, default_value: Variant = null) -> Variant`

Reads the component value; returns `default_value` if the entity is invalid or the component is
absent.

> ⚠️ If the value itself is a reference type (`Dictionary` / `Array` / `Object`), what you get
> back is **the same reference** — mutating its contents changes the data stored in the world.
> That doubles as a shortcut for in-place mutation, but beware of aliasing when several holders
> share the same data.

```gdscript
var health: int = world.get_component(player, &"health", 0)
```

#### `func set_component(entity_id: int, type_key: StringName, value: Variant) -> bool`

Overwrites an **existing** component's value. Returns `false` when the component is absent (use
`add_component` instead).

- Always **immediate**: a value write does not change any query's membership (the entity still
  matches whatever it matched), so it is safe inside ticks and iterations alike.
- High-frequency logic (input, movement, damage) should prefer this over "remove + add".

```gdscript
world.set_component(player, &"health", 75)
```

### Queries

#### `func query(criteria: Dictionary = {}) -> EcsQuery`

Gets (and caches) a query by criteria. `criteria` understands three keys:

| Key | Meaning | Example |
| --- | --- | --- |
| `"all"` | must own **every** listed component | `{"all": [&"position", &"velocity"]}` |
| `"any"` | must own **at least one** listed component | `{"any": [&"player", &"enemy"]}` |
| `"none"` | must own **none** of the listed components | `{"none": [&"dead"]}` |

Combine freely; an empty dictionary matches every entity:

```gdscript
var movers := world.query({
    "all": [&"position", &"velocity"],
    "any": [&"player", &"enemy"],
    "none": [&"dead"],
})
for entity_id in movers.entities():
    print(entity_id)
```

Caching semantics are described under [EcsQuery](#ecsquery).

### System scheduling

#### `func add_system(system: EcsSystem) -> bool`

Registers a system: adds it to the schedule, sorts by priority, then calls its
`configure(world)`.

- If the world already started, the system's `on_start()` runs immediately as well.
- Returns `false` for `null` or an instance that was already registered.
- Registering mid-tick is allowed; the new system starts receiving `on_update` next frame.

```gdscript
world.add_system(EcsMovementSystem.new())
```

#### `func remove_system(system: EcsSystem) -> bool`

Unregisters a system. On a running world its `on_stop()` is called first. Returns `false` for
unregistered systems. Removing mid-tick is also safe (it simply no longer updates this frame).

#### `func start() -> void`

Starts the world, calling `on_start()` on all registered systems in schedule order. Repeated
calls do nothing. `tick()` starts the world automatically, so explicit calls are rarely needed.

#### `func tick(delta: float) -> void`

Advances the simulation one step:

1. calls `start()` first if necessary;
2. invokes `on_update(world, delta)` on a stable snapshot of systems;
3. after updates, applies all queued structural changes (flush).

Reentrancy guard: calling `tick()` from within a tick does nothing.

```gdscript
func _physics_process(delta: float) -> void:
    world.tick(delta)
```

#### `func stop() -> void`

Stops the world, invoking `on_stop()` in **reverse** order, then flushes the command queue once.
No effect on an already stopped world.

#### `func flush() -> void`

Manually drains the command queue. Works only at a safe boundary (not ticking, not iterating),
otherwise does nothing. Rarely needed manually; typical use is forcing batch structural changes
made outside the frame loop to land immediately.

---

## EcsSystem

Base class of all systems. Subclass it and override whichever of the four virtuals you need to
process entities sharing a component shape.

```gdscript
class_name DamageSystem
extends EcsSystem


func _init() -> void:
    priority = 50   # lower values run earlier


func on_update(world: EcsWorld, delta: float) -> void:
    world.query({"all": [&"health"], "none": [&"invincible"]}).for_each(func(entity_id: int) -> void:
        var hp: int = world.get_component(entity_id, &"health", 0)
        if hp <= 0:
            world.destroy_entity(entity_id)   # destroying during iteration is safe (queued)
    )
```

### Members

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `priority` | `int` | `0` | Schedule priority; lower runs earlier, ties break by registration order |

Built-in reference points: input-style systems usually sit at `0`–`50`; `EcsMovementSystem` uses
`100`; presentation sync `EcsTransformSyncSystem` uses `1000` (so it writes nodes only after all
gameplay logic finished this frame).

### Lifecycle methods

| Method | When it runs | Typical use |
| --- | --- | --- |
| `configure(_world: EcsWorld)` | Exactly once, when added via `add_system` | Cache the world reference, initialize config |
| `on_start()` | When the world starts; immediately if added to an already-started world | Spawn initial entities, subscribe signals |
| `on_update(_world: EcsWorld, _delta: float)` | Every `world.tick(delta)` | Main behavior (most subclasses override only this) |
| `on_stop()` | When the world stops, or before removal from a running world | Release external resources, disconnect signals |

Base implementations are empty; override only what you need.

---

## EcsQuery

A condition-based query created by `world.query(criteria)`, caching results internally.

```gdscript
var movers := world.query({"all": [&"position", &"velocity"], "none": [&"dead"]})
for entity_id in movers.entities():
    print(entity_id)
```

### Caching semantics

- For the same world, **identical criteria** always return the same `EcsQuery` instance — repeat
  calls cost almost nothing, so feel free to keep query objects as system members.
- Criteria are normalized (deduplicated and sorted) before building the cache key, so
  `{"all": [&"a", &"b"]}` and `{"all": [&"b", &"a"]}` share one cached query.
- The cache rebuilds only when the world's topology (entity↔component membership) changes;
  `set_component` writes are not topology changes and never invalidate queries.

### Methods

#### `func entities() -> Array[int]`

Returns the list of currently matching entity IDs. Access lazily refreshes the cache; the return
value is a **copy** of the internal array, so external pushes/erases cannot pollute the cache
(the tradeoff is an O(n) copy per call — weigh call frequency on hot paths).

```gdscript
for entity_id in movers.entities():
    world.set_component(entity_id, &"health", 100)
```

#### `func for_each(callback: Callable) -> void`

Iterates matching entities, running the callback for each — a **structurally-safe** iterator with
three layers of protection:

1. Enters iteration state: structural changes made during traversal are queued;
2. Traverses a snapshot: mutations in callbacks never disturb the current loop's collection;
3. Re-checks `is_entity_alive` before every callback: entities destroyed after the snapshot was
   taken get skipped.

When the outermost traversal ends (and we're not inside a tick), queued changes apply at once —
so the world is consistent again by the time `for_each` returns.

```gdscript
movers.for_each(func(entity_id: int) -> void:
    world.set_component(entity_id, &"velocity", Vector2.ZERO)
)
```

---

## EcsRunner

A `Node` subclass hosting a world, translating node lifecycle into world lifecycle:

| Node callback | World call |
| --- | --- |
| `_ready()` (enters tree) | `world.start()` |
| `_process` or `_physics_process` | `world.tick(delta)` |
| `_exit_tree()` (leaves tree) | `world.stop()` |

Because `EcsWorld` itself needs no scene tree, this bridge is optional: unit tests and headless
simulations can just `tick()` manually.

### Members

#### `@export var use_physics_process: bool = false`

Clock selection:

- `false` (default): follow rendered frames via `_process`; delta is actual frame time — good for
  pure visual pacing;
- `true`: follow physics frames via `_physics_process`; fixed step length (1/60 s by default)
  keeps simulation deterministic — good for netcode, replays, and stability.

> ⚠️ This property decides which callback gets enabled, so set it **before**
> `add_child(runner)`. Changing it after entering the tree has no effect (until the node leaves
> and re-enters the tree).

#### `var world: EcsWorld = EcsWorld.new()`

The world instance owned by this runner, publicly accessible — call
`runner.world.create_entity()` directly. Each runner owns exactly one world; host multiple
runners for multiple isolated worlds. You may also populate `runner.world` with systems before
or after entering the tree, either order works.

### Methods

#### `func add_system(system: EcsSystem) -> bool`

Convenience proxy for `runner.world.add_system(system)` — a facade letting you build the whole
ECS through the runner alone.

---

## EcsMovementSystem (built-in)

2D movement system: performs `position += velocity * delta` for entities owning both components.

```gdscript
world.add_system(EcsMovementSystem.new())   # priority = 100

var entity := world.create_entity()
world.add_component(entity, &"position", Vector2.ZERO)
world.add_component(entity, &"velocity", Vector2(60, 0))   # moves 30 px in 0.5 s
```

| Constant | Value | Type | Description |
| --- | --- | --- | --- |
| `POSITION` | `&"position"` | `Vector2` | Position component updated by the system |
| `VELOCITY` | `&"velocity"` | `Vector2` | Velocity component sampled by the system |

The class never touches any node and runs fine on headless servers. If your game uses different
key names, the simplest fix is writing your own movement system modeled on this one.

---

## EcsNodeSync (optional presentation layer)

An explicit bidirectional binding table between entity IDs and `Node2D` instances — the bridge
between the ECS data world and the scene tree.

```gdscript
var sync := EcsNodeSync.new(world)

var entity := world.create_entity()
sync.bind(entity, $Player)

var player_node := sync.get_node(entity)
```

### Design points

- **One-way dependency**: this class knows `EcsWorld`, but the world knows nothing about nodes,
  keeping the core pure.
- **One-to-one bindings**: an entity binds at most one node and vice versa; re-binding releases
  the previous relation automatically.
- **Never creates or frees nodes**: where visuals come from and when they are freed is the
  presentation layer's job. Entity destruction only unbinds here.
- **Two desync defenses**:
  - entity dies first → automatic unbind via the `entity_destroyed` signal;
  - node dies first (freed externally) → `is_instance_valid` checks plus lazy cleanup on access.

### Construction

#### `func _init(world: EcsWorld = null)`

Pass a world to associate immediately; pass `null` and call `attach_world()` later.

### Methods

#### `func attach_world(world: EcsWorld) -> void`

Associates with (or switches to) a world: disconnects the old world's signals, clears all
bindings, then subscribes to the new world's `entity_destroyed`. Pass `null` to detach fully.
Useful when switching levels/worlds.

#### `func bind(entity_id: int, node: Node2D) -> bool`

Establishes the two-way binding. Fails (returns `false`) when no world is attached, the entity is
dead, or the node is invalid (`null` / freed).

```gdscript
sync.bind(enemy_entity, enemy_sprite)
```

#### `func unbind(entity_id: int) -> void`

Removes an entity's binding (both mapping tables cleaned together). Silently does nothing when
unbound; **does not free the node itself**.

#### `func get_node(entity_id: int) -> Node2D`

Forward lookup: entity → node. Returns `null` when unbound or the node has been freed (also
cleaning up the stale record). Render/sync systems use it to update display properties.

#### `func get_entity(node: Node2D) -> int`

Reverse lookup: node → entity. Returns the sentinel `-1` when the node is invalid or unbound.
Use it in interaction handlers (clicks, collisions) to translate scene nodes back into entities:

```gdscript
var clicked_entity := sync.get_entity(clicked_node)
if clicked_entity != -1:
    world.set_component(clicked_entity, &"health", 0)
```

#### `func clear() -> void`

Clears all binding records (touching neither entities nor nodes). Use when switching worlds or
resetting the whole presentation layer.

---

## EcsTransformSyncSystem (optional presentation layer)

A Godot 2D presentation adapter copying ECS transform components onto `Node2D` instances bound
through `EcsNodeSync`. **The ECS side is always authoritative** — nodes are just projections.

```gdscript
var sync := EcsNodeSync.new(world)
sync.bind(entity, $Player)
world.add_system(EcsTransformSyncSystem.new(sync))   # priority = 1000
```

| Constant | Value | Copied to | Required? |
| --- | --- | --- | --- |
| `POSITION` | `&"position"` | `node.position` (`Vector2`) | Required — entities without it are skipped |
| `ROTATION` | `&"rotation"` | `node.rotation` (`float`, radians) | Optional |
| `SCALE` | `&"scale"` | `node.scale` (`Vector2`) | Optional |

Construction requires the binding table: `EcsTransformSyncSystem.new(sync)`. The default priority
`1000` guarantees it runs after gameplay systems — i.e., "finish computing this frame's logic,
then write display nodes". Entities without bound nodes are silently skipped.

---

## Appendix A: operation semantics cheat sheet

| Operation | Outside protected boundaries | During tick / iteration / flush | Invalidates query cache? |
| --- | --- | --- | --- |
| `create_entity()` | immediate | immediate | yes |
| `destroy_entity()` | immediate | **queued** | yes |
| `add_component()` | immediate | **queued** | yes |
| `remove_component()` | immediate | **queued** | yes |
| `set_component()` | immediate | immediate | no |
| `get_component()` / `has_component()` | immediate | immediate | — |
| `add_system()` / `remove_system()` | immediate | immediate (current frame uses the snapshot) | — |
| `query()` | immediate (may trigger a cache rebuild) | immediate | — |

The command queue is applied at: end of `tick()`, end of `stop()`, completion of the outermost
`for_each`, and manual `flush()`.

## Appendix B: return-value cheat sheet

| Failure scenario | Return value |
| --- | --- |
| Invalid / destroyed entity ID (any operation) | `false` or `default_value` |
| `add_component` but component exists | `false` (use `set_component` to overwrite) |
| `set_component` but component absent | `false` (use `add_component` for the first write) |
| `EcsNodeSync.get_node()` with no binding or a freed node | `null` |
| `EcsNodeSync.get_entity()` finds nothing | `-1` |
| Structural request (destroy / add / remove) made during iteration | `true` (means queued, not yet applied) |
