# ECSmini API 参考

> 🌐 English edition: [api-reference.md](en/api-reference.md)

适用版本：`0.1.0`（对应 `ECSmini.VERSION`）。

## 阅读约定

- **组件键**：组件类型用 `StringName` 表示，字面量写作 `&"health"`。键名完全由应用层定义，插件不预置任何 schema。
- **立即 / 延迟**：标注「延迟」的操作在 `tick`、查询回调或 `flush` 期间不会立刻生效，而是进入命令队列，在安全边界统一应用；标注「立即」的操作总是当场生效。
- **错误处理**：所有 API 都不抛错。失败时安静地返回 `false` / 默认值 / `-1` 等哨兵值，可放心调用后直接判断返回值。
- **线程模型**：整个运行时为单线程设计（主线程），不要从 `Thread` 中并发调用世界 API。

## 类总览

| 类 | 基类 | 模块 | 说明 |
| --- | --- | --- | --- |
| `ECSmini` | `RefCounted` | runtime | 无状态静态入口 |
| `EcsWorld` | `RefCounted` | runtime | 世界核心 |
| `EcsQuery` | `RefCounted` | runtime | 缓存查询 |
| `EcsSystem` | `RefCounted` | runtime | 系统基类 |
| `EcsRunner` | `Node` | runtime | 场景树桥接节点 |
| `EcsComponentStore` | `RefCounted` | runtime | 内部稀疏集存储（一般不直接使用） |
| `EcsMovementSystem` | `EcsSystem` | systems/2d | 内置移动系统 |
| `EcsNodeSync` | `RefCounted` | integrations/godot_2d | 实体↔节点绑定表 |
| `EcsTransformSyncSystem` | `EcsSystem` | integrations/godot_2d | 变换同步系统 |

---

## ECSmini

无状态静态入口类，只提供工厂方法与版本号。

```gdscript
var world := ECSmini.create_world()
```

### 常量

| 名称 | 类型 | 值 | 说明 |
| --- | --- | --- | --- |
| `VERSION` | `String` | `"0.1.0"` | 运行时版本，可用于兼容性判断 |

### 静态方法

#### `static func create_world() -> EcsWorld`

创建一个空的 `EcsWorld`。世界不依附于任何节点：可手动调用 `tick()` 推进，或交给 `EcsRunner` 托管。

---

## EcsWorld

ECS 运行时根对象，拥有实体生命周期、组件存储、查询缓存与系统调度。

```gdscript
var world := EcsWorld.new()
var entity := world.create_entity()
world.add_component(entity, &"position", Vector2.ZERO)
world.tick(1.0 / 60.0)
```

### 信号

#### `signal entity_destroyed(entity_id: int)`

实体被销毁且其所有组件移除完毕**之后**发出。常用于清理表现层资源：

```gdscript
world.entity_destroyed.connect(func(entity_id: int) -> void:
    print("实体已销毁: ", entity_id)
)
```

注意：该信号可能在命令队列 flush 时批量触发（同一帧销毁 N 个实体会连发 N 次）。

### 实体管理

#### `func create_entity() -> int`

创建并返回一个新实体 ID。

- ID 编码为「槽位 + 代际」：高 32 位是代际号，低 32 位是存储槽位号。槽位复用时代际号递增，因此**旧 ID 在槽位被复用后会自动失效**。
- 有效实体 ID 都是很大的整数（≥ `4294967297`），不会与 `0` / `-1` 之类的哨兵值混淆。
- 本操作始终**立即生效**，即使在迭代中创建实体也会立刻拿到合法 ID。

```gdscript
var enemy := world.create_entity()
```

#### `func destroy_entity(entity_id: int) -> bool`

销毁实体并移除其全部组件。

- 实体不存在时返回 `false`。
- **延迟操作**：在 tick / 迭代 / flush 期间调用时，销毁请求进入队列，此时返回 `true` 只代表「请求已接受」，不代表已销毁；实际销毁发生在安全边界，随后才发出 `entity_destroyed` 信号。

```gdscript
if world.is_entity_alive(target):
    world.destroy_entity(target)
```

#### `func is_entity_alive(entity_id: int) -> bool`

当且仅当 `entity_id` 是本世界当前存活的实体时返回 `true`。跨帧保存的 ID 在使用前应先校验。

#### `func entity_count() -> int`

返回存活实体总数（无论有没有组件）。

### 组件操作

#### `func add_component(entity_id: int, type_key: StringName, value: Variant = null) -> bool`

为存活实体添加一个组件。

- 组件已存在、实体无效或键名为空时返回 `false`（**不会覆盖**已有值）。
- **延迟操作**：迭代期间调用会排队，返回 `true` 代表请求已接受。
- `value` 可以是任何 `Variant`：数值、`Vector2`、`Dictionary`、自定义对象等。

```gdscript
world.add_component(player, &"health", 100)
world.add_component(enemy, &"position", Vector2.ZERO)
world.add_component(projectile, &"owner_id", shooter_id)
```

#### `func remove_component(entity_id: int, type_key: StringName) -> bool`

移除组件。组件不存在或实体无效时返回 `false`。**延迟操作**。

```gdscript
world.remove_component(player, &"stunned")
```

#### `func has_component(entity_id: int, type_key: StringName) -> bool`

查询实体当前是否拥有某组件。实体无效时返回 `false`。

#### `func get_component(entity_id: int, type_key: StringName, default_value: Variant = null) -> Variant`

读取组件值；实体无效或组件缺失时返回 `default_value`。

> ⚠️ 若组件值本身是引用类型（`Dictionary` / `Array` / `Object`），返回的是**同一个引用**——通过它修改内容会直接影响世界中的数据。这可以当作「就地改值」的捷径使用，但要小心多个持有者共享同一份数据造成的别名问题。

```gdscript
var health: int = world.get_component(player, &"health", 0)
```

#### `func set_component(entity_id: int, type_key: StringName, value: Variant) -> bool`

覆盖一个**已存在**组件的值。组件不存在时返回 `false`（请改用 `add_component`）。

- **始终立即生效**：改值不影响任何查询的成员关系（实体仍匹配原有条件），因此在 `tick` 与迭代过程中都安全。
- 高频逻辑（输入、移动、血量扣减）应优先使用本方法而非「remove + add」。

```gdscript
world.set_component(player, &"health", 75)
```

### 查询

#### `func query(criteria: Dictionary = {}) -> EcsQuery`

按条件获取（并缓存）一个查询对象。`criteria` 支持三组键：

| 键 | 含义 | 示例 |
| --- | --- | --- |
| `"all"` | 必须同时拥有列表中**所有**组件 | `{"all": [&"position", &"velocity"]}` |
| `"any"` | 至少拥有列表中**任意一个**组件 | `{"any": [&"player", &"enemy"]}` |
| `"none"` | **不能拥有**列表中的任何组件 | `{"none": [&"dead"]}` |

三组可自由组合，也可传空字典表示「全部实体」：

```gdscript
var movers := world.query({
    "all": [&"position", &"velocity"],
    "any": [&"player", &"enemy"],
    "none": [&"dead"],
})
for entity_id in movers.entities():
    print(entity_id)
```

缓存语义详见 [EcsQuery](#ecsquery)。

### 系统调度

#### `func add_system(system: EcsSystem) -> bool`

注册系统：加入调度表、按优先级排序，随后调用其 `configure(world)`。

- 若世界已经启动（`start()` 之后），还会立刻调用该系统的 `on_start()`。
- 传入 `null` 或重复注册同一实例时返回 `false`。
- 允许在 tick 过程中注册；新系统从下一帧开始收到 `on_update`。

```gdscript
world.add_system(EcsMovementSystem.new())
```

#### `func remove_system(system: EcsSystem) -> bool`

注销系统。若世界正在运行，会先调用其 `on_stop()`。未注册的系统返回 `false`。在 tick 过程中注销同样安全（当帧不再更新它）。

#### `func start() -> void`

启动世界，按调度顺序对所有已注册系统调用 `on_start()`。重复调用无效果。`tick()` 会自动调用它，通常无需手动执行。

#### `func tick(delta: float) -> void`

推进模拟一步：

1. 未启动则先 `start()`；
2. 对系统的稳定快照依次调用 `on_update(world, delta)`；
3. 更新结束后统一应用排队中的结构变更（flush）。

重入保护：tick 过程中再次调用 `tick()` 会被忽略，避免嵌套驱动。

```gdscript
func _physics_process(delta: float) -> void:
    world.tick(delta)
```

#### `func stop() -> void`

停止世界，按**逆序**对系统调用 `on_stop()`，然后 flush 一次命令队列。对已停止的世界无效果。

#### `func flush() -> void`

手动排空命令队列。仅在世界的安全边界（非 tick、非迭代）有效，否则什么都不做。通常不需要手动调用；典型用途是在帧外做了批量结构变更后立即让它们落地。

---

## EcsSystem

所有系统的基类。子类按需覆写四个虚方法，处理拥有特定组件组合的实体。

```gdscript
class_name DamageSystem
extends EcsSystem


func _init() -> void:
    priority = 50   # 数值越小越先执行


func on_update(world: EcsWorld, delta: float) -> void:
    world.query({"all": [&"health"], "none": [&"invincible"]}).for_each(func(entity_id: int) -> void:
        var hp: int = world.get_component(entity_id, &"health", 0)
        if hp <= 0:
            world.destroy_entity(entity_id)   # 迭代中销毁是安全的（自动排队）
    )
```

### 成员

| 名称 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `priority` | `int` | `0` | 调度优先级，数值越小越先执行；相同值按注册先后执行 |

内置参考：输入类系统常用 `0`~`50`；`EcsMovementSystem` 为 `100`；表现同步 `EcsTransformSyncSystem` 为 `1000`（确保在本帧逻辑全部完成后才写节点）。

### 生命周期方法

| 方法 | 调用时机 | 典型用途 |
| --- | --- | --- |
| `configure(_world: EcsWorld)` | 被 `add_system` 注册时，恰好一次 | 缓存世界引用、初始化配置 |
| `on_start()` | 世界启动时；中途注册到已启动的世界则立即调用 | 生成初始实体、订阅信号 |
| `on_update(_world: EcsWorld, _delta: float)` | 每次 `world.tick(delta)` | 系统主体逻辑（绝大多数子类只需覆写这里） |
| `on_stop()` | 世界停止时，或从运行中的世界被移除前 | 释放外部资源、断开信号 |

基类的默认实现都是空函数，只覆写需要的即可。

---

## EcsQuery

由 `world.query(criteria)` 创建的条件查询，内部按条件缓存结果。

```gdscript
var movers := world.query({"all": [&"position", &"velocity"], "none": [&"dead"]})
for entity_id in movers.entities():
    print(entity_id)
```

### 缓存语义

- 同一世界中**相同条件**的 `query({...})` 永远返回同一个 `EcsQuery` 实例，重复调用几乎零开销，可以把查询对象长期保存在系统成员变量里。
- 条件会被归一化（去重、排序）后再生成缓存键，因此 `{"all": [&"a", &"b"]}` 与 `{"all": [&"b", &"a"]}` 是同一个查询。
- 只有世界的「拓扑」（实体与组件的从属关系）发生变化时缓存才会重建；`set_component` 改值不算拓扑变化，不会使查询失效。

### 方法

#### `func entities() -> Array[int]`

返回当前满足条件的实体 ID 列表。访问时会惰性刷新缓存；返回值是内部数组的**副本**，外部增删不会污染缓存（代价是每次调用有一次 O(n) 拷贝，超热路径可自行权衡调用频率）。

```gdscript
for entity_id in movers.entities():
    world.set_component(entity_id, &"health", 100)
```

#### `func for_each(callback: Callable) -> void`

遍历所有匹配实体并对每个执行回调，是**结构性变更安全**的迭代器，内含三重防护：

1. 进入迭代状态：遍历期间的结构变更自动排队；
2. 基于快照遍历：回调里的变更不会干扰本次循环的集合；
3. 回调前复查 `is_entity_alive`：快照拍摄后才被销毁的实体会被跳过。

遍历结束时（不在 tick 中的情况下）排队的变更会立即应用，因此 `for_each` 返回后世界状态已经一致。

```gdscript
movers.for_each(func(entity_id: int) -> void:
    world.set_component(entity_id, &"velocity", Vector2.ZERO)
)
```

---

## EcsRunner

继承 `Node` 的世界托管节点，把 Godot 的节点生命周期翻译成世界的生命周期：

| 节点回调 | 世界调用 |
| --- | --- |
| `_ready()`（进入树） | `world.start()` |
| `_process` 或 `_physics_process` | `world.tick(delta)` |
| `_exit_tree()`（离开树） | `world.stop()` |

因为 `EcsWorld` 本身不依赖场景树，这个桥接是可选的：单元测试和 headless 模拟可以直接手动 `tick()`。

### 成员

#### `@export var use_physics_process: bool = false`

时钟选择：

- `false`（默认）：跟随渲染帧 `_process`，delta 为实际帧耗时，适合纯视觉表现；
- `true`：跟随物理帧 `_physics_process`，步长固定（默认 1/60 秒），模拟确定，利于网络同步、录像回放与物理稳定。

> ⚠️ 该属性决定节点开启哪个回调，**必须在 `add_child(runner)` 之前设置**，加入树之后再改不会生效（除非把节点移出树再放回）。

#### `var world: EcsWorld = EcsWorld.new()`

Runner 托管的世界实例，公开可访问——可以直接 `runner.world.create_entity()` 等。每个 Runner 独占一个世界；需要多世界隔离时挂多个 Runner 即可。也可以先把系统加进 `runner.world` 再入树，顺序不限。

### 方法

#### `func add_system(system: EcsSystem) -> bool`

`runner.world.add_system(system)` 的便捷代理，提供「只接触 Runner 就能搭起整套 ECS」的门面用法。

---

## EcsMovementSystem（内置）

2D 移动系统：对同时拥有两个组件的实体执行 `position += velocity * delta`。

```gdscript
world.add_system(EcsMovementSystem.new())   # priority = 100

var entity := world.create_entity()
world.add_component(entity, &"position", Vector2.ZERO)
world.add_component(entity, &"velocity", Vector2(60, 0))   # 0.5 秒移动 30 像素
```

| 常量 | 值 | 类型 | 说明 |
| --- | --- | --- | --- |
| `POSITION` | `&"position"` | `Vector2` | 被更新的位置组件 |
| `VELOCITY` | `&"velocity"` | `Vector2` | 被采样的速度组件 |

本类完全不接触任何节点，可在 headless 服务器上运行。若你的游戏使用别的键名，最简单的办法是照抄它写一个自己的移动系统。

---

## EcsNodeSync（可选表现层）

实体 ID ↔ `Node2D` 的显式双向绑定表，是 ECS 数据世界与场景树之间的桥梁。

```gdscript
var sync := EcsNodeSync.new(world)

var entity := world.create_entity()
sync.bind(entity, $Player)

var player_node := sync.get_node(entity)
```

### 设计要点

- **单向依赖**：本类知道 `EcsWorld`，但世界不知道任何节点的存在，核心保持纯净。
- **一对一绑定**：一个实体最多绑一个节点、一个节点最多绑一个实体；重复绑定时旧关系自动解除（换绑）。
- **绝不创建或删除节点**：节点从哪来、什么时候 free，由表现层自己决定。实体销毁时本类只解除绑定。
- **两种失联防御**：
  - 实体先死 → 监听 `entity_destroyed` 自动解绑；
  - 节点先死（被外部 free）→ 查询时的 `is_instance_valid` 校验 + 惰性清理兜底。

### 构造

#### `func _init(world: EcsWorld = null)`

可直接传入世界完成关联；传 `null` 则稍后用 `attach_world()` 手动关联。

### 方法

#### `func attach_world(world: EcsWorld) -> void`

关联（或切换）世界：断开旧世界的信号连接、清空全部绑定，然后订阅新世界的 `entity_destroyed`。传 `null` 表示彻底解绑。适合关卡切换时换世界。

#### `func bind(entity_id: int, node: Node2D) -> bool`

建立双向绑定。世界未关联、实体已死亡或节点无效（`null` / 已 free）时返回 `false`。

```gdscript
sync.bind(enemy_entity, enemy_sprite)
```

#### `func unbind(entity_id: int) -> void`

解除实体的绑定（两张映射表同步清理）。未绑定时静默无操作；**不会删除节点本身**。

#### `func get_node(entity_id: int) -> Node2D`

正向查询：实体 → 节点。未绑定或节点已被释放时返回 `null`（顺带清掉这条脏数据）。渲染/同步系统用它取节点更新显示属性。

#### `func get_entity(node: Node2D) -> int`

反向查询：节点 → 实体。节点无效或未绑定时返回哨兵值 `-1`。交互事件（点击、碰撞）里用它把场景节点翻译回实体：

```gdscript
var clicked_entity := sync.get_entity(clicked_node)
if clicked_entity != -1:
    world.set_component(clicked_entity, &"health", 0)
```

#### `func clear() -> void`

清空所有绑定记录（不动任何实体和节点的存活状态）。用于切换世界或整体重置表现层。

---

## EcsTransformSyncSystem（可选表现层）

Godot 2D 表现适配器：把 ECS 的变换组件复制到经 `EcsNodeSync` 绑定的 `Node2D` 上。**ECS 始终是权威数据源**，节点只是投影。

```gdscript
var sync := EcsNodeSync.new(world)
sync.bind(entity, $Player)
world.add_system(EcsTransformSyncSystem.new(sync))   # priority = 1000
```

| 常量 | 值 | 复制到 | 是否必需 |
| --- | --- | --- | --- |
| `POSITION` | `&"position"` | `node.position`（`Vector2`） | 必需，没有则跳过该实体 |
| `ROTATION` | `&"rotation"` | `node.rotation`（`float` 弧度） | 可选 |
| `SCALE` | `&"scale"` | `node.scale`（`Vector2`） | 可选 |

构造时必须传入绑定表：`EcsTransformSyncSystem.new(sync)`。默认优先级 `1000` 保证它在所有玩法系统之后执行，即「本帧逻辑算完 → 才写显示节点」。未绑定节点的实体会被安静地跳过。

---

## 附录 A：操作语义速查表

| 操作 | 安全边界之外（普通时机） | tick / 迭代 / flush 期间 | 影响查询缓存？ |
| --- | --- | --- | --- |
| `create_entity()` | 立即 | 立即 | 是 |
| `destroy_entity()` | 立即 | **排队** | 是 |
| `add_component()` | 立即 | **排队** | 是 |
| `remove_component()` | 立即 | **排队** | 是 |
| `set_component()` | 立即 | 立即 | 否 |
| `get_component()` / `has_component()` | 立即 | 立即 | — |
| `add_system()` / `remove_system()` | 立即 | 立即（当帧用系统快照，不受影响） | — |
| `query()` | 立即（可能触发缓存重建） | 立即 | — |

命令队列的应用时机：`tick()` 末尾、`stop()` 末尾、最外层 `for_each` 结束时、以及手动 `flush()`。

## 附录 B：返回值速查

| 调用失败场景 | 返回值 |
| --- | --- |
| 实体 ID 无效 / 已销毁（各类操作） | `false` 或 `default_value` |
| `add_component` 但组件已存在 | `false`（想覆盖请用 `set_component`） |
| `set_component` 但组件不存在 | `false`（首次写入请用 `add_component`） |
| `EcsNodeSync.get_node()` 但实体未绑定或节点已释放 | `null` |
| `EcsNodeSync.get_entity()` 查不到绑定 | `-1` |
| 迭代中请求结构变更（destroy / add / remove） | `true`（表示命令已入队，不代表已生效） |
