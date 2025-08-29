# 🧩 MultiEntityPool

A concrete implementation of `IMultiEntityPool` that manages multiple entity pools through a key-based registry system with factory integration. Provides efficient entity creation and lifecycle management across multiple pool types.

## Overview

`MultiEntityPool` serves as a centralized registry for managing different entity pools, using factory patterns for entity creation and automatic pool routing for entity returns. Implements sophisticated pool management with lifecycle event hooks and performance optimization features.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic multi-pool using string keys and IEntity values.
    /// </summary>
    public class MultiEntityPool : MultiEntityPool<string, IEntity>, IMultiEntityPool
    {
        public MultiEntityPool(IMultiEntityFactory<string, IEntity> factory) : base(factory) { }
    }
    
    /// <summary>
    /// Generic multi-pool implementation managing entity pools by key.
    /// </summary>
    /// <typeparam name="TKey">The key type used to identify pools</typeparam>
    /// <typeparam name="E">The entity type managed by pools</typeparam>
    public class MultiEntityPool<TKey, E> : IMultiEntityPool<TKey, E> where E : IEntity
    {
        private readonly Dictionary<TKey, Stack<E>> _pooledEntities = new();
        private readonly Dictionary<E, TKey> _rentEntities = new();
        private readonly IMultiEntityFactory<TKey, E> _factory;

        public MultiEntityPool(IMultiEntityFactory<TKey, E> factory);
        
        public void Init(TKey key, int count);
        public E Rent(TKey key);
        public void Return(E entity);
        public void Dispose();
        
        protected virtual void OnCreate(E entity);
        protected virtual void OnRent(E entity);
        protected virtual void OnReturn(E entity);
        protected virtual void OnDispose(E entity);
    }
}
```

## Key Features

### Factory Integration
- Requires factory for entity creation on demand
- Supports runtime factory changes and updates
- Automatic entity creation when pools are exhausted

### Lifecycle Management
- Virtual hook methods for entity lifecycle events
- Automatic tracking of rented vs pooled entities
- Prevention of duplicate returns and pool contamination

### Memory Efficiency
- Stack-based pooling for efficient LIFO entity reuse
- Dictionary tracking for O(1) return routing
- Automatic pool cleanup and entity disposal

## Usage Examples

### Basic Multi-Pool Setup

```csharp
public class GameEntityPoolSystem : MonoBehaviour
{
    private MultiEntityPool _entityPool;
    private MultiEntityFactory _entityFactory;
    
    void Start()
    {
        // Create factory with entity creation logic
        _entityFactory = new MultiEntityFactory();
        RegisterEntityFactories();
        
        // Create multi-pool with factory
        _entityPool = new MultiEntityPool(_entityFactory);
        
        // Initialize pools for different entity types
        InitializePools();
    }
    
    private void RegisterEntityFactories()
    {
        // Player entity factory
        _entityFactory.Add("Player", new InlineEntityFactory(() =>
        {
            var player = new Entity("Player", 10, 20, 5);
            player.Set("Health", 100f);
            player.Set("MaxHealth", 100f);
            player.Set("Speed", 10f);
            player.Set("Level", 1);
            player.AddTag("Player");
            player.AddTag("Controllable");
            return player;
        }));
        
        // Enemy entity factory
        _entityFactory.Add("Enemy", new InlineEntityFactory(() =>
        {
            var enemy = new Entity("Enemy", 6, 12, 3);
            enemy.Set("Health", 50f);
            enemy.Set("MaxHealth", 50f);
            enemy.Set("Speed", 5f);
            enemy.Set("Damage", 15f);
            enemy.AddTag("Enemy");
            enemy.AddTag("Hostile");
            return enemy;
        }));
        
        // Projectile entity factory
        _entityFactory.Add("Projectile", new InlineEntityFactory(() =>
        {
            var projectile = new Entity("Projectile", 3, 8, 2);
            projectile.Set("Speed", 20f);
            projectile.Set("Damage", 10f);
            projectile.Set("Lifetime", 5f);
            projectile.AddTag("Projectile");
            return projectile;
        }));
        
        // Item entity factory
        _entityFactory.Add("Item", new InlineEntityFactory(() =>
        {
            var item = new Entity("Item", 4, 10, 1);
            item.Set("Value", 100);
            item.Set("Weight", 1f);
            item.AddTag("Item");
            item.AddTag("Collectible");
            return item;
        }));
    }
    
    private void InitializePools()
    {
        // Initialize pools with appropriate sizes
        _entityPool.Init("Player", 2);        // Few players
        _entityPool.Init("Enemy", 20);        // Many enemies
        _entityPool.Init("Projectile", 50);   // Lots of projectiles
        _entityPool.Init("Item", 30);         // Moderate items
        
        Debug.Log("All entity pools initialized");
    }
    
    public IEntity CreatePlayer(Vector3 position, string playerName)
    {
        var player = _entityPool.Rent("Player");
        player.Set("Position", position);
        player.Set("PlayerName", playerName);
        player.Set("SpawnTime", Time.time);
        player.AddTag("Active");
        return player;
    }
    
    public IEntity SpawnEnemy(Vector3 position, int level)
    {
        var enemy = _entityPool.Rent("Enemy");
        enemy.Set("Position", position);
        enemy.Set("Level", level);
        enemy.Set("SpawnTime", Time.time);
        
        // Apply level scaling
        float healthMultiplier = 1f + (level - 1) * 0.2f;
        float baseHealth = 50f;
        enemy.Set("Health", baseHealth * healthMultiplier);
        enemy.Set("MaxHealth", baseHealth * healthMultiplier);
        
        enemy.AddTag("Active");
        enemy.AddTag($"Level{level}");
        
        return enemy;
    }
    
    public IEntity FireProjectile(Vector3 position, Vector3 direction, float damage)
    {
        var projectile = _entityPool.Rent("Projectile");
        projectile.Set("Position", position);
        projectile.Set("Direction", direction);
        projectile.Set("Damage", damage);
        projectile.Set("LaunchTime", Time.time);
        projectile.AddTag("Active");
        
        // Auto-return after lifetime
        float lifetime = projectile.Get<float>("Lifetime");
        StartCoroutine(ReturnAfterDelay(projectile, lifetime));
        
        return projectile;
    }
    
    public IEntity SpawnItem(Vector3 position, string itemType, int value)
    {
        var item = _entityPool.Rent("Item");
        item.Set("Position", position);
        item.Set("ItemType", itemType);
        item.Set("Value", value);
        item.Set("SpawnTime", Time.time);
        item.AddTag("Active");
        return item;
    }
    
    public void ReturnEntity(IEntity entity)
    {
        // Clean entity state before return
        CleanEntityForReturn(entity);
        
        // Return to appropriate pool
        _entityPool.Return(entity);
    }
    
    private void CleanEntityForReturn(IEntity entity)
    {
        // Remove runtime tags
        entity.RemoveTag("Active");
        entity.RemoveTag("Selected");
        entity.RemoveTag("InUse");
        
        // Remove level-specific tags
        for (int i = 1; i <= 10; i++)
        {
            entity.RemoveTag($"Level{i}");
        }
        
        // Unsubscribe from events
        entity.UnsubscribeAll();
        
        // Reset common runtime values
        entity.Set("LastUsedTime", Time.time);
        
        // Keep base configuration intact
    }
    
    private IEnumerator ReturnAfterDelay(IEntity entity, float delay)
    {
        yield return new WaitForSeconds(delay);
        if (entity != null && entity.HasTag("Active"))
        {
            ReturnEntity(entity);
        }
    }
    
    void OnDestroy()
    {
        _entityPool?.Dispose();
    }
}
```

### Advanced Pool with Lifecycle Hooks

```csharp
public class AdvancedMultiEntityPool : MultiEntityPool<string, IEntity>
{
    // Pool statistics
    private readonly Dictionary<string, PoolStats> _poolStats = new();
    
    [System.Serializable]
    public struct PoolStats
    {
        public int totalCreated;
        public int totalRented;
        public int totalReturned;
        public int currentRented;
        public float averageLifetime;
        public float lastRentalTime;
    }
    
    public AdvancedMultiEntityPool(IMultiEntityFactory<string, IEntity> factory) : base(factory)
    {
    }
    
    public override void Init(string key, int count)
    {
        base.Init(key, count);
        
        // Initialize statistics
        _poolStats[key] = new PoolStats
        {
            totalCreated = count,
            lastRentalTime = Time.time
        };
        
        Debug.Log($"Initialized pool '{key}' with {count} entities");
    }
    
    protected override void OnCreate(IEntity entity)
    {
        // Configure newly created entity
        entity.Set("CreatedTime", Time.time);
        entity.Set("PoolCreated", true);
        entity.AddTag("Pooled");
        
        // Update statistics
        string poolKey = GetEntityPoolKey(entity);
        if (_poolStats.TryGetValue(poolKey, out var stats))
        {
            stats.totalCreated++;
            _poolStats[poolKey] = stats;
        }
        
        Debug.Log($"Created new entity for pool: {poolKey}");
    }
    
    protected override void OnRent(IEntity entity)
    {
        // Configure entity for rental
        entity.Set("RentTime", Time.time);
        entity.Set("RentCount", entity.Get<int>("RentCount") + 1);
        entity.AddTag("Rented");
        
        // Update statistics
        string poolKey = GetEntityPoolKey(entity);
        if (_poolStats.TryGetValue(poolKey, out var stats))
        {
            stats.totalRented++;
            stats.currentRented++;
            stats.lastRentalTime = Time.time;
            _poolStats[poolKey] = stats;
        }
        
        // Log high-frequency rentals
        int rentCount = entity.Get<int>("RentCount");
        if (rentCount > 10)
        {
            Debug.Log($"High-frequency entity rented {rentCount} times from {poolKey}");
        }
    }
    
    protected override void OnReturn(IEntity entity)
    {
        // Calculate usage lifetime
        float rentTime = entity.Get<float>("RentTime");
        float lifetime = Time.time - rentTime;
        
        // Update entity state
        entity.Set("LastReturnTime", Time.time);
        entity.Set("TotalUsageTime", entity.Get<float>("TotalUsageTime") + lifetime);
        entity.RemoveTag("Rented");
        
        // Update statistics
        string poolKey = GetEntityPoolKey(entity);
        if (_poolStats.TryGetValue(poolKey, out var stats))
        {
            stats.totalReturned++;
            stats.currentRented = Mathf.Max(0, stats.currentRented - 1);
            stats.averageLifetime = UpdateAverageLifetime(stats.averageLifetime, lifetime, stats.totalReturned);
            _poolStats[poolKey] = stats;
        }
        
        Debug.Log($"Entity returned to {poolKey} after {lifetime:F2}s usage");
    }
    
    protected override void OnDispose(IEntity entity)
    {
        // Log disposal
        string poolKey = GetEntityPoolKey(entity);
        int rentCount = entity.Get<int>("RentCount");
        float totalUsage = entity.Get<float>("TotalUsageTime");
        
        Debug.Log($"Disposing entity from {poolKey}: {rentCount} rentals, {totalUsage:F2}s total usage");
        
        // Clean up entity resources if needed
        entity.UnsubscribeAll();
    }
    
    private string GetEntityPoolKey(IEntity entity)
    {
        // Implementation would need to determine pool key from entity
        // This might be stored during creation or determined from entity properties
        return entity.Get<string>("PoolKey") ?? entity.Name;
    }
    
    private float UpdateAverageLifetime(float currentAverage, float newValue, int count)
    {
        if (count == 1) return newValue;
        return ((currentAverage * (count - 1)) + newValue) / count;
    }
    
    // Public API for monitoring
    public PoolStats GetPoolStats(string poolKey)
    {
        return _poolStats.TryGetValue(poolKey, out var stats) ? stats : new PoolStats();
    }
    
    public Dictionary<string, PoolStats> GetAllPoolStats()
    {
        return new Dictionary<string, PoolStats>(_poolStats);
    }
    
    public void LogPoolStatistics()
    {
        Debug.Log("=== Pool Statistics ===");
        foreach (var kvp in _poolStats)
        {
            var stats = kvp.Value;
            Debug.Log($"{kvp.Key}: Created={stats.totalCreated}, Rented={stats.totalRented}, " +
                     $"Returned={stats.totalReturned}, Active={stats.currentRented}, " +
                     $"AvgLifetime={stats.averageLifetime:F2}s");
        }
    }
}
```

### Pool with Dynamic Expansion

```csharp
public class DynamicMultiEntityPool : MultiEntityPool<string, IEntity>
{
    [System.Serializable]
    public struct PoolConfiguration
    {
        public int initialSize;
        public int maxSize;
        public float expansionThreshold;
        public int expansionAmount;
        public bool allowContraction;
        public float contractionThreshold;
    }
    
    private readonly Dictionary<string, PoolConfiguration> _poolConfigs = new();
    private readonly Dictionary<string, int> _poolSizes = new();
    
    public DynamicMultiEntityPool(IMultiEntityFactory<string, IEntity> factory) : base(factory)
    {
    }
    
    public void ConfigurePool(string key, PoolConfiguration config)
    {
        _poolConfigs[key] = config;
        _poolSizes[key] = config.initialSize;
        
        // Initialize with initial size
        Init(key, config.initialSize);
    }
    
    public override IEntity Rent(string key)
    {
        // Check if expansion is needed before renting
        CheckPoolExpansion(key);
        
        return base.Rent(key);
    }
    
    public override void Return(IEntity entity)
    {
        base.Return(entity);
        
        // Check if contraction is possible after return
        string poolKey = GetEntityPoolKey(entity);
        CheckPoolContraction(poolKey);
    }
    
    private void CheckPoolExpansion(string key)
    {
        if (!_poolConfigs.TryGetValue(key, out var config)) return;
        
        // Get current pool utilization
        int availableCount = GetAvailableCount(key);
        int totalSize = _poolSizes[key];
        
        float utilization = 1f - (float)availableCount / totalSize;
        
        // Expand if above threshold and below max size
        if (utilization > config.expansionThreshold && totalSize < config.maxSize)
        {
            int expansionAmount = Mathf.Min(config.expansionAmount, config.maxSize - totalSize);
            ExpandPool(key, expansionAmount);
        }
    }
    
    private void CheckPoolContraction(string key)
    {
        if (!_poolConfigs.TryGetValue(key, out var config) || !config.allowContraction) return;
        
        // Get current pool utilization
        int availableCount = GetAvailableCount(key);
        int totalSize = _poolSizes[key];
        
        float utilization = 1f - (float)availableCount / totalSize;
        
        // Contract if below threshold and above initial size
        if (utilization < config.contractionThreshold && totalSize > config.initialSize)
        {
            int contractionAmount = Mathf.Min(availableCount / 2, totalSize - config.initialSize);
            ContractPool(key, contractionAmount);
        }
    }
    
    private void ExpandPool(string key, int amount)
    {
        Debug.Log($"Expanding pool '{key}' by {amount} entities");
        
        Init(key, amount);
        _poolSizes[key] += amount;
    }
    
    private void ContractPool(string key, int amount)
    {
        Debug.Log($"Contracting pool '{key}' by {amount} entities");
        
        // Implementation would need to remove entities from pool
        // This is complex as it requires access to the internal stack
        _poolSizes[key] = Mathf.Max(_poolConfigs[key].initialSize, _poolSizes[key] - amount);
    }
    
    private int GetAvailableCount(string key)
    {
        // Implementation would need pool to expose available count
        // This is a limitation of the base class interface
        return 0; // Placeholder
    }
    
    private string GetEntityPoolKey(IEntity entity)
    {
        return entity.Get<string>("PoolKey") ?? entity.Name;
    }
    
    // Monitoring API
    public void LogPoolStatus()
    {
        Debug.Log("=== Dynamic Pool Status ===");
        foreach (var kvp in _poolConfigs)
        {
            string key = kvp.Key;
            var config = kvp.Value;
            int currentSize = _poolSizes[key];
            int available = GetAvailableCount(key);
            
            Debug.Log($"{key}: Size={currentSize}/{config.maxSize}, Available={available}, " +
                     $"Utilization={(1f - (float)available / currentSize):P}");
        }
    }
}
```

## Integration with Atomic Framework

### Reactive Multi-Pool System

```csharp
public class ReactiveMultiPoolManager : MonoBehaviour
{
    private AdvancedMultiEntityPool _multiPool;
    
    [Header("Reactive Pool Monitoring")]
    private readonly ReactiveCollection<string> _activePools = new();
    private readonly ReactiveDictionary<string, int> _poolUtilization = new();
    private readonly ReactiveInt _totalEntitiesRented = new(0);
    
    void Start()
    {
        SetupReactivePool();
        
        // React to pool state changes
        _activePools.OnAdded += OnPoolActivated;
        _activePools.OnRemoved += OnPoolDeactivated;
        _poolUtilization.OnChanged += OnUtilizationChanged;
        _totalEntitiesRented.Subscribe(OnTotalRentedChanged);
        
        // Start monitoring
        InvokeRepeating(nameof(UpdatePoolMetrics), 1f, 2f);
    }
    
    private void SetupReactivePool()
    {
        var factory = new MultiEntityFactory();
        RegisterFactories(factory);
        
        _multiPool = new AdvancedMultiEntityPool(factory);
        
        // Initialize standard pools
        InitializeStandardPools();
    }
    
    private void InitializeStandardPools()
    {
        string[] pools = { "Enemy", "Projectile", "Effect", "Item", "UI" };
        int[] sizes = { 20, 50, 30, 25, 15 };
        
        for (int i = 0; i < pools.Length; i++)
        {
            _multiPool.Init(pools[i], sizes[i]);
            _activePools.Add(pools[i]);
            _poolUtilization[pools[i]] = 0;
        }
    }
    
    public IEntity RentEntity(string poolKey, Dictionary<string, object> initialValues = null)
    {
        var entity = _multiPool.Rent(poolKey);
        
        // Apply initial values if provided
        if (initialValues != null)
        {
            foreach (var kvp in initialValues)
            {
                entity.Set(kvp.Key, kvp.Value);
            }
        }
        
        // Update reactive counters
        _totalEntitiesRented.Value++;
        
        return entity;
    }
    
    public void ReturnEntity(IEntity entity)
    {
        _multiPool.Return(entity);
        _totalEntitiesRented.Value = Mathf.Max(0, _totalEntitiesRented.Value - 1);
    }
    
    private void UpdatePoolMetrics()
    {
        var allStats = _multiPool.GetAllPoolStats();
        
        foreach (var kvp in allStats)
        {
            _poolUtilization[kvp.Key] = kvp.Value.currentRented;
        }
    }
    
    private void OnPoolActivated(string poolKey)
    {
        Debug.Log($"Pool activated: {poolKey}");
        GameEvents.OnPoolActivated?.Invoke(poolKey);
    }
    
    private void OnPoolDeactivated(string poolKey)
    {
        Debug.Log($"Pool deactivated: {poolKey}");
        GameEvents.OnPoolDeactivated?.Invoke(poolKey);
    }
    
    private void OnUtilizationChanged(string poolKey, int oldValue, int newValue)
    {
        float utilizationPercent = (float)newValue / 20f; // Assuming 20 base size
        
        if (utilizationPercent > 0.8f)
        {
            Debug.LogWarning($"High utilization in pool {poolKey}: {utilizationPercent:P}");
        }
        
        GameEvents.OnPoolUtilizationChanged?.Invoke(poolKey, utilizationPercent);
    }
    
    private void OnTotalRentedChanged(int count)
    {
        Debug.Log($"Total rented entities: {count}");
        
        if (count > 100)
        {
            Debug.LogWarning("Very high total entity count - performance may be affected");
        }
        
        GameEvents.OnTotalEntityCountChanged?.Invoke(count);
    }
    
    private void RegisterFactories(MultiEntityFactory factory)
    {
        factory.Add("Enemy", new InlineEntityFactory(() => new Entity("Enemy", 4, 8, 3)));
        factory.Add("Projectile", new InlineEntityFactory(() => new Entity("Projectile", 2, 5, 1)));
        factory.Add("Effect", new InlineEntityFactory(() => new Entity("Effect", 3, 4, 2)));
        factory.Add("Item", new InlineEntityFactory(() => new Entity("Item", 4, 6, 1)));
        factory.Add("UI", new InlineEntityFactory(() => new Entity("UI", 2, 4, 2)));
    }
}
```

## Implementation Notes

### Factory Dependency
- Requires factory for entity creation when pools are empty
- Factory must remain valid for the lifetime of the pool
- Consider factory updates and their impact on existing pools

### Memory Management
- Stack-based pooling provides LIFO entity reuse patterns
- Dictionary tracking enables O(1) return routing
- Dispose method cleans up all pooled and rented entities

### Thread Safety
- Default implementation is not thread-safe
- Consider synchronization for multi-threaded access patterns
- Use concurrent collections if needed for parallel scenarios

## Best Practices

### Pool Configuration
- Size pools based on expected peak usage patterns
- Monitor pool utilization to optimize sizing
- Consider implementing dynamic expansion for unexpected loads

### Entity Lifecycle
- Always clean entity state before return
- Implement proper disposal for entity resources
- Track entity usage patterns for optimization

### Performance Optimization
- Pre-initialize pools at application startup
- Use lifecycle hooks for performance monitoring
- Implement pool statistics for optimization insights

## Common Patterns

### Pool Manager Singleton

```csharp
public class PoolManager : MonoBehaviour
{
    public static PoolManager Instance { get; private set; }
    
    private MultiEntityPool _globalPool;
    
    void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
            InitializeGlobalPool();
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    private void InitializeGlobalPool()
    {
        var factory = new MultiEntityFactory();
        // Configure factory...
        
        _globalPool = new MultiEntityPool(factory);
        // Initialize pools...
    }
    
    public IEntity Rent(string key) => _globalPool.Rent(key);
    public void Return(IEntity entity) => _globalPool.Return(entity);
}
```

### Event-Driven Pool Management

```csharp
public class EventDrivenPoolManager : MonoBehaviour
{
    private MultiEntityPool _eventPool;
    
    void Start()
    {
        var factory = new MultiEntityFactory();
        _eventPool = new MultiEntityPool(factory);
        
        // Subscribe to game events
        GameEvents.OnEnemySpawnRequested += (pos, type) => SpawnEnemy(pos, type);
        GameEvents.OnEffectRequested += (pos, effect) => CreateEffect(pos, effect);
    }
    
    private void SpawnEnemy(Vector3 position, string enemyType)
    {
        var enemy = _eventPool.Rent("Enemy");
        enemy.Set("Position", position);
        enemy.Set("EnemyType", enemyType);
    }
    
    private void CreateEffect(Vector3 position, string effectType)
    {
        var effect = _eventPool.Rent("Effect");
        effect.Set("Position", position);
        effect.Set("EffectType", effectType);
    }
}
```

The `MultiEntityPool` provides a robust, efficient implementation for managing multiple entity pools with sophisticated lifecycle management and performance optimization features within the Atomic framework's reactive architecture.