# 🔄 Entity_Lifecycle

`Entity_Lifecycle` is a partial class implementation that manages entity lifecycle states, behavior orchestration, and update system integration. It provides the core state machine for entity spawning, activation, deactivation, and despawning with automatic behavior lifecycle management.

## Key Features

- **State Machine Management** – Spawn/Despawn and Active/Inactive states
- **Behavior Lifecycle Integration** – Automatic behavior event dispatching
- **Update System Registration** – Dynamic update behavior management  
- **Event System** – Comprehensive lifecycle event notifications
- **Performance Optimized** – Efficient behavior iteration and registration
- **Virtual Methods** – Extensible lifecycle processing

---

## Lifecycle States

### State Properties

```csharp
public bool IsSpawned { get; }                    // Entity spawn state
public bool IsActive { get; }                     // Entity activation state

private bool _spawned;                            // Internal spawn flag
private bool _active;                             // Internal active flag
```

### State Events

```csharp
public event Action OnSpawned;                    // Spawned event
public event Action OnDespawned;                  // Despawned event
public event Action OnActivated;                  // Activated event
public event Action OnDeactivated;                // Deactivated event
public event Action<float> OnUpdated;             // Update event
public event Action<float> OnFixedUpdated;        // Fixed update event
public event Action<float> OnLateUpdated;         // Late update event
```

---

## Update Management

### Update Arrays

```csharp
private IEntityUpdate[] _updates;                // Update behaviors
private IEntityFixedUpdate[] _fixedUpdates;      // Fixed update behaviors  
private IEntityLateUpdate[] _lateUpdates;        // Late update behaviors

private int _updateCount;                         // Active update count
private int _fixedUpdateCount;                    // Active fixed update count
private int _lateUpdateCount;                     // Active late update count
```

### Update Comparers

```csharp
private static readonly IEqualityComparer<IEntityUpdate> s_updateComparer;
private static readonly IEqualityComparer<IEntityFixedUpdate> s_fixedUpdateComparer;
private static readonly IEqualityComparer<IEntityLateUpdate> s_lateUpdateComparer;
```

---

## Lifecycle Methods

### Spawning

```csharp
public void Spawn()                              // Spawn entity
protected virtual void ProcessSpawn()           // Virtual spawn processing
```

**Spawn Process:**
1. Check if already spawned (early exit)
2. Call `ProcessSpawn()` → triggers `IEntitySpawn.OnSpawn()` on all behaviors
3. Set `_spawned = true`
4. Trigger `OnSpawned` event
5. Trigger `OnStateChanged` event

### Despawning

```csharp
public void Despawn()                           // Despawn entity
protected virtual void ProcessDespawn()        // Virtual despawn processing
```

**Despawn Process:**
1. Check if not spawned (early exit)
2. Deactivate if currently active
3. Call `ProcessDespawn()` → triggers `IEntityDespawn.OnDespawn()` on all behaviors
4. Set `_spawned = false`
5. Trigger `OnDespawned` event
6. Trigger `OnStateChanged` event

### Activation

```csharp
public void Activate()                          // Activate entity
protected virtual void ProcessActivate()       // Virtual activation processing
```

**Activation Process:**
1. Auto-spawn if not already spawned
2. Check if already active (early exit)  
3. Call `ProcessActivate()` → calls `ActivateBehaviour()` on all behaviors
4. Set `_active = true`
5. Trigger `OnActivated` event
6. Trigger `OnStateChanged` event

### Deactivation

```csharp
public void Deactivate()                        // Deactivate entity
protected virtual void ProcessInactivate()     // Virtual deactivation processing
```

**Deactivation Process:**
1. Check if not active (early exit)
2. Call `ProcessInactivate()` → calls `InactivateBehaviour()` on all behaviors  
3. Set `_active = false`
4. Trigger `OnStateChanged` event
5. Trigger `OnDeactivated` event

---

## Update Processing

### Frame Updates

```csharp
public void OnUpdate(float deltaTime)           // Process frame updates
protected virtual void ProcessUpdate(float deltaTime)  // Virtual update processing
```

### Fixed Updates

```csharp
public void OnFixedUpdate(float deltaTime)      // Process fixed updates
protected virtual void ProcessFixedUpdate(float deltaTime) // Virtual fixed update
```

### Late Updates

```csharp
public void OnLateUpdate(float deltaTime)       // Process late updates
protected virtual void ProcessLateUpdate(float deltaTime) // Virtual late update
```

---

## Usage Patterns

### Basic Lifecycle Management

```csharp
public class EntityLifecycleExample
{
    public void BasicLifecycleDemo()
    {
        var entity = new Entity("TestEntity");
        
        // Add behaviors before spawning
        entity.AddBehaviour(new LoggingBehaviour());
        entity.AddBehaviour(new MovementBehaviour());
        
        // Lifecycle progression
        entity.Spawn();      // Spawned, but not active
        entity.Activate();   // Now active and updating
        
        // Entity is now fully operational
        Debug.Log($"Entity state: Spawned={entity.IsSpawned}, Active={entity.IsActive}");
        
        // Shutdown sequence
        entity.Deactivate(); // Stop updating
        entity.Despawn();    // Cleanup and despawn
    }
}

public class LoggingBehaviour : IEntityBehaviour, IEntitySpawn, IEntityDespawn, 
    IEntityActivate, IEntityDeactivate, IEntityUpdate
{
    public void OnSpawn(Entity entity) => Debug.Log("Behavior spawned");
    public void OnDespawn(Entity entity) => Debug.Log("Behavior despawned");
    public void OnActivate(Entity entity) => Debug.Log("Behavior activated");
    public void OnDeactivate(Entity entity) => Debug.Log("Behavior deactivated");
    public void OnUpdate(Entity entity, float deltaTime) => Debug.Log($"Behavior updating: {deltaTime}");
}
```

### Custom Lifecycle Processing

```csharp
public class CustomEntity : Entity
{
    private readonly List<string> lifecycleLog = new();
    
    protected override void ProcessSpawn()
    {
        lifecycleLog.Add("Custom spawn processing started");
        
        // Custom spawn logic here
        InitializeCustomSystems();
        
        // Call base implementation to handle behaviors
        base.ProcessSpawn();
        
        lifecycleLog.Add("Custom spawn processing completed");
    }
    
    protected override void ProcessActivate()  
    {
        lifecycleLog.Add("Custom activation started");
        
        // Custom activation logic
        StartCustomSystems();
        
        // Call base implementation
        base.ProcessActivate();
        
        lifecycleLog.Add("Custom activation completed");
    }
    
    protected override void ProcessUpdate(float deltaTime)
    {
        // Custom update logic before behaviors
        ProcessCustomLogic(deltaTime);
        
        // Process behavior updates
        base.ProcessUpdate(deltaTime);
        
        // Custom update logic after behaviors
        PostProcessCustomLogic(deltaTime);
    }
    
    private void InitializeCustomSystems() { /* Custom initialization */ }
    private void StartCustomSystems() { /* Custom activation */ }
    private void ProcessCustomLogic(float deltaTime) { /* Custom processing */ }
    private void PostProcessCustomLogic(float deltaTime) { /* Post processing */ }
    
    public List<string> GetLifecycleLog() => new List<string>(lifecycleLog);
}
```

### Event-Driven Lifecycle Monitoring

```csharp
public class LifecycleEventMonitor
{
    private readonly Dictionary<Entity, LifecycleState> entityStates = new();
    
    public struct LifecycleState
    {
        public DateTime SpawnTime;
        public DateTime? DespawnTime;
        public int ActivationCount;
        public int DeactivationCount;
        public TimeSpan TotalActiveTime;
        public DateTime LastActivation;
    }
    
    public void MonitorEntity(Entity entity)
    {
        var state = new LifecycleState
        {
            SpawnTime = DateTime.MinValue,
            ActivationCount = 0,
            DeactivationCount = 0,
            TotalActiveTime = TimeSpan.Zero
        };
        entityStates[entity] = state;
        
        entity.OnSpawned += () => OnEntitySpawned(entity);
        entity.OnDespawned += () => OnEntityDespawned(entity);
        entity.OnActivated += () => OnEntityActivated(entity);
        entity.OnDeactivated += () => OnEntityDeactivated(entity);
    }
    
    private void OnEntitySpawned(Entity entity)
    {
        var state = entityStates[entity];
        state.SpawnTime = DateTime.Now;
        entityStates[entity] = state;
        
        Debug.Log($"Entity {entity.Name} spawned at {state.SpawnTime:HH:mm:ss}");
    }
    
    private void OnEntityActivated(Entity entity)
    {
        var state = entityStates[entity];
        state.ActivationCount++;
        state.LastActivation = DateTime.Now;
        entityStates[entity] = state;
        
        Debug.Log($"Entity {entity.Name} activated (count: {state.ActivationCount})");
    }
    
    private void OnEntityDeactivated(Entity entity)
    {
        var state = entityStates[entity];
        state.DeactivationCount++;
        
        if (state.LastActivation != DateTime.MinValue)
        {
            var activeTime = DateTime.Now - state.LastActivation;
            state.TotalActiveTime += activeTime;
        }
        
        entityStates[entity] = state;
        
        Debug.Log($"Entity {entity.Name} deactivated (total active time: {state.TotalActiveTime.TotalSeconds:F2}s)");
    }
    
    public LifecycleState GetEntityMetrics(Entity entity)
    {
        return entityStates.TryGetValue(entity, out var state) ? state : default;
    }
}
```

### Update System Integration

```csharp
public class EntityUpdateManager
{
    private readonly List<Entity> activeEntities = new();
    private readonly List<Entity> toRemove = new();
    
    public void RegisterEntity(Entity entity)
    {
        if (!activeEntities.Contains(entity))
        {
            activeEntities.Add(entity);
            
            // Monitor lifecycle events
            entity.OnActivated += () => OnEntityActivated(entity);
            entity.OnDeactivated += () => OnEntityDeactivated(entity);
            entity.OnDespawned += () => OnEntityDespawned(entity);
        }
    }
    
    public void UpdateEntities(float deltaTime)
    {
        // Update all active entities
        foreach (var entity in activeEntities)
        {
            if (entity.IsActive)
            {
                entity.OnUpdate(deltaTime);
            }
        }
        
        // Remove despawned entities
        if (toRemove.Count > 0)
        {
            foreach (var entity in toRemove)
            {
                activeEntities.Remove(entity);
            }
            toRemove.Clear();
        }
    }
    
    public void FixedUpdateEntities(float fixedDeltaTime)
    {
        foreach (var entity in activeEntities)
        {
            if (entity.IsActive)
            {
                entity.OnFixedUpdate(fixedDeltaTime);
            }
        }
    }
    
    public void LateUpdateEntities(float deltaTime)
    {
        foreach (var entity in activeEntities)
        {
            if (entity.IsActive)
            {
                entity.OnLateUpdate(deltaTime);
            }
        }
    }
    
    private void OnEntityActivated(Entity entity)
    {
        Debug.Log($"Entity {entity.Name} is now updating");
    }
    
    private void OnEntityDeactivated(Entity entity)
    {
        Debug.Log($"Entity {entity.Name} stopped updating");
    }
    
    private void OnEntityDespawned(Entity entity)
    {
        toRemove.Add(entity);
    }
}
```

### Performance-Optimized Update Processing

```csharp
public class OptimizedEntity : Entity
{
    private bool hasUpdateBehaviors;
    private bool hasFixedUpdateBehaviors;
    private bool hasLateUpdateBehaviors;
    
    protected override void ProcessActivate()
    {
        base.ProcessActivate();
        
        // Cache update behavior existence for faster update checks
        hasUpdateBehaviors = _updateCount > 0;
        hasFixedUpdateBehaviors = _fixedUpdateCount > 0;
        hasLateUpdateBehaviors = _lateUpdateCount > 0;
    }
    
    protected override void ProcessUpdate(float deltaTime)
    {
        if (!hasUpdateBehaviors) return;
        
        // Optimized update loop - avoid virtual calls when possible
        for (int i = 0; i < _updateCount && _active; i++)
        {
            _updates[i].OnUpdate(this, deltaTime);
        }
        
        // Trigger event after behavior updates
        OnUpdated?.Invoke(deltaTime);
    }
    
    protected override void ProcessFixedUpdate(float deltaTime)
    {
        if (!hasFixedUpdateBehaviors) return;
        
        for (int i = 0; i < _fixedUpdateCount && _active; i++)
        {
            _fixedUpdates[i].OnFixedUpdate(this, deltaTime);
        }
        
        OnFixedUpdated?.Invoke(deltaTime);
    }
    
    protected override void ProcessLateUpdate(float deltaTime)
    {
        if (!hasLateUpdateBehaviors) return;
        
        for (int i = 0; i < _lateUpdateCount && _active; i++)
        {
            _lateUpdates[i].OnLateUpdate(this, deltaTime);
        }
        
        OnLateUpdated?.Invoke(deltaTime);
    }
}
```

## Behavior Registration System

### Activation Process

When a behavior is activated:
1. **IEntityActivate** → `OnActivate()` called if implemented
2. **IEntityUpdate** → Added to `_updates` array if implemented  
3. **IEntityFixedUpdate** → Added to `_fixedUpdates` array if implemented
4. **IEntityLateUpdate** → Added to `_lateUpdates` array if implemented

### Deactivation Process

When a behavior is deactivated:
1. **IEntityDeactivate** → `OnDeactivate()` called if implemented
2. **Update arrays** → Behavior removed from all update arrays
3. **Array compaction** → Arrays compacted to remove gaps

## Performance Characteristics

### Lifecycle Operations
- **Spawn/Despawn**: O(n) where n = behavior count
- **Activate/Deactivate**: O(n) for behavior processing + O(k) for update registration
- **State checks**: O(1) - simple boolean flags

### Update Processing  
- **OnUpdate/OnFixedUpdate/OnLateUpdate**: O(k) where k = update behavior count
- **Behavior iteration**: Direct array access with count bounds
- **Early termination**: Stops processing if entity deactivated mid-update

## Best Practices

### Lifecycle Management
1. **Spawn before activate** – Entity auto-spawns on activation if needed
2. **Proper shutdown sequence** – Deactivate before despawn for clean shutdown
3. **Event subscription cleanup** – Unsubscribe from events when no longer needed
4. **State validation** – Check entity state before operations

### Performance Optimization
1. **Minimize lifecycle churn** – Avoid frequent spawn/despawn cycles
2. **Cache active state** – Store references to active entities for updates
3. **Batch lifecycle operations** – Group related lifecycle changes
4. **Override virtual methods** – Customize lifecycle processing for specific entities

## Thread Safety

- **Not thread-safe** – All lifecycle operations must be on main thread
- **Event callbacks** execute synchronously on calling thread  
- **Update processing** designed for single-threaded Unity update cycle
- **State modifications** not atomic across multiple operations