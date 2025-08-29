# 🧩 IMultiEntityFactory

A registry interface for storing, retrieving, and creating entities through multiple registered factories identified by keys. Enables centralized entity creation with dynamic factory management capabilities.

## Overview

`IMultiEntityFactory` provides a mutable registry for managing entity factories by key, offering create-on-demand functionality with runtime factory registration and removal. Unlike catalogs, multi-entity factories support modification and direct entity creation.

## Interface Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic registry for IEntity factories using string keys.
    /// </summary>
    public interface IMultiEntityFactory : IMultiEntityFactory<string, IEntity>
    {
    }

    /// <summary>
    /// Generic registry interface for storing and retrieving entity factories by key.
    /// </summary>
    /// <typeparam name="TKey">The type of the key used to identify factories</typeparam>
    /// <typeparam name="E">The type of entity created by the factories</typeparam>
    public interface IMultiEntityFactory<in TKey, E> where E : IEntity
    {
        /// <summary>
        /// Registers an entity factory with the specified key.
        /// </summary>
        void Add(TKey key, IEntityFactory<E> factory);

        /// <summary>
        /// Removes the entity factory associated with the specified key.
        /// </summary>
        void Remove(TKey key);

        /// <summary>
        /// Creates an entity using the factory associated with the specified key.
        /// </summary>
        E Create(TKey key);
    }
}
```

## Key Features

### Dynamic Factory Management
- Runtime registration and removal of entity factories
- Flexible key-based factory identification
- Mutable registry for changing factory configurations

### Direct Entity Creation
- Simplified entity creation through `Create(key)` method
- Automatic factory lookup and entity instantiation
- Exception handling for missing factories

### Type Safety
- Generic interface ensures type-safe factory registration
- Constrains entities to `IEntity` implementations
- Supports any key type for flexible organization schemes

## Usage Examples

### Basic Factory Registry

```csharp
public class EntityFactoryManager : MonoBehaviour
{
    private readonly IMultiEntityFactory _factory = new MultiEntityFactory();
    
    void Start()
    {
        // Register entity factories
        _factory.Add("Player", new InlineEntityFactory(() => CreatePlayerEntity()));
        _factory.Add("Enemy", new InlineEntityFactory(() => CreateEnemyEntity()));
        _factory.Add("Powerup", new InlineEntityFactory(() => CreatePowerupEntity()));
        
        // Create entities on demand
        IEntity player = _factory.Create("Player");
        IEntity enemy = _factory.Create("Enemy");
        
        // Remove factory when no longer needed
        _factory.Remove("Powerup");
    }
    
    private IEntity CreatePlayerEntity()
    {
        var entity = new Entity("Player", 5, 10, 3);
        entity.Set("Health", 100f);
        entity.Set("Speed", 10f);
        entity.AddTag("Player");
        return entity;
    }
    
    private IEntity CreateEnemyEntity()
    {
        var entity = new Entity("Enemy", 3, 8, 2);
        entity.Set("Health", 50f);
        entity.Set("Speed", 5f);
        entity.AddTag("Enemy");
        return entity;
    }
}
```

### Procedural Factory Configuration

```csharp
public class ProceduralEnemyFactory : MonoBehaviour
{
    private readonly IMultiEntityFactory _enemyFactory = new MultiEntityFactory();
    
    [System.Serializable]
    public struct EnemyConfig
    {
        public string name;
        public float health;
        public float speed;
        public float damage;
    }
    
    [SerializeField] private EnemyConfig[] _enemyConfigs;
    
    void Start()
    {
        // Register procedural enemy factories
        foreach (var config in _enemyConfigs)
        {
            _enemyFactory.Add(config.name, new InlineEntityFactory(() => CreateEnemy(config)));
        }
    }
    
    private IEntity CreateEnemy(EnemyConfig config)
    {
        var enemy = new Entity($"Enemy_{config.name}", 4, 6, 2);
        enemy.Set("Health", config.health);
        enemy.Set("MaxHealth", config.health);
        enemy.Set("Speed", config.speed);
        enemy.Set("Damage", config.damage);
        enemy.AddTag("Enemy");
        enemy.AddTag(config.name);
        return enemy;
    }
    
    public IEntity SpawnRandomEnemy()
    {
        var config = _enemyConfigs[Random.Range(0, _enemyConfigs.Length)];
        return _enemyFactory.Create(config.name);
    }
}
```

### Dynamic Factory System

```csharp
public class ModularEntitySystem : MonoBehaviour
{
    private readonly IMultiEntityFactory<Type, IEntity> _entityFactory = 
        new MultiEntityFactory<Type, IEntity>();
    
    void Start()
    {
        // Register factories by component type
        RegisterEntityFactories();
    }
    
    private void RegisterEntityFactories()
    {
        // Player entities
        _entityFactory.Add(typeof(PlayerController), new InlineEntityFactory(() =>
        {
            var entity = new Entity("Player", 8, 15, 5);
            entity.Set("Health", 100f);
            entity.Set("Mana", 50f);
            entity.Set("Level", 1);
            entity.AddTag("Player");
            return entity;
        }));
        
        // AI entities
        _entityFactory.Add(typeof(AIController), new InlineEntityFactory(() =>
        {
            var entity = new Entity("AI", 5, 10, 3);
            entity.Set("Health", 75f);
            entity.Set("Aggression", 0.7f);
            entity.AddTag("AI");
            return entity;
        }));
    }
    
    public T CreateEntityWith<T>() where T : Component
    {
        IEntity entity = _entityFactory.Create(typeof(T));
        // Additional component setup logic
        return null; // Return actual component instance
    }
    
    public void RegisterCustomFactory<T>(IEntityFactory<IEntity> factory) where T : Component
    {
        _entityFactory.Add(typeof(T), factory);
    }
    
    public void UnregisterFactory<T>() where T : Component
    {
        _entityFactory.Remove(typeof(T));
    }
}
```

## Integration with Atomic Framework

### Reactive Factory Management

```csharp
public class ReactiveFactorySystem : MonoBehaviour
{
    private readonly IMultiEntityFactory _factory = new MultiEntityFactory();
    private readonly ReactiveCollection<string> _availableFactories = new();
    
    void Start()
    {
        // React to factory availability changes
        _availableFactories.OnAdded += OnFactoryAdded;
        _availableFactories.OnRemoved += OnFactoryRemoved;
    }
    
    public void RegisterFactory(string key, IEntityFactory<IEntity> factory)
    {
        _factory.Add(key, factory);
        _availableFactories.Add(key);
    }
    
    public void UnregisterFactory(string key)
    {
        _factory.Remove(key);
        _availableFactories.Remove(key);
    }
    
    private void OnFactoryAdded(string key)
    {
        Debug.Log($"Factory '{key}' is now available");
        // Trigger UI updates, event notifications, etc.
    }
    
    private void OnFactoryRemoved(string key)
    {
        Debug.Log($"Factory '{key}' removed");
        // Handle cleanup, UI updates, etc.
    }
}
```

### Event-Driven Entity Creation

```csharp
public class EventEntityFactory : MonoBehaviour
{
    private readonly IMultiEntityFactory _factory = new MultiEntityFactory();
    
    [System.Serializable]
    public class EntityCreationEvent
    {
        public string entityType;
        public Vector3 position;
        public float delay;
    }
    
    void Start()
    {
        SetupFactories();
        
        // Subscribe to game events
        GameEvents.OnEntitySpawnRequested += HandleEntitySpawnRequest;
    }
    
    private void SetupFactories()
    {
        _factory.Add("Projectile", new InlineEntityFactory(CreateProjectile));
        _factory.Add("Explosion", new InlineEntityFactory(CreateExplosion));
        _factory.Add("Pickup", new InlineEntityFactory(CreatePickup));
    }
    
    private void HandleEntitySpawnRequest(EntityCreationEvent evt)
    {
        StartCoroutine(SpawnWithDelay(evt));
    }
    
    private IEnumerator SpawnWithDelay(EntityCreationEvent evt)
    {
        yield return new WaitForSeconds(evt.delay);
        
        try
        {
            IEntity entity = _factory.Create(evt.entityType);
            entity.Set("Position", evt.position);
            entity.Set("SpawnTime", Time.time);
        }
        catch (KeyNotFoundException)
        {
            Debug.LogWarning($"No factory registered for entity type: {evt.entityType}");
        }
    }
    
    private IEntity CreateProjectile() => CreateBaseEntity("Projectile", 2, 4, 1);
    private IEntity CreateExplosion() => CreateBaseEntity("Explosion", 1, 3, 1);
    private IEntity CreatePickup() => CreateBaseEntity("Pickup", 3, 5, 2);
    
    private IEntity CreateBaseEntity(string name, int tags, int values, int behaviors)
    {
        var entity = new Entity(name, tags, values, behaviors);
        entity.AddTag(name);
        entity.Set("CreatedTime", Time.time);
        return entity;
    }
}
```

## Implementation Notes

### Factory Storage
- Uses internal dictionary for O(1) lookup performance
- Supports null factory registration (overwrites existing)
- Throws `KeyNotFoundException` when creating with invalid keys

### Thread Safety
- Default implementations are not thread-safe
- Consider synchronization for multi-threaded scenarios
- Use concurrent collections if needed for parallel access

### Memory Management
- Factories remain in memory until explicitly removed
- Consider cleanup strategies for dynamic factory registration
- Monitor memory usage with large numbers of registered factories

## Best Practices

### Factory Organization
- Use consistent naming conventions for factory keys
- Group related factories by domain or functionality
- Document factory purposes and expected entity configurations

### Error Handling
- Always check for factory existence before creation
- Implement fallback strategies for missing factories
- Log factory registration and removal for debugging

### Performance Optimization
- Pre-register frequently used factories at startup
- Avoid frequent factory registration/removal in hot paths
- Cache factory references when possible

## Common Patterns

### Factory Builder Pattern

```csharp
public class EntityFactoryBuilder
{
    private readonly IMultiEntityFactory _factory;
    
    public EntityFactoryBuilder()
    {
        _factory = new MultiEntityFactory();
    }
    
    public EntityFactoryBuilder WithFactory(string key, IEntityFactory<IEntity> factory)
    {
        _factory.Add(key, factory);
        return this;
    }
    
    public EntityFactoryBuilder WithInlineFactory(string key, Func<IEntity> creator)
    {
        _factory.Add(key, new InlineEntityFactory(creator));
        return this;
    }
    
    public IMultiEntityFactory Build() => _factory;
}

// Usage
var factory = new EntityFactoryBuilder()
    .WithInlineFactory("Player", () => new Entity("Player"))
    .WithInlineFactory("Enemy", () => new Entity("Enemy"))
    .Build();
```

The `IMultiEntityFactory` interface provides a powerful abstraction for runtime entity factory management, enabling flexible and dynamic entity creation systems within the Atomic framework's reactive architecture.