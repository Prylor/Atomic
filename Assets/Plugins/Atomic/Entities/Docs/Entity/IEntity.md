# 🧩 IEntity

`IEntity` is the fundamental interface representing an entity in the Atomic framework. It follows the Entity-State-Behaviour pattern, serving as a container for data (values), identity (tags), and modular logic (behaviours).

## Key Features

- **State Management** – Dynamic key-value store for runtime data
- **Tag System** – Lightweight categorization and filtering  
- **Behaviour Composition** – Attach/detach modular logic at runtime
- **Lifecycle Control** – Built-in spawn, activate, update, and despawn phases
- **Event-Driven** – State change notifications for reactive programming
- **Unique Identity** – Runtime instance ID for entity tracking

---

## Interface Definition

```csharp
public partial interface IEntity : ISpawnable, IActivatable, IUpdatable
{
    /// <summary>
    /// Raised when the internal state of the entity changes.
    /// Useful for tracking structural or dynamic modifications.
    /// </summary>
    event Action OnStateChanged;

    /// <summary>
    /// The runtime-generated unique identifier for this entity instance.
    /// This value is valid only during runtime and should not be used for persistence or serialization.
    /// </summary>
    int InstanceID { get; }

    /// <summary>
    /// Optional user-defined name of the entity.
    /// Typically used for editor tooling, debugging, or runtime labeling.
    /// </summary>
    string Name { get; set; }
}
```

## Inheritance

- **ISpawnable** – Provides spawn/despawn lifecycle management
- **IActivatable** – Provides activate/deactivate functionality
- **IUpdatable** – Provides update loop integration

## Partial Interface Extensions

The `IEntity` interface is split across multiple files for organization:

- **IEntity_Values** – Value storage and retrieval functionality
- **IEntity_Tags** – Tag management functionality  
- **IEntity_Behaviours** – Behaviour attachment and management

## Properties

### InstanceID
```csharp
int InstanceID { get; }
```
- **Description**: Unique runtime identifier for the entity instance
- **Access**: Read-only
- **Notes**: 
  - Generated automatically when entity is created
  - Unique only during runtime session
  - Should not be used for serialization or persistence

### Name
```csharp
string Name { get; set; }
```
- **Description**: Optional human-readable name for the entity
- **Access**: Read-write
- **Usage**: Debugging, editor tools, runtime identification

## Events

### OnStateChanged
```csharp
event Action OnStateChanged;
```
- **Description**: Fired when entity's internal state changes
- **Common Triggers**:
  - Value added/removed/changed
  - Tag added/removed
  - Behaviour added/removed
  - Lifecycle state changes

## Example Usage

```csharp
// Working with IEntity interface
public class EntityManager
{
    private List<IEntity> entities = new List<IEntity>();
    
    public void RegisterEntity(IEntity entity)
    {
        entities.Add(entity);
        
        // Subscribe to state changes
        entity.OnStateChanged += () => OnEntityStateChanged(entity);
        
        // Access entity properties
        Debug.Log($"Registered entity: {entity.Name} (ID: {entity.InstanceID})");
    }
    
    private void OnEntityStateChanged(IEntity entity)
    {
        Debug.Log($"Entity {entity.Name} state changed");
    }
    
    public void UpdateAll(float deltaTime)
    {
        foreach (var entity in entities)
        {
            if (entity.Enabled)
            {
                entity.Update(deltaTime);
            }
        }
    }
}
```

## Procedural Pattern Example

Following Atomic's procedural approach:

```csharp
// Static utility methods operating on IEntity
public static class EntityUtils
{
    public static void SetHealth(IEntity entity, int health)
    {
        entity.SetValue(EntityNames.HEALTH, health);
    }
    
    public static int GetHealth(IEntity entity)
    {
        return entity.GetValue<int>(EntityNames.HEALTH);
    }
    
    public static bool IsAlive(IEntity entity)
    {
        return GetHealth(entity) > 0;
    }
    
    public static void Damage(IEntity entity, int amount)
    {
        int currentHealth = GetHealth(entity);
        SetHealth(entity, Math.Max(0, currentHealth - amount));
        
        if (!IsAlive(entity))
        {
            entity.Despawn();
        }
    }
}

// Usage
IEntity player = new Entity("Player");
EntityUtils.SetHealth(player, 100);
EntityUtils.Damage(player, 30);
```

## Notes

- **Reactive Programming** – Entities support reactive patterns through the `OnStateChanged` event
- **Separation of Concerns** – Interface focuses on entity contract, not implementation
- **Flexibility** – Can be implemented by both pure C# classes and Unity MonoBehaviours
- **Testability** – Interface-based design enables easy mocking and testing