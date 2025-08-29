# 🧩 IMultiEntityPool

A registry interface for managing multiple entity pools identified by keys, enabling centralized pooling across different entity types or categories. Provides unified access to heterogeneous entity pools through a single management interface.

## Overview

`IMultiEntityPool` extends the basic pooling concept to support multiple pools within a single registry, allowing different entity types or categories to be managed separately while providing unified access patterns. Essential for complex systems requiring organized pool management.

## Interface Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic alias using string keys and IEntity values.
    /// </summary>
    public interface IMultiEntityPool : IMultiEntityPool<string, IEntity>
    {
    }
    
    /// <summary>
    /// Registry interface for managing multiple entity pools by key.
    /// </summary>
    /// <typeparam name="TKey">The key type used to identify individual pools</typeparam>
    /// <typeparam name="E">The type of entity managed by the pools</typeparam>
    public interface IMultiEntityPool<in TKey, E> : IDisposable where E : IEntity
    {
        /// <summary>
        /// Initializes the pool associated with the specified key.
        /// </summary>
        void Init(TKey key, int count);

        /// <summary>
        /// Rents an entity from the pool associated with the specified key.
        /// </summary>
        E Rent(TKey key);

        /// <summary>
        /// Returns an entity to its corresponding pool.
        /// </summary>
        void Return(E entity);
    }
}
```

## Key Features

### Multi-Pool Management
- Manages multiple pools through a unified interface
- Key-based pool identification and access
- Automatic pool creation and management

### Unified Access Pattern
- Single interface for accessing heterogeneous pools
- Simplified entity rental across different types
- Centralized pool lifecycle management

### Automatic Pool Routing
- Entities remember their source pool for automatic return
- No need to specify pool key when returning entities
- Prevents cross-pool contamination

## Usage Examples

### Basic Multi-Pool Setup

```csharp
public class GameEntityPoolManager : MonoBehaviour
{
    private IMultiEntityPool _multiPool;
    
    [Header("Pool Configuration")]
    [SerializeField] private PoolConfig[] _poolConfigs;
    
    [System.Serializable]
    public struct PoolConfig
    {
        public string poolKey;
        public int initialSize;
        public string description;
    }
    
    void Start()
    {
        // Initialize multi-pool with factory
        var factory = new MultiEntityFactory();
        RegisterEntityFactories(factory);
        
        _multiPool = new MultiEntityPool(factory);
        
        // Initialize individual pools
        foreach (var config in _poolConfigs)
        {
            _multiPool.Init(config.poolKey, config.initialSize);
            Debug.Log($"Initialized {config.poolKey} pool with {config.initialSize} entities");
        }
    }
    
    private void RegisterEntityFactories(MultiEntityFactory factory)
    {
        // Register different entity types
        factory.Add("Enemy", new InlineEntityFactory(() => CreateEnemyEntity()));
        factory.Add("Projectile", new InlineEntityFactory(() => CreateProjectileEntity()));
        factory.Add("Effect", new InlineEntityFactory(() => CreateEffectEntity()));
        factory.Add("Item", new InlineEntityFactory(() => CreateItemEntity()));
        factory.Add("UI", new InlineEntityFactory(() => CreateUIEntity()));
    }
    
    public IEntity SpawnEnemy(Vector3 position)
    {
        var enemy = _multiPool.Rent("Enemy");
        enemy.Set("Position", position);
        enemy.Set("SpawnTime", Time.time);
        enemy.Set("Health", 100f);
        enemy.AddTag("Active");
        return enemy;
    }
    
    public IEntity FireProjectile(Vector3 position, Vector3 direction)
    {
        var projectile = _multiPool.Rent("Projectile");
        projectile.Set("Position", position);
        projectile.Set("Direction", direction);
        projectile.Set("Speed", 20f);
        projectile.AddTag("Active");
        
        // Auto-return projectile after 5 seconds
        StartCoroutine(ReturnAfterDelay(projectile, 5f));
        
        return projectile;
    }
    
    public IEntity CreateEffect(Vector3 position, string effectType)
    {
        var effect = _multiPool.Rent("Effect");
        effect.Set("Position", position);
        effect.Set("EffectType", effectType);
        effect.Set("Duration", 2f);
        effect.AddTag("Visual");
        
        StartCoroutine(ReturnAfterDelay(effect, 2f));
        
        return effect;
    }
    
    public void ReturnEntity(IEntity entity)
    {
        // Clean entity state
        entity.RemoveTag("Active");
        entity.RemoveTag("Visual");
        entity.UnsubscribeAll();
        
        // Return to appropriate pool automatically
        _multiPool.Return(entity);
    }
    
    private IEnumerator ReturnAfterDelay(IEntity entity, float delay)
    {
        yield return new WaitForSeconds(delay);
        ReturnEntity(entity);
    }
    
    private IEntity CreateEnemyEntity() => CreateBaseEntity("Enemy", 5, 10, 3);
    private IEntity CreateProjectileEntity() => CreateBaseEntity("Projectile", 2, 6, 1);
    private IEntity CreateEffectEntity() => CreateBaseEntity("Effect", 3, 4, 1);
    private IEntity CreateItemEntity() => CreateBaseEntity("Item", 4, 8, 2);
    private IEntity CreateUIEntity() => CreateBaseEntity("UI", 2, 5, 1);
    
    private IEntity CreateBaseEntity(string name, int tagCount, int valueCount, int behaviorCount)
    {
        var entity = new Entity(name, tagCount, valueCount, behaviorCount);
        entity.Set("CreatedTime", Time.time);
        entity.AddTag(name);
        return entity;
    }
    
    void OnDestroy()
    {
        _multiPool?.Dispose();
    }
}
```

### Type-Safe Multi-Pool System

```csharp
public enum EntityType
{
    Character,
    Weapon,
    Armor,
    Consumable,
    Environment,
    UI
}

public class TypedMultiEntityPool : MonoBehaviour
{
    private IMultiEntityPool<EntityType, IEntity> _typedPool;
    private readonly Dictionary<EntityType, int> _poolSizes = new()
    {
        { EntityType.Character, 20 },
        { EntityType.Weapon, 50 },
        { EntityType.Armor, 30 },
        { EntityType.Consumable, 100 },
        { EntityType.Environment, 200 },
        { EntityType.UI, 40 }
    };
    
    void Start()
    {
        // Setup factory with enum-based keys
        var factory = new MultiEntityFactory<EntityType, IEntity>();
        RegisterTypedFactories(factory);
        
        _typedPool = new MultiEntityPool<EntityType, IEntity>(factory);
        
        // Initialize pools by type
        foreach (var kvp in _poolSizes)
        {
            _typedPool.Init(kvp.Key, kvp.Value);
        }
    }
    
    private void RegisterTypedFactories(MultiEntityFactory<EntityType, IEntity> factory)
    {
        factory.Add(EntityType.Character, new InlineEntityFactory(() => CreateCharacterEntity()));
        factory.Add(EntityType.Weapon, new InlineEntityFactory(() => CreateWeaponEntity()));
        factory.Add(EntityType.Armor, new InlineEntityFactory(() => CreateArmorEntity()));
        factory.Add(EntityType.Consumable, new InlineEntityFactory(() => CreateConsumableEntity()));
        factory.Add(EntityType.Environment, new InlineEntityFactory(() => CreateEnvironmentEntity()));
        factory.Add(EntityType.UI, new InlineEntityFactory(() => CreateUIEntity()));
    }
    
    public IEntity CreateCharacter(string characterName, float health, float speed)
    {
        var character = _typedPool.Rent(EntityType.Character);
        character.Set("Name", characterName);
        character.Set("Health", health);
        character.Set("MaxHealth", health);
        character.Set("Speed", speed);
        character.AddTag("Character");
        character.AddTag("Living");
        return character;
    }
    
    public IEntity CreateWeapon(string weaponName, float damage, float range)
    {
        var weapon = _typedPool.Rent(EntityType.Weapon);
        weapon.Set("Name", weaponName);
        weapon.Set("Damage", damage);
        weapon.Set("Range", range);
        weapon.Set("Durability", 100f);
        weapon.AddTag("Weapon");
        weapon.AddTag("Equipment");
        return weapon;
    }
    
    public IEntity CreateConsumable(string itemName, float effectValue, float duration)
    {
        var item = _typedPool.Rent(EntityType.Consumable);
        item.Set("Name", itemName);
        item.Set("EffectValue", effectValue);
        item.Set("Duration", duration);
        item.Set("UsesRemaining", 1);
        item.AddTag("Consumable");
        item.AddTag("Item");
        return item;
    }
    
    public void ReturnToPool(IEntity entity)
    {
        // Cleanup entity state
        CleanEntityForReturn(entity);
        
        // Return to appropriate typed pool
        _typedPool.Return(entity);
    }
    
    private void CleanEntityForReturn(IEntity entity)
    {
        // Remove temporary tags
        entity.RemoveTag("Active");
        entity.RemoveTag("Selected");
        entity.RemoveTag("InUse");
        
        // Unsubscribe from all events
        entity.UnsubscribeAll();
        
        // Reset common runtime values
        entity.Set("LastUsedTime", Time.time);
        
        // Keep type identification and base properties
    }
    
    private IEntity CreateCharacterEntity() => new Entity("Character", 8, 15, 5);
    private IEntity CreateWeaponEntity() => new Entity("Weapon", 5, 10, 3);
    private IEntity CreateArmorEntity() => new Entity("Armor", 4, 8, 2);
    private IEntity CreateConsumableEntity() => new Entity("Consumable", 3, 6, 2);
    private IEntity CreateEnvironmentEntity() => new Entity("Environment", 6, 8, 1);
    private IEntity CreateUIEntity() => new Entity("UI", 4, 6, 3);
}
```

### Dynamic Pool Management System

```csharp
public class DynamicMultiPoolManager : MonoBehaviour
{
    private IMultiEntityPool _dynamicPool;
    private readonly Dictionary<string, PoolMetrics> _poolMetrics = new();
    
    [System.Serializable]
    public struct PoolMetrics
    {
        public int totalRentals;
        public int currentRented;
        public int peakUsage;
        public float averageUsageTime;
        public float lastUsedTime;
    }
    
    [Header("Dynamic Management")]
    [SerializeField] private int _defaultPoolSize = 10;
    [SerializeField] private float _expansionThreshold = 0.8f;
    [SerializeField] private float _contractionThreshold = 0.2f;
    [SerializeField] private int _maxPoolSize = 100;
    [SerializeField] private int _minPoolSize = 5;
    
    void Start()
    {
        var factory = new MultiEntityFactory();
        _dynamicPool = new MultiEntityPool(factory);
        
        // Start monitoring pools
        InvokeRepeating(nameof(MonitorPools), 1f, 1f);
    }
    
    public IEntity RentEntity(string poolKey)
    {
        // Create pool if it doesn't exist
        if (!_poolMetrics.ContainsKey(poolKey))
        {
            CreateNewPool(poolKey);
        }
        
        // Rent entity
        var entity = _dynamicPool.Rent(poolKey);
        
        // Update metrics
        UpdateRentalMetrics(poolKey);
        
        return entity;
    }
    
    public void ReturnEntity(IEntity entity)
    {
        // Extract pool key from entity
        string poolKey = entity.Get<string>("PoolKey");
        
        // Update metrics
        UpdateReturnMetrics(poolKey);
        
        // Return to pool
        _dynamicPool.Return(entity);
    }
    
    private void CreateNewPool(string poolKey)
    {
        Debug.Log($"Creating new dynamic pool: {poolKey}");
        
        // Register factory for new pool
        RegisterDynamicFactory(poolKey);
        
        // Initialize pool
        _dynamicPool.Init(poolKey, _defaultPoolSize);
        
        // Initialize metrics
        _poolMetrics[poolKey] = new PoolMetrics
        {
            lastUsedTime = Time.time
        };
    }
    
    private void RegisterDynamicFactory(string poolKey)
    {
        // This would require access to the factory to add new keys
        // Implementation depends on the specific multi-pool architecture
    }
    
    private void UpdateRentalMetrics(string poolKey)
    {
        if (_poolMetrics.TryGetValue(poolKey, out var metrics))
        {
            metrics.totalRentals++;
            metrics.currentRented++;
            metrics.peakUsage = Mathf.Max(metrics.peakUsage, metrics.currentRented);
            metrics.lastUsedTime = Time.time;
            
            _poolMetrics[poolKey] = metrics;
        }
    }
    
    private void UpdateReturnMetrics(string poolKey)
    {
        if (_poolMetrics.TryGetValue(poolKey, out var metrics))
        {
            metrics.currentRented = Mathf.Max(0, metrics.currentRented - 1);
            _poolMetrics[poolKey] = metrics;
        }
    }
    
    private void MonitorPools()
    {
        foreach (var kvp in _poolMetrics.ToList())
        {
            string poolKey = kvp.Key;
            PoolMetrics metrics = kvp.Value;
            
            // Check for expansion needs
            float utilization = (float)metrics.currentRented / _maxPoolSize; // Approximate
            
            if (utilization > _expansionThreshold && CanExpandPool(poolKey))
            {
                ExpandPool(poolKey);
            }
            else if (utilization < _contractionThreshold && CanContractPool(poolKey))
            {
                ConsiderPoolContraction(poolKey, metrics);
            }
            
            // Check for unused pools
            if (Time.time - metrics.lastUsedTime > 60f && metrics.currentRented == 0)
            {
                ConsiderPoolRemoval(poolKey);
            }
        }
    }
    
    private bool CanExpandPool(string poolKey)
    {
        // Logic to determine if pool can be expanded
        return true; // Simplified
    }
    
    private bool CanContractPool(string poolKey)
    {
        // Logic to determine if pool can be contracted
        return true; // Simplified
    }
    
    private void ExpandPool(string poolKey)
    {
        Debug.Log($"Expanding pool: {poolKey}");
        _dynamicPool.Init(poolKey, _defaultPoolSize); // Add more entities
    }
    
    private void ConsiderPoolContraction(string poolKey, PoolMetrics metrics)
    {
        if (metrics.currentRented == 0)
        {
            Debug.Log($"Pool {poolKey} underutilized, considering contraction");
        }
    }
    
    private void ConsiderPoolRemoval(string poolKey)
    {
        Debug.Log($"Pool {poolKey} unused for extended period, considering removal");
        // Implementation would need to support pool removal
    }
    
    // Debug information
    void OnGUI()
    {
        int y = 10;
        GUI.Label(new Rect(10, y, 300, 20), "Dynamic Pool Statistics:");
        y += 25;
        
        foreach (var kvp in _poolMetrics)
        {
            var metrics = kvp.Value;
            GUI.Label(new Rect(10, y, 400, 20), 
                $"{kvp.Key}: Rented={metrics.currentRented}, Peak={metrics.peakUsage}, Total={metrics.totalRentals}");
            y += 20;
        }
    }
}
```

## Integration with Atomic Framework

### Reactive Multi-Pool Monitoring

```csharp
public class ReactiveMultiPoolSystem : MonoBehaviour
{
    private IMultiEntityPool _multiPool;
    
    [Header("Reactive Pool Monitoring")]
    private readonly ReactiveCollection<string> _activePools = new();
    private readonly ReactiveDictionary<string, int> _poolUtilization = new();
    private readonly ReactiveFloat _totalMemoryUsage = new();
    
    void Start()
    {
        SetupMultiPool();
        
        // React to pool changes
        _activePools.OnAdded += OnPoolAdded;
        _activePools.OnRemoved += OnPoolRemoved;
        _poolUtilization.OnChanged += OnUtilizationChanged;
        
        // Monitor system periodically
        InvokeRepeating(nameof(UpdatePoolMetrics), 1f, 1f);
    }
    
    private void SetupMultiPool()
    {
        var factory = new MultiEntityFactory();
        RegisterFactories(factory);
        _multiPool = new MultiEntityPool(factory);
        
        // Initialize standard pools
        string[] standardPools = { "Enemy", "Projectile", "Effect", "UI" };
        foreach (string poolKey in standardPools)
        {
            _multiPool.Init(poolKey, 20);
            _activePools.Add(poolKey);
            _poolUtilization[poolKey] = 0;
        }
    }
    
    public IEntity RentFromPool(string poolKey)
    {
        var entity = _multiPool.Rent(poolKey);
        entity.Set("RentTime", Time.time);
        
        // Update reactive metrics
        UpdatePoolMetrics();
        
        return entity;
    }
    
    public void ReturnToPool(IEntity entity)
    {
        float rentDuration = Time.time - entity.Get<float>("RentTime");
        
        _multiPool.Return(entity);
        
        // Update metrics
        UpdatePoolMetrics();
        
        // Log usage patterns
        Debug.Log($"Entity used for {rentDuration:F2} seconds");
    }
    
    private void UpdatePoolMetrics()
    {
        float totalMemory = 0f;
        
        foreach (string poolKey in _activePools)
        {
            // Update utilization (implementation-dependent)
            // _poolUtilization[poolKey] = CalculateUtilization(poolKey);
            // totalMemory += CalculatePoolMemoryUsage(poolKey);
        }
        
        _totalMemoryUsage.Value = totalMemory;
    }
    
    private void OnPoolAdded(string poolKey)
    {
        Debug.Log($"New pool activated: {poolKey}");
        GameEvents.OnPoolActivated?.Invoke(poolKey);
    }
    
    private void OnPoolRemoved(string poolKey)
    {
        Debug.Log($"Pool deactivated: {poolKey}");
        GameEvents.OnPoolDeactivated?.Invoke(poolKey);
    }
    
    private void OnUtilizationChanged(string poolKey, int oldValue, int newValue)
    {
        Debug.Log($"Pool {poolKey} utilization: {oldValue} -> {newValue}");
        
        if (newValue > 15)
        {
            Debug.LogWarning($"High utilization in pool {poolKey}");
        }
    }
    
    private void RegisterFactories(MultiEntityFactory factory)
    {
        factory.Add("Enemy", new InlineEntityFactory(() => new Entity("Enemy", 4, 8, 3)));
        factory.Add("Projectile", new InlineEntityFactory(() => new Entity("Projectile", 2, 5, 1)));
        factory.Add("Effect", new InlineEntityFactory(() => new Entity("Effect", 3, 4, 2)));
        factory.Add("UI", new InlineEntityFactory(() => new Entity("UI", 2, 6, 2)));
    }
}
```

## Implementation Notes

### Pool Identification
- Entities must retain pool identification for automatic return routing
- Key-based pool access requires consistent key management
- Pool creation is typically lazy but can be pre-initialized

### Memory Management
- Dispose method should clean up all managed pools
- Individual pools may need separate disposal considerations
- Monitor total memory usage across all pools

### Performance Characteristics
- O(1) pool lookup through dictionary-based implementations
- Entity return requires pool identification lookup
- Consider caching frequently accessed pools

## Best Practices

### Pool Organization
- Use meaningful, consistent pool keys
- Group related entities in appropriate pools
- Document pool purposes and expected usage patterns

### Key Management
- Use enums or constants for pool keys to avoid typos
- Consider hierarchical naming schemes for complex systems
- Validate pool keys before operations

### Monitoring and Optimization
- Track pool utilization metrics for optimization
- Monitor cross-pool usage patterns
- Implement dynamic pool sizing based on usage

## Common Patterns

### Pool Registry Pattern

```csharp
public static class PoolRegistry
{
    public const string ENEMIES = "Enemies";
    public const string PROJECTILES = "Projectiles";
    public const string EFFECTS = "Effects";
    public const string UI_ELEMENTS = "UIElements";
    
    public static readonly string[] ALL_POOLS = 
    {
        ENEMIES, PROJECTILES, EFFECTS, UI_ELEMENTS
    };
}
```

### Pool Event System

```csharp
public class PoolEventSystem
{
    public static event Action<string, IEntity> OnEntityRented;
    public static event Action<string, IEntity> OnEntityReturned;
    public static event Action<string, int> OnPoolInitialized;
    
    public static void NotifyEntityRented(string poolKey, IEntity entity)
    {
        OnEntityRented?.Invoke(poolKey, entity);
    }
    
    public static void NotifyEntityReturned(string poolKey, IEntity entity)
    {
        OnEntityReturned?.Invoke(poolKey, entity);
    }
    
    public static void NotifyPoolInitialized(string poolKey, int size)
    {
        OnPoolInitialized?.Invoke(poolKey, size);
    }
}
```

The `IMultiEntityPool` interface provides powerful abstractions for managing complex pooling scenarios, enabling organized and efficient multi-pool systems within the Atomic framework's reactive architecture.