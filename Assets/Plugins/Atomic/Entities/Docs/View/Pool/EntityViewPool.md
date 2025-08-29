# 🏊 EntityViewPool

`EntityViewPool` is a concrete implementation of `IEntityViewPool` that provides efficient pooling of entity view instances. It manages object lifecycle, reduces memory allocations, and optimizes performance for frequently created and destroyed views.

## Key Features

- **Concrete Pool Implementation** – Ready-to-use view pooling system
- **Prefab-Based Creation** – Creates views from registered prefab templates
- **Automatic Scaling** – Dynamically adjusts pool sizes based on demand
- **Performance Optimized** – Minimizes garbage collection and object creation overhead
- **Resource Management** – Proper cleanup and memory management

---

## Core Implementation

```csharp
public class EntityViewPool : IEntityViewPool
{
    private readonly Dictionary<string, Stack<IReadOnlyEntityView>> pools;
    private readonly Dictionary<string, GameObject> prefabs;
    private readonly Dictionary<string, Transform> poolParents;
    
    // Pool configuration
    private int defaultInitialSize = 5;
    private int maxPoolSize = 100;
    private bool usePoolParents = true;
    
    public IReadOnlyEntityView Rent(string name) { /* Implementation */ }
    public void Return(string name, IReadOnlyEntityView view) { /* Implementation */ }
    public void Clear() { /* Implementation */ }
}
```

## Configuration Options

### Pool Sizing
```csharp
public class PoolConfiguration
{
    public string viewName;
    public GameObject prefab;
    public int initialSize = 5;
    public int maxSize = 100;
    public bool prewarm = true;
    public Transform parentTransform;
}
```

### Pool Settings
- **Initial Size**: Number of instances created at startup
- **Max Size**: Maximum number of pooled instances
- **Prewarm**: Create initial instances immediately
- **Parent Transform**: Hierarchy organization for pooled objects

## Usage Examples

### Basic Pool Setup

```csharp
public class ViewPoolManager : MonoBehaviour
{
    [Header("Pool Configuration")]
    [SerializeField] private PoolConfiguration[] configurations;
    
    private EntityViewPool pool;
    
    void Start()
    {
        InitializePool();
        PrewarmPools();
    }
    
    private void InitializePool()
    {
        pool = new EntityViewPool();
        
        foreach (var config in configurations)
        {
            pool.RegisterPrefab(config.viewName, config.prefab);
            pool.SetPoolConfiguration(config.viewName, config);
        }
    }
    
    private void PrewarmPools()
    {
        foreach (var config in configurations)
        {
            if (config.prewarm)
            {
                pool.PrewarmPool(config.viewName, config.initialSize);
            }
        }
    }
    
    public IReadOnlyEntityView GetView(string viewName)
    {
        return pool.Rent(viewName);
    }
    
    public void ReturnView(string viewName, IReadOnlyEntityView view)
    {
        pool.Return(viewName, view);
    }
    
    void OnDestroy()
    {
        pool?.Clear();
    }
}
```

### Advanced Pool with Monitoring

```csharp
public class MonitoredEntityViewPool : MonoBehaviour
{
    private EntityViewPool pool;
    private Dictionary<string, PoolMetrics> metrics;
    
    [System.Serializable]
    public class PoolMetrics
    {
        public int totalRented;
        public int totalReturned;
        public int currentActive;
        public int currentPooled;
        public int peakActive;
        public float averageLifetime;
    }
    
    void Start()
    {
        pool = new EntityViewPool();
        metrics = new Dictionary<string, PoolMetrics>();
        
        // Subscribe to pool events
        pool.OnViewRented += OnViewRented;
        pool.OnViewReturned += OnViewReturned;
    }
    
    private void OnViewRented(string name, IReadOnlyEntityView view)
    {
        if (!metrics.ContainsKey(name))
        {
            metrics[name] = new PoolMetrics();
        }
        
        var metric = metrics[name];
        metric.totalRented++;
        metric.currentActive++;
        metric.peakActive = Mathf.Max(metric.peakActive, metric.currentActive);
        
        // Track rental time
        if (view is MonoBehaviour mb)
        {
            var tracker = mb.GetComponent<ViewLifetimeTracker>();
            if (tracker == null)
            {
                tracker = mb.gameObject.AddComponent<ViewLifetimeTracker>();
            }
            tracker.StartTracking();
        }
    }
    
    private void OnViewReturned(string name, IReadOnlyEntityView view)
    {
        if (metrics.ContainsKey(name))
        {
            var metric = metrics[name];
            metric.totalReturned++;
            metric.currentActive--;
            
            // Calculate lifetime
            if (view is MonoBehaviour mb)
            {
                var tracker = mb.GetComponent<ViewLifetimeTracker>();
                if (tracker != null)
                {
                    float lifetime = tracker.GetLifetime();
                    metric.averageLifetime = (metric.averageLifetime * (metric.totalReturned - 1) + lifetime) / metric.totalReturned;
                }
            }
        }
    }
    
    public PoolMetrics GetMetrics(string viewName)
    {
        return metrics.TryGetValue(viewName, out var metric) ? metric : new PoolMetrics();
    }
    
    // Debug visualization
    void OnGUI()
    {
        if (!Application.isPlaying) return;
        
        GUILayout.BeginArea(new Rect(10, 10, 300, 200));
        GUILayout.Label("View Pool Statistics", GUI.skin.box);
        
        foreach (var kvp in metrics)
        {
            var name = kvp.Key;
            var metric = kvp.Value;
            
            GUILayout.Label($"{name}:");
            GUILayout.Label($"  Active: {metric.currentActive} | Peak: {metric.peakActive}");
            GUILayout.Label($"  Pooled: {metric.currentPooled}");
            GUILayout.Label($"  Avg Lifetime: {metric.averageLifetime:F2}s");
            GUILayout.Space(5);
        }
        
        GUILayout.EndArea();
    }
}

// Helper component for tracking view lifetimes
public class ViewLifetimeTracker : MonoBehaviour
{
    private float startTime;
    
    public void StartTracking()
    {
        startTime = Time.time;
    }
    
    public float GetLifetime()
    {
        return Time.time - startTime;
    }
}
```

### Async View Pool

```csharp
public class AsyncEntityViewPool : MonoBehaviour
{
    private EntityViewPool pool;
    private Dictionary<string, Queue<TaskCompletionSource<IReadOnlyEntityView>>> pendingRequests;
    
    void Start()
    {
        pool = new EntityViewPool();
        pendingRequests = new Dictionary<string, Queue<TaskCompletionSource<IReadOnlyEntityView>>>();
    }
    
    public async Task<IReadOnlyEntityView> RentAsync(string name)
    {
        // Try to get from pool immediately
        if (pool.HasAvailable(name))
        {
            return pool.Rent(name);
        }
        
        // Queue request for async fulfillment
        var tcs = new TaskCompletionSource<IReadOnlyEntityView>();
        
        if (!pendingRequests.ContainsKey(name))
        {
            pendingRequests[name] = new Queue<TaskCompletionSource<IReadOnlyEntityView>>();
        }
        
        pendingRequests[name].Enqueue(tcs);
        
        // Start background creation if needed
        StartCoroutine(CreateViewsAsync(name));
        
        return await tcs.Task;
    }
    
    private IEnumerator CreateViewsAsync(string name)
    {
        while (pendingRequests.ContainsKey(name) && pendingRequests[name].Count > 0)
        {
            // Create view over multiple frames to avoid hitches
            yield return null;
            
            var view = pool.CreateNewView(name);
            
            if (view != null && pendingRequests[name].Count > 0)
            {
                var tcs = pendingRequests[name].Dequeue();
                tcs.SetResult(view);
            }
            
            // Limit creation rate
            yield return new WaitForSeconds(0.016f); // ~60 FPS
        }
    }
    
    public void ReturnAsync(string name, IReadOnlyEntityView view)
    {
        pool.Return(name, view);
        
        // Fulfill any pending requests
        if (pendingRequests.ContainsKey(name) && pendingRequests[name].Count > 0)
        {
            var tcs = pendingRequests[name].Dequeue();
            tcs.SetResult(pool.Rent(name));
        }
    }
}
```

### Specialized View Pools

```csharp
// Pool for UI views
public class UIEntityViewPool : EntityViewPool
{
    private Canvas uiCanvas;
    
    protected override void PrepareRentedView(IReadOnlyEntityView view)
    {
        base.PrepareRentedView(view);
        
        if (view is MonoBehaviour mb)
        {
            // Ensure UI views are under canvas
            if (uiCanvas == null)
            {
                uiCanvas = FindObjectOfType<Canvas>();
            }
            
            if (uiCanvas != null)
            {
                mb.transform.SetParent(uiCanvas.transform, false);
            }
            
            // Reset UI-specific properties
            var rectTransform = mb.GetComponent<RectTransform>();
            if (rectTransform != null)
            {
                rectTransform.anchoredPosition = Vector2.zero;
                rectTransform.localScale = Vector3.one;
            }
        }
    }
    
    protected override void PrepareReturnedView(IReadOnlyEntityView view)
    {
        base.PrepareReturnedView(view);
        
        // Hide UI elements
        if (view is MonoBehaviour mb)
        {
            var canvasGroup = mb.GetComponent<CanvasGroup>();
            if (canvasGroup != null)
            {
                canvasGroup.alpha = 0;
                canvasGroup.interactable = false;
                canvasGroup.blocksRaycasts = false;
            }
        }
    }
}

// Pool for particle effect views
public class EffectEntityViewPool : EntityViewPool
{
    protected override void PrepareRentedView(IReadOnlyEntityView view)
    {
        base.PrepareRentedView(view);
        
        // Start particle systems
        if (view is MonoBehaviour mb)
        {
            var particleSystems = mb.GetComponentsInChildren<ParticleSystem>();
            foreach (var ps in particleSystems)
            {
                ps.Play();
            }
        }
    }
    
    protected override void PrepareReturnedView(IReadOnlyEntityView view)
    {
        base.PrepareReturnedView(view);
        
        // Stop and clear particle systems
        if (view is MonoBehaviour mb)
        {
            var particleSystems = mb.GetComponentsInChildren<ParticleSystem>();
            foreach (var ps in particleSystems)
            {
                ps.Stop();
                ps.Clear();
            }
        }
    }
    
    // Auto-return when particle effects complete
    public void RentWithAutoReturn(string name, float duration)
    {
        var view = Rent(name);
        StartCoroutine(AutoReturnCoroutine(name, view, duration));
    }
    
    private IEnumerator AutoReturnCoroutine(string name, IReadOnlyEntityView view, float duration)
    {
        yield return new WaitForSeconds(duration);
        
        // Check if all particle systems have stopped
        if (view is MonoBehaviour mb)
        {
            var particleSystems = mb.GetComponentsInChildren<ParticleSystem>();
            bool allStopped = true;
            
            foreach (var ps in particleSystems)
            {
                if (ps.isPlaying)
                {
                    allStopped = false;
                    break;
                }
            }
            
            if (allStopped)
            {
                Return(name, view);
            }
            else
            {
                // Wait for natural completion
                yield return new WaitUntil(() =>
                {
                    foreach (var ps in particleSystems)
                    {
                        if (ps.isPlaying) return false;
                    }
                    return true;
                });
                
                Return(name, view);
            }
        }
    }
}
```

## Performance Optimizations

### Memory Management
```csharp
public class OptimizedEntityViewPool : EntityViewPool
{
    private readonly Dictionary<string, ObjectPool<IReadOnlyEntityView>> nativePools;
    
    // Use Unity's ObjectPool for better performance
    protected override void InitializePool(string name, GameObject prefab)
    {
        nativePools[name] = new ObjectPool<IReadOnlyEntityView>(
            createFunc: () => CreateViewFromPrefab(prefab),
            actionOnGet: view => PrepareRentedView(view),
            actionOnRelease: view => PrepareReturnedView(view),
            actionOnDestroy: view => DestroyView(view),
            collectionCheck: true,
            defaultCapacity: defaultInitialSize,
            maxSize: maxPoolSize
        );
    }
    
    public override IReadOnlyEntityView Rent(string name)
    {
        if (nativePools.TryGetValue(name, out var pool))
        {
            return pool.Get();
        }
        
        throw new ArgumentException($"No pool configured for view: {name}");
    }
    
    public override void Return(string name, IReadOnlyEntityView view)
    {
        if (nativePools.TryGetValue(name, out var pool))
        {
            pool.Release(view);
        }
    }
}
```

## Best Practices

1. **Pool Sizing** – Configure pools based on actual usage patterns
2. **Prewarming** – Create initial instances to avoid runtime hitches
3. **Resource Cleanup** – Always clear pools during scene transitions
4. **Memory Limits** – Set maximum pool sizes to prevent memory bloat
5. **State Reset** – Ensure views are properly reset before reuse
6. **Performance Monitoring** – Track pool statistics for optimization
7. **Thread Safety** – Consider thread safety if accessing from multiple threads

## Integration Patterns

### Entity System Integration
```csharp
public class EntityViewSystem : MonoBehaviour
{
    private EntityViewPool viewPool;
    private Dictionary<IEntity, IReadOnlyEntityView> activeViews;
    
    public void CreateViewForEntity(IEntity entity, string viewType)
    {
        if (activeViews.ContainsKey(entity))
        {
            Debug.LogWarning($"Entity {entity.Name} already has a view");
            return;
        }
        
        var view = viewPool.Rent(viewType);
        view.BindToEntity(entity);
        activeViews[entity] = view;
        
        entity.OnDespawned += () => RemoveViewForEntity(entity);
    }
    
    private void RemoveViewForEntity(IEntity entity)
    {
        if (activeViews.TryGetValue(entity, out var view))
        {
            var viewType = GetViewType(view);
            viewPool.Return(viewType, view);
            activeViews.Remove(entity);
        }
    }
}
```

## Notes

- EntityViewPool is a production-ready implementation of IEntityViewPool
- Optimized for Unity's component system and object lifecycle
- Supports both immediate and asynchronous view creation patterns  
- Includes built-in performance monitoring and statistics tracking
- Can be specialized for specific view types (UI, effects, etc.)
- Integration with Unity's native ObjectPool provides additional performance benefits