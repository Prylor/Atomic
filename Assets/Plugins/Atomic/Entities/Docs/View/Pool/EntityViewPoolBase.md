# 🏊 EntityViewPoolBase

`EntityViewPoolBase` is an abstract base class that provides common functionality for implementing entity view pools. It offers a foundation for creating concrete pool implementations with shared behavior patterns, error handling, and resource management.

## Key Features

- **Abstract Base Implementation** – Common pooling logic foundation
- **Extensible Design** – Override methods for custom behavior
- **Resource Management** – Built-in cleanup and disposal patterns
- **Error Handling** – Standardized error handling approaches
- **Performance Optimizations** – Shared performance-critical implementations

---

## Base Implementation Structure

```csharp
public abstract class EntityViewPoolBase : IEntityViewPool
{
    // Abstract methods for concrete implementation
    public abstract IReadOnlyEntityView Rent(string name);
    public abstract void Return(string name, IReadOnlyEntityView view);
    public abstract void Clear();
    
    // Virtual methods for optional override
    protected virtual void OnViewRented(string name, IReadOnlyEntityView view) { }
    protected virtual void OnViewReturned(string name, IReadOnlyEntityView view) { }
    protected virtual void OnPoolCleared() { }
    
    // Utility methods for common operations
    protected void ValidateViewName(string name) { /* ... */ }
    protected void ValidateView(IReadOnlyEntityView view) { /* ... */ }
    protected void LogPoolOperation(string operation, string viewName) { /* ... */ }
}
```

## Implementation Examples

### Basic Concrete Implementation

```csharp
public class SimpleEntityViewPool : EntityViewPoolBase
{
    private readonly Dictionary<string, Stack<IReadOnlyEntityView>> pools
        = new Dictionary<string, Stack<IReadOnlyEntityView>>();
    
    private readonly Dictionary<string, GameObject> prefabs
        = new Dictionary<string, GameObject>();
    
    public void RegisterPrefab(string name, GameObject prefab)
    {
        ValidateViewName(name);
        if (prefab == null)
            throw new ArgumentNullException(nameof(prefab));
            
        prefabs[name] = prefab;
        if (!pools.ContainsKey(name))
        {
            pools[name] = new Stack<IReadOnlyEntityView>();
        }
    }
    
    public override IReadOnlyEntityView Rent(string name)
    {
        ValidateViewName(name);
        
        if (pools.TryGetValue(name, out var pool) && pool.Count > 0)
        {
            var view = pool.Pop();
            PrepareRentedView(view);
            OnViewRented(name, view);
            return view;
        }
        
        return CreateNewView(name);
    }
    
    public override void Return(string name, IReadOnlyEntityView view)
    {
        ValidateViewName(name);
        ValidateView(view);
        
        PrepareReturnedView(view);
        
        if (!pools.ContainsKey(name))
        {
            pools[name] = new Stack<IReadOnlyEntityView>();
        }
        
        pools[name].Push(view);
        OnViewReturned(name, view);
    }
    
    public override void Clear()
    {
        foreach (var pool in pools.Values)
        {
            while (pool.Count > 0)
            {
                var view = pool.Pop();
                DestroyPooledView(view);
            }
        }
        
        pools.Clear();
        OnPoolCleared();
    }
    
    private IReadOnlyEntityView CreateNewView(string name)
    {
        if (!prefabs.TryGetValue(name, out var prefab))
        {
            throw new ArgumentException($"No prefab registered for view: {name}");
        }
        
        var instance = Object.Instantiate(prefab);
        var view = instance.GetComponent<IReadOnlyEntityView>();
        
        if (view == null)
        {
            Object.Destroy(instance);
            throw new InvalidOperationException($"Prefab '{name}' does not have a view component");
        }
        
        PrepareRentedView(view);
        OnViewRented(name, view);
        return view;
    }
    
    protected virtual void PrepareRentedView(IReadOnlyEntityView view)
    {
        if (view is MonoBehaviour mb)
        {
            mb.gameObject.SetActive(true);
        }
        
        if (view is IResettableView resettable)
        {
            resettable.Reset();
        }
    }
    
    protected virtual void PrepareReturnedView(IReadOnlyEntityView view)
    {
        if (view is MonoBehaviour mb)
        {
            mb.gameObject.SetActive(false);
            mb.transform.SetParent(null);
        }
        
        if (view is IPoolableView poolable)
        {
            poolable.PrepareForPooling();
        }
    }
    
    protected virtual void DestroyPooledView(IReadOnlyEntityView view)
    {
        if (view is MonoBehaviour mb)
        {
            Object.Destroy(mb.gameObject);
        }
    }
}
```

### Advanced Pool with Statistics

```csharp
public class StatisticsEntityViewPool : EntityViewPoolBase
{
    public class PoolStatistics
    {
        public int TotalRents;
        public int TotalReturns;
        public int CurrentPooled;
        public int PeakPooled;
        public int TotalCreated;
        public float AverageLifetime;
        public DateTime LastActivity;
    }
    
    private readonly Dictionary<string, Stack<IReadOnlyEntityView>> pools
        = new Dictionary<string, Stack<IReadOnlyEntityView>>();
    
    private readonly Dictionary<string, PoolStatistics> statistics
        = new Dictionary<string, PoolStatistics>();
    
    private readonly Dictionary<IReadOnlyEntityView, DateTime> rentTimes
        = new Dictionary<IReadOnlyEntityView, DateTime>();
    
    public override IReadOnlyEntityView Rent(string name)
    {
        ValidateViewName(name);
        UpdateStatistics(name, stats => stats.TotalRents++);
        
        var view = GetOrCreateView(name);
        rentTimes[view] = DateTime.Now;
        
        OnViewRented(name, view);
        return view;
    }
    
    public override void Return(string name, IReadOnlyEntityView view)
    {
        ValidateViewName(name);
        ValidateView(view);
        
        UpdateStatistics(name, stats =>
        {
            stats.TotalReturns++;
            
            if (rentTimes.TryGetValue(view, out var rentTime))
            {
                var lifetime = (float)(DateTime.Now - rentTime).TotalSeconds;
                stats.AverageLifetime = (stats.AverageLifetime * (stats.TotalReturns - 1) + lifetime) / stats.TotalReturns;
                rentTimes.Remove(view);
            }
        });
        
        ReturnToPool(name, view);
        OnViewReturned(name, view);
    }
    
    public override void Clear()
    {
        foreach (var kvp in pools)
        {
            var poolName = kvp.Key;
            var pool = kvp.Value;
            
            while (pool.Count > 0)
            {
                var view = pool.Pop();
                DestroyPooledView(view);
            }
            
            UpdateStatistics(poolName, stats => stats.CurrentPooled = 0);
        }
        
        pools.Clear();
        rentTimes.Clear();
        OnPoolCleared();
    }
    
    public PoolStatistics GetStatistics(string name)
    {
        return statistics.TryGetValue(name, out var stats) ? stats : new PoolStatistics();
    }
    
    public Dictionary<string, PoolStatistics> GetAllStatistics()
    {
        return new Dictionary<string, PoolStatistics>(statistics);
    }
    
    private void UpdateStatistics(string name, System.Action<PoolStatistics> update)
    {
        if (!statistics.ContainsKey(name))
        {
            statistics[name] = new PoolStatistics();
        }
        
        var stats = statistics[name];
        update(stats);
        stats.LastActivity = DateTime.Now;
        
        // Update current pooled count
        stats.CurrentPooled = pools.TryGetValue(name, out var pool) ? pool.Count : 0;
        stats.PeakPooled = Math.Max(stats.PeakPooled, stats.CurrentPooled);
    }
    
    protected override void OnViewRented(string name, IReadOnlyEntityView view)
    {
        base.OnViewRented(name, view);
        LogPoolOperation("RENT", name);
    }
    
    protected override void OnViewReturned(string name, IReadOnlyEntityView view)
    {
        base.OnViewReturned(name, view);
        LogPoolOperation("RETURN", name);
    }
}
```

## Common Base Utilities

### Validation Methods
```csharp
protected void ValidateViewName(string name)
{
    if (string.IsNullOrEmpty(name))
        throw new ArgumentException("View name cannot be null or empty", nameof(name));
}

protected void ValidateView(IReadOnlyEntityView view)
{
    if (view == null)
        throw new ArgumentNullException(nameof(view));
}

protected void ValidateCapacity(int capacity)
{
    if (capacity < 0)
        throw new ArgumentOutOfRangeException(nameof(capacity), "Capacity cannot be negative");
}
```

### Logging Methods
```csharp
protected void LogPoolOperation(string operation, string viewName)
{
    Debug.Log($"[EntityViewPool] {operation}: {viewName} at {DateTime.Now:HH:mm:ss}");
}

protected void LogPoolState(string poolName, int count)
{
    Debug.Log($"[EntityViewPool] Pool '{poolName}' has {count} instances");
}

protected void LogError(string message, System.Exception exception = null)
{
    if (exception != null)
        Debug.LogError($"[EntityViewPool] ERROR: {message}\n{exception}");
    else
        Debug.LogError($"[EntityViewPool] ERROR: {message}");
}
```

### Resource Management
```csharp
protected virtual void DisposeView(IReadOnlyEntityView view)
{
    if (view is System.IDisposable disposable)
    {
        disposable.Dispose();
    }
    
    if (view is MonoBehaviour mb)
    {
        Object.Destroy(mb.gameObject);
    }
}

protected virtual void ResetViewState(IReadOnlyEntityView view)
{
    if (view is IResettableView resettable)
    {
        resettable.Reset();
    }
    
    if (view is IPoolableView poolable)
    {
        poolable.OnRentFromPool();
    }
}
```

## Extension Patterns

### Thread-Safe Pool Base
```csharp
public abstract class ThreadSafeEntityViewPoolBase : EntityViewPoolBase
{
    protected readonly object lockObject = new object();
    
    public override IReadOnlyEntityView Rent(string name)
    {
        lock (lockObject)
        {
            return base.Rent(name);
        }
    }
    
    public override void Return(string name, IReadOnlyEntityView view)
    {
        lock (lockObject)
        {
            base.Return(name, view);
        }
    }
    
    public override void Clear()
    {
        lock (lockObject)
        {
            base.Clear();
        }
    }
}
```

### Disposable Pool Base
```csharp
public abstract class DisposableEntityViewPoolBase : EntityViewPoolBase, System.IDisposable
{
    private bool disposed = false;
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!disposed)
        {
            if (disposing)
            {
                Clear();
            }
            disposed = true;
        }
    }
    
    protected void ThrowIfDisposed()
    {
        if (disposed)
            throw new ObjectDisposedException(GetType().Name);
    }
    
    public override IReadOnlyEntityView Rent(string name)
    {
        ThrowIfDisposed();
        return base.Rent(name);
    }
}
```

## Best Practices for Base Classes

1. **Abstract Design** – Keep base class focused on common functionality
2. **Virtual Methods** – Provide extension points for customization
3. **Error Handling** – Implement consistent error handling patterns
4. **Resource Management** – Include proper cleanup and disposal
5. **Documentation** – Document virtual methods and extension points
6. **Performance** – Optimize common operations in base class
7. **Thread Safety** – Consider thread safety requirements in base design

## Integration Benefits

### Consistency
- Standardized error messages and handling
- Consistent logging and debugging output
- Uniform validation and safety checks

### Maintainability
- Shared code reduces duplication
- Centralized updates affect all implementations
- Common patterns make code more readable

### Extensibility
- Virtual methods allow customization
- Abstract methods enforce interface compliance
- Base utilities reduce implementation complexity

## Notes

- EntityViewPoolBase provides structure without enforcing specific storage mechanisms
- Virtual methods enable customization while maintaining core functionality
- Built-in validation and error handling improve reliability
- Logging and debugging support aids in development and troubleshooting
- Resource management patterns help prevent memory leaks
- Statistics and monitoring capabilities can be built into base classes