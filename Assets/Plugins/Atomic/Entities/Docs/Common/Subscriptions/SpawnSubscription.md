# 🐣 SpawnSubscription

`SpawnSubscription` is a disposable subscription handle that automatically unregisters a callback from an `ISpawnable`'s `OnSpawned` event when disposed. It provides safe event subscription management for handling entity spawn events.

## Key Features

- **Automatic Cleanup** – Unsubscribes callback on disposal
- **Spawn Event Focus** – Specifically handles entity spawn events
- **Memory Safe** – Prevents event handler memory leaks
- **Struct Design** – Lightweight, no heap allocation
- **Null Safe** – Handles null sources and callbacks gracefully

---

## Structure Definition

```csharp
public readonly struct SpawnSubscription : IDisposable
{
    private readonly ISpawnable _source;
    private readonly Action _callback;
}
```

### Constructor

```csharp
internal SpawnSubscription(ISpawnable source, Action callback)
```

- **source**: The spawnable object to subscribe to
- **callback**: The action to invoke when spawned

### Disposal

```csharp
public void Dispose()
```

Safely unsubscribes the callback from the source's `OnSpawned` event.

---

## Usage Patterns

### Basic Spawn Monitoring

```csharp
public class SpawnMonitor
{
    private SpawnSubscription subscription;
    
    public void StartMonitoring(ISpawnable entity)
    {
        subscription = entity.SubscribeToSpawn(OnEntitySpawned);
    }
    
    public void StopMonitoring()
    {
        subscription.Dispose();
    }
    
    private void OnEntitySpawned()
    {
        Debug.Log("Entity spawned!");
        HandleSpawnEvent();
    }
}
```

### Spawn Counter System

```csharp
public class SpawnCounter
{
    private readonly Dictionary<Type, int> spawnCounts = new();
    private readonly List<SpawnSubscription> subscriptions = new();
    
    public void MonitorSpawns<T>(IEnumerable<T> spawnables) where T : ISpawnable
    {
        foreach (var spawnable in spawnables)
        {
            var subscription = spawnable.SubscribeToSpawn(() => OnEntitySpawned(typeof(T)));
            subscriptions.Add(subscription);
        }
    }
    
    private void OnEntitySpawned(Type entityType)
    {
        if (!spawnCounts.ContainsKey(entityType))
            spawnCounts[entityType] = 0;
            
        spawnCounts[entityType]++;
        Debug.Log($"Total {entityType.Name} spawns: {spawnCounts[entityType]}");
        
        UpdateSpawnStatistics(entityType);
    }
    
    public int GetSpawnCount(Type entityType)
    {
        return spawnCounts.TryGetValue(entityType, out int count) ? count : 0;
    }
    
    public void Reset()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
        spawnCounts.Clear();
    }
}
```

### Entity Pool Monitoring

```csharp
public class PoolSpawnMonitor
{
    private readonly Dictionary<IEntityPool, List<SpawnSubscription>> poolSubscriptions = new();
    
    public void MonitorPool(IEntityPool pool)
    {
        var subscriptions = new List<SpawnSubscription>();
        
        // Monitor existing entities in pool
        foreach (var entity in pool.GetActiveEntities())
        {
            if (entity is ISpawnable spawnable)
            {
                var subscription = spawnable.SubscribeToSpawn(() => OnPoolEntitySpawned(pool, spawnable));
                subscriptions.Add(subscription);
            }
        }
        
        poolSubscriptions[pool] = subscriptions;
        
        // Also monitor when new entities are added to pool
        pool.OnEntityCreated += (entity) => 
        {
            if (entity is ISpawnable spawnable)
            {
                var subscription = spawnable.SubscribeToSpawn(() => OnPoolEntitySpawned(pool, spawnable));
                subscriptions.Add(subscription);
            }
        };
    }
    
    private void OnPoolEntitySpawned(IEntityPool pool, ISpawnable entity)
    {
        UpdatePoolStatistics(pool);
        Debug.Log($"Pool entity spawned. Pool utilization: {pool.ActiveCount}/{pool.TotalCount}");
    }
    
    public void StopMonitoringPool(IEntityPool pool)
    {
        if (poolSubscriptions.TryGetValue(pool, out var subscriptions))
        {
            foreach (var subscription in subscriptions)
            {
                subscription.Dispose();
            }
            poolSubscriptions.Remove(pool);
        }
    }
}
```

### Spawn Effect System

```csharp
public class SpawnEffectManager
{
    private readonly Dictionary<ISpawnable, SpawnSubscription> effectSubscriptions = new();
    
    public void RegisterSpawnEffect(ISpawnable entity, GameObject effectPrefab)
    {
        var subscription = entity.SubscribeToSpawn(() => PlaySpawnEffect(entity, effectPrefab));
        effectSubscriptions[entity] = subscription;
    }
    
    private void PlaySpawnEffect(ISpawnable entity, GameObject effectPrefab)
    {
        if (entity is Component component)
        {
            // Instantiate effect at entity position
            var effect = Object.Instantiate(effectPrefab, component.transform.position, component.transform.rotation);
            
            // Schedule cleanup
            Object.Destroy(effect, 3f);
        }
        
        PlaySpawnSound(entity);
        UpdateSpawnVisualEffects(entity);
    }
    
    public void UnregisterSpawnEffect(ISpawnable entity)
    {
        if (effectSubscriptions.TryGetValue(entity, out var subscription))
        {
            subscription.Dispose();
            effectSubscriptions.Remove(entity);
        }
    }
}
```

### Spawn Queue System

```csharp
public class SpawnQueueManager
{
    private readonly Queue<ISpawnable> spawnQueue = new();
    private readonly Dictionary<ISpawnable, SpawnSubscription> queueSubscriptions = new();
    private bool processingQueue;
    
    public void QueueForSpawn(ISpawnable entity, float delay = 0f)
    {
        // Subscribe to know when entity actually spawns
        var subscription = entity.SubscribeToSpawn(() => OnQueuedEntitySpawned(entity));
        queueSubscriptions[entity] = subscription;
        
        if (delay > 0)
        {
            StartCoroutine(DelayedQueueAdd(entity, delay));
        }
        else
        {
            spawnQueue.Enqueue(entity);
            ProcessSpawnQueue();
        }
    }
    
    private IEnumerator DelayedQueueAdd(ISpawnable entity, float delay)
    {
        yield return new WaitForSeconds(delay);
        spawnQueue.Enqueue(entity);
        ProcessSpawnQueue();
    }
    
    private void ProcessSpawnQueue()
    {
        if (processingQueue || spawnQueue.Count == 0) return;
        
        processingQueue = true;
        StartCoroutine(ProcessQueueCoroutine());
    }
    
    private IEnumerator ProcessQueueCoroutine()
    {
        while (spawnQueue.Count > 0)
        {
            var entity = spawnQueue.Dequeue();
            entity.Spawn();
            
            // Wait a frame between spawns to avoid performance spikes
            yield return null;
        }
        
        processingQueue = false;
    }
    
    private void OnQueuedEntitySpawned(ISpawnable entity)
    {
        Debug.Log("Queued entity successfully spawned");
        
        // Clean up subscription
        if (queueSubscriptions.TryGetValue(entity, out var subscription))
        {
            subscription.Dispose();
            queueSubscriptions.Remove(entity);
        }
    }
}
```

### Spawn Analytics System

```csharp
public class SpawnAnalytics
{
    private readonly List<SpawnSubscription> subscriptions = new();
    private readonly Dictionary<string, SpawnMetrics> metrics = new();
    
    public struct SpawnMetrics
    {
        public int TotalSpawns;
        public DateTime FirstSpawn;
        public DateTime LastSpawn;
        public TimeSpan AverageSpawnInterval;
    }
    
    public void StartTracking(IEnumerable<ISpawnable> entities)
    {
        foreach (var entity in entities)
        {
            var subscription = entity.SubscribeToSpawn(() => RecordSpawn(entity));
            subscriptions.Add(subscription);
        }
    }
    
    private void RecordSpawn(ISpawnable entity)
    {
        string entityType = entity.GetType().Name;
        var now = DateTime.Now;
        
        if (!metrics.ContainsKey(entityType))
        {
            metrics[entityType] = new SpawnMetrics
            {
                TotalSpawns = 0,
                FirstSpawn = now,
                LastSpawn = now,
                AverageSpawnInterval = TimeSpan.Zero
            };
        }
        
        var metric = metrics[entityType];
        metric.TotalSpawns++;
        metric.LastSpawn = now;
        
        // Calculate average spawn interval
        if (metric.TotalSpawns > 1)
        {
            var totalTime = metric.LastSpawn - metric.FirstSpawn;
            metric.AverageSpawnInterval = new TimeSpan(totalTime.Ticks / (metric.TotalSpawns - 1));
        }
        
        metrics[entityType] = metric;
        
        Debug.Log($"{entityType} spawn #{metric.TotalSpawns}, Average interval: {metric.AverageSpawnInterval.TotalSeconds:F2}s");
    }
    
    public SpawnMetrics GetMetrics(string entityType)
    {
        return metrics.TryGetValue(entityType, out var metric) ? metric : default;
    }
    
    public void StopTracking()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
    }
}
```

### Entity Registry Integration

```csharp
public class SpawnRegistry
{
    private readonly Dictionary<int, ISpawnable> spawnedEntities = new();
    private readonly List<SpawnSubscription> subscriptions = new();
    
    public void RegisterEntity(ISpawnable entity)
    {
        int entityId = entity.GetHashCode();
        
        var subscription = entity.SubscribeToSpawn(() => OnEntitySpawned(entityId, entity));
        subscriptions.Add(subscription);
    }
    
    private void OnEntitySpawned(int entityId, ISpawnable entity)
    {
        spawnedEntities[entityId] = entity;
        NotifyEntitySpawned(entity);
        UpdateSpawnedEntityCount();
    }
    
    public bool IsEntitySpawned(int entityId)
    {
        return spawnedEntities.ContainsKey(entityId);
    }
    
    public ISpawnable GetSpawnedEntity(int entityId)
    {
        return spawnedEntities.TryGetValue(entityId, out var entity) ? entity : null;
    }
    
    public IEnumerable<ISpawnable> GetAllSpawnedEntities()
    {
        return spawnedEntities.Values;
    }
    
    private void NotifyEntitySpawned(ISpawnable entity)
    {
        Debug.Log($"Entity {entity.GetHashCode()} registered as spawned");
        OnEntitySpawned?.Invoke(entity);
    }
    
    public event Action<ISpawnable> OnEntitySpawned;
    
    public void Cleanup()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
        spawnedEntities.Clear();
    }
}
```

## Integration with Game Systems

### Level Loading Integration

```csharp
public class LevelSpawnManager
{
    private readonly List<SpawnSubscription> levelSubscriptions = new();
    private int entitiesSpawned;
    private int totalEntitiesInLevel;
    
    public void LoadLevel(LevelData levelData)
    {
        totalEntitiesInLevel = levelData.Entities.Count;
        entitiesSpawned = 0;
        
        foreach (var entityData in levelData.Entities)
        {
            var entity = CreateEntity(entityData);
            if (entity is ISpawnable spawnable)
            {
                var subscription = spawnable.SubscribeToSpawn(() => OnLevelEntitySpawned());
                levelSubscriptions.Add(subscription);
                
                // Trigger spawn
                spawnable.Spawn();
            }
        }
    }
    
    private void OnLevelEntitySpawned()
    {
        entitiesSpawned++;
        float progress = (float)entitiesSpawned / totalEntitiesInLevel;
        UpdateLoadingProgress(progress);
        
        if (entitiesSpawned >= totalEntitiesInLevel)
        {
            OnLevelFullyLoaded();
        }
    }
    
    private void OnLevelFullyLoaded()
    {
        Debug.Log("All level entities have been spawned");
        TriggerLevelReadyEvents();
        
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

### Subscription Management
1. **Track subscriptions** – Store subscriptions for proper cleanup
2. **Use using statements** – For temporary subscriptions
3. **Batch operations** – Group related spawn handling
4. **Avoid redundant subscriptions** – Check before subscribing

### Performance Considerations
1. **Lightweight callbacks** – Keep spawn handlers fast
2. **Batch spawn effects** – Group visual/audio effects
3. **Profile spawn bursts** – Monitor performance during mass spawns
4. **Cache references** – Store frequently accessed objects

### Error Handling
1. **Null safety** – Check for null entities and callbacks
2. **Exception handling** – Wrap spawn logic in try-catch
3. **State validation** – Verify entity state before processing

## Common Use Cases

- **Spawn effects** and visual feedback
- **Entity registration** and tracking systems
- **Performance monitoring** for spawn operations
- **Analytics and metrics** collection
- **Level loading progress** tracking
- **Pool management** and utilization monitoring

## Thread Safety

- **Main thread only** – Use only on Unity's main thread
- **Event safety** – Callbacks execute on main thread
- **Disposal safety** – Safe to dispose from any thread
- **No synchronization needed** – Single-threaded by design