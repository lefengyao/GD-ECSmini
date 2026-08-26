# Tutorial: Building a 2D Demo from Scratch

> 🌐 中文版：[../tutorial-2d-demo.md](../tutorial-2d-demo.md)

This tutorial walks you through building a **complete, runnable** 2D scene with ECSmini:

- A white square (the player) controlled with **WASD / arrow keys**;
- **Left-clicking** anywhere spawns a crimson square (an enemy) that gets a random velocity and
  starts drifting;
- Each enemy **self-destructs after 5 seconds**, and its display node is cleaned up as well.

The demo is small, but it covers every core loop of ECS development: an input system, a movement
system, render sync, entity spawning and destruction, and a cleanup callback on the destroy
signal. Budget about 10 minutes.

## Prerequisites

- Godot 4.x (4.2+ recommended)
- The plugin installed per the README: `addons/ecsmini` in place and enabled.

---

## Step 1: Create the scene

1. Create a new project (or use an existing one).
2. Create a new scene: root node type **Node2D**, renamed to `Main`.
3. Save the scene as `main.tscn`.
4. Select `Main` and attach a new script `main.gd` (contents given in Step 5).

We build everything "code-first" — the runner node and system registration all happen in scripts,
avoiding ambiguity about editor steps. You can equally add the `EcsRunner` node by hand in the
editor; both ways are equivalent.

## Step 2: Configure the input map

Open **Project → Project Settings → Input Map** and add these four actions:

| Action name | Suggested keys |
| --- | --- |
| `move_left` | `A`, `←` |
| `move_right` | `D`, `→` |
| `move_up` | `W`, `↑` |
| `move_down` | `S`, `↓` |

The action names must match the strings used in the input system script below exactly.

## Step 3: Write the player input system

Create a script `player_input_system.gd`:

```gdscript
class_name PlayerInputSystem
extends EcsSystem

## Converts device input into the player's velocity component.
## Must run first, hence priority = 0.

const SPEED := 260.0


func _init() -> void:
	priority = 0   # Input runs before every other system


func on_update(world: EcsWorld, _delta: float) -> void:
	var direction := Input.get_vector("move_left", "move_right", "move_up", "move_down")

	world.query({"all": [&"player"]}).for_each(func(entity_id: int) -> void:
		# set_component takes effect immediately and is iteration-safe:
		# MovementSystem reads the fresh velocity later within the same frame.
		world.set_component(entity_id, &"velocity", direction * SPEED)
	)
```

Key points:

- Subclass `EcsSystem` and override `on_update(world, delta)` — that is most of what a system
  ever is.
- Use `world.query({"all": [&"player"]})` to find entities carrying an `&"player"` component —
  components are pure convention; nothing needs pre-registering.
- Write data through `set_component`: it is the only write operation that takes effect
  **immediately**, even mid-iteration.

## Step 4: Write the lifetime system

Create a script `lifetime_system.gd`:

```gdscript
class_name LifetimeSystem
extends EcsSystem

## Counts down the &"lifetime" component of owning entities, destroying them at zero.

const LIFETIME := &"lifetime"


func on_update(world: EcsWorld, delta: float) -> void:
	world.query({"all": [LIFETIME]}).for_each(func(entity_id: int) -> void:
		var remaining: float = world.get_component(entity_id, LIFETIME, 0.0) - delta
		if remaining <= 0.0:
			# Destroying during iteration is safe: the request is queued,
			# applied together when for_each returns, then entity_destroyed fires.
			world.destroy_entity(entity_id)
		else:
			world.set_component(entity_id, LIFETIME, remaining)
	)
```

Key points:

- Calling `destroy_entity` inside `for_each` is idiomatic ECSmini — no need to collect first and
  delete later. The world guarantees this traversal runs over a consistent snapshot, and queued
  destruction lands together at the end of the frame.

You don't need to write movement yourself: reuse the built-in `EcsMovementSystem` (it performs
`position += velocity * delta` for entities owning both `&"position"` and `&"velocity"`).

## Step 5: Write the main script

Replace `main.gd` with the complete listing:

```gdscript
# main.gd — attached to the scene root Main (Node2D)
extends Node2D

const ENEMY_SPEED_MIN := 40.0
const ENEMY_SPEED_MAX := 140.0
const ENEMY_LIFETIME := 5.0

var _world: EcsWorld
var _sync: EcsNodeSync
var _visuals: Dictionary = {}   # entity_id -> Node2D, to release nodes after destruction


func _ready() -> void:
	# --- 1. Create the runner and pick the physics clock ---
	var runner := EcsRunner.new()
	runner.use_physics_process = true      # fixed-step drive; must be set BEFORE add_child!
	add_child(runner)                      # entering the tree calls world.start()

	_world = runner.world

	# --- 2. Register systems: execution order follows priority ---
	_world.add_system(PlayerInputSystem.new())            # priority 0    input
	_world.add_system(EcsMovementSystem.new())            # priority 100  movement (built-in)
	_world.add_system(LifetimeSystem.new())               # default 0     lifetime
	_sync = EcsNodeSync.new(_world)
	_world.add_system(EcsTransformSyncSystem.new(_sync))  # priority 1000 render sync

	# --- 3. Clean up display nodes when entities are destroyed ---
	_world.entity_destroyed.connect(_on_entity_destroyed)

	# --- 4. Create the player entity and bind its visual node ---
	var player := _world.create_entity()
	_world.add_component(player, &"player", true)
	_world.add_component(player, &"position", Vector2.ZERO)
	_world.add_component(player, &"velocity", Vector2.ZERO)
	var player_node := _make_square(Color.WHITE)
	add_child(player_node)
	_bind_entity_to_node(player, player_node)


func _unhandled_input(event: InputEvent) -> void:
	# Left mouse click: spawn an enemy entity at the click position
	if event is InputEventMouseButton and event.pressed \
			and event.button_index == MOUSE_BUTTON_LEFT:
		_spawn_enemy(get_global_mouse_position())


func _spawn_enemy(at: Vector2) -> void:
	var enemy := _world.create_entity()
	var direction := Vector2.RIGHT.rotated(randf() * TAU)
	var speed := randf_range(ENEMY_SPEED_MIN, ENEMY_SPEED_MAX)

	_world.add_component(enemy, &"position", at)
	_world.add_component(enemy, &"velocity", direction * speed)
	_world.add_component(enemy, &"lifetime", ENEMY_LIFETIME)

	var node := _make_square(Color.CRIMSON)
	add_child(node)
	node.position = at
	_bind_entity_to_node(enemy, node)


func _bind_entity_to_node(entity_id: int, node: Node2D) -> void:
	_sync.bind(entity_id, node)     # register in the sync system's binding table
	_visuals[entity_id] = node      # keep our own copy to free nodes after destruction


func _make_square(color: Color) -> Node2D:
	var half := 14.0
	var square := Polygon2D.new()
	square.polygon = PackedVector2Array([
		Vector2(-half, -half), Vector2(half, -half),
		Vector2(half, half), Vector2(-half, half),
	])
	square.color = color
	return square


func _on_entity_destroyed(entity_id: int) -> void:
	# EcsNodeSync has already unbound automatically; recycling the node itself is the
	# presentation layer's job — which is right here.
	var node: Node2D = _visuals.get(entity_id)
	if node != null and is_instance_valid(node):
		node.queue_free()
	_visuals.erase(entity_id)
```

## Step 6: Run

Press `F5` (choose `main.tscn` as the main scene if asked). You should see:

1. The white square moves smoothly with WASD / arrows;
2. Left-clicking spawns a crimson square that drifts in a random direction;
3. Each red square disappears after ~5 s (entity destroyed → signal fired → node `queue_free`d).

---

## Recap: what happens in one frame

Taking one physics step as an example, here's the whole pipeline:

```
_physics_process(delta)
└─ world.tick(delta)
   ├─ PlayerInputSystem      (priority 0)    read input -> write velocity (immediate)
   ├─ LifetimeSystem         (priority 0)    count down -> queue destroys for zeros
   ├─ EcsMovementSystem      (priority 100)  position += velocity * delta (immediate)
   ├─ EcsTransformSyncSystem (priority 1000) copy position/rotation/scale onto nodes
   └─ flush: apply queued destroys -> emit entity_destroyed per entity
                                             └─ main cleans up the matching node
```

Three design intents are worth remembering:

1. **Priority encodes dependency direction**: input (0) → logic (100) → presentation (1000).
   Rendering always consumes data that finished computing this frame.
2. **High-frequency reads/writes use only `set_component` / `get_component`**: immediate,
   iteration-safe, and they never invalidate query caches.
3. **Structural changes go through the command buffer**: `destroy_entity` mid-traversal is fine;
   the `entity_destroyed` signal is the presentation layer's single reliable cleanup hook.

## Troubleshooting

| Symptom | Cause & fix |
| --- | --- |
| Player doesn't move | Input-map action names don't match `"move_left"` etc.; or another control stole focus |
| Nothing moves at all | `use_physics_process` was set after `add_child(runner)` — move it before |
| Enemies never despawn | Forgot the `lifetime` component or never registered `LifetimeSystem` |
| Errors around `bind()` | Make sure `_sync` exists and its world is the same one being ticked |
| Clicks do nothing | No viewport layer receives the event; you're using `_unhandled_input`, so check no UI intercepts it |

## Next exercises

Roughly increasing difficulty:

1. **Click to kill**: add an `Area2D` to squares; on hit, use `EcsNodeSync.get_entity(node)` to
   find the entity, apply damage, destroy at zero HP;
2. **Chase behavior**: write a `ChaseSystem` (priority 200) reading the player's position and
   steering enemies toward it — feel how "data in, data out" systems work;
3. **Object pooling**: instead of destroying entities, remove components and park them asleep;
   compare query performance of both approaches;
4. **Level switching**: create a second world and migrate bindings smoothly with
   `EcsNodeSync.attach_world()`;
5. **Headless testing**: write a GUT / built-in test script that ticks the world 100 frames
   without any scene tree and asserts the movement integrator's result — that payoff is exactly
   why the core avoids depending on nodes.
