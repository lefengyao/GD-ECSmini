# 上手教程：从零做一个 2D 小 Demo

> 🌐 English edition: [tutorial-2d-demo.md](en/tutorial-2d-demo.md)

本教程带你用 ECSmini 从零搭建一个**完整可运行**的 2D 小场景：

- 一个白色方块（玩家），用 **WASD / 方向键** 控制；
- **鼠标左键点击**场景任意位置生成红色方块（敌人），敌人获得随机速度开始漂移；
- 每个敌人生成 **5 秒后自动销毁**，对应的显示节点一并清理。

运行效果虽小，但覆盖了 ECS 开发的全部核心环节：输入系统、移动系统、渲染同步、
实体生成与销毁、销毁信号的清理回调。完成大约需要 10 分钟。

## 前置条件

- Godot 4.x（建议 4.2+）
- 已按 README 完成插件安装：`addons/ecsmini` 就位并启用。

---

## 第 1 步：创建场景

1. 新建项目（或使用现有项目）。
2. 新建一个场景：根节点选择 **Node2D**，重命名为 `Main`。
3. 将场景保存为 `main.tscn`。
4. 选中 `Main` 节点，附加一个新脚本 `main.gd`（内容在第 5 步给出）。

我们用「代码优先」的方式搭建一切——Runner 节点、系统注册都在脚本里完成，
避免编辑器操作步骤的歧义。你也可以改用编辑器手动添加 `EcsRunner` 节点，
两种方式等价。

## 第 2 步：配置输入映射

打开 **项目 → 项目设置 → 输入映射** 标签页，添加以下 4 个动作：

| 动作名 | 建议绑定的按键 |
| --- | --- |
| `move_left` | `A`、`←` |
| `move_right` | `D`、`→` |
| `move_up` | `W`、`↑` |
| `move_down` | `S`、`↓` |

动作名必须与下面输入系统脚本里的字符串完全一致。

## 第 3 步：编写玩家输入系统

新建脚本 `player_input_system.gd`：

```gdscript
class_name PlayerInputSystem
extends EcsSystem

## 把输入设备的方向转化为玩家的 velocity 组件。
## 必须最先执行，因此 priority = 0。

const SPEED := 260.0


func _init() -> void:
	priority = 0   # 输入先于其他所有系统


func on_update(world: EcsWorld, _delta: float) -> void:
	var direction := Input.get_vector("move_left", "move_right", "move_up", "move_down")

	world.query({"all": [&"player"]}).for_each(func(entity_id: int) -> void:
		# set_component 立即生效且迭代安全：
		# 后续 MovementSystem 在同一帧内就能读到最新速度。
		world.set_component(entity_id, &"velocity", direction * SPEED)
	)
```

要点：

- 继承 `EcsSystem` 并覆写 `on_update(world, delta)`，这是绝大多数系统的全部内容。
- 用 `world.query({"all": [&"player"]})` 找到带 `&"player"` 组件的实体——组件就是约定，
  不需要任何预注册。
- 写数据统一走 `set_component`：它是唯一在迭代中**立即生效**的写操作。

## 第 4 步：编写寿命系统

新建脚本 `lifetime_system.gd`：

```gdscript
class_name LifetimeSystem
extends EcsSystem

## 为拥有 &"lifetime" 组件的实体倒计时，归零后销毁实体。

const LIFETIME := &"lifetime"


func on_update(world: EcsWorld, delta: float) -> void:
	world.query({"all": [LIFETIME]}).for_each(func(entity_id: int) -> void:
		var remaining: float = world.get_component(entity_id, LIFETIME, 0.0) - delta
		if remaining <= 0.0:
			# 迭代中销毁是安全的：请求进入队列，
			# for_each 返回后统一应用，随后触发 entity_destroyed。
			world.destroy_entity(entity_id)
		else:
			world.set_component(entity_id, LIFETIME, remaining)
	)
```

要点：

- 在 `for_each` 里调用 `destroy_entity` 是 ECSmini 的常规用法——不必先收集再删除，
  世界会保证这次遍历基于一致的快照，被排队的销毁在本帧末尾统一落地。

移动逻辑不需要自己写：直接复用内置的 `EcsMovementSystem`
（它对同时拥有 `&"position"` 和 `&"velocity"` 的实体执行 `position += velocity * delta`）。

## 第 5 步：编写主脚本

把 `main.gd` 替换为下面的完整内容：

```gdscript
# main.gd —— 挂在场景根节点 Main（Node2D）上
extends Node2D

const ENEMY_SPEED_MIN := 40.0
const ENEMY_SPEED_MAX := 140.0
const ENEMY_LIFETIME := 5.0

var _world: EcsWorld
var _sync: EcsNodeSync
var _visuals: Dictionary = {}   # entity_id -> Node2D，用于销毁后清理节点


func _ready() -> void:
	# --- 1. 创建 Runner 并选择物理帧时钟 ---
	var runner := EcsRunner.new()
	runner.use_physics_process = true      # 固定步长驱动；必须在 add_child 之前设置！
	add_child(runner)                      # 进入树时自动 world.start()

	_world = runner.world

	# --- 2. 注册系统：按 priority 决定执行顺序 ---
	_world.add_system(PlayerInputSystem.new())            # priority 0    输入
	_world.add_system(EcsMovementSystem.new())            # priority 100  移动（内置）
	_world.add_system(LifetimeSystem.new())               # 默认 0        寿命
	_sync = EcsNodeSync.new(_world)
	_world.add_system(EcsTransformSyncSystem.new(_sync))  # priority 1000 渲染同步

	# --- 3. 实体销毁时清理对应的表现节点 ---
	_world.entity_destroyed.connect(_on_entity_destroyed)

	# --- 4. 创建玩家实体并绑定视觉节点 ---
	var player := _world.create_entity()
	_world.add_component(player, &"player", true)
	_world.add_component(player, &"position", Vector2.ZERO)
	_world.add_component(player, &"velocity", Vector2.ZERO)
	var player_node := _make_square(Color.WHITE)
	add_child(player_node)
	_bind_entity_to_node(player, player_node)


func _unhandled_input(event: InputEvent) -> void:
	# 鼠标左键点击：在点击处生成一个敌人实体
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
	_sync.bind(entity_id, node)     # 注册进同步系统的绑定表
	_visuals[entity_id] = node      # 自己留一份，用于销毁后释放节点


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
	# EcsNodeSync 已经自动解除绑定；节点本身的回收由表现层负责——也就是这里。
	var node: Node2D = _visuals.get(entity_id)
	if node != null and is_instance_valid(node):
		node.queue_free()
	_visuals.erase(entity_id)
```

## 第 6 步：运行

按 `F5` 运行场景（首次会要求选择主场景，选 `main.tscn`）。你应该看到：

1. 白色方块可以用 WASD / 方向键平滑移动；
2. 左键点击处出现红色方块，朝随机方向漂移；
3. 每个红方块约 5 秒后消失（实体销毁 → 信号触发 → 节点被 `queue_free`）。

---

## 回顾：这一帧里发生了什么

以一帧物理步长为例，整条流水线是：

```
_physics_process(delta)
└─ world.tick(delta)
   ├─ PlayerInputSystem   (priority 0)    读输入 → 写 velocity（立即生效）
   ├─ LifetimeSystem      (priority 0)    倒计时 → 归零者排队 destroy
   ├─ EcsMovementSystem   (priority 100)  position += velocity * delta（立即生效）
   ├─ EcsTransformSyncSystem (priority 1000) 把 position/rotation/scale 复制到节点
   └─ flush：应用排队的销毁 → 逐个发出 entity_destroyed → main 清理对应节点
```

三个设计意图值得记住：

1. **priority 编排依赖方向**：输入(0) → 逻辑(100) → 表现(1000)。渲染永远消费
   「本帧已经算完」的数据。
2. **高频读写只用 `set_component` / `get_component`**：立即生效，迭代安全，
   且不会让查询缓存失效。
3. **结构变更交给命令缓冲**：`destroy_entity` 出现在遍历中间也没关系；
   `entity_destroyed` 信号是表现层做清理的唯一可靠入口。

## 常见问题排查

| 现象 | 原因与解法 |
| --- | --- |
| 玩家不动 | 输入映射的动作名与脚本中的 `"move_left"` 等不一致；或焦点被其他控件吞掉 |
| 一切都静止 | `use_physics_process` 是在 `add_child(runner)` 之后才设置的——把它挪到前面 |
| 敌人不消失 | 忘记添加 `lifetime` 组件，或 `LifetimeSystem` 没有注册 |
| 报错「Invalid call to 'bind'」之类 | 先确认 `_sync` 已创建并且传入的世界与实际 tick 的是同一个 |
| 点击没反应 | 场景里没有可接收鼠标事件的视口层级；确认用的是 `_unhandled_input` 且没有 UI 拦截 |

## 下一步练习

按难度递增，试着扩展这个 Demo：

1. **点击消灭**：给方块加 `Area2D`，在点击命中时用 `EcsNodeSync.get_entity(node)`
   反查实体并扣血，血量归零销毁；
2. **追踪行为**：写一个 `ChaseSystem`（priority 200），读取玩家位置，向最近的敌人
   反向施加速度——体会「纯数据进出」的系统写法；
3. **对象池**：不销毁实体而是移除组件、加入休眠集合，对比两种做法的查询表现；
4. **关卡切换**：创建第二个世界，用 `EcsNodeSync.attach_world()` 平滑迁移绑定表；
5. **headless 测试**：写一个 GUT/内置测试脚本，脱离场景树手动 `tick()` 一百帧，
   断言移动系统的积分结果——这正是核心层不依赖节点的回报。
