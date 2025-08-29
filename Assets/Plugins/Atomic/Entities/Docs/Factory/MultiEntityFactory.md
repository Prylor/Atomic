# 🧩 MultiEntityFactory

A concrete implementation of `IMultiEntityFactory` that manages entity factories in a key-based registry with runtime registration, removal, and creation capabilities. Provides efficient factory management for dynamic entity creation systems.

## Overview

`MultiEntityFactory` serves as a central registry for entity factories, enabling organized entity creation through key-based lookup. Supports runtime factory management and provides efficient O(1) creation performance through internal dictionary storage.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic multi-factory using string keys and IEntity values.
    /// </summary>
    public class MultiEntityFactory : MultiEntityFactory<string, IEntity>, IMultiEntityFactory
    {
        public MultiEntityFactory() { }
        public MultiEntityFactory(IReadOnlyDictionary<string, IEntityFactory<IEntity>> factories) : base(factories) { }
        public MultiEntityFactory(IEnumerable<KeyValuePair<string, IEntityFactory<IEntity>>> factories) : base(factories) { }
        public MultiEntityFactory(params KeyValuePair<string, IEntityFactory<IEntity>>[] factory) : base(factory) { }
    }

    /// <summary>
    /// Generic multi-factory implementation managing entity factories by key.
    /// </summary>
    /// <typeparam name="TKey">The type of keys used to retrieve factories</typeparam>
    /// <typeparam name="E">The type of entity created by factories</typeparam>
    public class MultiEntityFactory<TKey, E> : IMultiEntityFactory<TKey, E> where E : IEntity
    {
        private readonly Dictionary<TKey, IEntityFactory<E>> _factories;

        public void Add(TKey key, IEntityFactory<E> factory);
        public void Remove(TKey key);
        public E Create(TKey key);
    }
}
```

## Key Features

### Runtime Factory Management
- Dynamic factory registration and removal during execution
- Multiple constructor overloads for flexible initialization
- Thread-safe operations on internal factory dictionary

### Efficient Lookup Performance
- O(1) factory retrieval through dictionary implementation
- Optimized for frequent create operations
- Minimal overhead for factory management operations

### Flexible Initialization
- Empty constructor for runtime-only factory registration
- Collection-based constructors for bulk factory initialization
- Support for dictionary, enumerable, and array initialization patterns

## Usage Examples

### Basic Factory Registry Setup

```csharp
public class GameEntityManager : MonoBehaviour
{
    private readonly MultiEntityFactory _entityFactory = new();
    
    void Start()
    {
        SetupEntityFactories();
        CreateGameEntities();
    }
    
    private void SetupEntityFactories()
    {
        // Register player factory
        _entityFactory.Add("Player", new InlineEntityFactory(() =>
        {
            var player = new Entity("Player", 8, 15, 5);
            player.Set("Health", 100f);
            player.Set("MaxHealth", 100f);
            player.Set("Speed", 10f);
            player.Set("Level", 1);
            player.AddTag("Player");
            player.AddTag("Controllable");
            return player;
        }));
        
        // Register enemy factories
        _entityFactory.Add("BasicEnemy", new InlineEntityFactory(() =>
        {
            var enemy = new Entity("BasicEnemy", 4, 8, 3);
            enemy.Set("Health", 50f);
            enemy.Set("Speed", 5f);
            enemy.Set("Damage", 15f);
            enemy.AddTag("Enemy");
            return enemy;
        }));
        
        _entityFactory.Add("BossEnemy", new InlineEntityFactory(() =>
        {
            var boss = new Entity("BossEnemy", 6, 12, 4);
            boss.Set("Health", 500f);
            boss.Set("Speed", 3f);
            boss.Set("Damage", 50f);
            boss.Set("Experience", 1000);
            boss.AddTag("Enemy");
            boss.AddTag("Boss");
            return boss;
        }));
    }
    
    private void CreateGameEntities()
    {
        // Create entities using registered factories
        IEntity player = _entityFactory.Create("Player");
        IEntity enemy = _entityFactory.Create("BasicEnemy");
        IEntity boss = _entityFactory.Create("BossEnemy");
        
        // Position entities in world
        player.Set("Position", Vector3.zero);
        enemy.Set("Position", new Vector3(10, 0, 0));
        boss.Set("Position", new Vector3(0, 0, 20));
    }
    
    public void SpawnEnemy(string enemyType, Vector3 position)
    {
        try
        {
            IEntity enemy = _entityFactory.Create(enemyType);
            enemy.Set("Position", position);
            enemy.Set("SpawnTime", Time.time);
        }
        catch (KeyNotFoundException)
        {
            Debug.LogError($"Unknown enemy type: {enemyType}");
        }
    }
}
```

### Pre-Initialized Factory Collection

```csharp
public class PrebuiltFactorySystem : MonoBehaviour
{
    [System.Serializable]
    public struct EntityTemplate
    {
        public string key;
        public string name;
        public float health;
        public float speed;
        public string[] tags;
    }
    
    [SerializeField] private EntityTemplate[] _entityTemplates;
    private MultiEntityFactory _factory;
    
    void Start()
    {
        // Build factory collection from templates
        var factoryPairs = _entityTemplates.Select(template => 
            new KeyValuePair<string, IEntityFactory<IEntity>>(
                template.key,
                new InlineEntityFactory(() => CreateFromTemplate(template))
            )
        );
        
        // Initialize multi-factory with pre-built collection
        _factory = new MultiEntityFactory(factoryPairs);
        
        // Test entity creation
        foreach (var template in _entityTemplates)
        {
            IEntity entity = _factory.Create(template.key);
            Debug.Log($"Created {entity.Get<string>("Name")} with {entity.Get<float>("Health")} health");
        }
    }
    
    private IEntity CreateFromTemplate(EntityTemplate template)
    {
        var entity = new Entity(template.name, template.tags.Length, 5, 2);
        entity.Set("Name", template.name);
        entity.Set("Health", template.health);
        entity.Set("MaxHealth", template.health);
        entity.Set("Speed", template.speed);
        
        foreach (string tag in template.tags)
        {
            entity.AddTag(tag);
        }
        
        return entity;
    }
}
```

### Dynamic Factory Modification

```csharp
public class DynamicFactorySystem : MonoBehaviour
{
    private readonly MultiEntityFactory _factory = new();
    private readonly List<string> _registeredFactories = new();
    
    void Start()
    {
        RegisterBaseFactories();
    }
    
    private void RegisterBaseFactories()
    {
        RegisterFactory("Warrior", () => CreateWarrior());
        RegisterFactory("Mage", () => CreateMage());
        RegisterFactory("Archer", () => CreateArcher());
    }
    
    public void RegisterFactory(string key, Func<IEntity> creator)
    {
        _factory.Add(key, new InlineEntityFactory(creator));
        _registeredFactories.Add(key);
        Debug.Log($"Registered factory: {key}");
    }
    
    public void UnregisterFactory(string key)
    {
        if (_registeredFactories.Contains(key))
        {
            _factory.Remove(key);
            _registeredFactories.Remove(key);
            Debug.Log($"Unregistered factory: {key}");
        }
    }
    
    public void RegisterTemporaryFactory(string key, Func<IEntity> creator, float duration)
    {
        RegisterFactory(key, creator);
        StartCoroutine(RemoveFactoryAfterDelay(key, duration));
    }
    
    private IEnumerator RemoveFactoryAfterDelay(string key, float delay)
    {
        yield return new WaitForSeconds(delay);
        UnregisterFactory(key);
    }
    
    private IEntity CreateWarrior()
    {
        var warrior = new Entity("Warrior", 4, 8, 3);
        warrior.Set("Health", 120f);
        warrior.Set("Strength", 15f);
        warrior.Set("Defense", 10f);
        warrior.AddTag("Warrior");
        warrior.AddTag("Melee");
        return warrior;
    }
    
    private IEntity CreateMage()
    {
        var mage = new Entity("Mage", 4, 8, 3);
        mage.Set("Health", 60f);
        mage.Set("Mana", 100f);
        mage.Set("Intelligence", 18f);
        mage.AddTag("Mage");
        mage.AddTag("Caster");
        return mage;
    }
    
    private IEntity CreateArcher()
    {
        var archer = new Entity("Archer", 4, 8, 3);
        archer.Set("Health", 80f);
        archer.Set("Dexterity", 16f);
        archer.Set("Accuracy", 90f);
        archer.AddTag("Archer");
        archer.AddTag("Ranged");
        return archer;
    }
}
```

## Integration with Atomic Framework

### Reactive Factory Management

```csharp
public class ReactiveFactoryRegistry : MonoBehaviour
{
    private readonly MultiEntityFactory _factory = new();
    private readonly ReactiveCollection<string> _availableFactories = new();
    private readonly ReactiveValue<int> _factoryCount = new();
    
    void Start()
    {
        // React to factory availability changes
        _availableFactories.OnAdded += OnFactoryRegistered;
        _availableFactories.OnRemoved += OnFactoryUnregistered;
        _factoryCount.Subscribe(OnFactoryCountChanged);
        
        SetupInitialFactories();
    }
    
    private void SetupInitialFactories()
    {
        var factories = new Dictionary<string, IEntityFactory<IEntity>>
        {
            ["Player"] = new InlineEntityFactory(() => CreatePlayerEntity()),
            ["Enemy"] = new InlineEntityFactory(() => CreateEnemyEntity()),
            ["Item"] = new InlineEntityFactory(() => CreateItemEntity())
        };
        
        _factory = new MultiEntityFactory(factories);
        
        // Update reactive collections
        foreach (string key in factories.Keys)
        {
            _availableFactories.Add(key);
        }
        
        _factoryCount.Value = factories.Count;
    }
    
    public void RegisterNewFactory(string key, Func<IEntity> creator)
    {
        _factory.Add(key, new InlineEntityFactory(creator));
        _availableFactories.Add(key);
        _factoryCount.Value++;
    }
    
    public void RemoveFactory(string key)
    {
        _factory.Remove(key);
        _availableFactories.Remove(key);
        _factoryCount.Value--;
    }
    
    private void OnFactoryRegistered(string key)
    {
        Debug.Log($"New factory available: {key}");
        // Trigger UI updates, notifications, etc.
    }
    
    private void OnFactoryUnregistered(string key)
    {
        Debug.Log($"Factory removed: {key}");
        // Handle cleanup, UI updates, etc.
    }
    
    private void OnFactoryCountChanged(int count)
    {
        Debug.Log($"Total factories: {count}");
        // Update analytics, UI counters, etc.
    }
}
```

### Event-Driven Entity Creation

```csharp
public class EventBasedEntityFactory : MonoBehaviour
{
    private readonly MultiEntityFactory _factory = new();
    
    [System.Serializable]
    public class EntitySpawnEvent
    {
        public string entityType;
        public Vector3 position;
        public Quaternion rotation;
        public Dictionary<string, object> properties;
    }
    
    void Start()
    {
        SetupEventFactories();
        
        // Subscribe to game events
        GameEvents.OnEntitySpawnRequested += HandleEntitySpawn;
        GameEvents.OnWaveStart += HandleWaveStart;
        GameEvents.OnBossEncounter += HandleBossEncounter;
    }
    
    private void SetupEventFactories()
    {
        _factory.Add("SpawnPortal", new InlineEntityFactory(() =>
        {
            var portal = new Entity("SpawnPortal", 3, 6, 2);
            portal.Set("ActivationTime", 2f);
            portal.Set("SpawnRadius", 5f);
            portal.AddTag("Portal");
            return portal;
        }));
        
        _factory.Add("WaveIndicator", new InlineEntityFactory(() =>
        {
            var indicator = new Entity("WaveIndicator", 2, 4, 1);
            indicator.Set("Duration", 3f);
            indicator.Set("IntensityLevel", Random.Range(1, 5));
            indicator.AddTag("Effect");
            return indicator;
        }));
        
        _factory.Add("BossAura", new InlineEntityFactory(() =>
        {
            var aura = new Entity("BossAura", 4, 8, 3);
            aura.Set("EffectRadius", 15f);
            aura.Set("IntensityMultiplier", 2.5f);
            aura.Set("Duration", 60f);
            aura.AddTag("Boss");
            aura.AddTag("Aura");
            return aura;
        }));
    }
    
    private void HandleEntitySpawn(EntitySpawnEvent evt)
    {
        try
        {
            IEntity entity = _factory.Create(evt.entityType);
            entity.Set("Position", evt.position);
            entity.Set("Rotation", evt.rotation);
            
            // Apply custom properties
            foreach (var property in evt.properties)
            {
                entity.Set(property.Key, property.Value);
            }
            
            entity.Set("SpawnTime", Time.time);
        }
        catch (KeyNotFoundException)
        {
            Debug.LogError($"No factory registered for entity type: {evt.entityType}");
        }
    }
    
    private void HandleWaveStart(int waveNumber)
    {
        IEntity waveIndicator = _factory.Create("WaveIndicator");
        waveIndicator.Set("WaveNumber", waveNumber);
        waveIndicator.Set("Position", Vector3.zero);
    }
    
    private void HandleBossEncounter(string bossType)
    {
        IEntity bossAura = _factory.Create("BossAura");
        bossAura.Set("BossType", bossType);
        bossAura.Set("Position", Vector3.zero);
    }
}
```

## Implementation Notes

### Dictionary Management
- Internal `Dictionary<TKey, IEntityFactory<E>>` provides efficient factory storage
- Add/Remove operations modify dictionary directly
- Create operations perform dictionary lookup and factory invocation

### Exception Handling
- `Create` method throws `KeyNotFoundException` when key doesn't exist
- Constructor validation ensures non-null factory collections
- No additional exception handling for factory creation failures

### Memory Considerations
- Factories remain in memory until explicitly removed
- Dictionary grows as factories are added but doesn't shrink when removed
- Consider periodic cleanup for dynamic factory scenarios

## Best Practices

### Factory Organization
- Use consistent, meaningful names for factory keys
- Group related factories by domain or system
- Document factory purposes and expected entity configurations

### Error Handling
- Always validate factory keys before calling Create
- Implement fallback strategies for missing factories
- Log factory operations for debugging and monitoring

### Performance Optimization
- Pre-register frequently used factories at startup
- Avoid frequent Add/Remove operations in performance-critical paths
- Consider caching factory references for hot paths

## Common Patterns

### Factory Builder Pattern

```csharp
public class EntityFactoryBuilder
{
    private readonly Dictionary<string, IEntityFactory<IEntity>> _factories = new();
    
    public EntityFactoryBuilder Add(string key, IEntityFactory<IEntity> factory)
    {
        _factories[key] = factory;
        return this;
    }
    
    public EntityFactoryBuilder Add(string key, Func<IEntity> creator)
    {
        return Add(key, new InlineEntityFactory(creator));
    }
    
    public MultiEntityFactory Build()
    {
        return new MultiEntityFactory(_factories);
    }
}

// Usage
var factory = new EntityFactoryBuilder()
    .Add("Player", () => new Entity("Player"))
    .Add("Enemy", () => new Entity("Enemy"))
    .Build();
```

### Configuration-Driven Factory

```csharp
[System.Serializable]
public class EntityFactoryConfig
{
    public string key;
    public string entityName;
    public EntityData initialData;
}

public class ConfigurableEntityFactory : MonoBehaviour
{
    [SerializeField] private EntityFactoryConfig[] _configs;
    private MultiEntityFactory _factory;
    
    void Start()
    {
        _factory = new MultiEntityFactory(_configs.ToDictionary(
            config => config.key,
            config => (IEntityFactory<IEntity>)new InlineEntityFactory(() => CreateFromConfig(config))
        ));
    }
    
    private IEntity CreateFromConfig(EntityFactoryConfig config)
    {
        var entity = new Entity(config.entityName, 10, 15, 5);
        // Apply configuration data to entity
        return entity;
    }
}
```

The `MultiEntityFactory` provides a robust, efficient implementation for managing entity factories in dynamic systems, supporting both design-time configuration and runtime factory management within the Atomic framework's reactive architecture.