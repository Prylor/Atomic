# 🏊 IEntityViewPool

`IEntityViewPool` is an interface that defines a contract for managing pools of `EntityViewBase` instances. It provides a standardized approach to view object pooling, enabling efficient memory management and performance optimization by reusing view instances instead of constantly creating and destroying them.

## Key Features

- **Pool Management Interface** – Standardized pooling contract for view instances
- **Name-Based Identification** – Views are identified by string names for flexible categorization
- **Memory Optimization** – Reduces garbage collection through object reuse
- **Performance Enhancement** – Minimizes allocation overhead for frequently used views
- **Resource Management** – Centralized cleanup and resource management

---

## Interface Definition

```csharp
public interface IEntityViewPool
{
    /// <summary>
    /// Retrieves a view instance from the pool associated with the specified name.
    /// If no available instance exists, a new one may be created.
    /// </summary>
    /// <param name="name">The name identifying the type of view to rent.</param>
    /// <returns>An active EntityViewBase instance.</returns>
    IReadOnlyEntityView Rent(string name);

    /// <summary>
    /// Returns a previously rented view instance back to the pool for reuse.
    /// </summary>
    /// <param name="name">The name identifying the type of view being returned.</param>
    /// <param name="view">The EntityViewBase instance to return to the pool.</param>
    void Return(string name, IReadOnlyEntityView view);

    /// <summary>
    /// Clears all view instances from the pool, releasing any resources held.
    /// </summary>
    void Clear();
}
```

## Method Details

### Rent(string name)
- **Purpose**: Obtains a view instance from the pool
- **Parameter**: `name` - String identifier for the view type
- **Returns**: `IReadOnlyEntityView` instance ready for use
- **Behavior**: May create new instance if pool is empty

### Return(string name, IReadOnlyEntityView view)
- **Purpose**: Returns a used view instance back to the pool
- **Parameters**: 
  - `name` - String identifier for the view type
  - `view` - The view instance being returned
- **Behavior**: Prepares view for reuse and stores in pool

### Clear()
- **Purpose**: Empties the entire pool and releases resources
- **Behavior**: Destroys or clears all pooled instances
- **Usage**: Typically called during cleanup or scene transitions

## Implementation Examples

### Basic Pool Implementation

```csharp
public class BasicEntityViewPool : IEntityViewPool
{
    private readonly Dictionary<string, Stack<IReadOnlyEntityView>> pools 
        = new Dictionary<string, Stack<IReadOnlyEntityView>>();
    
    private readonly Dictionary<string, GameObject> prefabs
        = new Dictionary<string, GameObject>();
    
    public void RegisterPrefab(string name, GameObject prefab)
    {
        prefabs[name] = prefab;
        if (!pools.ContainsKey(name))
        {
            pools[name] = new Stack<IReadOnlyEntityView>();
        }
    }
    
    public IReadOnlyEntityView Rent(string name)
    {
        if (pools.TryGetValue(name, out var pool) && pool.Count > 0)
        {
            // Return existing instance from pool
            var view = pool.Pop();
            ActivateView(view);
            return view;
        }
        
        // Create new instance if pool is empty
        if (prefabs.TryGetValue(name, out var prefab))
        {
            var instance = Object.Instantiate(prefab);
            var view = instance.GetComponent<IReadOnlyEntityView>();
            if (view != null)
            {
                ActivateView(view);
                return view;
            }
        }
        
        throw new ArgumentException($"No prefab registered for view name: {name}");
    }
    
    public void Return(string name, IReadOnlyEntityView view)
    {
        if (view == null) return;
        
        DeactivateView(view);
        
        if (pools.TryGetValue(name, out var pool))
        {
            pool.Push(view);
        }
        else
        {
            // Create pool if it doesn't exist
            pools[name] = new Stack<IReadOnlyEntityView>();
            pools[name].Push(view);
        }
    }
    
    public void Clear()
    {
        foreach (var pool in pools.Values)
        {
            while (pool.Count > 0)
            {
                var view = pool.Pop();
                if (view is MonoBehaviour mb)
                {
                    Object.Destroy(mb.gameObject);
                }
            }
        }
        
        pools.Clear();
    }
    
    private void ActivateView(IReadOnlyEntityView view)
    {
        if (view is MonoBehaviour mb)
        {
            mb.gameObject.SetActive(true);
        }
        
        // Reset view state for reuse
        if (view is IResettableView resettable)
        {
            resettable.Reset();
        }
    }
    
    private void DeactivateView(IReadOnlyEntityView view)
    {
        if (view is MonoBehaviour mb)
        {
            mb.gameObject.SetActive(false);
        }
        
        // Prepare view for pooling
        if (view is IPoolableView poolable)
        {
            poolable.PrepareForPooling();
        }
    }
}
```

### Advanced Pool with Prewarming

```csharp
public class PrewarmingEntityViewPool : IEntityViewPool
{
    [System.Serializable]
    public class PoolConfiguration
    {
        public string viewName;
        public GameObject prefab;
        public int prewarmCount = 5;
        public int maxPoolSize = 50;
    }
    
    private readonly Dictionary<string, Stack<IReadOnlyEntityView>> pools
        = new Dictionary<string, Stack<IReadOnlyEntityView>>();
    
    private readonly Dictionary<string, PoolConfiguration> configurations
        = new Dictionary<string, PoolConfiguration>();
    
    public void Initialize(PoolConfiguration[] configs)
    {
        foreach (var config in configs)
        {
            configurations[config.viewName] = config;
            pools[config.viewName] = new Stack<IReadOnlyEntityView>();
            
            PrewarmPool(config);
        }
    }
    
    private void PrewarmPool(PoolConfiguration config)
    {
        var pool = pools[config.viewName];
        
        for (int i = 0; i < config.prewarmCount; i++)
        {
            var instance = Object.Instantiate(config.prefab);
            var view = instance.GetComponent<IReadOnlyEntityView>();
            
            if (view != null)
            {
                instance.SetActive(false);
                pool.Push(view);
            }
            else
            {
                Object.Destroy(instance);
            }
        }
        
        Debug.Log($"Prewarmed pool '{config.viewName}' with {pool.Count} instances");
    }
    
    public IReadOnlyEntityView Rent(string name)
    {
        if (!pools.ContainsKey(name))
        {
            throw new ArgumentException($"Pool not initialized for view: {name}");
        }
        
        var pool = pools[name];
        
        if (pool.Count > 0)
        {
            var view = pool.Pop();
            ActivatePooledView(view);
            return view;
        }
        
        // Pool exhausted, create new instance if allowed
        if (configurations.TryGetValue(name, out var config))
        {
            var instance = Object.Instantiate(config.prefab);
            var view = instance.GetComponent<IReadOnlyEntityView>();
            
            if (view != null)
            {
                ActivatePooledView(view);
                Debug.LogWarning($"Pool '{name}' exhausted, created new instance");
                return view;
            }
            
            Object.Destroy(instance);
        }
        
        throw new InvalidOperationException($"Cannot create new view for: {name}");
    }
    
    public void Return(string name, IReadOnlyEntityView view)
    {
        if (view == null || !pools.ContainsKey(name)) return;
        
        var pool = pools[name];
        var config = configurations[name];
        
        // Check pool size limits
        if (pool.Count >= config.maxPoolSize)
        {
            // Destroy excess instances
            if (view is MonoBehaviour mb)
            {
                Object.Destroy(mb.gameObject);
            }
            return;
        }
        
        PrepareViewForPooling(view);
        pool.Push(view);
    }
    
    public void Clear()
    {
        foreach (var kvp in pools)
        {
            var pool = kvp.Value;
            while (pool.Count > 0)
            {
                var view = pool.Pop();
                if (view is MonoBehaviour mb)
                {
                    Object.Destroy(mb.gameObject);
                }
            }
        }
        
        pools.Clear();
        configurations.Clear();
    }
    
    private void ActivatePooledView(IReadOnlyEntityView view)
    {
        if (view is MonoBehaviour mb)
        {
            mb.gameObject.SetActive(true);
        }
        
        if (view is IPoolableView poolable)
        {
            poolable.OnRentFromPool();
        }
    }
    
    private void PrepareViewForPooling(IReadOnlyEntityView view)
    {
        if (view is MonoBehaviour mb)
        {
            mb.gameObject.SetActive(false);
            mb.transform.SetParent(null);
        }
        
        if (view is IPoolableView poolable)
        {
            poolable.OnReturnToPool();
        }
    }
    
    // Diagnostic methods
    public int GetPoolSize(string name)
    {
        return pools.TryGetValue(name, out var pool) ? pool.Count : 0;
    }
    
    public Dictionary<string, int> GetAllPoolSizes()
    {
        var sizes = new Dictionary<string, int>();
        foreach (var kvp in pools)
        {
            sizes[kvp.Key] = kvp.Value.Count;
        }
        return sizes;
    }
}
```

### Async Pool Implementation

```csharp
public class AsyncEntityViewPool : IEntityViewPool
{
    private readonly Dictionary<string, Queue<IReadOnlyEntityView>> pools
        = new Dictionary<string, Queue<IReadOnlyEntityView>>();
    
    private readonly Dictionary<string, AssetReference> assetReferences
        = new Dictionary<string, AssetReference>();
    
    private readonly Dictionary<string, TaskCompletionSource<GameObject>> loadingTasks
        = new Dictionary<string, TaskCompletionSource<GameObject>>();
    
    public void RegisterAssetReference(string name, AssetReference assetRef)
    {
        assetReferences[name] = assetRef;
        pools[name] = new Queue<IReadOnlyEntityView>();
    }
    
    public async Task<IReadOnlyEntityView> RentAsync(string name)
    {
        // Return from pool if available
        if (pools.TryGetValue(name, out var pool) && pool.Count > 0)
        {
            var pooledView = pool.Dequeue();
            ActivateView(pooledView);
            return pooledView;
        }
        
        // Load asset asynchronously if not in pool
        var prefab = await LoadPrefabAsync(name);
        var instance = Object.Instantiate(prefab);
        var view = instance.GetComponent<IReadOnlyEntityView>();
        
        if (view != null)
        {
            ActivateView(view);
            return view;
        }
        
        Object.Destroy(instance);
        throw new InvalidOperationException($"Prefab for '{name}' does not have a view component");
    }
    
    // Synchronous fallback (may block)
    public IReadOnlyEntityView Rent(string name)
    {
        try
        {
            return RentAsync(name).GetAwaiter().GetResult();
        }
        catch (System.Exception e)
        {
            throw new InvalidOperationException($"Failed to rent view '{name}': {e.Message}");
        }
    }
    
    public void Return(string name, IReadOnlyEntityView view)
    {
        if (view == null) return;
        
        DeactivateView(view);
        
        if (pools.TryGetValue(name, out var pool))
        {
            pool.Enqueue(view);
        }
    }
    
    public void Clear()
    {
        foreach (var pool in pools.Values)
        {
            while (pool.Count > 0)
            {
                var view = pool.Dequeue();
                if (view is MonoBehaviour mb)
                {
                    Object.Destroy(mb.gameObject);
                }
            }
        }
        
        pools.Clear();
        
        // Release asset references
        foreach (var assetRef in assetReferences.Values)
        {
            if (assetRef.IsValid())
            {
                Addressables.Release(assetRef);
            }
        }
        
        assetReferences.Clear();
    }
    
    private async Task<GameObject> LoadPrefabAsync(string name)
    {
        // Check if already loading
        if (loadingTasks.TryGetValue(name, out var existingTask))
        {
            return await existingTask.Task;
        }
        
        // Start loading
        var tcs = new TaskCompletionSource<GameObject>();
        loadingTasks[name] = tcs;
        
        try
        {
            var assetRef = assetReferences[name];
            var handle = Addressables.LoadAssetAsync<GameObject>(assetRef);
            var prefab = await handle.Task;
            
            tcs.SetResult(prefab);
            return prefab;
        }
        catch (System.Exception e)
        {
            tcs.SetException(e);
            throw;
        }
        finally
        {
            loadingTasks.Remove(name);
        }
    }
}
```

## Supporting Interfaces

### IPoolableView
```csharp
public interface IPoolableView
{
    void OnRentFromPool();
    void OnReturnToPool();
    void PrepareForPooling();
}
```

### IResettableView
```csharp
public interface IResettableView
{
    void Reset();
}
```

## Integration Patterns

### Pool Manager
```csharp
public class ViewPoolManager : MonoBehaviour
{
    [SerializeField] private PrewarmingEntityViewPool.PoolConfiguration[] poolConfigurations;
    private PrewarmingEntityViewPool pool;
    
    void Start()
    {
        pool = new PrewarmingEntityViewPool();
        pool.Initialize(poolConfigurations);
    }
    
    public static ViewPoolManager Instance { get; private set; }
    
    void Awake()
    {
        Instance = this;
    }
    
    public IReadOnlyEntityView RentView(string name) => pool.Rent(name);
    public void ReturnView(string name, IReadOnlyEntityView view) => pool.Return(name, view);
}
```

## Best Practices

1. **Pool Sizing** – Configure appropriate pool sizes based on usage patterns
2. **Resource Management** – Always clear pools during scene transitions
3. **Error Handling** – Handle cases where pools are exhausted or misconfigured
4. **Performance Monitoring** – Track pool usage statistics for optimization
5. **Memory Limits** – Implement maximum pool sizes to prevent memory bloat
6. **State Reset** – Ensure pooled objects are properly reset before reuse
7. **Async Support** – Consider async loading for large or remote assets

## Performance Considerations

### Benefits
- Reduced garbage collection pressure
- Faster object creation (from pool)
- Predictable memory usage patterns
- Lower CPU overhead for frequent spawning

### Trade-offs  
- Memory overhead from pooled objects
- Complexity in state management
- Potential for memory leaks if not cleared properly

## Notes

- Interface is designed for flexibility in implementation approaches
- Name-based identification allows for dynamic view type management
- Works with both synchronous and asynchronous loading patterns
- Compatible with Unity's component system and object lifecycle
- Supports integration with asset loading systems like Addressables