# 🧩 EntityPool

`EntityPool` is a high-performance object pool implementation for entities in the Atomic framework. It reduces memory allocation overhead by reusing entity instances, making it essential for performance-critical scenarios with frequent entity creation and destruction.

## Key Features

- **Memory Efficiency** – Reuses entity instances to reduce GC pressure
- **Factory Integration** – Works seamlessly with `IEntityFactory` implementations
- **Lifecycle Hooks** – Virtual methods for customizing pool behavior
- **Generic Type Safety** – Strongly typed pools with `EntityPool<T>`
- **Automatic Tracking** – Monitors rented vs. pooled entities
- **Pre-allocation Support** – Initialize with desired pool size
- **Procedural Integration** – Designed for Atomic's procedural patterns

---

## Class Definition

### Generic EntityPool
```csharp
public class EntityPool<E> : IEntityPool<E> where E : IEntity
{
    // Core pool operations
    public E Rent();
    public void Return(E entity);
    public void Init(int count);
    public void Dispose();
    
    // Lifecycle hooks
    protected virtual void OnCreate(E entity);
    protected virtual void OnDispose(E entity);
    protected virtual void OnRent(E entity);
    protected virtual void OnReturn(E entity);
}
```

### Non-Generic EntityPool
```csharp
public class EntityPool : EntityPool<IEntity>, IEntityPool
{
    public EntityPool(IEntityFactory<IEntity> factory) : base(factory);
}
```

## Constructor

```csharp
public EntityPool(IEntityFactory<E> factory)
```
- **factory**: Factory used to create new entity instances when pool is empty
- Throws `ArgumentNullException` if factory is null

## Core Methods

### Init
```csharp
public void Init(int count)
```
- Pre-populates the pool with specified number of entities
- Creates entities using the provided factory
- Calls `OnCreate` hook for each created entity
- Recommended for performance optimization

### Rent
```csharp
public E Rent()
```
- Retrieves an entity from the pool or creates new one if empty
- Marks entity as rented (tracked internally)
- Calls `OnRent` hook before returning
- Returns ready-to-use entity instance

### Return
```csharp
public void Return(E entity)
```
- Returns entity to pool for future reuse
- Only accepts entities that were previously rented
- Calls `OnReturn` hook before pooling
- Logs warning if entity wasn't rented from this pool

### Dispose
```csharp
public void Dispose()
```
- Clears all pooled and rented entities
- Calls `OnDispose` hook for each entity
- Releases all internal collections
- Pool becomes unusable after disposal

## Lifecycle Hooks

### OnCreate
```csharp
protected virtual void OnCreate(E entity)
```
- Called when new entity is created by factory
- Allows custom initialization
- Called during `Init()` and when pool is empty

### OnRent
```csharp
protected virtual void OnRent(E entity)
```
- Called when entity is rented from pool
- Allows activation or reset logic
- Called after internal tracking is updated

### OnReturn
```csharp
protected virtual void OnReturn(E entity)
```
- Called when entity is returned to pool
- Allows cleanup or reset logic
- Called before adding back to pool

### OnDispose
```csharp
protected virtual void OnDispose(E entity)
```
- Called during pool disposal
- Allows final cleanup of entities
- Called for both pooled and rented entities

## Implementation Examples

### Basic Entity Pool
```csharp
public class GameEntityPool
{
    private readonly EntityPool<Entity> pool;
    
    public GameEntityPool(IEntityFactory<Entity> factory)
    {
        pool = new EntityPool<Entity>(factory);
        
        // Pre-populate for better performance
        pool.Init(50);
    }
    
    public Entity GetEntity()
    {
        return pool.Rent();
    }
    
    public void ReturnEntity(Entity entity)
    {
        pool.Return(entity);
    }
}
```

### Custom Entity Pool with Reset Logic
```csharp
public class ProjectilePool : EntityPool<Entity>
{
    public ProjectilePool(IEntityFactory<Entity> factory) : base(factory)
    {
        Init(100); // Pre-create projectiles
    }
    
    protected override void OnCreate(Entity entity)
    {
        // Set up projectile defaults
        entity.AddTag(EntityTags.PROJECTILE);
        entity.AddValue(EntityNames.SPEED, 10.0f);
        entity.AddBehaviour(new ProjectileMovementBehaviour());
    }
    
    protected override void OnRent(Entity entity)
    {
        // Activate projectile
        entity.Spawn();
    }
    
    protected override void OnReturn(Entity entity)
    {
        // Clean up projectile state
        entity.Despawn();
        entity.DelValue(EntityNames.DIRECTION);
        entity.DelValue(EntityNames.LIFETIME);
        entity.DelValue(EntityNames.DAMAGE);
        
        // Reset position
        entity.SetValue(EntityNames.POSITION, Vector3.zero);
    }
    
    protected override void OnDispose(Entity entity)
    {
        entity.Dispose();
    }
}
```

### Specialized Pool Types
```csharp
public class EnemyPool : EntityPool<Entity>
{
    private readonly EnemyConfiguration config;
    
    public EnemyPool(IEntityFactory<Entity> factory, EnemyConfiguration config) 
        : base(factory)
    {
        this.config = config;
        Init(config.poolSize);
    }
    
    protected override void OnCreate(Entity entity)
    {
        // Apply enemy configuration
        entity.AddTag(EntityTags.ENEMY);
        entity.SetValue(EntityNames.HEALTH, config.baseHealth);
        entity.SetValue(EntityNames.DAMAGE, config.baseDamage);
        
        // Add AI behavior
        entity.AddBehaviour(new EnemyAIBehaviour());
        entity.AddBehaviour(new HealthBehaviour());
    }
    
    protected override void OnRent(Entity entity)
    {
        // Randomize enemy attributes
        var healthVariation = UnityEngine.Random.Range(-20, 21);
        var baseHealth = config.baseHealth + healthVariation;
        
        entity.SetValue(EntityNames.HEALTH, baseHealth);
        entity.SetValue(EntityNames.MAX_HEALTH, baseHealth);
        entity.SetValue(EntityNames.ALIVE, true);
        
        entity.Spawn();
    }
    
    protected override void OnReturn(Entity entity)
    {
        // Reset enemy state
        entity.Despawn();
        entity.DelTag(EntityTags.DEAD);
        entity.ClearValues();
        
        // Reapply base configuration
        entity.SetValue(EntityNames.HEALTH, config.baseHealth);
        entity.SetValue(EntityNames.DAMAGE, config.baseDamage);
    }
}
```

## Usage Patterns

### Pool Manager Pattern
```csharp
public class EntityPoolManager
{
    private readonly Dictionary<string, IEntityPool> pools;
    
    public EntityPoolManager()
    {
        pools = new Dictionary<string, IEntityPool>();
    }
    
    public void RegisterPool(string key, IEntityPool pool)
    {
        pools[key] = pool;
    }
    
    public T RentEntity<T>(string poolKey) where T : IEntity
    {
        if (pools.TryGetValue(poolKey, out var pool) && pool is IEntityPool<T> typedPool)
        {
            return typedPool.Rent();
        }
        
        throw new ArgumentException($"No suitable pool found for key: {poolKey}");
    }
    
    public void ReturnEntity<T>(string poolKey, T entity) where T : IEntity
    {
        if (pools.TryGetValue(poolKey, out var pool) && pool is IEntityPool<T> typedPool)
        {
            typedPool.Return(entity);
        }
    }
    
    public void InitializePools()
    {
        foreach (var pool in pools.Values)
        {
            pool.Init(20); // Default pool size
        }
    }
    
    public void Dispose()
    {
        foreach (var pool in pools.Values)
        {
            pool.Dispose();
        }
        pools.Clear();
    }
}
```

### Game Object Pool Integration
```csharp
public class SceneEntityPool : EntityPool<SceneEntity>
{
    private readonly Transform poolContainer;
    
    public SceneEntityPool(IEntityFactory<SceneEntity> factory, Transform container) 
        : base(factory)
    {
        poolContainer = container;
    }
    
    protected override void OnCreate(SceneEntity entity)
    {
        // Parent to pool container and deactivate
        entity.transform.SetParent(poolContainer);
        entity.gameObject.SetActive(false);
    }
    
    protected override void OnRent(SceneEntity entity)
    {
        // Activate and remove from container
        entity.gameObject.SetActive(true);
        entity.transform.SetParent(null);
        entity.Spawn();
    }
    
    protected override void OnReturn(SceneEntity entity)
    {
        // Deactivate and return to container
        entity.Despawn();
        entity.gameObject.SetActive(false);
        entity.transform.SetParent(poolContainer);
        entity.transform.localPosition = Vector3.zero;
        entity.transform.localRotation = Quaternion.identity;
    }
    
    protected override void OnDispose(SceneEntity entity)
    {
        if (entity.gameObject != null)
        {
            Object.Destroy(entity.gameObject);
        }
    }
}
```

### Multi-Pool System
```csharp
public class CombatPoolSystem
{
    private readonly EntityPool<Entity> bulletPool;
    private readonly EntityPool<Entity> explosionPool;
    private readonly EntityPool<Entity> pickupPool;
    
    public CombatPoolSystem(
        IEntityFactory<Entity> bulletFactory,
        IEntityFactory<Entity> explosionFactory,
        IEntityFactory<Entity> pickupFactory)
    {
        bulletPool = new BulletPool(bulletFactory);
        explosionPool = new ExplosionPool(explosionFactory);
        pickupPool = new PickupPool(pickupFactory);
        
        // Initialize all pools
        InitializePools();
    }
    
    public Entity SpawnBullet(Vector3 position, Vector3 direction)
    {
        var bullet = bulletPool.Rent();
        bullet.SetValue(EntityNames.POSITION, position);
        bullet.SetValue(EntityNames.DIRECTION, direction);
        return bullet;
    }
    
    public Entity SpawnExplosion(Vector3 position, float radius)
    {
        var explosion = explosionPool.Rent();
        explosion.SetValue(EntityNames.POSITION, position);
        explosion.SetValue(EntityNames.RADIUS, radius);
        return explosion;
    }
    
    public void ReturnBullet(Entity bullet) => bulletPool.Return(bullet);
    public void ReturnExplosion(Entity explosion) => explosionPool.Return(explosion);
    public void ReturnPickup(Entity pickup) => pickupPool.Return(pickup);
    
    private void InitializePools()
    {
        bulletPool.Init(200);    // Expect many bullets
        explosionPool.Init(50);  // Fewer explosions
        pickupPool.Init(30);     // Occasional pickups
    }
    
    public void Dispose()
    {
        bulletPool.Dispose();
        explosionPool.Dispose();
        pickupPool.Dispose();
    }
}
```

### Procedural Pool Operations
```csharp
public static class PoolOperations
{
    public static Entity RentConfiguredEntity<T>(EntityPool<T> pool, 
                                               Action<T> configureAction) 
        where T : IEntity
    {
        var entity = pool.Rent();
        configureAction?.Invoke(entity);
        return entity;
    }
    
    public static void ReturnAndReset<T>(EntityPool<T> pool, T entity) 
        where T : IEntity
    {
        // Standard reset procedure
        entity.Despawn();
        entity.ClearTags();
        entity.ClearValues();
        
        pool.Return(entity);
    }
    
    public static void WarmupPool<T>(EntityPool<T> pool, int targetSize) 
        where T : IEntity
    {
        // Rent entities to reach target size, then return them
        var entities = new List<T>();
        
        for (int i = 0; i < targetSize; i++)
        {
            entities.Add(pool.Rent());
        }
        
        foreach (var entity in entities)
        {
            pool.Return(entity);
        }
    }
    
    public static void BatchReturn<T>(EntityPool<T> pool, IEnumerable<T> entities) 
        where T : IEntity
    {
        foreach (var entity in entities)
        {
            if (entity != null)
            {
                pool.Return(entity);
            }
        }
    }
}
```

### Pool with Lifetime Management
```csharp
public class TimedEntityPool : EntityPool<Entity>
{
    private readonly Dictionary<Entity, float> entityLifetimes;
    private readonly float maxLifetime;
    
    public TimedEntityPool(IEntityFactory<Entity> factory, float maxLifetime = 10.0f) 
        : base(factory)
    {
        this.maxLifetime = maxLifetime;
        entityLifetimes = new Dictionary<Entity, float>();
    }
    
    protected override void OnRent(Entity entity)
    {
        base.OnRent(entity);
        entityLifetimes[entity] = Time.time;
    }
    
    protected override void OnReturn(Entity entity)
    {
        entityLifetimes.Remove(entity);
        base.OnReturn(entity);
    }
    
    public void UpdateLifetimes()
    {
        var currentTime = Time.time;
        var expiredEntities = new List<Entity>();
        
        foreach (var kvp in entityLifetimes)
        {
            if (currentTime - kvp.Value > maxLifetime)
            {
                expiredEntities.Add(kvp.Key);
            }
        }
        
        // Auto-return expired entities
        foreach (var entity in expiredEntities)
        {
            Return(entity);
        }
    }
    
    protected override void OnDispose(Entity entity)
    {
        entityLifetimes.Remove(entity);
        base.OnDispose(entity);
    }
}
```

## Best Practices

1. **Pre-allocate Pools** – Use `Init()` to create entities during startup
2. **Reset State on Return** – Clear entity state in `OnReturn` hook
3. **Monitor Pool Health** – Track rent/return ratios to detect leaks
4. **Size Pools Appropriately** – Balance memory usage vs. allocation frequency
5. **Use Typed Pools** – Prefer `EntityPool<T>` over generic `EntityPool`
6. **Handle Disposal** – Always dispose pools when no longer needed

## Performance Considerations

- **Memory Footprint** – Pools consume memory even when entities unused
- **GC Pressure** – Dramatically reduces garbage collection from entity allocation
- **Pool Sizing** – Larger pools use more memory, smaller pools allocate more
- **Return Discipline** – Unreturned entities cause memory leaks
- **Hook Overhead** – Complex lifecycle hooks can impact rent/return performance

## Common Pitfalls

- **Forgetting to Return** – Leads to pool exhaustion and memory leaks
- **Double Return** – Returning same entity twice (pool detects and warns)
- **State Contamination** – Not properly resetting entity state between uses
- **Pool Sizing** – Creating too many pools or pools that are too large
- **Lifecycle Hooks** – Expensive operations in hooks impact performance

## Integration with Atomic Systems

### With Entity Worlds
```csharp
public class PooledEntityWorld : EntityWorld<Entity>
{
    private readonly EntityPool<Entity> pool;
    
    public PooledEntityWorld(EntityPool<Entity> pool) 
    {
        this.pool = pool;
    }
    
    public Entity AddPooledEntity()
    {
        var entity = pool.Rent();
        Add(entity);
        return entity;
    }
    
    protected override void OnRemove(Entity entity)
    {
        base.OnRemove(entity);
        pool.Return(entity);
    }
}
```

### With Factory Patterns
```csharp
public class PooledFactory : IEntityFactory<Entity>
{
    private readonly EntityPool<Entity> pool;
    
    public PooledFactory(IEntityFactory<Entity> baseFactory)
    {
        pool = new EntityPool<Entity>(baseFactory);
        pool.Init(25);
    }
    
    public Entity Create()
    {
        return pool.Rent();
    }
    
    public void Recycle(Entity entity)
    {
        pool.Return(entity);
    }
}
```

The `EntityPool` is essential for high-performance entity management in Atomic, providing memory-efficient reuse patterns that integrate seamlessly with the framework's procedural architecture.