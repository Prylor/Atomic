# 📘 Atomic.Entities

`Atomic.Entities` is the core module of the Atomic reactive procedural framework for Unity and C#. It implements the Entity-State-Behaviour pattern where entities serve as containers for both data (state) and modular logic (behaviours), emphasizing separation of concerns and procedural programming over traditional object-oriented approaches.

## 🔍 Table of Contents

- **Core Entity System**
  - [IEntity](Entity/IEntity.md)
  - [Entity](Entity/Entity.md)
  - [SceneEntity](Entity/SceneEntity.md)
  - [SceneEntityProxy](Entity/SceneEntityProxy.md)
  - [EntitySingleton](Entity/EntitySingleton.md)
  - [Entity Registry](Entity/EntityRegistry.md)
  - [Entity Extensions](Entity/Extensions.md)

- **Entity Behaviours**
  - [IEntityBehaviour](Behaviours/IEntityBehaviour.md)
  - [IEntitySpawn](Behaviours/IEntitySpawn.md)
  - [IEntityDespawn](Behaviours/IEntityDespawn.md)
  - [IEntityActivate](Behaviours/IEntityActivate.md)
  - [IEntityDeactivate](Behaviours/IEntityDeactivate.md)
  - [IEntityUpdate](Behaviours/IEntityUpdate.md)
  - [IEntityFixedUpdate](Behaviours/IEntityFixedUpdate.md)
  - [IEntityLateUpdate](Behaviours/IEntityLateUpdate.md)
  - [IEntityGizmos](Behaviours/IEntityGizmos.md)

- **Entity Values & Tags**
  - [IEntity_Values](Entity/IEntity_Values.md)
  - [IEntity_Tags](Entity/IEntity_Tags.md)
  - [IEntity_Behaviours](Entity/IEntity_Behaviours.md)

- **Collections & Filters**
  - [IEntityCollection](Collections/IEntityCollection.md)
  - [EntityCollection](Collections/EntityCollection.md)
  - [IReadOnlyEntityCollection](Collections/IReadOnlyEntityCollection.md)
  - [EntityFilter](Filters/EntityFilter.md)
  - [Collection Extensions](Collections/Extensions.md)

- **Entity Factory System**
  - [IEntityFactory](Factory/IEntityFactory.md)
  - [InlineEntityFactory](Factory/InlineEntityFactory.md)
  - [SceneEntityFactory](Factory/SceneEntityFactory.md)
  - [ScriptableEntityFactory](Factory/ScriptableEntityFactory.md)
  - [IMultiEntityFactory](Factory/IMultiEntityFactory.md)
  - [MultiEntityFactory](Factory/MultiEntityFactory.md)
  - [IEntityFactoryCatalog](Factory/IEntityFactoryCatalog.md)
  - [ScriptableEntityCatalog](Factory/ScriptableEntityCatalog.md)

- **Entity Pooling**
  - [IEntityPool](Pooling/IEntityPool.md)
  - [EntityPool](Pooling/EntityPool.md)
  - [IMultiEntityPool](Pooling/IMultiEntityPool.md)
  - [MultiEntityPool](Pooling/MultiEntityPool.md)
  - [IPrefabEntityPool](Pooling/IPrefabEntityPool.md)
  - [PrefabEntityPool](Pooling/PrefabEntityPool.md)
  - [SceneEntityPool](Pooling/SceneEntityPool.md)

- **Entity World**
  - [IEntityWorld](World/IEntityWorld.md)
  - [EntityWorld](World/EntityWorld.md)
  - [SceneEntityWorld](World/SceneEntityWorld.md)

- **Entity Views**
  - [IEntityView](View/IEntityView.md)
  - [EntityView](View/EntityView.md)
  - [EntityViewBase](View/EntityViewBase.md)
  - [IEntityCollectionView](View/IEntityCollectionView.md)
  - [EntityCollectionView](View/EntityCollectionView.md)
  - [EntityViewCatalog](View/EntityViewCatalog.md)
  - [IEntityViewPool](View/IEntityViewPool.md)
  - [EntityViewPool](View/EntityViewPool.md)

- **Entity Installers**
  - [IEntityInstaller](Installer/IEntityInstaller.md)
  - [SceneEntityInstaller](Installer/SceneEntityInstaller.md)
  - [ScriptableEntityInstaller](Installer/ScriptableEntityInstaller.md)

- **Entity Triggers**
  - [IEntityTrigger](Trigger/IEntityTrigger.md)
  - [EntityTriggerBase](Trigger/EntityTriggerBase.md)
  - [InlineEntityTrigger](Trigger/InlineEntityTrigger.md)
  - [TagEntityTrigger](Trigger/TagEntityTrigger.md)
  - [ValueEntityTrigger](Trigger/ValueEntityTrigger.md)
  - [SubscriptionEntityTrigger](Trigger/SubscriptionEntityTrigger.md)

- **Baking System**
  - [SceneEntityBaker](Baking/SceneEntityBaker.md)

- **Common Contracts**
  - [ISpawnable](Common/ISpawnable.md)
  - [IActivatable](Common/IActivatable.md)
  - [IUpdatable](Common/IUpdatable.md)

- **Utilities**
  - [EntityUtils](Utils/EntityUtils.md)
  - [EntityNames](Names/EntityNames.md)
  - [UpdateLoop](Internal/UpdateLoop.md)

- **Best Practices**
  - [Best Practices](BestPractices.md)
  - [Performance Guide](Performance.md)

## 🚀 Quick Start

### Creating an Entity

```csharp
// Create a simple entity
var player = new Entity("Player");

// Add tags for identification
player.AddTag(EntityNames.PLAYER);
player.AddTag(EntityNames.CONTROLLABLE);

// Add values (state)
player.AddValue(EntityNames.HEALTH, 100);
player.AddValue(EntityNames.POSITION, Vector3.zero);

// Add behaviours
player.AddBehaviour(new PlayerMovementBehaviour());
player.AddBehaviour(new HealthSystemBehaviour());

// Spawn the entity to activate it
player.Spawn();
```

### Using SceneEntity in Unity

```csharp
public class PlayerEntity : SceneEntity
{
    [SerializeField] private int maxHealth = 100;
    [SerializeField] private float moveSpeed = 5f;
    
    protected override void OnSpawn()
    {
        base.OnSpawn();
        
        // Initialize entity state
        this.AddValue(EntityNames.HEALTH, maxHealth);
        this.AddValue(EntityNames.MOVE_SPEED, moveSpeed);
        
        // Add runtime behaviours
        this.AddBehaviour(new PlayerInputBehaviour());
    }
}
```

### Entity World Management

```csharp
// Create a world to manage entities
var gameWorld = new EntityWorld("GameWorld");

// Add entities to the world
gameWorld.Add(player);
gameWorld.Add(enemy);

// The world manages lifecycle updates
gameWorld.Spawn();    // Spawns all entities
gameWorld.Activate();  // Activates all entities
gameWorld.Update(deltaTime);  // Updates all entities
```

## 🏗️ Architecture Overview

The Entities module implements the reactive procedural pattern of Atomic framework where:

- **Entities** are data containers that hold state (values), identity (tags), and modular logic (behaviours)
- **State-Behaviour Separation** - Data and logic are strictly separated, with behaviours operating on entity state
- **Behaviours** define modular, reusable logic units that can be dynamically attached/detached
- **Values** store runtime state as reactive key-value pairs that can trigger updates
- **Tags** provide lightweight entity categorization for filtering and identification
- **Procedural Approach** - Logic is implemented in static methods and standalone functions rather than instance methods
- **Worlds** manage collections of entities and coordinate their lifecycle
- **Factories** handle entity creation with configurable templates
- **Pools** optimize memory by reusing entity instances
- **Views** separate presentation logic from core entity logic

## 💡 Key Features

- **Modular Composition** - Build complex entities from simple, reusable behaviours
- **Dynamic State** - Add/remove values and behaviours at runtime
- **Lifecycle Management** - Automatic handling of spawn, activate, update, deactivate, despawn phases
- **Performance Optimized** - Entity pooling, efficient collections, and minimal allocations
- **Unity Integration** - SceneEntity for MonoBehaviour compatibility
- **Testable** - Pure C# entities can be tested without Unity
- **Flexible Architecture** - Supports both inheritance and composition patterns

## 🔧 Advanced Features

- **Entity Filtering** - Query entities by tags and values
- **Entity Triggers** - React to entity state changes
- **Entity Installers** - Configure entities with reusable setups
- **Entity Baking** - Convert GameObjects to entities
- **Multi-Factory Support** - Chain multiple factories for complex creation
- **View System** - Separate presentation from logic
- **Singleton Entities** - Global entity instances