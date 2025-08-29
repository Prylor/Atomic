# 🔛 ActivateSubscription

`ActivateSubscription` is a disposable subscription handle that automatically unregisters a callback from an `IActivatable`'s `OnActivated` event when disposed. It provides safe and deterministic event subscription management for temporary or scoped listeners.

## Key Features

- **Automatic Cleanup** – Unsubscribes callback on disposal
- **Memory Safe** – Prevents event handler memory leaks
- **Scoped Lifetime** – Perfect for temporary subscriptions
- **Struct Design** – Lightweight, no heap allocation
- **Null Safe** – Handles null sources and callbacks gracefully

---

## Structure Definition

```csharp
public readonly struct ActivateSubscription : IDisposable
{
    private readonly IActivatable _source;
    private readonly Action _callback;
}
```

### Constructor

```csharp
internal ActivateSubscription(IActivatable source, Action callback)
```

- **source**: The activatable object to subscribe to
- **callback**: The action to invoke when activated

### Disposal

```csharp
public void Dispose()
```

Safely unsubscribes the callback from the source's `OnActivated` event.

---

## Usage Patterns

### Basic Subscription

```csharp
public class ActivationListener
{
    private ActivateSubscription subscription;
    
    public void StartListening(IActivatable entity)
    {
        // Create subscription to activation events
        subscription = entity.SubscribeToActivate(OnEntityActivated);
    }
    
    public void StopListening()
    {
        // Automatically unsubscribes
        subscription.Dispose();
    }
    
    private void OnEntityActivated()
    {
        Debug.Log("Entity was activated!");
    }
}
```

### Using Statement Pattern

```csharp
public class TemporaryActivationWatcher
{
    public void MonitorActivation(IActivatable entity, float duration)
    {
        // Automatic cleanup when using block exits
        using var subscription = entity.SubscribeToActivate(() => 
        {
            Debug.Log($"Entity {entity} activated!");
            OnActivationDetected(entity);
        });
        
        // Wait for specified duration
        Wait(duration);
        
        // Subscription automatically disposed here
    }
}
```

### Event Chain Management

```csharp
public class EntityStateTracker
{
    private List<ActivateSubscription> activeSubscriptions = new List<ActivateSubscription>();
    
    public void TrackEntities(IEnumerable<IActivatable> entities)
    {
        foreach (var entity in entities)
        {
            var subscription = entity.SubscribeToActivate(() => OnEntityActivated(entity));
            activeSubscriptions.Add(subscription);
        }
    }
    
    public void StopTracking()
    {
        // Clean up all subscriptions
        foreach (var subscription in activeSubscriptions)
        {
            subscription.Dispose();
        }
        activeSubscriptions.Clear();
    }
    
    private void OnEntityActivated(IActivatable entity)
    {
        UpdateEntityState(entity, true);
    }
}
```

### Conditional Subscription

```csharp
public class ConditionalActivationHandler
{
    private ActivateSubscription? subscription;
    private bool shouldListen;
    
    public void SetListening(IActivatable entity, bool enabled)
    {
        if (enabled && !shouldListen)
        {
            // Start listening
            subscription = entity.SubscribeToActivate(OnActivated);
            shouldListen = true;
        }
        else if (!enabled && shouldListen)
        {
            // Stop listening
            subscription?.Dispose();
            subscription = null;
            shouldListen = false;
        }
    }
    
    private void OnActivated()
    {
        ProcessActivation();
    }
}
```

### Scoped Event Handling

```csharp
public class GamePhaseManager
{
    public void RunGamePhase(IActivatable gameSystem, Action onPhaseComplete)
    {
        Debug.Log("Starting game phase...");
        
        // Subscribe to activation only for this phase
        using var activationSub = gameSystem.SubscribeToActivate(() => 
        {
            Debug.Log("Game system activated for this phase");
            HandleSystemActivation();
        });
        
        // Run the phase logic
        ExecutePhaseLogic();
        
        // Subscription automatically cleaned up when method exits
        onPhaseComplete?.Invoke();
    }
}
```

### Multiple Entity Monitoring

```csharp
public class ActivationAggregator
{
    private Dictionary<IActivatable, ActivateSubscription> subscriptions = new();
    private int activatedCount;
    private readonly int requiredCount;
    
    public ActivationAggregator(int requiredActivations)
    {
        requiredCount = requiredActivations;
    }
    
    public void MonitorEntities(IEnumerable<IActivatable> entities)
    {
        foreach (var entity in entities)
        {
            var subscription = entity.SubscribeToActivate(() => OnEntityActivated(entity));
            subscriptions[entity] = subscription;
        }
    }
    
    private void OnEntityActivated(IActivatable entity)
    {
        activatedCount++;
        Debug.Log($"Entity activated. Count: {activatedCount}/{requiredCount}");
        
        if (activatedCount >= requiredCount)
        {
            OnAllEntitiesActivated();
            CleanupSubscriptions();
        }
    }
    
    private void OnAllEntitiesActivated()
    {
        Debug.Log("All required entities have been activated!");
    }
    
    private void CleanupSubscriptions()
    {
        foreach (var subscription in subscriptions.Values)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
    }
}
```

### Lifecycle-Aware Subscription

```csharp
public class EntityLifecycleMonitor : IDisposable
{
    private readonly List<IDisposable> subscriptions = new();
    
    public void MonitorEntity(IActivatable entity)
    {
        // Track activation with automatic cleanup
        var activateSubscription = entity.SubscribeToActivate(() => 
        {
            Debug.Log($"Entity {entity.GetHashCode()} activated");
            OnEntityStateChanged(entity, true);
        });
        
        subscriptions.Add(activateSubscription);
    }
    
    private void OnEntityStateChanged(IActivatable entity, bool activated)
    {
        UpdateEntityDatabase(entity, activated);
        TriggerStateChangeEvents(entity, activated);
    }
    
    public void Dispose()
    {
        // Clean up all subscriptions
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
    }
}
```

### Performance-Conscious Usage

```csharp
public class OptimizedActivationHandler
{
    private readonly Dictionary<int, ActivateSubscription> entitySubscriptions = new();
    private readonly HashSet<int> activeEntities = new();
    
    public void RegisterEntity(IActivatable entity)
    {
        int entityId = entity.GetHashCode();
        
        // Avoid duplicate subscriptions
        if (entitySubscriptions.ContainsKey(entityId))
            return;
            
        var subscription = entity.SubscribeToActivate(() => OnEntityActivated(entityId));
        entitySubscriptions[entityId] = subscription;
    }
    
    public void UnregisterEntity(IActivatable entity)
    {
        int entityId = entity.GetHashCode();
        
        if (entitySubscriptions.TryGetValue(entityId, out var subscription))
        {
            subscription.Dispose();
            entitySubscriptions.Remove(entityId);
            activeEntities.Remove(entityId);
        }
    }
    
    private void OnEntityActivated(int entityId)
    {
        activeEntities.Add(entityId);
        ProcessActivatedEntities();
    }
    
    private void ProcessActivatedEntities()
    {
        // Batch process all activated entities
        foreach (int entityId in activeEntities)
        {
            UpdateEntityState(entityId);
        }
    }
}
```

## Integration Examples

### With Entity System

```csharp
public class EntityActivationSystem
{
    private readonly List<ActivateSubscription> subscriptions = new();
    
    public void Initialize(IEntityWorld world)
    {
        // Subscribe to all existing activatable entities
        foreach (var entity in world.Entities.OfType<IActivatable>())
        {
            var subscription = entity.SubscribeToActivate(() => HandleActivation(entity));
            subscriptions.Add(subscription);
        }
        
        // Subscribe to new entities
        world.OnEntityAdded += OnEntityAdded;
    }
    
    private void OnEntityAdded(IEntity entity)
    {
        if (entity is IActivatable activatable)
        {
            var subscription = activatable.SubscribeToActivate(() => HandleActivation(activatable));
            subscriptions.Add(subscription);
        }
    }
    
    private void HandleActivation(IActivatable entity)
    {
        // Process entity activation
        UpdateActivationStatistics();
        TriggerActivationEffects(entity);
    }
    
    public void Shutdown()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        subscriptions.Clear();
    }
}
```

## Best Practices

### Subscription Management
1. **Always dispose** – Use using statements or explicit disposal
2. **Check subscription validity** – Handle null sources gracefully
3. **Avoid memory leaks** – Don't forget to unsubscribe
4. **Use scoped subscriptions** – Prefer using statements for temporary subscriptions

### Performance Considerations
1. **Batch subscriptions** – Group related subscriptions for efficient management
2. **Avoid frequent subscribe/unsubscribe** – Cache subscriptions when possible
3. **Clean up promptly** – Dispose subscriptions as soon as they're no longer needed
4. **Monitor subscription count** – Track active subscriptions in debug builds

### Error Handling
1. **Null-safe operations** – Handle null sources and callbacks
2. **Exception safety** – Wrap subscription logic in try-catch blocks
3. **Defensive programming** – Validate inputs before creating subscriptions

## Common Anti-patterns

### Don't Do This
```csharp
// ❌ Forgetting to dispose
var subscription = entity.SubscribeToActivate(callback);
// Missing disposal - memory leak!

// ❌ Creating multiple subscriptions for same entity
foreach (var entity in entities)
{
    entity.SubscribeToActivate(callback); // Multiple subscriptions!
}
```

### Do This Instead
```csharp
// ✅ Proper disposal
using var subscription = entity.SubscribeToActivate(callback);

// ✅ Check for existing subscriptions
if (!alreadySubscribed)
{
    var subscription = entity.SubscribeToActivate(callback);
    subscriptions[entity] = subscription;
}
```

## Thread Safety

- **Not thread-safe** – Use only on main thread
- **Event handling** – Callbacks invoked on event thread
- **Disposal safety** – Safe to dispose from any thread
- **Synchronization** – Use locks if shared across threads