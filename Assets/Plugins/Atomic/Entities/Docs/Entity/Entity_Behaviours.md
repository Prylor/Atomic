# ⚙️ Entity_Behaviours

`Entity_Behaviours` is a partial class implementation that manages entity behavior attachment, lifecycle, and execution. It provides dynamic behavior composition with automatic lifecycle integration and efficient array-based storage.

## Key Features

- **Dynamic Behavior Composition** – Runtime behavior attachment/detachment
- **Automatic Lifecycle Integration** – Behaviors automatically receive lifecycle events
- **Array-Based Storage** – Dynamic array with automatic resizing
- **Type Safety** – Generic methods for type-safe behavior access
- **Event System** – Behavior addition/removal notifications
- **ArrayPool Optimization** – Shared array pool for temporary operations

---

## Behavior Storage Architecture

### Storage Fields

```csharp
private IEntityBehaviour[] _behaviours;           // Dynamic behavior array
private int _behaviourCount;                      // Current behavior count

private static readonly ArrayPool<IEntityBehaviour> s_behaviourPool = 
    ArrayPool<IEntityBehaviour>.Shared;          // Shared pool for operations
```

### Events

```csharp
public event Action<IEntity, IEntityBehaviour> OnBehaviourAdded;    // Behavior added
public event Action<IEntity, IEntityBehaviour> OnBehaviourDeleted;  // Behavior removed
```

---

## Public API

### Properties

```csharp
public int BehaviourCount { get; }                // Total behavior count
```

### Behavior Queries

```csharp
public bool HasBehaviour(IEntityBehaviour behaviour)      // Check specific instance
public bool HasBehaviour<T>() where T : IEntityBehaviour  // Check by type
```

### Behavior Management

```csharp
public void AddBehaviour(IEntityBehaviour behaviour)      // Add behavior
public bool DelBehaviour<T>() where T : IEntityBehaviour  // Remove first of type
public void DelBehaviours<T>() where T : IEntityBehaviour // Remove all of type
public bool DelBehaviour(IEntityBehaviour behaviour)      // Remove specific instance
public bool DelBehaviourAt(int index)                     // Remove by index
public void ClearBehaviours()                             // Remove all behaviors
```

### Behavior Access

```csharp
public T GetBehaviour<T>() where T : IEntityBehaviour             // Get first of type
public bool TryGetBehaviour<T>(out T behaviour) where T : IEntityBehaviour  // Try get
public IEntityBehaviour GetBehaviourAt(int index)                 // Get by index
public IEntityBehaviour[] GetBehaviours()                         // Get all behaviors
public T[] GetBehaviours<T>() where T : IEntityBehaviour          // Get all of type
public int CopyBehaviours(IEntityBehaviour[] results)             // Copy to array
public int CopyBehaviours<T>(T[] results) where T : IEntityBehaviour // Copy typed
```

### Enumeration

```csharp
public BehaviourEnumerator GetBehaviourEnumerator()               // Struct enumerator
IEnumerator<IEntityBehaviour> IEntity.GetBehaviourEnumerator()   // Interface impl
```

---

## Usage Patterns

### Basic Behavior Management

```csharp
public class EntityBehaviorSetup
{
    public void SetupPlayerEntity(Entity entity)
    {
        // Add various behaviors
        entity.AddBehaviour(new MovementBehaviour());
        entity.AddBehaviour(new HealthBehaviour());
        entity.AddBehaviour(new InputBehaviour());
        entity.AddBehaviour(new AnimationBehaviour());
        
        Debug.Log($"Player entity has {entity.BehaviourCount} behaviors");
        
        // Check for specific behavior types
        if (entity.HasBehaviour<MovementBehaviour>())
        {
            Debug.Log("Entity can move");
        }
        
        // Spawn entity to trigger behavior lifecycle
        entity.Spawn();
        entity.Activate();
    }
    
    public void ModifyEntityBehaviors(Entity entity)
    {
        // Remove specific behavior type
        if (entity.DelBehaviour<InputBehaviour>())
        {
            Debug.Log("Removed input behavior - entity is now AI controlled");
        }
        
        // Add AI behavior replacement
        entity.AddBehaviour(new AIBehaviour());
        
        // Get specific behavior for configuration
        if (entity.TryGetBehaviour<MovementBehaviour>(out var movement))
        {
            movement.SetSpeed(10f);
        }
    }
}
```

### Behavior Event Monitoring

```csharp
public class BehaviorEventSystem
{
    private readonly Dictionary<Type, List<Action<Entity>>> behaviorAddedCallbacks = new();
    private readonly Dictionary<Type, List<Action<Entity>>> behaviorRemovedCallbacks = new();
    
    public void MonitorEntity(Entity entity)
    {
        entity.OnBehaviourAdded += OnBehaviorAdded;
        entity.OnBehaviourDeleted += OnBehaviorRemoved;
    }
    
    public void RegisterBehaviorCallback<T>(Action<Entity> onAdded, Action<Entity> onRemoved = null) 
        where T : IEntityBehaviour
    {
        var type = typeof(T);
        
        if (onAdded != null)
        {
            if (!behaviorAddedCallbacks.ContainsKey(type))
                behaviorAddedCallbacks[type] = new List<Action<Entity>>();
            behaviorAddedCallbacks[type].Add(onAdded);
        }
        
        if (onRemoved != null)
        {
            if (!behaviorRemovedCallbacks.ContainsKey(type))
                behaviorRemovedCallbacks[type] = new List<Action<Entity>>();
            behaviorRemovedCallbacks[type].Add(onRemoved);
        }
    }
    
    private void OnBehaviorAdded(IEntity entity, IEntityBehaviour behavior)
    {
        var type = behavior.GetType();
        if (behaviorAddedCallbacks.TryGetValue(type, out var callbacks))
        {
            foreach (var callback in callbacks)
                callback(entity as Entity);
        }
        
        Debug.Log($"Behavior added: {type.Name} to entity {entity.Name}");
    }
    
    private void OnBehaviorRemoved(IEntity entity, IEntityBehaviour behavior)
    {
        var type = behavior.GetType();
        if (behaviorRemovedCallbacks.TryGetValue(type, out var callbacks))
        {
            foreach (var callback in callbacks)
                callback(entity as Entity);
        }
        
        Debug.Log($"Behavior removed: {type.Name} from entity {entity.Name}");
    }
}
```

### Behavior-Based System Processing

```csharp
public class BehaviorProcessingSystem
{
    public void ProcessMovementBehaviors(IEnumerable<Entity> entities, float deltaTime)
    {
        foreach (var entity in entities)
        {
            var movements = entity.GetBehaviours<MovementBehaviour>();
            foreach (var movement in movements)
            {
                movement.ProcessMovement(entity, deltaTime);
            }
        }
    }
    
    public void ProcessAllBehaviorTypes(Entity entity, float deltaTime)
    {
        // Process all behaviors using generic enumeration
        foreach (var behavior in entity.GetBehaviours())
        {
            ProcessBehaviorByType(behavior, entity, deltaTime);
        }
    }
    
    private void ProcessBehaviorByType(IEntityBehaviour behavior, Entity entity, float deltaTime)
    {
        switch (behavior)
        {
            case IEntityUpdate updateBehavior:
                updateBehavior.OnUpdate(entity, deltaTime);
                break;
                
            case IEntityFixedUpdate fixedUpdateBehavior:
                // Note: FixedUpdate would be called separately with fixed delta time
                break;
                
            case IEntityLateUpdate lateUpdateBehavior:
                // Note: LateUpdate would be called separately
                break;
                
            default:
                // Handle other behavior types
                break;
        }
    }
}
```

### Behavior Validation and Management

```csharp
public class BehaviorValidator
{
    private readonly Dictionary<Type, IBehaviorValidator> validators = new();
    
    public interface IBehaviorValidator
    {
        bool IsValidForEntity(IEntityBehaviour behavior, Entity entity);
        string GetValidationError(IEntityBehaviour behavior, Entity entity);
    }
    
    public void RegisterValidator<T>(IBehaviorValidator validator) where T : IEntityBehaviour
    {
        validators[typeof(T)] = validator;
    }
    
    public bool ValidateEntityBehaviors(Entity entity)
    {
        bool isValid = true;
        
        foreach (var behavior in entity.GetBehaviours())
        {
            if (!ValidateBehavior(behavior, entity))
            {
                isValid = false;
            }
        }
        
        return isValid;
    }
    
    private bool ValidateBehavior(IEntityBehaviour behavior, Entity entity)
    {
        var behaviorType = behavior.GetType();
        
        if (validators.TryGetValue(behaviorType, out var validator))
        {
            if (!validator.IsValidForEntity(behavior, entity))
            {
                string error = validator.GetValidationError(behavior, entity);
                Debug.LogError($"Behavior validation failed: {error}");
                return false;
            }
        }
        
        return true;
    }
}

// Example validator
public class MovementBehaviorValidator : BehaviorValidator.IBehaviorValidator
{
    public bool IsValidForEntity(IEntityBehaviour behavior, Entity entity)
    {
        // Movement requires position value
        return entity.HasValue(EntityNames.NameToId("Position"));
    }
    
    public string GetValidationError(IEntityBehaviour behavior, Entity entity)
    {
        return "MovementBehaviour requires Position value on entity";
    }
}
```

### Dynamic Behavior Composition

```csharp
public class DynamicBehaviorComposer
{
    private readonly Dictionary<string, Func<IEntityBehaviour>> behaviorFactories = new();
    
    public void RegisterBehaviorFactory<T>(string name, Func<T> factory) where T : IEntityBehaviour
    {
        behaviorFactories[name] = () => factory();
    }
    
    public void ComposeBehaviorsFromConfiguration(Entity entity, string[] behaviorNames)
    {
        foreach (string behaviorName in behaviorNames)
        {
            if (behaviorFactories.TryGetValue(behaviorName, out var factory))
            {
                var behavior = factory();
                entity.AddBehaviour(behavior);
                Debug.Log($"Added behavior: {behaviorName}");
            }
            else
            {
                Debug.LogWarning($"Unknown behavior: {behaviorName}");
            }
        }
    }
    
    public void RecomposeBehaviors(Entity entity, string[] newBehaviorNames)
    {
        // Save current state
        bool wasSpawned = entity.IsSpawned;
        bool wasActive = entity.IsActive;
        
        // Clear existing behaviors
        entity.ClearBehaviours();
        
        // Add new behaviors
        ComposeBehaviorsFromConfiguration(entity, newBehaviorNames);
        
        // Restore state
        if (wasSpawned) entity.Spawn();
        if (wasActive) entity.Activate();
    }
}

// Usage example
public class BehaviorCompositionExample
{
    public void SetupBehaviorComposer(DynamicBehaviorComposer composer)
    {
        composer.RegisterBehaviorFactory<MovementBehaviour>("Movement", () => new MovementBehaviour());
        composer.RegisterBehaviorFactory<HealthBehaviour>("Health", () => new HealthBehaviour());
        composer.RegisterBehaviorFactory<AIBehaviour>("AI", () => new AIBehaviour());
        composer.RegisterBehaviorFactory<InputBehaviour>("Input", () => new InputBehaviour());
    }
    
    public void CreateDynamicEntities(DynamicBehaviorComposer composer)
    {
        var playerEntity = new Entity("Player");
        composer.ComposeBehaviorsFromConfiguration(playerEntity, new[] { "Movement", "Health", "Input" });
        
        var enemyEntity = new Entity("Enemy");
        composer.ComposeBehaviorsFromConfiguration(enemyEntity, new[] { "Movement", "Health", "AI" });
        
        var npcEntity = new Entity("NPC");
        composer.ComposeBehaviorsFromConfiguration(npcEntity, new[] { "AI" });
    }
}
```

### Behavior Performance Monitoring

```csharp
public class BehaviorPerformanceMonitor
{
    private readonly Dictionary<Type, BehaviorMetrics> behaviorMetrics = new();
    
    public struct BehaviorMetrics
    {
        public int InstanceCount;
        public int AdditionCount;
        public int RemovalCount;
        public DateTime LastActivity;
        public TimeSpan AverageLifetime;
        public List<DateTime> AdditionTimes;
        public List<DateTime> RemovalTimes;
    }
    
    public void MonitorEntity(Entity entity)
    {
        entity.OnBehaviourAdded += (e, behavior) => RecordBehaviorAdded(behavior);
        entity.OnBehaviourDeleted += (e, behavior) => RecordBehaviorRemoved(behavior);
    }
    
    private void RecordBehaviorAdded(IEntityBehaviour behavior)
    {
        var type = behavior.GetType();
        var now = DateTime.Now;
        
        if (!behaviorMetrics.TryGetValue(type, out var metrics))
        {
            metrics = new BehaviorMetrics
            {
                AdditionTimes = new List<DateTime>(),
                RemovalTimes = new List<DateTime>()
            };
        }
        
        metrics.InstanceCount++;
        metrics.AdditionCount++;
        metrics.LastActivity = now;
        metrics.AdditionTimes.Add(now);
        
        behaviorMetrics[type] = metrics;
    }
    
    private void RecordBehaviorRemoved(IEntityBehaviour behavior)
    {
        var type = behavior.GetType();
        var now = DateTime.Now;
        
        if (behaviorMetrics.TryGetValue(type, out var metrics))
        {
            metrics.InstanceCount = Math.Max(0, metrics.InstanceCount - 1);
            metrics.RemovalCount++;
            metrics.LastActivity = now;
            metrics.RemovalTimes.Add(now);
            
            // Calculate average lifetime if we have paired add/remove times
            if (metrics.AdditionTimes.Count > 0 && metrics.RemovalTimes.Count > 0)
            {
                CalculateAverageLifetime(ref metrics);
            }
            
            behaviorMetrics[type] = metrics;
        }
    }
    
    private void CalculateAverageLifetime(ref BehaviorMetrics metrics)
    {
        var lifetimes = new List<TimeSpan>();
        int pairs = Math.Min(metrics.AdditionTimes.Count, metrics.RemovalTimes.Count);
        
        for (int i = 0; i < pairs; i++)
        {
            var lifetime = metrics.RemovalTimes[i] - metrics.AdditionTimes[i];
            lifetimes.Add(lifetime);
        }
        
        if (lifetimes.Count > 0)
        {
            var totalTicks = lifetimes.Sum(t => t.Ticks);
            metrics.AverageLifetime = new TimeSpan(totalTicks / lifetimes.Count);
        }
    }
    
    public void PrintMetrics()
    {
        Debug.Log("=== Behavior Performance Metrics ===");
        foreach (var kvp in behaviorMetrics)
        {
            var type = kvp.Key;
            var metrics = kvp.Value;
            
            Debug.Log($"{type.Name}:");
            Debug.Log($"  Current Instances: {metrics.InstanceCount}");
            Debug.Log($"  Total Additions: {metrics.AdditionCount}");
            Debug.Log($"  Total Removals: {metrics.RemovalCount}");
            Debug.Log($"  Average Lifetime: {metrics.AverageLifetime.TotalSeconds:F2}s");
            Debug.Log($"  Last Activity: {metrics.LastActivity:HH:mm:ss}");
        }
    }
}
```

## Lifecycle Integration

### Automatic Lifecycle Events

When behaviors are added to an entity:
1. **If entity is spawned** → `IEntitySpawn.OnSpawn()` called immediately
2. **If entity is active** → `IEntityActivate.OnActivate()` called immediately
3. **Update registration** → Update behaviors automatically registered

When behaviors are removed:
1. **If entity is active** → `IEntityDeactivate.OnDeactivate()` called first  
2. **If entity is spawned** → `IEntityDespawn.OnDespawn()` called
3. **Update unregistration** → Update behaviors automatically unregistered

### Behavior Interface Support

The system automatically detects and handles these interfaces:
- `IEntitySpawn` → OnSpawn lifecycle
- `IEntityDespawn` → OnDespawn lifecycle  
- `IEntityActivate` → OnActivate lifecycle
- `IEntityDeactivate` → OnDeactivate lifecycle
- `IEntityUpdate` → Frame update registration
- `IEntityFixedUpdate` → Fixed update registration
- `IEntityLateUpdate` → Late update registration

## Performance Characteristics

### Time Complexity
- **Add/Remove**: O(n) - must check for duplicates and shift array
- **Find by type**: O(n) - linear search through behavior array
- **Access by index**: O(1) - direct array access
- **Clear**: O(n) - must process each behavior for events

### Space Complexity
- **Array storage** – Dynamic array with 2x growth factor
- **No boxing** – Interface references stored directly
- **ArrayPool** – Temporary arrays pooled for memory efficiency

## Best Practices

### Performance Optimization
1. **Minimize behavior churn** – Avoid frequent add/remove operations
2. **Pre-size behavior arrays** – Use capacity constructor parameter
3. **Cache behavior references** – Store references instead of repeated lookups
4. **Batch behavior operations** when possible

### Design Patterns
1. **Composition over inheritance** – Use behaviors for entity capabilities
2. **Single responsibility** – Each behavior handles one concern
3. **Interface segregation** – Implement only needed lifecycle interfaces
4. **Behavior validation** – Validate behavior compatibility

## Thread Safety

- **Not thread-safe** – All operations must be on main thread
- **Lifecycle events** execute synchronously on calling thread
- **Array modifications** not atomic across operations
- **Enumeration** not safe during concurrent modifications