# 🔄 UpdateSubscription

`UpdateSubscription` is a disposable subscription handle that automatically unregisters a callback from an `IUpdatable`'s `OnUpdated` event when disposed. It provides safe event subscription management for frame-based update events.

## Key Features

- **Frame-Based Updates** – Handles standard Update cycle events
- **Automatic Cleanup** – Unsubscribes callback on disposal
- **Delta Time Support** – Callbacks receive deltaTime parameter
- **Memory Safe** – Prevents event handler memory leaks
- **Performance Focused** – Lightweight struct design

---

## Structure Definition

```csharp
public readonly struct UpdateSubscription : IDisposable
{
    private readonly IUpdatable _source;
    private readonly Action<float> _callback;
}
```

### Constructor

```csharp
internal UpdateSubscription(IUpdatable source, Action<float> callback)
```

- **source**: The updatable object to subscribe to
- **callback**: The action to invoke on each update (receives deltaTime)

### Disposal

```csharp
public void Dispose()
```

Safely unsubscribes the callback from the source's `OnUpdated` event.

---

## Usage Patterns

### Basic Update Monitoring

```csharp
public class UpdateMonitor
{
    private UpdateSubscription subscription;
    
    public void StartMonitoring(IUpdatable system)
    {
        subscription = system.SubscribeToUpdate(OnSystemUpdated);
    }
    
    public void StopMonitoring()
    {
        subscription.Dispose();
    }
    
    private void OnSystemUpdated(float deltaTime)
    {
        Debug.Log($"System updated with deltaTime: {deltaTime:F3}");
        ProcessUpdateEvent(deltaTime);
    }
}
```

### Frame Rate Analysis

```csharp
public class FrameRateAnalyzer
{
    private readonly List<UpdateSubscription> subscriptions = new();
    private readonly List<float> frameTimes = new();
    private float timeAccumulator;
    private int frameCount;
    
    public void StartAnalyzing(IEnumerable<IUpdatable> systems)
    {
        foreach (var system in systems)
        {
            var subscription = system.SubscribeToUpdate(OnSystemUpdate);
            subscriptions.Add(subscription);
        }
    }
    
    private void OnSystemUpdate(float deltaTime)
    {
        frameTimes.Add(deltaTime);
        timeAccumulator += deltaTime;
        frameCount++;
        
        // Report every second
        if (timeAccumulator >= 1f)
        {
            float averageFps = frameCount / timeAccumulator;
            float minFrameTime = frameTimes.Min();
            float maxFrameTime = frameTimes.Max();
            
            Debug.Log($"FPS: {averageFps:F1}, Frame time range: {minFrameTime:F3}ms - {maxFrameTime:F3}ms");
            
            // Reset
            frameTimes.Clear();
            timeAccumulator = 0f;
            frameCount = 0;
        }
    }
    
    public void StopAnalyzing()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
        frameTimes.Clear();
    }
}
```

### Conditional Update Processing

```csharp
public class ConditionalUpdateProcessor
{
    private readonly Dictionary<IUpdatable, (Func<bool> condition, UpdateSubscription subscription)> conditionalSubscriptions = new();
    
    public void RegisterConditionalUpdate(IUpdatable system, Func<bool> condition, Action<float> processor)
    {
        var subscription = system.SubscribeToUpdate(deltaTime => 
        {
            if (condition())
            {
                processor(deltaTime);
            }
        });
        
        conditionalSubscriptions[system] = (condition, subscription);
    }
    
    public void UnregisterConditionalUpdate(IUpdatable system)
    {
        if (conditionalSubscriptions.TryGetValue(system, out var data))
        {
            data.subscription.Dispose();
            conditionalSubscriptions.Remove(system);
        }
    }
}

// Usage example
public class GameUpdateManager
{
    private ConditionalUpdateProcessor processor = new();
    
    public void SetupConditionalUpdates(IUpdatable gameSystem)
    {
        // Only process updates during gameplay
        processor.RegisterConditionalUpdate(
            gameSystem,
            condition: () => GameState.CurrentState == GameState.Playing,
            processor: (deltaTime) => ProcessGameplayUpdate(deltaTime)
        );
        
        // Only process updates when not paused
        processor.RegisterConditionalUpdate(
            gameSystem,
            condition: () => !GameState.IsPaused,
            processor: (deltaTime) => ProcessNonPausedUpdate(deltaTime)
        );
    }
}
```

### Update Performance Profiler

```csharp
public class UpdateProfiler
{
    private readonly Dictionary<IUpdatable, UpdateMetrics> systemMetrics = new();
    private readonly List<UpdateSubscription> subscriptions = new();
    
    public struct UpdateMetrics
    {
        public int UpdateCount;
        public float TotalTime;
        public float AverageTime;
        public float MaxTime;
        public float MinTime;
    }
    
    public void StartProfiling(IEnumerable<IUpdatable> systems)
    {
        foreach (var system in systems)
        {
            var subscription = system.SubscribeToUpdate(deltaTime => ProfileUpdate(system, deltaTime));
            subscriptions.Add(subscription);
            
            systemMetrics[system] = new UpdateMetrics
            {
                MinTime = float.MaxValue
            };
        }
    }
    
    private void ProfileUpdate(IUpdatable system, float deltaTime)
    {
        var startTime = Time.realtimeSinceStartup;
        
        // The actual update logic would be measured here
        // For subscription monitoring, we just track the delta time
        var metrics = systemMetrics[system];
        
        metrics.UpdateCount++;
        metrics.TotalTime += deltaTime;
        metrics.AverageTime = metrics.TotalTime / metrics.UpdateCount;
        metrics.MaxTime = Mathf.Max(metrics.MaxTime, deltaTime);
        metrics.MinTime = Mathf.Min(metrics.MinTime, deltaTime);
        
        systemMetrics[system] = metrics;
        
        // Log performance warnings
        if (deltaTime > 0.016f) // 60 FPS threshold
        {
            Debug.LogWarning($"Slow frame detected: {deltaTime:F3}s for {system.GetType().Name}");
        }
    }
    
    public UpdateMetrics GetMetrics(IUpdatable system)
    {
        return systemMetrics.TryGetValue(system, out var metrics) ? metrics : default;
    }
    
    public void PrintReport()
    {
        Debug.Log("=== Update Performance Report ===");
        foreach (var kvp in systemMetrics)
        {
            var system = kvp.Key;
            var metrics = kvp.Value;
            
            Debug.Log($"{system.GetType().Name}:");
            Debug.Log($"  Updates: {metrics.UpdateCount}");
            Debug.Log($"  Avg Time: {metrics.AverageTime:F3}s");
            Debug.Log($"  Max Time: {metrics.MaxTime:F3}s");
            Debug.Log($"  Min Time: {metrics.MinTime:F3}s");
        }
    }
}
```

### Update Throttling System

```csharp
public class UpdateThrottler
{
    private readonly Dictionary<IUpdatable, (float interval, float lastUpdate, UpdateSubscription subscription)> throttledSystems = new();
    
    public void RegisterThrottledSystem(IUpdatable system, float updateInterval, Action<float> throttledCallback)
    {
        var subscription = system.SubscribeToUpdate(deltaTime => 
            ProcessThrottledUpdate(system, deltaTime, throttledCallback));
            
        throttledSystems[system] = (updateInterval, 0f, subscription);
    }
    
    private void ProcessThrottledUpdate(IUpdatable system, float deltaTime, Action<float> callback)
    {
        if (throttledSystems.TryGetValue(system, out var data))
        {
            data.lastUpdate += deltaTime;
            
            if (data.lastUpdate >= data.interval)
            {
                // Execute throttled callback with accumulated time
                callback(data.lastUpdate);
                
                // Reset timer
                data.lastUpdate = 0f;
                throttledSystems[system] = (data.interval, data.lastUpdate, data.subscription);
            }
            else
            {
                // Update timer
                throttledSystems[system] = (data.interval, data.lastUpdate, data.subscription);
            }
        }
    }
    
    public void UnregisterThrottledSystem(IUpdatable system)
    {
        if (throttledSystems.TryGetValue(system, out var data))
        {
            data.subscription.Dispose();
            throttledSystems.Remove(system);
        }
    }
}

// Usage
public class GameManager
{
    private UpdateThrottler throttler = new();
    
    public void SetupThrottledSystems(IUpdatable aiSystem, IUpdatable uiSystem)
    {
        // AI updates every 0.1 seconds instead of every frame
        throttler.RegisterThrottledSystem(aiSystem, 0.1f, 
            (accumulatedTime) => ProcessAIUpdate(accumulatedTime));
            
        // UI updates every 0.05 seconds for responsive feel
        throttler.RegisterThrottledSystem(uiSystem, 0.05f,
            (accumulatedTime) => ProcessUIUpdate(accumulatedTime));
    }
}
```

### Update Event Aggregator

```csharp
public class UpdateEventAggregator
{
    public event Action<IUpdatable, float> OnAnySystemUpdated;
    private readonly List<UpdateSubscription> subscriptions = new();
    
    public void AggregateUpdates(IEnumerable<IUpdatable> systems)
    {
        foreach (var system in systems)
        {
            var subscription = system.SubscribeToUpdate(deltaTime => 
            {
                OnAnySystemUpdated?.Invoke(system, deltaTime);
            });
            subscriptions.Add(subscription);
        }
    }
    
    public void Dispose()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
    }
}

// Usage
public class UpdateLogger
{
    private UpdateEventAggregator aggregator = new();
    
    public void Initialize(IEnumerable<IUpdatable> systems)
    {
        aggregator.AggregateUpdates(systems);
        aggregator.OnAnySystemUpdated += LogSystemUpdate;
    }
    
    private void LogSystemUpdate(IUpdatable system, float deltaTime)
    {
        Debug.Log($"System {system.GetType().Name} updated: {deltaTime:F4}s");
        RecordSystemActivity(system, deltaTime);
    }
}
```

### Smooth Value Interpolator

```csharp
public class SmoothValueInterpolator
{
    private readonly Dictionary<string, (float current, float target, float speed, UpdateSubscription subscription)> values = new();
    
    public void RegisterValue(IUpdatable system, string valueName, float initialValue, float speed = 1f)
    {
        var subscription = system.SubscribeToUpdate(deltaTime => UpdateValue(valueName, deltaTime));
        values[valueName] = (initialValue, initialValue, speed, subscription);
    }
    
    public void SetTarget(string valueName, float targetValue)
    {
        if (values.TryGetValue(valueName, out var data))
        {
            values[valueName] = (data.current, targetValue, data.speed, data.subscription);
        }
    }
    
    private void UpdateValue(string valueName, float deltaTime)
    {
        if (values.TryGetValue(valueName, out var data))
        {
            if (Mathf.Approximately(data.current, data.target))
                return;
                
            float newValue = Mathf.MoveTowards(data.current, data.target, data.speed * deltaTime);
            values[valueName] = (newValue, data.target, data.speed, data.subscription);
            
            OnValueChanged?.Invoke(valueName, newValue);
        }
    }
    
    public float GetValue(string valueName)
    {
        return values.TryGetValue(valueName, out var data) ? data.current : 0f;
    }
    
    public event Action<string, float> OnValueChanged;
    
    public void UnregisterValue(string valueName)
    {
        if (values.TryGetValue(valueName, out var data))
        {
            data.subscription.Dispose();
            values.Remove(valueName);
        }
    }
}
```

## Integration with Game Systems

### Entity Update Monitoring

```csharp
public class EntityUpdateSystem
{
    private readonly List<UpdateSubscription> entitySubscriptions = new();
    
    public void MonitorEntityUpdates(IEntityWorld world)
    {
        foreach (var entity in world.Entities.OfType<IUpdatable>())
        {
            var subscription = entity.SubscribeToUpdate(deltaTime => ProcessEntityUpdate(entity, deltaTime));
            entitySubscriptions.Add(subscription);
        }
        
        world.OnEntityAdded += OnEntityAdded;
    }
    
    private void OnEntityAdded(IEntity entity)
    {
        if (entity is IUpdatable updatable)
        {
            var subscription = updatable.SubscribeToUpdate(deltaTime => ProcessEntityUpdate(updatable, deltaTime));
            entitySubscriptions.Add(subscription);
        }
    }
    
    private void ProcessEntityUpdate(IUpdatable entity, float deltaTime)
    {
        // Process entity-specific update logic
        UpdateEntityState(entity, deltaTime);
        CheckEntityConditions(entity);
    }
}
```

## Best Practices

### Performance Considerations
1. **Lightweight callbacks** – Keep update handlers fast
2. **Avoid allocations** – Don't create objects in update callbacks
3. **Batch operations** – Group related updates together
4. **Profile regularly** – Monitor update performance impact

### Memory Management
1. **Dispose properly** – Always dispose subscriptions when done
2. **Avoid closures** – Use static methods to prevent captures
3. **Cache references** – Store frequently used objects
4. **Monitor subscriptions** – Track active subscription count

### Update Logic
1. **Use deltaTime** – Always use deltaTime for frame-independent behavior
2. **Guard conditions** – Check preconditions before processing
3. **Exception handling** – Wrap update logic in try-catch blocks
4. **Early exits** – Return early when no work needed

## Common Use Cases

- **Performance profiling** and frame rate analysis
- **Conditional processing** based on game state
- **Update throttling** for expensive operations
- **Value interpolation** and smooth transitions
- **Event aggregation** across multiple systems
- **Debug monitoring** and logging systems

## Thread Safety

- **Main thread only** – All operations must be on main thread
- **Unity integration** – Callbacks execute during Unity's update cycle
- **No synchronization** – Single-threaded by design
- **Disposal safety** – Safe to dispose from main thread