# 投射式 ECS (Projected ECS) - 一种概念性架构

这是一个用于介绍 **投射式 ECS (Projected ECS)** 思想的文档。本仓库不包含具体实现，旨在阐述一种对经典 [实体-组件-系统 (ECS)](https://en.wikipedia.org/wiki/Entity_component_system) 架构的扩展，以解决领域解耦和逻辑复用的深层问题。

---

## 为什么需要 Projected ECS？

### 传统方法的困境

*   **面向对象 (OOP)**: 对象常被设计为高度耦合的聚合体，内部状态和逻辑紧密嵌套。这导致了 **领域隔离**，使得跨领域的功能复用变得异常困难。接口（Interface）虽然提供了一种解决方案，但带来了接口方法重名、需要提前设计所有实现、难以插件化扩充等新问题。
*   **经典 ECS**: 经典 ECS 通过将数据（组件）和逻辑（系统）分离，极大地提升了灵活性和性能。然而，它在处理组件间的“视为”关系时显得有些笨拙。

**一个典型的例子**：我们有一个 `RenderSystem`，它需要一个 `MeshComponent` 来进行渲染。现在，一个实体只有一个 `StringComponent`，其内容是一个指向模型文件的路径。在经典 ECS 中，`RenderSystem` 必须包含额外的逻辑：“如果找不到 `MeshComponent`，就去尝试查找 `StringComponent`，然后加载它，再进行渲染”。这种“胶水代码”污染了 `RenderSystem` 的纯粹性，使其与 `StringComponent` 产生了不必要的耦合。

### Projected ECS 的价值主张

Projected ECS 的核心思想是：**将组件之间的“视图转换”或“接口化”能力，从系统内部的胶水代码，提升为架构的一等公民，称之为“投射 (Projection)”。**

你只需要定义好投射规则，就可以轻松地将一个领域的问题，搬到另一个领域上解决。

*   **极致的领域解耦**: 系统只关心它需要的组件“接口”，而无需知道这个组件是真实的，还是由其他组件（如字符串、文件、网络数据流）动态投射而来的。
*   **超高的逻辑复用**: 不仅系统逻辑可以复用，连“转换逻辑”（即投射器）本身也可以被复用。
*   **强大的组合能力**: 一个问题可以被分解，然后投射到各个成熟的领域去解决。甚至可以将一个数据源投射成多个不同的视图，供多个独立的系统同时消费，而这些系统之间互不感知。

最终，Projected ECS 让我们能专注于解决特定领域的问题，而不是把精力浪费在编写大量的桥接、解包代码，或纠结于复杂的继承和实现上，从而避免创造出一个无人能维护的系统。

## 核心概念

### 实体 (Entity)

> 一个 ID。

与经典 ECS 一致，实体本身只是一个唯一的标识符，不包含任何数据和逻辑。

### 组件 (Component)

> **定义**: `接口 | 投射(组件)`

这是 Projected ECS 的核心扩展。一个组件可以是：
1.  **接口 (Interface)**: 一个具体的数据结构，是数据的真实载体。例如 `PositionComponent { x: 10, y: 20 }`。
2.  **投射 (Projection)**: 一个由其他组件动态生成、符合某个组件接口的“视图”或“代理”。

> **约定**: 组件只能影响自身的内部状态，不应带有影响其他组件和实体的副作用。

#### 虚拟实体 (Virtual Entity)

> **介绍**: 虚拟实体为子组件的寻址提供路径。

它解决了组件内部复杂数据（如列表、嵌套对象）的复用问题。

*   **场景**: 一个 `Player` 实体拥有一个 `InventoryComponent`，其内部有一个物品列表 `items: [Item, Item]`。我们希望一个通用的 `TooltipSystem` 能够显示任何 `Item` 的信息。
*   **传统做法**: `TooltipSystem` 需要知道 `InventoryComponent` 的内部结构，遍历 `items` 才能操作 `Item`。
*   **虚拟实体做法**: `InventoryComponent` 可以提供路径解析。我们可以通过一个虚拟实体 ID，如 `player_entity/inventory/0`，来直接寻址到第一个 `Item`。`TooltipSystem` 只需接收这个虚拟实体 ID，就可以像操作普通实体一样获取其 `Item` 组件，而完全无需关心“库存”这个概念。

### 实体-组件映射表 (Entity-Component Map)

> **定义**: `(实体ID, 组件类型) -> 组件实例`

这是 ECS 世界（World）的核心数据结构，但在 Projected ECS 中，它的 `get` 操作被赋予了投射能力。

#### 动态投射 (Dynamic Projection)

> **介绍**: 动态投射允许一个组件在被访问时，通过一个投射函数，动态地生成另一个类型的组件视图。投射函数应提供数据代理机制，将对投射结果的变更反馈回投射源。
>
> **定义**:
> ```
> world.set(ent, A.type, b, projector)
> world.get(ent, A.type) -> projector(b)
> ```

*   **场景**: 一个实体有一个 `JsonConfigComponent { path: "./config.json" }`。我们希望一个 `SettingsSystem` 能像操作普通 `SettingsComponent { theme: "dark" }` 一样读写这份配置。
*   **实现**:
    1.  定义一个从 `JsonConfigComponent` 到 `SettingsComponent` 的 **投射器 (Projector)**。
    2.  当 `SettingsSystem` 请求 `world.get(ent, SettingsComponent)` 时，`world` 发现没有真实的 `SettingsComponent`，但找到了 `JsonConfigComponent` 和与之关联的投射器。
    3.  `world` 执行投射器，该投射器会：
        a. 读取 `./config.json` 文件并解析。
        b. 返回一个 **代理对象 (Proxy)** 作为 `SettingsComponent` 的视图。
    4.  当 `SettingsSystem` 修改这个代理对象时（如 `settings.theme = "light"`），代理机制会自动将变更写回 `./config.json` 文件。
*   **效果**: `SettingsSystem` 与文件 IO 完全解耦，它只与纯粹的 `SettingsComponent` 数据接口交互。

### 系统 (System)

> **定义**: 系统提供操作组件和实体的具体逻辑。

为了保证代码的清晰和可维护性，我们将系统内的逻辑划分为三个层次：

#### 分类

1.  **组件 Operation (Component Operation)**
    *   **职责**: 对单个组件进行无副作用的原子读写。通常是纯函数。
    *   **示例**: `get_position(pos_comp)`、`set_position(pos_comp, x, y)`。
2.  **组件 Command (Component Command)**
    *   **职责**: 实现对单个组件的复杂操作，可能产生副作用（仅限于组件内部状态的改变）。通过调用组件 Operation 实现。
    *   **示例**: `move(pos_comp, dx, dy)`，内部调用 `get_position` 和 `set_position`。
3.  **实体 Command (Entity Command)**
    *   **职责**: 实现涉及多个组件的、复杂的实体级别行为，可能产生广泛的副作用。通过调用组件 Operation 和组件 Command 实现。
    *   **示例**: `player_jump(world, entity_id)`，可能需要读取 `InputComponent`，修改 `PhysicsComponent` 的速度，并更新 `AnimationComponent` 的状态。
