# 🔗 Extensions (Common)

`Extensions` provides extension methods for event-based subscriptions to lifecycle interfaces. These extensions offer a fluent API for subscribing to entity lifecycle events with automatic cleanup through disposable subscription handles.

## Key Features

- **Fluent API** – Chain-able extension methods for clean syntax
- **Automatic Cleanup** – Returns disposable subscriptions for safe cleanup
- **Event Subscription** – Simplified event subscription patterns
- **Performance Optimized** – Aggressively inlined for zero overhead
- **Type Safe** – Strongly typed event parameters and return values

---

## Extension Methods

### Spawn Event Subscription

```csharp
public static SpawnSubscription WhenSpawn(this ISpawnable source, Action action)
```

Subscribes to the `OnSpawned` event of an `ISpawnable` object:
- **source**: The spawnable object to monitor
- **action**: Callback to invoke when entity spawns
- **Returns**: `SpawnSubscription` for automatic cleanup

### Despawn Event Subscription

```csharp
public static DespawnSubscription WhenDespawn(this ISpawnable source, Action action)
```

Subscribes to the `OnDespawned` event of an `ISpawnable` object:
- **source**: The spawnable object to monitor
- **action**: Callback to invoke when entity despawns
- **Returns**: `DespawnSubscription` for automatic cleanup

### Activation Event Subscription

```csharp
public static ActivateSubscription WhenActivate(this IActivatable source, Action action)
```

Subscribes to the `OnActivated` event of an `IActivatable` object:
- **source**: The activatable object to monitor
- **action**: Callback to invoke when entity activates
- **Returns**: `ActivateSubscription` for automatic cleanup

### Deactivation Event Subscription

```csharp
public static DeactivateSubscription WhenDeactivate(this IActivatable source, Action action)
```

Subscribes to the `OnDeactivated` event of an `IActivatable` object:
- **source**: The activatable object to monitor
- **action**: Callback to invoke when entity deactivates
- **Returns**: `DeactivateSubscription` for automatic cleanup

### Update Event Subscription

```csharp
public static UpdateSubscription WhenUpdate(this IUpdatable source, Action<float> action)
```

Subscribes to the `OnUpdated` event of an `IUpdatable` object:
- **source**: The updatable object to monitor
- **action**: Callback to invoke each frame (receives deltaTime)
- **Returns**: `UpdateSubscription` for automatic cleanup

### Fixed Update Event Subscription

```csharp
public static FixedUpdateSubscription WhenFixedUpdate(this IUpdatable source, Action<float> action)
```

Subscribes to the `OnFixedUpdated` event of an `IUpdatable` object:
- **source**: The updatable object to monitor
- **action**: Callback to invoke each fixed timestep (receives fixed deltaTime)
- **Returns**: `FixedUpdateSubscription` for automatic cleanup

### Late Update Event Subscription

```csharp
public static LateUpdateSubscription WhenLateUpdate(this IUpdatable source, Action<float> action)
```

Subscribes to the `OnLateUpdated` event of an `IUpdatable` object:
- **source**: The updatable object to monitor
- **action**: Callback to invoke each late update (receives deltaTime)
- **Returns**: `LateUpdateSubscription` for automatic cleanup

---

## Usage Patterns

### Basic Event Subscription

```csharp
public class EntityEventHandler
{
    public void SetupEntityMonitoring(Entity entity)
    {
        // Using extension methods for clean, fluent syntax
        var spawnSubscription = entity.WhenSpawn(() => 
        {
            Debug.Log("Entity spawned!");
            OnEntitySpawned(entity);
        });
        
        var despawnSubscription = entity.WhenDespawn(() => 
        {
            Debug.Log("Entity despawned!");
            OnEntityDespawned(entity);
        });
        
        var activateSubscription = entity.WhenActivate(() => 
        {
            Debug.Log("Entity activated!");
            OnEntityActivated(entity);
        });
        
        // Store subscriptions for later cleanup
        StoreSubscriptions(spawnSubscription, despawnSubscription, activateSubscription);
    }
}
```

### Fluent Chaining Pattern

```csharp
public class FluentEntitySetup
{
    public void ConfigureEntity(Entity entity)
    {
        // Chain multiple subscriptions in a fluent manner
        using var spawnSub = entity.WhenSpawn(() => InitializeEntitySystems(entity));
        using var activateSub = entity.WhenActivate(() => StartEntityLogic(entity));
        using var updateSub = entity.WhenUpdate(deltaTime => UpdateEntityLogic(entity, deltaTime));
        using var despawnSub = entity.WhenDespawn(() => CleanupEntitySystems(entity));
        
        // Entity lifecycle fully managed with automatic cleanup
        entity.Spawn();
    }
}
```

### Conditional Event Handling

```csharp
public class ConditionalEntityHandler
{
    private Dictionary<Entity, List<IDisposable>> entitySubscriptions = new();
    
    public void RegisterEntity(Entity entity, bool monitorSpawn, bool monitorUpdate, bool monitorActivation)
    {
        var subscriptions = new List<IDisposable>();
        
        if (monitorSpawn)
        {
            subscriptions.Add(entity.WhenSpawn(() => HandleSpawnEvent(entity)));
            subscriptions.Add(entity.WhenDespawn(() => HandleDespawnEvent(entity)));
        }
        
        if (monitorUpdate)
        {
            subscriptions.Add(entity.WhenUpdate(deltaTime => HandleUpdateEvent(entity, deltaTime)));
            subscriptions.Add(entity.WhenFixedUpdate(deltaTime => HandleFixedUpdateEvent(entity, deltaTime)));
            subscriptions.Add(entity.WhenLateUpdate(deltaTime => HandleLateUpdateEvent(entity, deltaTime)));
        }
        
        if (monitorActivation)
        {
            subscriptions.Add(entity.WhenActivate(() => HandleActivateEvent(entity)));
            subscriptions.Add(entity.WhenDeactivate(() => HandleDeactivateEvent(entity)));
        }
        
        entitySubscriptions[entity] = subscriptions;
    }
    
    public void UnregisterEntity(Entity entity)
    {
        if (entitySubscriptions.TryGetValue(entity, out var subscriptions))
        {
            foreach (var subscription in subscriptions)
            {
                subscription.Dispose();
            }
            entitySubscriptions.Remove(entity);
        }
    }
}
```

### Event Aggregation System

```csharp
public class EntityEventAggregator
{
    public event Action<Entity> OnAnyEntitySpawned;
    public event Action<Entity> OnAnyEntityDespawned;
    public event Action<Entity> OnAnyEntityActivated;
    public event Action<Entity> OnAnyEntityDeactivated;
    public event Action<Entity, float> OnAnyEntityUpdated;
    
    private readonly List<IDisposable> subscriptions = new();
    
    public void AggregateEvents(IEnumerable<Entity> entities)
    {
        foreach (var entity in entities)
        {
            // Aggregate all entity events into centralized events
            subscriptions.Add(entity.WhenSpawn(() => OnAnyEntitySpawned?.Invoke(entity)));
            subscriptions.Add(entity.WhenDespawn(() => OnAnyEntityDespawned?.Invoke(entity)));
            subscriptions.Add(entity.WhenActivate(() => OnAnyEntityActivated?.Invoke(entity)));
            subscriptions.Add(entity.WhenDeactivate(() => OnAnyEntityDeactivated?.Invoke(entity)));
            subscriptions.Add(entity.WhenUpdate(deltaTime => OnAnyEntityUpdated?.Invoke(entity, deltaTime)));
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
public class GameEventLogger
{
    private EntityEventAggregator aggregator = new();
    
    public void Initialize(IEnumerable<Entity> entities)
    {
        aggregator.AggregateEvents(entities);
        
        aggregator.OnAnyEntitySpawned += entity => Debug.Log($"Entity {entity.Name} spawned");
        aggregator.OnAnyEntityDespawned += entity => Debug.Log($"Entity {entity.Name} despawned");
        aggregator.OnAnyEntityActivated += entity => Debug.Log($"Entity {entity.Name} activated");
        aggregator.OnAnyEntityDeactivated += entity => Debug.Log($"Entity {entity.Name} deactivated");
        aggregator.OnAnyEntityUpdated += (entity, dt) => Debug.Log($"Entity {entity.Name} updated: {dt:F4}s");
    }
}
```

### State Machine Integration

```csharp
public class EntityStateMachine
{
    private Entity entity;
    private State currentState;
    private readonly List<IDisposable> stateSubscriptions = new();
    
    public enum State
    {
        Inactive,
        Spawning,
        Active,
        Despawning
    }
    
    public void Initialize(Entity targetEntity)
    {
        entity = targetEntity;
        
        // Set up state transitions using extension methods
        stateSubscriptions.Add(entity.WhenSpawn(() => TransitionToState(State.Spawning)));
        stateSubscriptions.Add(entity.WhenActivate(() => TransitionToState(State.Active)));
        stateSubscriptions.Add(entity.WhenDeactivate(() => TransitionToState(State.Inactive)));
        stateSubscriptions.Add(entity.WhenDespawn(() => TransitionToState(State.Despawning)));
    }
    
    private void TransitionToState(State newState)
    {
        var previousState = currentState;
        currentState = newState;
        
        Debug.Log($"Entity {entity.Name} transitioned from {previousState} to {newState}");
        OnStateChanged?.Invoke(previousState, newState);
    }
    
    public event Action<State, State> OnStateChanged;
    
    public void Dispose()
    {
        foreach (var subscription in stateSubscriptions)
        {
            subscription.Dispose();
        }
        stateSubscriptions.Clear();
    }
}
```

### Performance Monitoring System

```csharp
public class EntityPerformanceMonitor
{
    private readonly Dictionary<Entity, EntityMetrics> entityMetrics = new();
    private readonly List<IDisposable> subscriptions = new();
    
    public struct EntityMetrics
    {
        public DateTime SpawnTime;
        public DateTime LastUpdateTime;
        public int UpdateCount;
        public float TotalUpdateTime;
        public bool IsActive;
    }
    
    public void StartMonitoring(IEnumerable<Entity> entities)
    {
        foreach (var entity in entities)
        {
            var metrics = new EntityMetrics();
            entityMetrics[entity] = metrics;
            
            // Use extension methods to monitor performance
            subscriptions.Add(entity.WhenSpawn(() => 
            {
                var m = entityMetrics[entity];
                m.SpawnTime = DateTime.Now;
                entityMetrics[entity] = m;
            }));
            
            subscriptions.Add(entity.WhenActivate(() => 
            {
                var m = entityMetrics[entity];
                m.IsActive = true;
                entityMetrics[entity] = m;
            }));
            
            subscriptions.Add(entity.WhenDeactivate(() => 
            {
                var m = entityMetrics[entity];
                m.IsActive = false;
                entityMetrics[entity] = m;
            }));
            
            subscriptions.Add(entity.WhenUpdate(deltaTime => 
            {
                var m = entityMetrics[entity];
                m.UpdateCount++;
                m.TotalUpdateTime += deltaTime;
                m.LastUpdateTime = DateTime.Now;
                entityMetrics[entity] = m;
            }));
        }
    }
    
    public EntityMetrics GetMetrics(Entity entity)
    {
        return entityMetrics.TryGetValue(entity, out var metrics) ? metrics : default;
    }
    
    public void PrintPerformanceReport()
    {
        Debug.Log("=== Entity Performance Report ===");
        foreach (var kvp in entityMetrics)
        {
            var entity = kvp.Key;
            var metrics = kvp.Value;
            
            var lifetime = DateTime.Now - metrics.SpawnTime;
            var avgUpdateTime = metrics.UpdateCount > 0 ? metrics.TotalUpdateTime / metrics.UpdateCount : 0f;
            
            Debug.Log($"Entity {entity.Name}:");
            Debug.Log($"  Lifetime: {lifetime.TotalSeconds:F2}s");
            Debug.Log($"  Updates: {metrics.UpdateCount}");
            Debug.Log($"  Avg Update Time: {avgUpdateTime:F4}s");
            Debug.Log($"  Is Active: {metrics.IsActive}");
        }
    }
}
```

### Reactive Entity System

```csharp
public class ReactiveEntitySystem
{
    private readonly Dictionary<Entity, ReactiveState> entityStates = new();
    private readonly List<IDisposable> subscriptions = new();
    
    public struct ReactiveState
    {
        public bool HasSpawned;
        public bool IsActive;
        public Vector3 LastPosition;
        public float LastUpdateTime;
    }
    
    public void RegisterReactiveEntity(Entity entity)
    {
        var state = new ReactiveState();
        entityStates[entity] = state;
        
        // React to all lifecycle events
        subscriptions.Add(entity.WhenSpawn(() => OnEntitySpawned(entity)));
        subscriptions.Add(entity.WhenDespawn(() => OnEntityDespawned(entity)));
        subscriptions.Add(entity.WhenActivate(() => OnEntityActivated(entity)));
        subscriptions.Add(entity.WhenDeactivate(() => OnEntityDeactivated(entity)));
        subscriptions.Add(entity.WhenUpdate(deltaTime => OnEntityUpdated(entity, deltaTime)));
    }
    
    private void OnEntitySpawned(Entity entity)
    {
        var state = entityStates[entity];
        state.HasSpawned = true;
        entityStates[entity] = state;
        
        // Trigger reactive behaviors
        TriggerSpawnReactions(entity);
    }
    
    private void OnEntityActivated(Entity entity)
    {
        var state = entityStates[entity];
        state.IsActive = true;
        entityStates[entity] = state;
        
        // Trigger activation reactions
        TriggerActivationReactions(entity);
    }
    
    private void OnEntityUpdated(Entity entity, float deltaTime)
    {
        var state = entityStates[entity];
        state.LastUpdateTime = Time.time;
        
        // Update position tracking
        if (entity.TryGetValue<Vector3>("Position", out var position))
        {
            if (Vector3.Distance(position, state.LastPosition) > 0.1f)
            {
                TriggerMovementReactions(entity, state.LastPosition, position);
                state.LastPosition = position;
            }
        }
        
        entityStates[entity] = state;
    }
    
    private void TriggerSpawnReactions(Entity entity)
    {
        // React to entity spawn
        NotifyNearbyEntities(entity, "EntitySpawned");
    }
    
    private void TriggerActivationReactions(Entity entity)
    {
        // React to entity activation
        UpdateEntitySystems(entity);
    }
    
    private void TriggerMovementReactions(Entity entity, Vector3 oldPos, Vector3 newPos)
    {
        // React to entity movement
        CheckProximityTriggers(entity, newPos);
    }
}
```

### Middleware Pattern Implementation

```csharp
public class EntityEventMiddleware
{
    private readonly List<IEntityEventMiddleware> middlewares = new();
    
    public interface IEntityEventMiddleware
    {
        void OnBeforeSpawn(Entity entity);
        void OnAfterSpawn(Entity entity);
        void OnBeforeUpdate(Entity entity, float deltaTime);
        void OnAfterUpdate(Entity entity, float deltaTime);
    }
    
    public void AddMiddleware(IEntityEventMiddleware middleware)
    {
        middlewares.Add(middleware);
    }
    
    public void ApplyMiddleware(Entity entity)
    {
        var subscriptions = new List<IDisposable>();
        
        subscriptions.Add(entity.WhenSpawn(() => 
        {
            foreach (var middleware in middlewares)
                middleware.OnBeforeSpawn(entity);
                
            // Original spawn logic would happen here
            
            foreach (var middleware in middlewares)
                middleware.OnAfterSpawn(entity);
        }));
        
        subscriptions.Add(entity.WhenUpdate(deltaTime => 
        {
            foreach (var middleware in middlewares)
                middleware.OnBeforeUpdate(entity, deltaTime);
                
            // Original update logic would happen here
            
            foreach (var middleware in middlewares)
                middleware.OnAfterUpdate(entity, deltaTime);
        }));
        
        // Store subscriptions for cleanup
        StoreEntitySubscriptions(entity, subscriptions);
    }
}

// Example middleware
public class LoggingMiddleware : EntityEventMiddleware.IEntityEventMiddleware
{
    public void OnBeforeSpawn(Entity entity) => Debug.Log($"Before spawn: {entity.Name}");
    public void OnAfterSpawn(Entity entity) => Debug.Log($"After spawn: {entity.Name}");
    public void OnBeforeUpdate(Entity entity, float deltaTime) { /* Log if needed */ }
    public void OnAfterUpdate(Entity entity, float deltaTime) { /* Log if needed */ }
}

public class ValidationMiddleware : EntityEventMiddleware.IEntityEventMiddleware
{
    public void OnBeforeSpawn(Entity entity) => ValidateEntityState(entity);
    public void OnAfterSpawn(Entity entity) => VerifySpawnSuccess(entity);
    public void OnBeforeUpdate(Entity entity, float deltaTime) => ValidateUpdatePreconditions(entity);
    public void OnAfterUpdate(Entity entity, float deltaTime) => ValidateUpdatePostconditions(entity);
}
```

## Integration Examples

### Unity MonoBehaviour Integration

```csharp
public class EntityMonoBehaviourBridge : MonoBehaviour
{
    [SerializeField] private Entity targetEntity;
    private readonly List<IDisposable> subscriptions = new();
    
    private void Start()
    {
        if (targetEntity != null)
        {
            SetupEntityBridge();
        }
    }
    
    private void SetupEntityBridge()
    {
        // Bridge entity events to Unity MonoBehaviour lifecycle
        subscriptions.Add(targetEntity.WhenSpawn(() => OnEntitySpawned()));
        subscriptions.Add(targetEntity.WhenDespawn(() => OnEntityDespawned()));
        subscriptions.Add(targetEntity.WhenActivate(() => OnEntityActivated()));
        subscriptions.Add(targetEntity.WhenDeactivate(() => OnEntityDeactivated()));
        subscriptions.Add(targetEntity.WhenUpdate(deltaTime => OnEntityUpdate(deltaTime)));
    }
    
    private void OnEntitySpawned() => Debug.Log($"MonoBehaviour received entity spawn event");
    private void OnEntityDespawned() => Destroy(gameObject);
    private void OnEntityActivated() => gameObject.SetActive(true);
    private void OnEntityDeactivated() => gameObject.SetActive(false);
    private void OnEntityUpdate(float deltaTime) => UpdateGameObjectState(deltaTime);
    
    private void OnDestroy()
    {
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
    }
}
```

## Best Practices

### Subscription Management
1. **Always dispose** – Use `using` statements or store for manual disposal
2. **Avoid memory leaks** – Ensure subscriptions are cleaned up
3. **Batch subscriptions** – Group related subscriptions together
4. **Use fluent patterns** – Chain extension methods for readability

### Performance Considerations
1. **Aggressive inlining** – Extensions are optimized for performance
2. **Minimal overhead** – Direct event subscription with no extra layers
3. **Cache subscriptions** – Store subscription handles efficiently
4. **Profile event handlers** – Monitor callback performance

### Error Handling
1. **Null safety** – Check for null sources before subscribing
2. **Exception safety** – Wrap event handlers in try-catch blocks
3. **Graceful degradation** – Handle subscription failures gracefully
4. **Cleanup on errors** – Ensure subscriptions are disposed on failures

## Common Anti-patterns

### Don't Do This
```csharp
// ❌ Forgetting to dispose subscriptions
entity.WhenSpawn(() => Debug.Log("Spawned")); // Memory leak!

// ❌ Subscribing in loops without cleanup
foreach (var entity in entities)
{
    entity.WhenUpdate(deltaTime => { /* No cleanup */ }); // Multiple leaks!
}
```

### Do This Instead
```csharp
// ✅ Proper disposal with using statements
using var spawnSub = entity.WhenSpawn(() => Debug.Log("Spawned"));

// ✅ Store subscriptions for proper cleanup
var subscriptions = entities.Select(entity => 
    entity.WhenUpdate(deltaTime => ProcessUpdate(entity, deltaTime))).ToList();

// Later: dispose all subscriptions
subscriptions.ForEach(sub => sub.Dispose());
```

## Thread Safety

- **Main thread only** – All extension methods should be called on main thread
- **Event thread safety** – Callbacks execute on the event thread
- **Disposal safety** – Subscriptions can be safely disposed from any thread
- **No synchronization needed** – Single-threaded by design for Unity integration

## Performance Characteristics

- **Zero allocation** – Extension methods create no additional objects
- **Aggressive inlining** – All methods are inlined for performance
- **Direct event subscription** – No proxy or wrapper objects
- **Minimal overhead** – Same performance as manual event subscription