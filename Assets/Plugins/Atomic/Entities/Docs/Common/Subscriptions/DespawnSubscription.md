# 💀 DespawnSubscription

`DespawnSubscription` is a disposable subscription handle that automatically unregisters a callback from an `ISpawnable`'s `OnDespawned` event when disposed. It provides safe event subscription management for handling entity despawn events and cleanup operations.

## Key Features

- **Automatic Cleanup** – Unsubscribes callback on disposal
- **Despawn Event Focus** – Specifically handles entity despawn events
- **Memory Safe** – Prevents event handler memory leaks
- **Struct Design** – Lightweight, no heap allocation
- **Cleanup Integration** – Perfect for resource cleanup on despawn

---

## Structure Definition

```csharp
public readonly struct DespawnSubscription : IDisposable
{
    private readonly ISpawnable _source;
    private readonly Action _callback;
}
```

### Constructor

```csharp
internal DespawnSubscription(ISpawnable source, Action callback)
```

- **source**: The spawnable object to subscribe to
- **callback**: The action to invoke when despawned

### Disposal

```csharp
public void Dispose()
```

Safely unsubscribes the callback from the source's `OnDespawned` event.

---

## Usage Patterns

### Basic Despawn Handling

```csharp
public class DespawnHandler
{
    private DespawnSubscription subscription;
    
    public void StartMonitoring(ISpawnable entity)
    {
        subscription = entity.SubscribeToDespawn(OnEntityDespawned);
    }
    
    public void StopMonitoring()
    {
        subscription.Dispose();
    }
    
    private void OnEntityDespawned()
    {
        Debug.Log("Entity despawned!");
        HandleDespawnCleanup();
    }
}
```

### Resource Cleanup System

```csharp
public class EntityResourceManager
{
    private readonly Dictionary<ISpawnable, (List<IDisposable> resources, DespawnSubscription subscription)> entityResources = new();
    
    public void RegisterEntity(ISpawnable entity)
    {
        var resources = new List<IDisposable>();
        var subscription = entity.SubscribeToDespawn(() => CleanupEntityResources(entity));
        
        entityResources[entity] = (resources, subscription);
    }
    
    public void AddResource(ISpawnable entity, IDisposable resource)
    {
        if (entityResources.TryGetValue(entity, out var data))
        {
            data.resources.Add(resource);
        }
    }
    
    private void CleanupEntityResources(ISpawnable entity)
    {
        if (entityResources.TryGetValue(entity, out var data))
        {
            Debug.Log($"Cleaning up {data.resources.Count} resources for despawned entity");
            
            // Dispose all resources
            foreach (var resource in data.resources)
            {
                try
                {
                    resource?.Dispose();
                }
                catch (Exception e)
                {
                    Debug.LogError($"Error disposing resource: {e}");
                }
            }
            
            // Clean up subscription and remove from tracking
            data.subscription.Dispose();
            entityResources.Remove(entity);
        }
    }
    
    public void ForceCleanup(ISpawnable entity)
    {
        CleanupEntityResources(entity);
    }
}
```

### Entity Pool Return System

```csharp
public class PoolReturnManager
{
    private readonly Dictionary<ISpawnable, (IEntityPool pool, DespawnSubscription subscription)> pooledEntities = new();
    
    public void RegisterPooledEntity(ISpawnable entity, IEntityPool pool)
    {
        var subscription = entity.SubscribeToDespawn(() => ReturnToPool(entity, pool));
        pooledEntities[entity] = (pool, subscription);
    }
    
    private void ReturnToPool(ISpawnable entity, IEntityPool pool)
    {
        Debug.Log("Returning entity to pool after despawn");
        
        try
        {
            pool.Return(entity);
            OnEntityReturnedToPool(entity, pool);
        }
        catch (Exception e)
        {
            Debug.LogError($"Failed to return entity to pool: {e}");
        }
        
        // Clean up tracking
        if (pooledEntities.TryGetValue(entity, out var data))
        {
            data.subscription.Dispose();
            pooledEntities.Remove(entity);
        }
    }
    
    private void OnEntityReturnedToPool(ISpawnable entity, IEntityPool pool)
    {
        UpdatePoolStatistics(pool);
        TriggerPoolReturnEffects(entity);
    }
    
    public void UnregisterPooledEntity(ISpawnable entity)
    {
        if (pooledEntities.TryGetValue(entity, out var data))
        {
            data.subscription.Dispose();
            pooledEntities.Remove(entity);
        }
    }
}
```

### Despawn Analytics

```csharp
public class DespawnAnalytics
{
    private readonly Dictionary<string, DespawnMetrics> metrics = new();
    private readonly List<DespawnSubscription> subscriptions = new();
    
    public struct DespawnMetrics
    {
        public int TotalDespawns;
        public TimeSpan AverageLifetime;
        public DateTime LastDespawn;
        public List<TimeSpan> Lifetimes;
    }
    
    public void StartTracking(IEnumerable<ISpawnable> entities)
    {
        foreach (var entity in entities)
        {
            var subscription = entity.SubscribeToDespawn(() => RecordDespawn(entity));
            subscriptions.Add(subscription);
        }
    }
    
    private void RecordDespawn(ISpawnable entity)
    {
        string entityType = entity.GetType().Name;
        var now = DateTime.Now;
        
        // Calculate lifetime if we know spawn time
        TimeSpan lifetime = TimeSpan.Zero;
        if (TryGetSpawnTime(entity, out var spawnTime))
        {
            lifetime = now - spawnTime;
        }
        
        if (!metrics.ContainsKey(entityType))
        {
            metrics[entityType] = new DespawnMetrics
            {
                TotalDespawns = 0,
                AverageLifetime = TimeSpan.Zero,
                LastDespawn = now,
                Lifetimes = new List<TimeSpan>()
            };
        }
        
        var metric = metrics[entityType];
        metric.TotalDespawns++;
        metric.LastDespawn = now;
        
        if (lifetime != TimeSpan.Zero)
        {
            metric.Lifetimes.Add(lifetime);
            
            // Calculate average lifetime
            var totalTicks = metric.Lifetimes.Sum(t => t.Ticks);
            metric.AverageLifetime = new TimeSpan(totalTicks / metric.Lifetimes.Count);
        }
        
        metrics[entityType] = metric;
        
        Debug.Log($"{entityType} despawned. Total: {metric.TotalDespawns}, Avg lifetime: {metric.AverageLifetime.TotalSeconds:F2}s");
    }
    
    public DespawnMetrics GetMetrics(string entityType)
    {
        return metrics.TryGetValue(entityType, out var metric) ? metric : default;
    }
}
```

### Despawn Effect System

```csharp
public class DespawnEffectManager
{
    private readonly Dictionary<ISpawnable, (GameObject effectPrefab, DespawnSubscription subscription)> effectSubscriptions = new();
    
    public void RegisterDespawnEffect(ISpawnable entity, GameObject effectPrefab)
    {
        var subscription = entity.SubscribeToDespawn(() => PlayDespawnEffect(entity, effectPrefab));
        effectSubscriptions[entity] = (effectPrefab, subscription);
    }
    
    private void PlayDespawnEffect(ISpawnable entity, GameObject effectPrefab)
    {
        if (entity is Component component)
        {
            // Play effect at last known position
            var lastPosition = component.transform.position;
            var effect = Object.Instantiate(effectPrefab, lastPosition, Quaternion.identity);
            
            // Schedule cleanup
            Object.Destroy(effect, 2f);
        }
        
        PlayDespawnSound(entity);
        UpdateDespawnParticles(entity);
        TriggerScreenShake(entity);
    }
    
    public void UnregisterDespawnEffect(ISpawnable entity)
    {
        if (effectSubscriptions.TryGetValue(entity, out var data))
        {
            data.subscription.Dispose();
            effectSubscriptions.Remove(entity);
        }
    }
    
    private void PlayDespawnSound(ISpawnable entity)
    {
        // Play despawn audio effect
        AudioManager.PlaySound("EntityDespawn");
    }
}
```

### Save System Integration

```csharp
public class DespawnSaveManager
{
    private readonly Dictionary<ISpawnable, DespawnSubscription> saveSubscriptions = new();
    
    public void RegisterSaveableEntity(ISpawnable entity, ISaveable saveData)
    {
        var subscription = entity.SubscribeToDespawn(() => SaveEntityState(entity, saveData));
        saveSubscriptions[entity] = subscription;
    }
    
    private void SaveEntityState(ISpawnable entity, ISaveable saveData)
    {
        try
        {
            Debug.Log("Saving entity state before despawn");
            saveData.Save();
            OnEntityStateSaved(entity);
        }
        catch (Exception e)
        {
            Debug.LogError($"Failed to save entity state on despawn: {e}");
        }
    }
    
    private void OnEntityStateSaved(ISpawnable entity)
    {
        UpdateSaveStatistics();
        TriggerSaveConfirmation(entity);
    }
    
    public void UnregisterSaveableEntity(ISpawnable entity)
    {
        if (saveSubscriptions.TryGetValue(entity, out var subscription))
        {
            subscription.Dispose();
            saveSubscriptions.Remove(entity);
        }
    }
}
```

### Network Despawn Synchronization

```csharp
public class NetworkDespawnSynchronizer
{
    private readonly Dictionary<ISpawnable, DespawnSubscription> networkSubscriptions = new();
    
    public void RegisterNetworkedEntity(ISpawnable entity, int networkId)
    {
        var subscription = entity.SubscribeToDespawn(() => SynchronizeDespawn(networkId));
        networkSubscriptions[entity] = subscription;
    }
    
    private void SynchronizeDespawn(int networkId)
    {
        // Send despawn event to network
        NetworkManager.Instance.SendDespawnEvent(networkId);
        Debug.Log($"Synchronized despawn for network entity {networkId}");
    }
    
    public void UnregisterNetworkedEntity(ISpawnable entity)
    {
        if (networkSubscriptions.TryGetValue(entity, out var subscription))
        {
            subscription.Dispose();
            networkSubscriptions.Remove(entity);
        }
    }
}
```

### Memory Pool Integration

```csharp
public class MemoryPoolDespawnHandler
{
    private readonly Dictionary<Type, IMemoryPool> typePools = new();
    private readonly List<DespawnSubscription> subscriptions = new();
    
    public void RegisterMemoryPoolType<T>(IMemoryPool<T> pool) where T : class, ISpawnable
    {
        typePools[typeof(T)] = pool;
    }
    
    public void RegisterEntity(ISpawnable entity)
    {
        var subscription = entity.SubscribeToDespawn(() => ReturnToMemoryPool(entity));
        subscriptions.Add(subscription);
    }
    
    private void ReturnToMemoryPool(ISpawnable entity)
    {
        var entityType = entity.GetType();
        
        if (typePools.TryGetValue(entityType, out var pool))
        {
            try
            {
                pool.Return(entity);
                Debug.Log($"Returned {entityType.Name} to memory pool");
            }
            catch (Exception e)
            {
                Debug.LogError($"Failed to return entity to memory pool: {e}");
            }
        }
    }
    
    public void Cleanup()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
    }
}
```

## Integration with Game Systems

### Level Cleanup

```csharp
public class LevelCleanupManager
{
    private readonly List<DespawnSubscription> levelSubscriptions = new();
    private int entitiesDespawned;
    private int totalLevelEntities;
    
    public void StartLevelCleanup(IEnumerable<ISpawnable> levelEntities)
    {
        var entities = levelEntities.ToList();
        totalLevelEntities = entities.Count;
        entitiesDespawned = 0;
        
        foreach (var entity in entities)
        {
            var subscription = entity.SubscribeToDespawn(() => OnLevelEntityDespawned());
            levelSubscriptions.Add(subscription);
            
            // Initiate despawn
            entity.Despawn();
        }
    }
    
    private void OnLevelEntityDespawned()
    {
        entitiesDespawned++;
        float progress = (float)entitiesDespawned / totalLevelEntities;
        UpdateCleanupProgress(progress);
        
        if (entitiesDespawned >= totalLevelEntities)
        {
            OnLevelFullyCleaned();
        }
    }
    
    private void OnLevelFullyCleaned()
    {
        Debug.Log("All level entities have been despawned and cleaned up");
        TriggerLevelCleanupComplete();
        
        // Clean up subscriptions
        foreach (var subscription in levelSubscriptions)
        {
            subscription.Dispose();
        }
        levelSubscriptions.Clear();
    }
}
```

## Best Practices

### Resource Management
1. **Always cleanup** – Use despawn events for resource cleanup
2. **Exception handling** – Wrap cleanup code in try-catch blocks
3. **Batch cleanup** – Group related cleanup operations
4. **Validate state** – Check entity state before cleanup

### Performance Considerations
1. **Lightweight callbacks** – Keep despawn handlers fast
2. **Avoid blocking operations** – Don't block on I/O during despawn
3. **Pool returns** – Use despawn for efficient pool management
4. **Memory management** – Release references to prevent leaks

### Error Handling
1. **Graceful degradation** – Handle cleanup failures gracefully
2. **Logging** – Log important despawn events for debugging
3. **State consistency** – Ensure consistent state after despawn
4. **Recovery mechanisms** – Implement fallback cleanup strategies

## Common Use Cases

- **Resource cleanup** when entities are destroyed
- **Pool management** for entity recycling
- **Save state** preservation before entity destruction
- **Network synchronization** for multiplayer games
- **Analytics collection** for entity lifecycle metrics
- **Visual effects** for entity destruction feedback

## Thread Safety

- **Main thread only** – Use only on Unity's main thread
- **Callback execution** – All callbacks execute on main thread
- **Disposal safety** – Safe to dispose from any thread
- **Synchronization** – No additional synchronization needed