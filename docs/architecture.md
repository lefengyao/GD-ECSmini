# ECSmini 架构与内部实现

> 🌐 English edition: [architecture.md](en/architecture.md)

本文面向想深入理解或参与贡献的开发者，解释 ECSmini 的模块分层与关键机制的实现方式。
API 用法请先阅读 [README](../README.md) 与 [API 参考](api-reference.md)。

## 总体分层

```
┌───────────────────────────────────────────────────────┐
│                     你的游戏代码                        │
│         自定义 System · 组件读写 · 查询 · 场景          │
├───────────────────────────┬───────────────────────────┤
│     表现层集成（可选）       │        运行时核心          │
│                           │                           │
│   EcsNodeSync             │   EcsWorld    世界/调度    │
│   （实体↔Node2D 绑定表）    │   EcsQuery    缓存查询     │
│   EcsTransformSyncSystem  │   EcsSystem   系统基类     │
│   （变换复制到节点）        │   EcsRunner   帧循环桥接   │
│   EcsMovementSystem*      │   EcsComponentStore       │
│                           │   （稀疏集存储，内部）      │
├───────────────────────────┴───────────────────────────┤
│                 Godot 4 / GDScript                     │
└───────────────────────────────────────────────────────┘
```

`*` `EcsMovementSystem` 虽然放在 `systems/2d/` 目录，但它只做数值积分、不接触任何节点，
因此同样可以在 headless 环境运行。

**依赖方向是单向的**：表现层知道核心，核心对场景树一无所知。
`EcsWorld` 继承 `RefCounted` 而非 `Node`，这带来三个直接收益：

1. 单元测试可以脱离场景树手动 `tick()`，结果确定、可回放；
2. headless 服务器可以直接跑同一套模拟代码；
3. 一个场景里可以并存多个世界（多个 `EcsRunner`），互不干扰。

---

## 实体 ID：槽位 + 代际编码

实体不是对象，而是一个打包了两个信息的 64 位整数：

```
entity_id = (generation << 32) | slot
              高 32 位：代际号      低 32 位：槽位号
```

- **槽位（slot）**：世界内部的数组下标。销毁实体时槽位进入空闲列表
  （`_free_slots`），下次 `create_entity()` 优先复用空闲槽位——避免数组无限增长。
- **代际（generation）**：每次复用槽位时递增。旧 ID 里记录的还是旧代际，
  与当前代际不符即判定失效。

这就是经典的「代际指针」方案，解决的是 ABA 问题：

```
1. create_entity() -> id_A (slot=3, gen=1)
2. destroy_entity(id_A)          // slot=3 进入空闲表
3. create_entity() -> id_B (slot=3, gen=2)   // 复用槽位！
4. is_entity_alive(id_A) == false            // 旧 ID 自动失效，不会误伤 id_B
```

两个附带性质：

- 槽位 `0` 被保留（初始化时占位且永不分配），因此任何有效实体 ID 都 ≥ `2^32 + 1`，
  天然不会和 `0`、`-1` 等哨兵值混淆；
- 所有 API 入口都会做 `is_entity_alive` 校验，持有失效 ID 调用任何方法只会得到
  失败返回值，不会破坏数据。

## 组件存储：稀疏集

每个组件类型（一个 `StringName` 键）对应一个 `EcsComponentStore`，采用经典的三表联动：

```
_values   字典    entity_id -> 组件值          O(1) 存取
_entities 稠密数组 [id0, id1, id2, ...]        连续存储，遍历缓存友好
_indices  字典    entity_id -> 数组下标        O(1) 定位，支撑交换删除
```

**删除使用「交换删除」（swap-remove）**：把稠密数组末尾元素搬到被删位置再弹出，
避免 O(n) 的整体搬移，代价是剩余元素的相对顺序会变。因此：

> ⚠️ 遍历顺序没有稳定性保证。需要顺序时请在应用层携带排序键组件自行排序。

其他行为约定：

- `add()` 对已存在的实体会拒绝（返回 `false`），保证三表索引一致；
- `set_value()` 只改 `_values`，不动数组结构——这就是 `world.set_component`
  「立即生效且不影响查询成员关系」的底层依据；
- 组件 store 在首个组件加入时创建、在最后一个组件移除时销毁（世界按需持有）；
- `entities()` 返回的是内部数组的副本，外部修改不会污染存储（代价是 O(n) 拷贝）。

## 查询系统：条件缓存 + 驱动集合剪枝

一次查询的生命周期如下：

```
query({"all": [...], "any": [...], "none": [...]})
        │
        ▼
 ① 归一化：去空、去重、排序 ──► 生成稳定缓存键（长度前缀编码，避免分隔符冲突）
        │
        ▼
 ② 同条件的查询在世界内共享同一个 EcsQuery 实例（重复调用近乎零开销）
        │
        ▼
 ③ entities() 访问时惰性刷新：
      若 query._compiled_topology_version == world._topology_version → 直接用缓存
      否则重建：
          a. 向世界索要「驱动集合」：
             - 有 all 条件：取 all 中【最小的那个 store】作为候选（选择性剪枝）
             - 只有 any：取各 any store 的并集
             - 都没有：全部存活实体
          b. 对候选逐个执行 all/any/none 完整判定
          c. 对齐版本号，缓存生效直到下一次拓扑变更
```

关键设计点：

- **拓扑版本号（`_topology_version`）**是世界内的单调计数器，任何改变
  「实体↔组件从属关系」的操作（建/删实体、加/删组件）都会使其 +1；
  `set_component` 改值不影响成员关系，因此**不会**使查询失效。
- 归一化后排序意味着条件书写顺序无关：`{"all": [&"a", &"b"]}` 与
  `{"all": [&"b", &"a"]}` 命中同一份缓存。
- 缓存键使用「长度前缀」编码各键名，杜绝键名中出现分隔符导致的碰撞。

## 命令缓冲：结构性变更的安全边界

「结构性变更」指 `destroy_entity`、`add_component`、`remove_component` 三类操作。
它们若发生在遍历中途，直接修改容器会让迭代器失效、让同帧内不同系统看到不一致的世界。
ECSmini 的解法是命令缓冲（command buffering）：

```
_must_defer() 为真的时机：
    _is_ticking        —— tick 正在执行系统更新
    _iteration_depth>0 —— 有 for_each 回调正在执行（支持嵌套）
    _is_flushing       —— 命令队列正在被应用

此时三类操作进入 FIFO 队列 _commands，立即返回 true（表示请求已接受）。

队列的应用时机（安全边界）：
    1. tick() 中所有系统的 on_update 结束后
    2. stop() 末尾
    3. 最外层 for_each 结束时（不在 tick 中的情况下立即应用）
    4. 手动调用 flush()（同样只在安全边界有效）
```

应用过程（`_flush_commands`）按批次进行：

```
while 队列非空:
    取出当前全部命令为一批，换上新空队列
    逐条执行（此时走「立即路径」，真正修改容器）
    执行期间新产生的结构变更（如 entity_destroyed 回调里又删别的实体）
    会因 _is_flushing 标记进入下一批，直到队列彻底排空
```

几个由此推出的重要语义：

- **for_each 快照一致性**：遍历基于开始时的快照 + 每次回调前的存活复查；
  回调中的结构变更既不影响本次循环，也会在遍历结束后立刻落地。
- **信号时机**：`entity_destroyed` 在实际销毁时才发出——批量销毁会在同一批
  flush 里连发多次信号，而不是发出时立刻发。
- ⚠️ **理论上的活锁**：如果 flush 过程中的回调每次都制造新的结构变更
  （例如在 `entity_destroyed` 回调里无中生有地继续销毁），队列将永远排不完。
  请保持销毁回调「只清理、不再引发新的级联结构变更」，或确保级联有限。

与之对照，`set_component` 与 `create_entity` 是刻意设计成始终立即生效的：
改值不动容器结构；新建实体只是追加而非修改既有遍历目标。

## 系统调度

```
add_system(system):
    追加到 _systems，记录注册序号
    稳定排序：priority 升序，相同 priority 按注册先后
    system.configure(world)
    若世界已 start → 立刻补调 system.on_start()

tick(delta):
    重入保护：已在 tick 中则忽略本次调用
    未启动则自动 start()
    对 _systems.duplicate()（快照）依次 on_update
        ↑ 使用快照使得「on_update 里 add/remove_system」是安全的：
          当帧仍执行旧序列，变更从下一帧开始体现

stop():
    对快照的逆序依次 on_stop()
    flush 一次命令队列
```

内置系统的优先级编排体现了推荐的分层惯例：

| priority | 层次 | 示例 |
| --- | --- | --- |
| 0 ~ 50 | 输入 / 决策 | 自定义输入系统、AI 决策 |
| 100 | 逻辑积分 | 内置 `EcsMovementSystem` |
| 200 ~ 900 | 后处理玩法逻辑 | 碰撞裁决、状态流转 |
| 1000 | 表现同步 | 内置 `EcsTransformSyncSystem` |

## 一帧完整时序（以 EcsRunner + 物理帧为例）

```
Godot 主循环
└─ EcsRunner._physics_process(delta)
   └─ EcsWorld.tick(delta)
      ├─ [未启动?] start(): 按 priority 顺序调用各系统 on_start()
      ├─ for system in snapshot:          ← 快照，允许当帧增删系统
      │    system.on_update(world, delta)
      │      ├─ query(...)               ← 命中缓存则零开销
      │      ├─ set_component(...)       ← 立即写入稀疏集
      │      ├─ destroy_entity(...)      ← 排队
      │      └─ add_component(...)       ← 排队
      └─ _flush_commands()               ← 应用队列：
           真正销毁/增删 → 更新拓扑版本号 → 发出 entity_destroyed
                                          └─ EcsNodeSync 自动解绑等清理发生在这里
下一渲染帧: EcsTransformSyncSystem 已在本 tick 内把最新 position 写到 Node2D
```

## 线程模型

整个运行时假设**单线程（主线程）访问**：没有锁、没有原子操作，命令缓冲与
缓存机制都依赖严格的单线程时序。多线程并行化（例如按系统并行）是未来可能的
演进方向，但当前请勿从工作线程触碰任何世界 API。

## 设计权衡备忘

给贡献者与评估者的决策记录：

- **为什么组件键是 `StringName` 而不是脚本类？**
  换来零注册成本与最大灵活性（任意 `Variant` 都能当组件），配合字典稀疏集在
  GDScript 语境下已足够快；类型安全的收益可以通过在应用层封装读写辅助函数获得。
- **为什么 `EcsWorld` 是 `RefCounted` 而非 `Node`？**
  见「总体分层」。核心纯净性优先于便利性，接入帧循环的成本被压缩到
  `EcsRunner` 这一个薄适配器上。
- **为什么 `entities()` / 查询结果返回副本？**
  防止外部增删污染内部缓存与存储。热路径上的拷贝开销可用「减少调用频率、
  在系统内复用局部变量」缓解；若未来成为瓶颈，可提供显式的只读迭代接口。
- **为什么失败一律静默返回而不 push_error？**
  让「校验后再操作」与「直接操作后判返回值」两种风格都能顺畅工作；
  也便于在测试中对非法输入断言返回值。代价是拼写错误的组件键不会报错——
  建议项目里集中定义组件键常量（参照内置系统的做法）。
