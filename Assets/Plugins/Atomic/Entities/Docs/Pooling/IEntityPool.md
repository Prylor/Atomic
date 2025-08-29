# 🧩 IEntityPool

A base interface for entity pooling systems that manage reusable entity instances to reduce memory allocation overhead. Provides standardized methods for entity rental, return, and pool initialization.

## Overview

`IEntityPool` defines the contract for object pooling patterns applied to entities, enabling efficient entity reuse through rental and return mechanics. Designed to minimize garbage collection pressure and improve performance in entity-intensive applications.

## Interface Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic alias for IEntityPool&lt;IEntity&gt;.
    /// </summary>
    public interface IEntityPool : IEntityPool<IEntity>
    {
    }

    /// <summary>
    /// Represents a pool for reusing IEntity instances to reduce allocation overhead.
    /// </summary>
    /// <typeparam name="E">The type of entity managed by the pool</typeparam>
    public interface IEntityPool<E> : IDisposable where E : IEntity
    {
        /// <summary>
        /// Retrieves an entity instance from the pool.
        /// </summary>
        E Rent();

        /// <summary>
        /// Returns an entity back to the pool for future reuse.
        /// </summary>
        void Return(E entity);

        /// <summary>
        /// Initializes the pool with a specified number of preallocated entities.
        /// </summary>
        void Init(int initialCount);
    }
}
```

## Key Features

### Memory Efficiency
- Reduces garbage collection pressure through entity reuse
- Preallocates entities to avoid runtime allocation spikes
- Manages entity lifecycle efficiently with rental/return pattern

### Simple Interface
- Minimal method surface for easy implementation
- Generic type support for specific entity types
- IDisposable implementation for proper cleanup

### Performance Optimization
- O(1) rental and return operations in typical implementations
- Configurable preallocation for predictable performance
- Eliminates frequent entity instantiation costs

## Usage Examples

### Basic Pool Usage

```csharp
public class EntityPoolExample : MonoBehaviour
{
    [SerializeField] private EntityPool _entityPool;
    
    void Start()
    {
        // Initialize pool with 10 preallocated entities
        _entityPool.Init(10);
        
        // Rent entities from pool
        var entity1 = _entityPool.Rent();
        var entity2 = _entityPool.Rent();
        
        // Configure entities
        entity1.Set("Position", new Vector3(0, 0, 0));
        entity1.Set("Health", 100f);
        entity1.AddTag("Active");
        
        entity2.Set("Position", new Vector3(5, 0, 0));
        entity2.Set("Health", 150f);
        entity2.AddTag("Active");
        
        // Return entities when no longer needed
        StartCoroutine(ReturnEntitiesAfterDelay(entity1, entity2, 5f));
    }
    
    private IEnumerator ReturnEntitiesAfterDelay(IEntity entity1, IEntity entity2, float delay)
    {
        yield return new WaitForSeconds(delay);
        
        // Clear entity state before returning
        entity1.RemoveTag("Active");
        entity1.UnsubscribeAll();
        
        entity2.RemoveTag("Active");
        entity2.UnsubscribeAll();
        
        // Return to pool for reuse
        _entityPool.Return(entity1);
        _entityPool.Return(entity2);
    }
    
    void OnDestroy()
    {
        // Properly dispose pool resources
        _entityPool?.Dispose();
    }
}
```

### Pool-Based Entity Manager

```csharp
public class PooledEntityManager : MonoBehaviour
{
    [System.Serializable]
    public struct PoolConfiguration
    {
        public string poolName;
        public int initialSize;
        public int maxSize;
        public IEntityPool pool;
    }
    
    [SerializeField] private PoolConfiguration[] _poolConfigurations;
    private readonly Dictionary<string, IEntityPool> _namedPools = new();
    private readonly Dictionary<IEntity, string> _entityToPool = new();
    
    void Start()
    {
        InitializePools();
    }
    
    private void InitializePools()
    {
        foreach (var config in _poolConfigurations)
        {
            config.pool.Init(config.initialSize);
            _namedPools[config.poolName] = config.pool;
            Debug.Log($"Initialized pool '{config.poolName}' with {config.initialSize} entities");
        }
    }
    
    public IEntity RentEntity(string poolName)
    {
        if (_namedPools.TryGetValue(poolName, out var pool))
        {
            var entity = pool.Rent();
            _entityToPool[entity] = poolName;
            
            // Configure entity for use
            entity.Set("RentTime", Time.time);
            entity.Set("PoolSource", poolName);
            entity.AddTag("Pooled");
            
            return entity;
        }
        
        Debug.LogWarning($"Pool '{poolName}' not found");
        return null;
    }
    
    public void ReturnEntity(IEntity entity)
    {
        if (_entityToPool.TryGetValue(entity, out var poolName))
        {
            // Clean entity state
            CleanEntityForReturn(entity);
            
            // Return to appropriate pool
            var pool = _namedPools[poolName];
            pool.Return(entity);
            
            _entityToPool.Remove(entity);
        }
        else
        {
            Debug.LogWarning("Attempted to return entity not managed by pools");
        }
    }
    
    private void CleanEntityForReturn(IEntity entity)
    {
        // Remove runtime-specific data
        entity.UnsubscribeAll();
        entity.RemoveTag("Active");
        entity.RemoveTag("Selected");
        
        // Reset common values
        entity.Set("RentTime", 0f);
        entity.Set("LastUsedTime", Time.time);
        
        // Keep pool identification
        // entity.Set("PoolSource", poolName); // Keep for tracking
    }
    
    public Dictionary<string, int> GetPoolStatistics()
    {
        var stats = new Dictionary<string, int>();
        
        foreach (var kvp in _namedPools)
        {
            // Implementation would depend on pool's ability to report statistics
            // stats[kvp.Key] = kvp.Value.AvailableCount;
        }
        
        return stats;
    }
    
    void OnDestroy()
    {
        foreach (var pool in _namedPools.Values)
        {
            pool.Dispose();
        }
    }
}
```

### Performance-Optimized Pool Usage

```csharp
public class HighPerformanceEntityPool : MonoBehaviour
{
    private IEntityPool<ProjectileEntity> _projectilePool;
    private readonly Queue<ProjectileEntity> _activeProjectiles = new();
    
    [Header("Pool Configuration")]
    [SerializeField] private int _initialPoolSize = 100;
    [SerializeField] private int _maxActiveProjectiles = 50;
    
    [Header("Performance Monitoring")]
    [SerializeField] private bool _enableProfiling = false;
    private int _totalRentals;
    private int _totalReturns;
    private float _averageRentTime;
    
    void Start()
    {
        // Initialize pool with sufficient capacity
        _projectilePool = new SceneEntityPool<ProjectileEntity>();
        _projectilePool.Init(_initialPoolSize);
    }
    
    public void FireProjectile(Vector3 position, Vector3 direction, float speed)
    {
        // Enforce active projectile limit
        if (_activeProjectiles.Count >= _maxActiveProjectiles)
        {
            ReturnOldestProjectile();
        }
        
        // Rent projectile from pool
        var startTime = _enableProfiling ? Time.realtimeSinceStartup : 0f;
        
        var projectile = _projectilePool.Rent();
        
        if (_enableProfiling)
        {
            var rentTime = Time.realtimeSinceStartup - startTime;
            UpdateRentTimeStatistics(rentTime);
        }
        
        // Configure projectile
        ConfigureProjectile(projectile, position, direction, speed);
        
        // Track active projectile
        _activeProjectiles.Enqueue(projectile);
        _totalRentals++;
        
        // Schedule automatic return
        StartCoroutine(ReturnProjectileAfterLifetime(projectile, 5f));
    }
    
    private void ConfigureProjectile(ProjectileEntity projectile, Vector3 position, Vector3 direction, float speed)
    {
        projectile.Set("Position", position);
        projectile.Set("Direction", direction);
        projectile.Set("Speed", speed);
        projectile.Set("LaunchTime", Time.time);
        projectile.Set("Lifetime", 5f);
        
        projectile.AddTag("Active");
        projectile.AddTag("Projectile");
        
        // Subscribe to collision events
        projectile.OnCollision += OnProjectileCollision;
    }
    
    private void ReturnOldestProjectile()
    {
        if (_activeProjectiles.Count > 0)
        {
            var oldest = _activeProjectiles.Dequeue();
            ReturnProjectile(oldest);
        }
    }
    
    private IEnumerator ReturnProjectileAfterLifetime(ProjectileEntity projectile, float lifetime)
    {
        yield return new WaitForSeconds(lifetime);
        
        if (projectile != null && projectile.HasTag("Active"))
        {
            ReturnProjectile(projectile);
        }
    }
    
    private void OnProjectileCollision(ProjectileEntity projectile, IEntity target)
    {
        // Handle collision effects
        CreateImpactEffect(projectile.Get<Vector3>("Position"));
        
        // Return projectile immediately on collision
        ReturnProjectile(projectile);
    }
    
    private void ReturnProjectile(ProjectileEntity projectile)
    {
        if (!projectile.HasTag("Active")) return;
        
        // Clean up projectile state
        projectile.RemoveTag("Active");
        projectile.UnsubscribeAll();
        projectile.OnCollision -= OnProjectileCollision;
        
        // Return to pool
        _projectilePool.Return(projectile);
        _totalReturns++;
        
        // Remove from active tracking if present
        if (_activeProjectiles.Contains(projectile))
        {
            var tempList = _activeProjectiles.ToList();
            tempList.Remove(projectile);
            _activeProjectiles.Clear();
            foreach (var p in tempList)
            {
                _activeProjectiles.Enqueue(p);
            }
        }
    }
    
    private void UpdateRentTimeStatistics(float rentTime)
    {
        _averageRentTime = (_averageRentTime * (_totalRentals - 1) + rentTime) / _totalRentals;
    }
    
    private void CreateImpactEffect(Vector3 position)
    {
        // Create impact effect (could also use pooled effect entities)
    }
    
    void OnDestroy()
    {
        // Return all active projectiles
        while (_activeProjectiles.Count > 0)
        {
            var projectile = _activeProjectiles.Dequeue();
            _projectilePool.Return(projectile);
        }
        
        _projectilePool?.Dispose();
    }
    
    // Performance monitoring
    void OnGUI()
    {
        if (!_enableProfiling) return;
        
        GUI.Label(new Rect(10, 10, 200, 20), $"Total Rentals: {_totalRentals}");
        GUI.Label(new Rect(10, 30, 200, 20), $"Total Returns: {_totalReturns}");
        GUI.Label(new Rect(10, 50, 200, 20), $"Active: {_activeProjectiles.Count}");
        GUI.Label(new Rect(10, 70, 200, 20), $"Avg Rent Time: {_averageRentTime:F6}s");
    }
}
```

## Integration with Atomic Framework

### Reactive Pool Monitoring

```csharp
public class ReactiveEntityPool : MonoBehaviour
{
    private IEntityPool _pool;
    
    [Header("Reactive Monitoring")]
    private readonly ReactiveInt _rentedEntities = new(0);
    private readonly ReactiveInt _availableEntities = new(0);
    private readonly ReactiveFloat _poolUtilization = new(0f);
    
    void Start()
    {
        _pool = new SceneEntityPool();
        _pool.Init(20);
        
        // React to pool statistics changes
        _rentedEntities.Subscribe(OnRentedCountChanged);
        _availableEntities.Subscribe(OnAvailableCountChanged);
        _poolUtilization.Subscribe(OnUtilizationChanged);
        
        // Update statistics periodically
        InvokeRepeating(nameof(UpdatePoolStatistics), 0f, 1f);
    }
    
    public IEntity RentEntity()
    {
        var entity = _pool.Rent();
        UpdatePoolStatistics();
        return entity;
    }
    
    public void ReturnEntity(IEntity entity)
    {
        _pool.Return(entity);
        UpdatePoolStatistics();
    }
    
    private void UpdatePoolStatistics()
    {
        // These would require pool implementation to expose statistics
        // _rentedEntities.Value = _pool.RentedCount;
        // _availableEntities.Value = _pool.AvailableCount;
        // _poolUtilization.Value = (float)_pool.RentedCount / _pool.TotalCount;
    }
    
    private void OnRentedCountChanged(int count)
    {
        Debug.Log($"Rented entities: {count}");
        
        if (count > 15)
        {
            Debug.LogWarning("Pool utilization high - consider expanding");
        }
    }
    
    private void OnAvailableCountChanged(int count)
    {
        Debug.Log($"Available entities: {count}");
        
        if (count < 3)
        {
            Debug.LogWarning("Pool running low - performance may degrade");
        }
    }
    
    private void OnUtilizationChanged(float utilization)
    {
        Debug.Log($"Pool utilization: {utilization:P}");
        
        // Trigger pool expansion or contraction based on utilization
        if (utilization > 0.9f)
        {
            // Consider pool expansion
        }
        else if (utilization < 0.1f)
        {
            // Consider pool contraction
        }
    }
}
```

## Implementation Notes

### Thread Safety
- Default interface doesn't guarantee thread safety
- Implementations should document thread safety characteristics
- Consider concurrent collections for multi-threaded scenarios

### Entity State Management
- Entities should be cleaned before return to pool
- Pool implementations may apply automatic state reset
- Consider entity validation before rental

### Memory Management
- IDisposable implementation ensures proper cleanup
- Pools should dispose all managed entities on disposal
- Consider memory growth patterns for long-running applications

## Best Practices

### Pool Sizing
- Initialize pools based on expected peak usage
- Monitor pool utilization to optimize sizing
- Consider dynamic pool expansion strategies

### Entity Lifecycle
- Always pair Rent calls with Return calls
- Clean entity state thoroughly before return
- Validate entity state after rental

### Performance Optimization
- Profile pool operations in performance-critical paths
- Consider pool pre-warming for predictable performance
- Monitor garbage collection impact

## Common Patterns

### Pool Factory Pattern

```csharp
public interface IEntityPoolFactory
{
    IEntityPool<T> CreatePool<T>(int initialSize) where T : IEntity;
}

public class StandardEntityPoolFactory : IEntityPoolFactory
{
    public IEntityPool<T> CreatePool<T>(int initialSize) where T : IEntity
    {
        var pool = new SceneEntityPool<T>();
        pool.Init(initialSize);
        return pool;
    }
}
```

### Pooled Entity Wrapper

```csharp
public struct PooledEntity : IDisposable
{
    private readonly IEntity _entity;
    private readonly IEntityPool _pool;
    
    public PooledEntity(IEntity entity, IEntityPool pool)
    {
        _entity = entity;
        _pool = pool;
    }
    
    public IEntity Entity => _entity;
    
    public void Dispose()
    {
        _pool.Return(_entity);
    }
}

// Usage with using statement for automatic return
using var pooledEntity = new PooledEntity(pool.Rent(), pool);
// Use pooledEntity.Entity
// Automatically returned to pool when using block exits
```

The `IEntityPool` interface provides a foundation for efficient entity reuse patterns, enabling high-performance entity management within the Atomic framework's reactive architecture.