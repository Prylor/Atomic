# 🧩 IEntityFactoryCatalog

A catalog interface for managing and accessing multiple entity factories through a dictionary-like interface. Provides centralized access to factory instances by key lookup for efficient entity creation workflows.

## Overview

`IEntityFactoryCatalog` represents a read-only catalog that maps keys to entity factories, enabling organized factory management and lookup. It extends `IReadOnlyDictionary` to provide familiar dictionary semantics while maintaining type safety for entity creation.

## Interface Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// A string-keyed specialization for generic IEntity factories.
    /// </summary>
    public interface IEntityFactoryCatalog : IEntityFactoryCatalog<string, IEntity>
    {
    }

    /// <summary>
    /// Represents a read-only catalog of entity factories, indexed by a key.
    /// </summary>
    /// <typeparam name="TKey">The type of the key used to identify factories</typeparam>
    /// <typeparam name="E">The type of entity each factory creates</typeparam>
    public interface IEntityFactoryCatalog<TKey, E> : IReadOnlyDictionary<TKey, IEntityFactory<E>> 
        where E : IEntity
    {
    }
}
```

## Key Features

### Type Safety
- Generic interface ensures type-safe access to entity factories
- Constrains entity types to `IEntity` implementations
- Supports any key type for flexible organization

### Dictionary Semantics
- Extends `IReadOnlyDictionary` for familiar lookup patterns
- Provides indexed access, enumeration, and existence checking
- Maintains immutable catalog interface for safe shared access

### Factory Management
- Centralizes factory registration and access
- Enables organized factory collections by category, name, or type
- Supports runtime factory discovery and lookup

## Usage Examples

### Basic Factory Catalog Access

```csharp
public class GameEntityManager : MonoBehaviour
{
    [SerializeField] private ScriptableEntityCatalog _entityCatalog;
    
    void Start()
    {
        // Access factory by string key
        if (_entityCatalog.TryGetValue("Enemy", out IEntityFactory<IEntity> factory))
        {
            IEntity enemy = factory.Create();
            enemy.Set("Health", 100);
            enemy.Set("Position", transform.position);
        }
        
        // Enumerate all available factories
        foreach (var kvp in _entityCatalog)
        {
            Debug.Log($"Available factory: {kvp.Key}");
        }
    }
}
```

### Procedural Factory Selection

```csharp
public class WaveSpawner : MonoBehaviour
{
    [SerializeField] private IEntityFactoryCatalog _enemyCatalog;
    [SerializeField] private string[] _waveEnemies;
    
    public void SpawnWave()
    {
        foreach (string enemyType in _waveEnemies)
        {
            if (_enemyCatalog.ContainsKey(enemyType))
            {
                IEntity enemy = _enemyCatalog[enemyType].Create();
                ConfigureEnemy(enemy, enemyType);
            }
        }
    }
    
    private void ConfigureEnemy(IEntity enemy, string type)
    {
        // Apply procedural configuration based on type
        switch (type)
        {
            case "FastEnemy":
                enemy.Set("Speed", 15f);
                enemy.Set("Health", 50);
                break;
            case "TankEnemy":
                enemy.Set("Speed", 5f);
                enemy.Set("Health", 200);
                break;
        }
    }
}
```

### Dynamic Factory Discovery

```csharp
public class EntityBuilder
{
    private readonly IEntityFactoryCatalog _catalog;
    
    public EntityBuilder(IEntityFactoryCatalog catalog)
    {
        _catalog = catalog;
    }
    
    public IEntity CreateRandomEntity()
    {
        // Get random factory from available options
        string[] factoryKeys = _catalog.Keys.ToArray();
        string randomKey = factoryKeys[UnityEngine.Random.Range(0, factoryKeys.Length)];
        
        return _catalog[randomKey].Create();
    }
    
    public List<IEntity> CreateEntitiesByPattern(string pattern)
    {
        var entities = new List<IEntity>();
        
        // Find all factories matching pattern
        foreach (var kvp in _catalog)
        {
            if (kvp.Key.Contains(pattern))
            {
                entities.Add(kvp.Value.Create());
            }
        }
        
        return entities;
    }
}
```

## Integration with Atomic Framework

### Reactive Factory Selection

```csharp
public class FactorySelector : MonoBehaviour
{
    [SerializeField] private IEntityFactoryCatalog _catalog;
    private readonly ReactiveValue<string> _selectedFactory = new();
    
    void Start()
    {
        // React to factory selection changes
        _selectedFactory.Subscribe(OnFactoryChanged);
    }
    
    private void OnFactoryChanged(string factoryKey)
    {
        if (_catalog.TryGetValue(factoryKey, out var factory))
        {
            IEntity entity = factory.Create();
            entity.AddTag("Selected");
            entity.Set("CreationTime", Time.time);
        }
    }
    
    public void SelectFactory(string key)
    {
        if (_catalog.ContainsKey(key))
            _selectedFactory.Value = key;
    }
}
```

### Event-Driven Entity Creation

```csharp
public class EventFactorySystem : MonoBehaviour
{
    [SerializeField] private IEntityFactoryCatalog _catalog;
    
    void Start()
    {
        // Subscribe to game events for entity creation
        GameEvents.OnEnemySpawnRequested += SpawnEnemy;
        GameEvents.OnPowerupRequested += SpawnPowerup;
    }
    
    private void SpawnEnemy(string enemyType, Vector3 position)
    {
        if (_catalog.TryGetValue(enemyType, out var factory))
        {
            IEntity enemy = factory.Create();
            enemy.Set("Position", position);
            enemy.Set("SpawnTime", Time.time);
            enemy.AddTag("Enemy");
        }
    }
    
    private void SpawnPowerup(string powerupType, Vector3 position)
    {
        if (_catalog.TryGetValue(powerupType, out var factory))
        {
            IEntity powerup = factory.Create();
            powerup.Set("Position", position);
            powerup.Set("Duration", 30f);
            powerup.AddTag("Powerup");
        }
    }
}
```

## Implementation Notes

### Catalog Initialization
- Catalogs are typically populated at design time or during application startup
- ScriptableObject-based implementations provide Unity integration
- Runtime catalogs can be built programmatically for dynamic scenarios

### Performance Considerations
- Dictionary lookup provides O(1) access time for factory retrieval
- Consider caching frequently accessed factories for optimal performance
- Immutable interface prevents accidental modification of catalog contents

### Best Practices
- Use meaningful, consistent naming conventions for factory keys
- Group related factories in separate catalogs by domain or system
- Validate factory existence before creation to handle missing entries gracefully
- Consider using enums or constants for factory keys to avoid string typos

## Common Patterns

### Factory Registration Pattern

```csharp
public class EntityFactoryRegistry
{
    private readonly Dictionary<string, IEntityFactory<IEntity>> _factories = new();
    
    public void RegisterFactory(string key, IEntityFactory<IEntity> factory)
    {
        _factories[key] = factory;
    }
    
    public IEntityFactoryCatalog<string, IEntity> BuildCatalog()
    {
        return new ReadOnlyFactoryCatalog(_factories);
    }
}
```

The `IEntityFactoryCatalog` interface provides a clean abstraction for managing collections of entity factories, enabling flexible and organized entity creation workflows within the Atomic framework's reactive architecture.