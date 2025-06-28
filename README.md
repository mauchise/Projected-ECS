# Projected ECS: A Conceptual Framework

This repository introduces the concept of **Projected ECS**. It does not contain an implementation but serves as a document to explain an extension to the classic [Entity-Component-System (ECS)](https://en.wikipedia.org/wiki/Entity_component_system) architectural pattern. The goal of Projected ECS is to solve deep-seated problems related to domain decoupling and logic reuse.

---

## Why Projected ECS?

### The Limitations of Traditional Approaches

*   **Object-Oriented Programming (OOP)**: Objects are often designed as tightly coupled aggregates of state and logic. This creates **domain silos**, making it difficult to reuse functionality across different domains. While interfaces offer a solution, they introduce their own problems, such as method name collisions, the need for upfront implementation, and difficulties with pluggable extensions.

*   **Classic ECS**: By separating data (Components) from logic (Systems), classic ECS greatly enhances flexibility and performance. However, it can be clumsy when handling "view-as" relationships between components.

**A Classic Example**: A `RenderSystem` needs a `MeshComponent` to render an object. However, an entity only has a `StringComponent` containing a path to a model file. In a classic ECS, the `RenderSystem` must include glue code: "If `MeshComponent` is not found, try to find a `StringComponent`, load the model from its path, and then render it." This pollutes the purity of the `RenderSystem` and creates an unnecessary coupling with `StringComponent`.

### The Value Proposition of Projected ECS

The core idea of Projected ECS is to elevate the "view-as" or "interface" capability from internal system glue code to a first-class citizen of the architecture, called a **Projection**.

By defining projection rules, you can effortlessly translate a problem from one domain to be solved in another.

*   **Ultimate Decoupling**: Systems only care about the component "interface" they need. They are completely unaware of whether this component is backed by real data or dynamically projected from another source (like a string, a file, or a network stream).
*   **Maximum Reusability**: Not only is system logic reusable, but the "translation logic" (i.e., the projectors) themselves become reusable assets.
*   **Powerful Composition**: A single problem can be decomposed and projected onto multiple mature domains for a solution. A single data source can even be projected into multiple different views to be consumed by several independent systems simultaneously, without any of them being aware of each other.

Ultimately, Projected ECS allows us to focus on solving domain-specific problems instead of wasting energy writing bridge/adapter code, wrestling with complex inheritance, and creating an unmaintainable system.

## Core Concepts

### Entity

> An ID.

Identical to classic ECS, an entity is merely a unique identifier, containing no data or logic.

### Component

> **Definition**: `Interface | Projection(Component)`

This is the core extension in Projected ECS. A component can be one of two things:
1.  **Interface**: A concrete data structure that is the source of truth. E.g., `PositionComponent { x: 10, y: 20 }`.
2.  **Projection**: A "view" or "proxy" that conforms to a component's interface but is dynamically generated from another component.

> **Convention**: A component should only affect its own internal state and should not have side effects that influence other components or entities.

#### Virtual Entity

> **Purpose**: Provides a path for addressing sub-components.

This concept solves the reusability problem for complex data within a component, such as lists or nested objects.

*   **Scenario**: A `Player` entity has an `InventoryComponent` containing a list of items: `items: [Item, Item]`. We want a generic `TooltipSystem` to be able to display information for any `Item`.
*   **Classic Approach**: The `TooltipSystem` would need to know the internal structure of `InventoryComponent` to iterate through `items` and access an `Item`.
*   **Virtual Entity Approach**: The `InventoryComponent` can provide path resolution. We can use a virtual entity ID, such as `player_entity/inventory/0`, to directly address the first `Item`. The `TooltipSystem` simply receives this virtual ID and can get its `Item` component just like any regular entity, remaining completely oblivious to the concept of an "inventory".

### Entity-Component Map

> **Definition**: `(EntityID, ComponentType) -> ComponentInstance`

This is the core data structure of the ECS World. In Projected ECS, its `get` operation is super-charged with projection capabilities.

#### Dynamic Projection

> **Purpose**: Dynamic Projection allows a component to dynamically generate a view of another component type via a projector function when accessed. This function should provide a data proxy mechanism to project changes made to the result back to the source.
>
> **Definition**:
> ```
> world.set(ent, A.type, b, projector)
> world.get(ent, A.type) -> projector(b)
> ```

*   **Scenario**: An entity has a `JsonConfigComponent { path: "./config.json" }`. We want a `SettingsSystem` to read and write this configuration as if it were a plain `SettingsComponent { theme: "dark" }`.
*   **Implementation**:
    1.  Define a **Projector** function that translates from `JsonConfigComponent` to `SettingsComponent`.
    2.  When `SettingsSystem` calls `world.get(ent, SettingsComponent)`, the `world` finds no real `SettingsComponent` but discovers the `JsonConfigComponent` and its associated projector.
    3.  The `world` executes the projector, which:
        a. Reads and parses the `./config.json` file.
        b. Returns a **Proxy object** as a view of the `SettingsComponent`.
    4.  When the `SettingsSystem` modifies this proxy object (e.g., `settings.theme = "light"`), the proxy mechanism automatically writes the changes back to the `./config.json` file.
*   **Result**: The `SettingsSystem` is completely decoupled from file I/O. It interacts only with the pure `SettingsComponent` data interface.

### System

> **Definition**: A System provides the concrete logic for operating on components and entities.

To ensure clarity and maintainability, we classify the logic within systems into three distinct layers:

#### Classification

1.  **Component Operation**
    *   **Responsibility**: Atomic, side-effect-free read/write operations on a single component. These are typically pure functions.
    *   **Example**: `get_position(pos_comp)`, `set_position(pos_comp, x, y)`.
2.  **Component Command**
    *   **Responsibility**: Implements complex logic for a single component, potentially with side effects (limited to the component's internal state). It is built by calling Component Operations.
    *   **Example**: `move(pos_comp, dx, dy)`, which internally calls `get_position` and `set_position`.
3.  **Entity Command**
    *   **Responsibility**: Implements complex, entity-level behaviors that involve multiple components and may produce broad side effects. It is built by calling Component Operations and Component Commands.
    *   **Example**: `player_jump(world, entity_id)`, which might read an `InputComponent`, modify the velocity in a `PhysicsComponent`, and update the state of an `AnimationComponent`.
