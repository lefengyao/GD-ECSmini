# ECSmini

**中文** | [English](README_EN.md)

**轻量级、纯 GDScript 的实体组件系统（ECS）运行时，为 Godot 4 设计。**

ECSmini 把游戏逻辑拆分为「数据（组件）+ 行为（系统）」，用组合代替继承来组织实体。
世界（`EcsWorld`）是一个与场景树完全解耦的纯逻辑对象，可以独立 tick、单元测试或在
headless 服务器上运行；需要接入画面时，再通过可选的表现层把实体绑定到 `Node2D` 上。

- 当前版本：`0.1.0`（与 `ECSmini.VERSION` 一致）
- 运行环境：Godot 4.x（使用了类型化数组、`StringName` 等 4.x 特性，建议 4.2 及以上）
- 语言：GDScript（无需 C# / GDExtension）

---

## 特性

- 🧩 **极简心智模型**：实体就是一个 `int`，组件是挂在实体上的任意值（键为 `StringName`），系统是普通的 GDScript 类。
- ⚡ **高效查询**：稀疏集组件存储 + 按条件缓存的查询 + 「驱动集合」剪枝优化。
- 🛡️ **迭代安全**：在 `tick` 或查询回调中进行的结构性变更（增删实体/组件）会自动排队，在安全边界统一应用；改值操作则立即生效。
- 🔌 **与场景树解耦**：`EcsWorld` 继承自 `RefCounted`，不依赖任何节点；`EcsRunner` 节点一行代码即可把世界接入 `_process` / `_physics_process`。
- 🎨 **可选的 2D 表现层**：`EcsNodeSync` 提供实体↔节点的双向绑定表，`EcsTransformSyncSystem` 自动把 ECS 变换同步到节点。不需要画面时整个表现层都可以不用。
- 📦 **零侵入安装**：单个插件文件夹，不注册任何编辑器面板或自定义检查器。

## 适用场景

| 适合 | 不太适合 |
| --- | --- |
| 中小规模实体数（数百至数千）的 2D 游戏 | 十万级实体的超大规模模拟（建议 C++ ECS） |
| 逻辑与表现分离、需要确定性模拟/回放的项目 | 直接开箱即用的 3D 项目（可参照 `godot_2d` 自行扩展集成层） |
| 希望逻辑可脱离场景树做单元测试的项目 | |

---

## 安装

1. 将 `addons/ecsmini` 整个文件夹复制到你项目的 `addons/` 目录下：
   ```
   你的项目/
   └── addons/
       └── ecsmini/
   ```
2. 打开编辑器，进入 **项目 → 项目设置 → 插件** 标签页，勾选启用 **ECSmini**。

> **说明**：当前版本的插件入口脚本不做任何编辑器侧注册，所有运行时类都通过
> `class_name` 全局可用——即使不启用插件，脚本中也能直接使用这些类。按标准流程启用
> 只是为了保持安装习惯的一致，并便于将来版本加入编辑器功能。

## 30 秒快速上手

下面的示例完全不依赖场景树，可以在任何脚本的 `_ready()` 里直接运行：

```gdscript
var world := ECSmini.create_world()          # 等价于 EcsWorld.new()

# 一个实体 = 一个 int；组件 = 挂在实体上的任意值
var player := world.create_entity()
world.add_component(player, &"position", Vector2.ZERO)
world.add_component(player, &"velocity", Vector2.RIGHT * 120.0)

# 注册一个内置系统：position += velocity * delta
world.add_system(EcsMovementSystem.new())

for i in 60:
    world.tick(1.0 / 60.0)                   # 推进 60 帧

print(world.get_component(player, &"position"))   # 约 (120, 0)
```

## 接入 Godot 场景树（推荐方式）

实际游戏中用 `EcsRunner` 节点托管世界，让 Godot 每帧自动驱动模拟：

```gdscript
extends Node2D   # 场景根节点上的任意脚本


func _ready() -> void:
    var runner := EcsRunner.new()
    runner.use_physics_process = true        # 固定物理步长驱动（必须在 add_child 之前设置）
    add_child(runner)                        # 进入树时自动 world.start()

    runner.add_system(EcsMovementSystem.new())          # priority=100

    # 可选：把 ECS 状态渲染出来
    var sync := EcsNodeSync.new(runner.world)
    runner.add_system(EcsTransformSyncSystem.new(sync))  # priority=1000

    var entity := runner.world.create_entity()
    runner.world.add_component(entity, &"position", Vector2(100, 100))
    runner.world.add_component(entity, &"velocity", Vector2(50, 0))
```

完整可运行的分步教程见 **[docs/tutorial-2d-demo.md](docs/tutorial-2d-demo.md)**。

---

## 核心概念

| 概念 | 类 | 一句话说明 |
| --- | --- | --- |
| 实体 Entity | `int` | 一个带「代际」校验的整数 ID，本身不含数据 |
| 组件 Component | 任意值 | 用 `StringName` 键标记的数据（如 `&"health"`），无预定义 schema |
| 系统 System | `EcsSystem` 子类 | 每帧被调用一次的行为，通过查询找到目标实体 |
| 世界 World | `EcsWorld` | 拥有实体、组件存储、查询缓存和系统调度 |
| 查询 Query | `EcsQuery` | 按 `all / any / none` 三组条件筛选实体，结果自动缓存 |
| 运行器 Runner | `EcsRunner`（节点） | 把 Godot 帧循环翻译成 `world.start() / tick() / stop()` |
| 节点桥 NodeSync | `EcsNodeSync` | 实体 ID ↔ `Node2D` 的双向绑定表（可选） |

### 一帧内发生了什么

1. `EcsRunner` 收到 `_process` / `_physics_process` 回调，调用 `world.tick(delta)`；
2. 若世界尚未启动，先按调度顺序调用所有系统的 `on_start()`；
3. 按 `priority` 从小到大依次调用每个系统的 `on_update(world, delta)`；
4. 系统内部通过 `query({...})` 遍历目标实体，读写组件；
5. 过程中的结构性变更（`destroy_entity` / `add_component` / `remove_component`）不会立刻生效，而是进入命令队列；
6. 所有系统更新完毕后，队列被统一应用，随后触发 `entity_destroyed` 等信号。

> 改值操作 `set_component` 是**即时生效**的，且在迭代期间安全——输入、移动等高频逻辑都应使用它。

---

## 内置内容一览

| 类 | 路径 | 作用 | 默认优先级 |
| --- | --- | --- | --- |
| `ECSmini` | `runtime/ecsmini.gd` | 静态入口：`create_world()`、`VERSION` | — |
| `EcsWorld` | `runtime/ecs_world.gd` | 世界核心：实体/组件/查询/系统调度 | — |
| `EcsQuery` | `runtime/ecs_query.gd` | 缓存查询与安全的遍历迭代器 | — |
| `EcsSystem` | `runtime/ecs_system.gd` | 系统基类（四个生命周期虚方法） | 0 |
| `EcsRunner` | `runtime/ecs_runner.gd` | 场景树桥接节点 | — |
| `EcsComponentStore` | `runtime/component_store.gd` | 单组件类型的稀疏集存储（内部类） | — |
| `EcsMovementSystem` | `systems/2d/movement_system.gd` | 内置移动系统：`position += velocity * delta` | 100 |
| `EcsNodeSync` | `integrations/godot_2d/ecs_node_sync.gd` | 实体↔`Node2D` 双向绑定表 | — |
| `EcsTransformSyncSystem` | `integrations/godot_2d/transform_sync_system.gd` | 把 ECS 变换复制到绑定的节点 | 1000 |

优先级数值越小越先执行；相同优先级按注册顺序执行。

## 目录结构

```
ECSmini/
├── README.md                        ← 本文件（中文）
├── README_EN.md                     ← 英文版总览
├── docs/
│   ├── api-reference.md             ← 完整 API 参考（中文）
│   ├── tutorial-2d-demo.md          ← 上手教程：从零做一个 2D 小 Demo（中文）
│   ├── architecture.md              ← 架构与内部实现说明（中文）
│   └── en/                          ← 英文版文档
│       ├── api-reference.md
│       ├── tutorial-2d-demo.md
│       └── architecture.md
└── addons/
    └── ecsmini/
        ├── plugin.cfg / plugin.gd   ← 插件清单与入口
        ├── runtime/                 ← 核心运行时
        │   ├── ecsmini.gd           ECSmini      静态入口
        │   ├── ecs_world.gd         EcsWorld     世界核心
        │   ├── ecs_query.gd         EcsQuery     缓存查询
        │   ├── ecs_system.gd        EcsSystem    系统基类
        │   ├── ecs_runner.gd        EcsRunner    场景树桥接节点
        │   └── component_store.gd   EcsComponentStore 稀疏集存储（内部）
        ├── systems/
        │   └── 2d/
        │       └── movement_system.gd            EcsMovementSystem 内置移动
        └── integrations/
            └── godot_2d/
                ├── ecs_node_sync.gd              EcsNodeSync 实体↔节点绑定
                └── transform_sync_system.gd      EcsTransformSyncSystem 变换同步
```

## 文档导航

| 文档 | 内容 |
| --- | --- |
| [使用教程](docs/tutorial-2d-demo.md) | 从零搭建一个可运行的 2D Demo：WASD 移动、点击生成敌人、寿命到期自动销毁 |
| [API 参考](docs/api-reference.md) | 所有公开类的完整方法签名、参数语义、返回值与示例 |
| [架构说明](docs/architecture.md) | 实体 ID 编码、稀疏集存储、查询缓存、命令缓冲等内部机制 |

## 常见问题（精选）

**Q：在 `for_each` 回调里 `add_component` 之后，紧接着查询为什么查不到？**
A：迭代期间的结构变更是延迟应用的，会在本次遍历结束（或本帧 `tick` 结束）后统一执行。这是刻意设计，保证一次遍历看到的是一致的快照。

**Q：`set_component` 返回了 `false`？**
A：它只能覆盖**已存在**的组件。首次写入请用 `add_component`；也可以先用 `has_component` 判断。

**Q：修改了 `runner.use_physics_process` 但没生效？**
A：该属性决定节点开启哪个回调，请在 `add_child(runner)` **之前**设置。

**Q：实体 ID 能保存起来跨帧使用吗？**
A：能。ID 内含代际号，槽位被复用后旧 ID 会自动失效；跨帧持有前用 `is_entity_alive(id)` 校验即可，失效的 ID 调用任何 API 都只会安静地返回失败值，不会误伤新实体。

**Q：性能预期如何？**
A：纯 GDScript 实现，数百到数千个活跃实体、每帧十几个系统是舒适区。更大的规模建议拆分多个世界或迁移到原生扩展方案。

更多问答见 [API 参考附录](docs/api-reference.md) 与 [架构说明](docs/architecture.md)。
