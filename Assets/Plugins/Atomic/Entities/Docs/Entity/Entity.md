# 🧩 Entity

`Entity` is the core implementation of the `IEntity` interface in the Atomic framework. It provides a complete entity implementation with support for values, tags, behaviours, and lifecycle management.

## Key Features

- **Complete Implementation** – Full IEntity interface implementation
- **Lifecycle Management** – Built-in spawn, activate, update, despawn support
- **Dynamic Composition** – Runtime attachment of behaviours
- **Event System** – Comprehensive event notifications
- **Registry Integration** – Automatic registration with EntityRegistry
- **Memory Efficient** – Pre-allocation support for collections

---

## Class Definition

```csharp
public partial class Entity : IEntity, IDisposable
{
    public virtual event Action OnStateChanged;
    public int InstanceID { get; }
    public string Name { get; set; }
    
    // Lifecycle properties
    public bool Spawned { get; private set; }
    public bool Enabled { get; private set; }
    
    // Collection counts
    public int ValueCount { get; }
    public int TagCount { get; }
    public int BehaviourCount { get; }
}
```

## Constructors

### Basic Constructor
```csharp
public Entity(string name = null, int tagCapacity = 0, int valueCapacity = 0, int behaviourCapacity = 0)
```
- **name**: Optional entity name for debugging
- **tagCapacity**: Initial tag collection capacity
- **valueCapacity**: Initial value dictionary capacity
- **behaviourCapacity**: Initial behaviour list capacity

### Full Constructor
```csharp
public Entity(
    string name,
    IEnumerable<int> tags,
    IEnumerable<KeyValuePair<int, object>> values,
    IEnumerable<IEntityBehaviour> behaviours
)
```
- Initializes entity with provided collections
- Automatically adds all provided items

## Lifecycle Methods

### Spawn
```csharp
public void Spawn()
```
- Transitions entity to spawned state
- Calls `OnSpawn` on all behaviours implementing `IEntitySpawn`
- Triggers `OnSpawned` event
- Automatically activates if not already active

### Despawn
```csharp
public void Despawn()
```
- Deactivates entity if active
- Calls `OnDespawn` on all behaviours implementing `IEntityDespawn`
- Triggers `OnDespawned` event
- Transitions to despawned state

### Activate
```csharp
public void Activate()
```
- Enables entity for updates
- Calls `OnActivate` on all behaviours implementing `IEntityActivate`
- Triggers `OnActivated` event

### Deactivate
```csharp
public void Deactivate()
```
- Disables entity updates
- Calls `OnDeactivate` on all behaviours implementing `IEntityDeactivate`
- Triggers `OnDeactivated` event

### Update Methods
```csharp
public void Update(float deltaTime)
public void FixedUpdate(float deltaTime)
public void LateUpdate(float deltaTime)
```
- Called during respective update phases
- Invokes corresponding behaviour interfaces
- Only processes when entity is enabled

## Disposal

```csharp
public void Dispose()
```
- Despawns entity if spawned
- Clears all tags, values, and behaviours
- Unsubscribes from all events
- Unregisters from EntityRegistry

## Example Usage

### Basic Entity Creation

```csharp
// Create a simple entity
var entity = new Entity("Player");

// Add initial state
entity.AddTag(EntityTags.PLAYER);
entity.AddValue(EntityNames.HEALTH, 100);
entity.AddValue(EntityNames.POSITION, Vector3.zero);

// Add behaviours
entity.AddBehaviour(new MovementBehaviour());
entity.AddBehaviour(new HealthBehaviour());

// Spawn to activate
entity.Spawn();
```

### Pre-allocated Entity

```csharp
// Create with pre-allocated capacity for performance
var entity = new Entity(
    name: "Enemy",
    tagCapacity: 10,      // Expects ~10 tags
    valueCapacity: 20,    // Expects ~20 values
    behaviourCapacity: 5  // Expects ~5 behaviours
);

// Add components without reallocation
for (int i = 0; i < 10; i++)
{
    entity.AddTag(i);
}
```

### Entity with Initial Configuration

```csharp
// Create fully configured entity
var entity = new Entity(
    name: "Boss",
    tags: new[] { EntityTags.ENEMY, EntityTags.BOSS },
    values: new[]
    {
        new KeyValuePair<int, object>(EntityNames.HEALTH, 1000),
        new KeyValuePair<int, object>(EntityNames.DAMAGE, 50),
        new KeyValuePair<int, object>(EntityNames.SPEED, 3.5f)
    },
    behaviours: new IEntityBehaviour[]
    {
        new BossAIBehaviour(),
        new AttackBehaviour(),
        new HealthBehaviour()
    }
);
```

### Lifecycle Management

```csharp
public class GameManager
{
    private List<Entity> entities = new List<Entity>();
    
    public void SpawnEntity(Entity entity)
    {
        entities.Add(entity);
        entity.Spawn();
        
        // Subscribe to lifecycle events
        entity.OnDespawned += () => OnEntityDespawned(entity);
    }
    
    public void UpdateEntities(float deltaTime)
    {
        foreach (var entity in entities)
        {
            if (entity.Enabled)
            {
                entity.Update(deltaTime);
            }
        }
    }
    
    private void OnEntityDespawned(Entity entity)
    {
        entities.Remove(entity);
        entity.Dispose();
    }
}
```

### Dynamic Entity Modification

```csharp
public static class EntityModifier
{
    public static void UpgradeEntity(Entity entity)
    {
        // Increase stats
        var currentHealth = entity.GetValue<int>(EntityNames.HEALTH);
        entity.SetValue(EntityNames.HEALTH, currentHealth + 50);
        
        // Add new capabilities
        entity.AddTag(EntityTags.UPGRADED);
        entity.AddBehaviour(new ShieldBehaviour());
        
        // Trigger state change
        entity.OnStateChanged?.Invoke();
    }
    
    public static void ApplyStatusEffect(Entity entity, int effectTag, float duration)
    {
        entity.AddTag(effectTag);
        
        // Schedule removal
        Timer.Schedule(duration, () =>
        {
            if (entity.HasTag(effectTag))
            {
                entity.DelTag(effectTag);
            }
        });
    }
}
```

### Entity Pooling

```csharp
public class EntityPool
{
    private Stack<Entity> pool = new Stack<Entity>();
    
    public Entity Get(string name)
    {
        Entity entity;
        
        if (pool.Count > 0)
        {
            entity = pool.Pop();
            entity.Name = name;
        }
        else
        {
            entity = new Entity(name, 
                tagCapacity: 5, 
                valueCapacity: 10, 
                behaviourCapacity: 3);
        }
        
        return entity;
    }
    
    public void Return(Entity entity)
    {
        entity.Despawn();
        entity.ClearTags();
        entity.ClearValues();
        entity.ClearBehaviours();
        pool.Push(entity);
    }
}
```

## Procedural Usage Pattern

Following Atomic's procedural approach:

```csharp
public static class EntityOperations
{
    public static Entity CreateCharacter(string name, int health, float speed)
    {
        var entity = new Entity(name);
        InitializeCharacter(entity, health, speed);
        return entity;
    }
    
    public static void InitializeCharacter(Entity entity, int health, float speed)
    {
        entity.AddValue(EntityNames.HEALTH, health);
        entity.AddValue(EntityNames.MAX_HEALTH, health);
        entity.AddValue(EntityNames.SPEED, speed);
        entity.AddTag(EntityTags.CHARACTER);
    }
    
    public static void DamageEntity(Entity entity, int damage)
    {
        if (!entity.HasTag(EntityTags.DAMAGEABLE))
            return;
            
        var health = entity.GetValue<int>(EntityNames.HEALTH);
        health = Math.Max(0, health - damage);
        entity.SetValue(EntityNames.HEALTH, health);
        
        if (health <= 0)
        {
            KillEntity(entity);
        }
    }
    
    public static void KillEntity(Entity entity)
    {
        entity.AddTag(EntityTags.DEAD);
        entity.DelTag(EntityTags.ALIVE);
        entity.Deactivate();
        
        // Schedule despawn
        Timer.Schedule(2f, () => entity.Despawn());
    }
}
```

## Best Practices

1. **Pre-allocate Capacity** – Use capacity parameters for known collection sizes
2. **Dispose Properly** – Always call Dispose() when entity is no longer needed
3. **Check Lifecycle State** – Verify Spawned/Enabled before operations
4. **Use Events Wisely** – Subscribe to events for reactive behaviour
5. **Batch Operations** – Group multiple changes before triggering OnStateChanged
6. **Avoid Deep Hierarchies** – Prefer composition over inheritance

## Performance Considerations

- **Registry Overhead** – Each entity registers with global registry
- **Event Invocations** – Events add overhead; batch changes when possible
- **Collection Growth** – Pre-allocate to avoid reallocation
- **Behaviour Iteration** – Update methods iterate all behaviours
- **Boxing/Unboxing** – Use generic value methods to minimize boxing

## Thread Safety

- Entity is **NOT thread-safe**
- All operations should be performed on the main thread
- Use synchronization if accessing from multiple threads

## Memory Management

- Implements `IDisposable` for proper cleanup
- Clears all references in Dispose()
- Unregisters from global registry
- Consider pooling for frequently created/destroyed entities