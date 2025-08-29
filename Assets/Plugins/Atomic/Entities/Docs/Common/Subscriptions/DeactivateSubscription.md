# 🔄 DeactivateSubscription

`DeactivateSubscription` is a disposable subscription handle that automatically unregisters a callback from an `IActivatable`'s `OnDeactivated` event when disposed. It provides safe event subscription management for handling entity deactivation events.

## Key Features

- **Automatic Cleanup** – Unsubscribes callback on disposal
- **Memory Safe** – Prevents event handler memory leaks
- **Deactivation Focus** – Specifically handles deactivation events
- **Struct Design** – Lightweight, no heap allocation
- **Null Safe** – Handles null sources and callbacks gracefully

---

## Structure Definition

```csharp
public readonly struct DeactivateSubscription : IDisposable
{
    private readonly IActivatable _source;
    private readonly Action _callback;
}
```

### Constructor

```csharp
internal DeactivateSubscription(IActivatable source, Action callback)
```

- **source**: The activatable object to subscribe to
- **callback**: The action to invoke when deactivated

### Disposal

```csharp
public void Dispose()
```

Safely unsubscribes the callback from the source's `OnDeactivated` event.

---

## Usage Patterns

### Basic Deactivation Handling

```csharp
public class DeactivationHandler
{
    private DeactivateSubscription subscription;
    
    public void StartMonitoring(IActivatable entity)
    {
        subscription = entity.SubscribeToDeactivate(OnEntityDeactivated);
    }
    
    public void StopMonitoring()
    {
        subscription.Dispose();
    }
    
    private void OnEntityDeactivated()
    {
        Debug.Log("Entity was deactivated!");
        HandleDeactivation();
    }
}
```

### Resource Cleanup on Deactivation

```csharp
public class ResourceManager
{
    private Dictionary<IActivatable, DeactivateSubscription> cleanupSubscriptions = new();
    private Dictionary<IActivatable, List<IResource>> entityResources = new();
    
    public void RegisterEntity(IActivatable entity)
    {
        // Subscribe to deactivation for automatic resource cleanup
        var subscription = entity.SubscribeToDeactivate(() => CleanupResources(entity));
        cleanupSubscriptions[entity] = subscription;
        entityResources[entity] = new List<IResource>();
    }
    
    public void AllocateResource(IActivatable entity, IResource resource)
    {
        if (entityResources.ContainsKey(entity))
        {
            entityResources[entity].Add(resource);
        }
    }
    
    private void CleanupResources(IActivatable entity)
    {
        if (entityResources.TryGetValue(entity, out var resources))
        {
            foreach (var resource in resources)
            {
                resource.Dispose();
            }
            resources.Clear();
            Debug.Log($"Cleaned up {resources.Count} resources for deactivated entity");
        }
        
        // Clean up the subscription itself
        if (cleanupSubscriptions.TryGetValue(entity, out var subscription))
        {
            subscription.Dispose();
            cleanupSubscriptions.Remove(entity);
        }
    }
}
```

### State Transition Monitoring

```csharp
public class StateTransitionLogger
{
    private readonly Dictionary<IActivatable, (ActivateSubscription, DeactivateSubscription)> subscriptions = new();
    
    public void MonitorEntity(IActivatable entity)
    {
        var activateSubscription = entity.SubscribeToActivate(() => LogStateChange(entity, true));
        var deactivateSubscription = entity.SubscribeToDeactivate(() => LogStateChange(entity, false));
        
        subscriptions[entity] = (activateSubscription, deactivateSubscription);
    }
    
    private void LogStateChange(IActivatable entity, bool activated)
    {
        string state = activated ? "activated" : "deactivated";
        Debug.Log($"Entity {entity.GetHashCode()} {state} at {DateTime.Now}");
        UpdateStateHistory(entity, activated);
    }
    
    public void StopMonitoring(IActivatable entity)
    {
        if (subscriptions.TryGetValue(entity, out var subs))
        {
            subs.Item1.Dispose(); // ActivateSubscription
            subs.Item2.Dispose(); // DeactivateSubscription
            subscriptions.Remove(entity);
        }
    }
}
```

### Performance Monitor

```csharp
public class EntityPerformanceMonitor
{
    private readonly Dictionary<IActivatable, DateTime> activationTimes = new();
    private readonly List<DeactivateSubscription> subscriptions = new();
    
    public void StartMonitoring(IEnumerable<IActivatable> entities)
    {
        foreach (var entity in entities)
        {
            // Track when entity gets deactivated for performance metrics
            var subscription = entity.SubscribeToDeactivate(() => RecordDeactivation(entity));
            subscriptions.Add(subscription);
            
            if (entity.Enabled) // Already active
            {
                activationTimes[entity] = DateTime.Now;
            }
        }
    }
    
    private void RecordDeactivation(IActivatable entity)
    {
        if (activationTimes.TryGetValue(entity, out var activationTime))
        {
            var duration = DateTime.Now - activationTime;
            RecordActiveDuration(entity, duration);
            activationTimes.Remove(entity);
        }
    }
    
    private void RecordActiveDuration(IActivatable entity, TimeSpan duration)
    {
        Debug.Log($"Entity was active for {duration.TotalSeconds:F2} seconds");
        UpdatePerformanceMetrics(entity, duration);
    }
    
    public void StopMonitoring()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
        activationTimes.Clear();
    }
}
```

### Conditional Deactivation Response

```csharp
public class ConditionalDeactivationHandler
{
    private readonly Dictionary<IActivatable, DeactivateSubscription> conditionalSubscriptions = new();
    
    public void RegisterConditionalHandler(IActivatable entity, Func<bool> condition, Action response)
    {
        var subscription = entity.SubscribeToDeactivate(() => 
        {
            if (condition())
            {
                response();
            }
        });
        
        conditionalSubscriptions[entity] = subscription;
    }
    
    public void UnregisterHandler(IActivatable entity)
    {
        if (conditionalSubscriptions.TryGetValue(entity, out var subscription))
        {
            subscription.Dispose();
            conditionalSubscriptions.Remove(entity);
        }
    }
}

// Usage example
public class GameManager
{
    private ConditionalDeactivationHandler deactivationHandler = new();
    
    public void SetupPlayerDeactivationHandling(IActivatable player)
    {
        deactivationHandler.RegisterConditionalHandler(
            player,
            condition: () => GameState.IsInCombat,
            response: () => TriggerCombatEndSequence()
        );
    }
}
```

### Event Aggregation

```csharp
public class DeactivationEventAggregator
{
    public event Action<IActivatable> OnAnyEntityDeactivated;
    private readonly List<DeactivateSubscription> subscriptions = new();
    
    public void AggregateDeactivationEvents(IEnumerable<IActivatable> entities)
    {
        foreach (var entity in entities)
        {
            var subscription = entity.SubscribeToDeactivate(() => 
            {
                OnAnyEntityDeactivated?.Invoke(entity);
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
public class DeactivationProcessor
{
    private DeactivationEventAggregator aggregator = new();
    
    public void Initialize(IEnumerable<IActivatable> entities)
    {
        aggregator.AggregateDeactivationEvents(entities);
        aggregator.OnAnyEntityDeactivated += HandleEntityDeactivated;
    }
    
    private void HandleEntityDeactivated(IActivatable entity)
    {
        ProcessDeactivation(entity);
        UpdateDeactivationStatistics();
    }
}
```

## Integration with Game Systems

### UI System Integration

```csharp
public class UIDeactivationHandler
{
    private readonly Dictionary<IActivatable, DeactivateSubscription> uiSubscriptions = new();
    
    public void RegisterUIElement(IActivatable uiElement, GameObject visualElement)
    {
        var subscription = uiElement.SubscribeToDeactivate(() => 
        {
            // Hide visual element when logical element deactivates
            visualElement.SetActive(false);
            OnUIElementDeactivated(uiElement);
        });
        
        uiSubscriptions[uiElement] = subscription;
    }
    
    private void OnUIElementDeactivated(IActivatable element)
    {
        UpdateUIState();
        TriggerUIAnimations();
    }
    
    public void UnregisterUIElement(IActivatable uiElement)
    {
        if (uiSubscriptions.TryGetValue(uiElement, out var subscription))
        {
            subscription.Dispose();
            uiSubscriptions.Remove(uiElement);
        }
    }
}
```

### Save System Integration

```csharp
public class SaveStateManager
{
    private readonly List<DeactivateSubscription> saveSubscriptions = new();
    
    public void RegisterSaveableEntity(IActivatable entity, ISaveable saveData)
    {
        // Auto-save when entity deactivates
        var subscription = entity.SubscribeToDeactivate(() => 
        {
            if (ShouldSaveOnDeactivation(entity))
            {
                SaveEntityState(entity, saveData);
            }
        });
        
        saveSubscriptions.Add(subscription);
    }
    
    private bool ShouldSaveOnDeactivation(IActivatable entity)
    {
        // Only save if entity has been modified
        return HasUnsavedChanges(entity);
    }
    
    private void SaveEntityState(IActivatable entity, ISaveable saveData)
    {
        Debug.Log($"Auto-saving entity state on deactivation");
        saveData.Save();
    }
}
```

## Best Practices

### Lifecycle Management
1. **Pair with activation** – Often used alongside ActivateSubscription
2. **Clean up resources** – Use deactivation for resource cleanup
3. **State consistency** – Ensure state remains consistent after deactivation
4. **Timing considerations** – Handle deactivation at appropriate game loop phases

### Performance Optimization
1. **Batch deactivation handling** – Process multiple deactivations together
2. **Avoid heavy operations** – Keep deactivation callbacks lightweight
3. **Cache subscription references** – Store subscriptions for efficient cleanup
4. **Profile callback performance** – Monitor deactivation handler performance

### Error Handling
1. **Exception safety** – Wrap deactivation logic in try-catch
2. **Null checks** – Validate entity state during deactivation
3. **Graceful degradation** – Handle failed deactivation gracefully

## Common Use Cases

- **Resource cleanup** when entities are no longer active
- **UI state management** for deactivated interface elements  
- **Performance monitoring** to track active/inactive durations
- **Save system triggers** for persistent state management
- **Event logging** for debugging and analytics
- **State machine transitions** based on deactivation events

## Thread Safety

- **Main thread only** – Use only on Unity's main thread
- **Event thread safety** – Callbacks execute on event thread
- **Disposal safety** – Safe to dispose from any thread
- **Synchronization** – Add locks if shared across threads