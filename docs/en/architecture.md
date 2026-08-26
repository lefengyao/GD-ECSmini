# ECSmini Architecture & Internals

> 🌐 中文版：[../architecture.md](../architecture.md)

This document targets developers who want a deeper understanding of ECSmini or wish to contribute.
For API usage, read the [README](../../README_EN.md) and the [API reference](api-reference.md)
first.

## Overall layering

```
┌───────────────────────────────────────────────────────┐
│                    Your game code                     │
│      custom systems · component I/O · queries         │
├───────────────────────────┬───────────────────────────┤
│  Presentation (optional)  │       Runtime core        │
│                           │                           │
│   EcsNodeSync             │   EcsWorld    world/sched │
│   (entity↔Node2D table)   │   EcsQuery    cached query│
│   EcsTransformSyncSystem  │   EcsSystem   base class  │
│   (transforms to nodes)   │   EcsRunner   frame bridge│
│   EcsMovementSystem*      │   EcsComponentStore       │
│                           │   (sparse set, internal)  │
├───────────────────────────┴───────────────────────────┤
│                 Godot 4 / GDScript                    │
└───────────────────────────────────────────────────────┘
```

`*` `EcsMovementSystem` lives under `systems/2d/`, but it only integrates numbers and never
touches a node, so it runs on headless setups too.

**The dependency direction is one-way**: presentation knows the core; the core knows nothing about
the scene tree. `EcsWorld` extends `RefCounted` rather than `Node`, which buys three things:

1. Unit tests can `tick()` manually without any scene tree — deterministic, replayable results;
2. Headless servers run the exact same simulation code;
3. Several worlds can coexist in one scene (multiple `EcsRunner`s) without interfering.

---

## Entity IDs: slot + generation encoding

An entity is not an object but a single 64-bit integer packing two numbers:

```
entity_id = (generation << 32) | slot
              high 32: generation   low 32: storage slot
```

- **Slot**: an index into internal arrays. Destroying an entity returns its slot to a free list
  (`_free_slots`); the next `create_entity()` recycles it first, keeping arrays from growing
  forever.
- **Generation**: bumped every time a slot is recycled. Old IDs still carry the old generation,
  which no longer matches — hence they are invalid.

This is the classic generational-index scheme solving the ABA problem:

```
1. create_entity() -> id_A (slot=3, gen=1)
2. destroy_entity(id_A)          // slot=3 goes to the free list
3. create_entity() -> id_B (slot=3, gen=2)   // slot recycled!
4. is_entity_alive(id_A) == false            // old ID auto-invalidated, id_B unharmed
```

Two side properties:

- Slot `0` is reserved (occupied at initialization, never allocated), so every valid entity ID is
  ≥ `2^32 + 1` and can never collide with sentinels like `0` or `-1`;
- Every API entry validates via `is_entity_alive`; calling anything with a stale ID just returns
  failure values without ever corrupting data.

## Component storage: sparse sets

Each component type (one `StringName` key) maps to an `EcsComponentStore`, using the classic
three-structure layout:

```
_values   dictionary  entity_id -> component value     O(1) access
_entities dense array  [id0, id1, id2, ...]            contiguous, cache-friendly iteration
_indices  dictionary  entity_id -> array index          O(1) location for swap-remove
```

**Removal uses swap-remove**: move the last dense element into the removed slot and pop, avoiding
O(n) shifting at the cost of reordering remaining elements. Therefore:

> ⚠️ Iteration order has no stability guarantee. Sort in application code when order matters.

Other behavioral contracts:

- `add()` refuses duplicates (returns `false`), keeping all three structures consistent;
- `set_value()` touches only `_values`, leaving array structures alone — this is exactly why
  `world.set_component` is "immediate and invisible to query membership";
- A store is created when its first component appears and destroyed when its last is removed
  (the world holds stores on demand);
- `entities()` returns a copy of the internal array so external edits cannot pollute storage
  (at O(n) copy cost).

## Queries: condition caching + driver-set pruning

The life cycle of one query:

```
query({"all": [...], "any": [...], "none": [...]})
        │
        ▼
 ① normalize: drop empties, dedupe, sort ──► stable cache key
    (length-prefixed encoding prevents separator collisions)
        │
        ▼
 ② identical criteria share ONE EcsQuery instance per world
    (repeat calls are nearly free)
        │
        ▼
 ③ entities() lazily refreshes:
      if query._compiled_topology_version == world._topology_version → serve cache
      else rebuild:
          a. ask the world for the "driver set":
             - with "all": pick the SMALLEST store among "all" as candidates (pruning)
             - only "any": union of the "any" stores
             - neither:   every live entity
          b. full all/any/none matching per candidate
          c. align version numbers; cache holds until the next topology change
```

Key design points:

- The **topology version** (`_topology_version`) is a monotonic counter inside the world. Every
  operation that alters entity↔component membership (create/destroy entities, add/remove
  components) bumps it; `set_component` does not affect membership and therefore does **not**
  invalidate queries.
- Normalization sorts keys before hashing, so criterion ordering is irrelevant:
  `{"all": [&"a", &"b"]}` hits the same cache as `{"all": [&"b", &"a"]}`.
- Cache keys length-prefix each name, eliminating collisions from separator characters inside
  names.

## Command buffer: safe boundaries for structural changes

"Structural changes" means `destroy_entity`, `add_component`, `remove_component`. Applied
mid-traversal they would invalidate iterators and let different systems see inconsistent worlds
within one frame. ECSmini's answer is command buffering:

```
_must_defer() is true while:
    _is_ticking        —— tick is currently running system updates
    _iteration_depth>0 —— some for_each callback is executing (nesting supported)
    _is_flushing       —— the queue is currently being applied

In those states the three operations enter the FIFO queue _commands
and immediately return true ("accepted").

Queue application points (safe boundaries):
    1. end of tick(), after every system's on_update
    2. end of stop()
    3. completion of the outermost for_each (applied right away outside ticks)
    4. manual flush() (also only effective at safe boundaries)
```

Application (`_flush_commands`) proceeds batch by batch:

```
while queue not empty:
    take everything currently queued as one batch; swap in a fresh queue
    execute each command (immediate paths now really mutate containers)
    structural changes produced during execution (e.g. destroy_entity inside an
    entity_destroyed handler) defer into the next batch because _is_flushing is set,
    looping until the queue drains completely
```

Consequences worth knowing:

- **for_each snapshot consistency**: traversal runs over the snapshot taken at start plus a
  liveness re-check per callback; mutations requested in callbacks neither disturb the current
  loop nor linger — they land the moment the outermost traversal ends.
- **Signal timing**: `entity_destroyed` fires at actual destruction time — mass destructions
  within one flush emit the signal repeatedly in sequence rather than at request time.
- ⚠️ **A theoretical livelock**: if callbacks during a flush keep producing new structural
  changes forever (e.g. each `entity_destroyed` handler destroys yet another entity ad
  infinitum), the queue never drains. Keep destroy handlers cleanup-only, or guarantee cascades
  are finite.

By contrast, `set_component` and `create_entity` are deliberately always-immediate: value writes
never touch container structure, and creating an entity only appends rather than mutating
existing traversal targets.

## System scheduling

```
add_system(system):
    append to _systems, record registration order
    stable sort: priority ascending, ties by registration order
    system.configure(world)
    if world already started → call system.on_start() right away

tick(delta):
    reentrancy guard: ignore nested calls
    auto-start if needed
    iterate _systems.duplicate() (a snapshot) calling on_update
        ↑ the snapshot makes add/remove_system during on_update safe:
          the current frame finishes with the old sequence; changes apply next frame

stop():
    call on_stop() over the snapshot in reverse
    flush the command queue once
```

Built-in priorities illustrate the recommended layering convention:

| priority | Layer | Example |
| --- | --- | --- |
| 0 – 50 | input / decision-making | custom input systems, AI decisions |
| 100 | logic integration | built-in `EcsMovementSystem` |
| 200 – 900 | post-processing gameplay | collision adjudication, state transitions |
| 1000 | presentation sync | built-in `EcsTransformSyncSystem` |

## Full frame timeline (EcsRunner + physics clock)

```
Godot main loop
└─ EcsRunner._physics_process(delta)
   └─ EcsWorld.tick(delta)
      ├─ [not started?] start(): per-priority on_start() calls
      ├─ for system in snapshot:          ← snapshot; add/remove safe mid-frame
      │    system.on_update(world, delta)
      │      ├─ query(...)               ← zero-cost on cache hit
      │      ├─ set_component(...)       ← written into the sparse set now
      │      ├─ destroy_entity(...)      ← queued
      │      └─ add_component(...)       ← queued
      └─ _flush_commands()               ← apply queue:
           real destroys/adds/removes → topology version bumps → entity_destroyed
                                                      └─ EcsNodeSync unbinds, etc.
next rendered frame: EcsTransformSyncSystem already wrote fresh positions onto Node2Ds
```

## Threading model

The runtime assumes **single-threaded (main thread) access**: no locks, no atomics; command
buffering and caching rely on strict single-threaded sequencing. Parallelizing across systems is
a possible future evolution, but today never touch world APIs from worker threads.

## Design trade-off notes

Decision records for contributors and evaluators:

- **Why `StringName` component keys instead of script classes?**
  Zero registration overhead and maximal flexibility (any `Variant` can be a component). In a
  GDScript context, dictionary-backed sparse sets are fast enough; type safety can be regained by
  wrapping read/write helpers at the application layer.
- **Why is `EcsWorld` a `RefCounted` instead of a `Node`?**
  See "Overall layering". Core purity beats convenience; the cost of hooking into the frame loop
  is compressed into the thin `EcsRunner` adapter.
- **Why do `entities()` and queries return copies?**
  To keep external pushes/erases from polluting internal caches and stores. Hot-path copying can
  be mitigated by lowering call frequency and reusing locals inside systems; if it ever becomes a
  bottleneck, an explicit read-only iteration interface could be added.
- **Why silent failures returning values instead of push_error?**
  Both "validate first, then act" and "act, then check the return" styles work smoothly, and
  tests can assert return values on illegal input. The tradeoff: misspelled component keys never
  error — define component-key constants centrally in your project (as the built-in systems do).
