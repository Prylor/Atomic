# ⚡ SceneEntity_Lifecycle

The `SceneEntity_Lifecycle` partial class manages the complete lifecycle of `SceneEntity` instances, providing spawn/despawn, activation/deactivation, and update phase management. It coordinates behaviour lifecycle events and maintains efficient update loops for entity behaviours.

## Key Features

- **Complete Lifecycle Management** – Spawn, activate, deactivate, despawn states
- **Update Phase Integration** – Update, FixedUpdate, LateUpdate support
- **Behaviour Coordination** – Automatic behaviour lifecycle synchronization  
- **Event-Driven Architecture** – Comprehensive lifecycle events
- **Performance Optimized** – Efficient update loop management
- **State Validation** – Proper state transitions and validation

---

## Lifecycle States

### Spawned State
- **Property**: `bool IsSpawned { get; }`
- **Description**: Indicates if entity has been spawned and initialized
- **Methods**: `Spawn()`, `Despawn()`

### Active State  
- **Property**: `bool IsActive { get; }`
- **Description**: Indicates if entity is currently active and receiving updates
- **Methods**: `Activate()`, `Deactivate()`

## Lifecycle Events

```csharp
// Lifecycle state events
public event Action OnSpawned;
public event Action OnDespawned;
public event Action OnActivated;
public event Action OnDeactivated;

// Update phase events
public event Action<float> OnUpdated;
public event Action<float> OnFixedUpdated;
public event Action<float> OnLateUpdated;
```

## Core Lifecycle Methods

### Spawn()
```csharp
public void Spawn()
```
- Initializes the entity and calls `OnSpawn()` on all behaviours
- Sets `IsSpawned` to true
- Fires `OnSpawned` event
- Idempotent - safe to call multiple times

### Despawn()
```csharp
public void Despawn()
```
- Deactivates entity if currently active
- Calls `OnDespawn()` on all behaviours
- Sets `IsSpawned` to false
- Fires `OnDespawned` event

### Activate()
```csharp
public void Activate()
```
- Spawns entity if not already spawned
- Calls `OnActivate()` on all behaviours
- Registers update behaviours with update loops
- Sets `IsActive` to true
- Fires `OnActivated` event

### Deactivate()
```csharp
public void Deactivate()
```
- Calls `OnDeactivate()` on all behaviours
- Unregisters update behaviours from update loops
- Sets `IsActive` to false
- Fires `OnDeactivated` event

## Update Methods

### OnUpdate(float deltaTime)
```csharp
public void OnUpdate(float deltaTime)
```
- Calls `OnUpdate()` on all registered `IEntityUpdate` behaviours
- Only processes updates when entity is active
- Fires `OnUpdated` event

### OnFixedUpdate(float deltaTime)
```csharp
public void OnFixedUpdate(float deltaTime)
```
- Calls `OnFixedUpdate()` on all registered `IEntityFixedUpdate` behaviours
- Only processes updates when entity is active
- Fires `OnFixedUpdated` event

### OnLateUpdate(float deltaTime)
```csharp
public void OnLateUpdate(float deltaTime)
```
- Calls `OnLateUpdate()` on all registered `IEntityLateUpdate` behaviours
- Only processes updates when entity is active
- Fires `OnLateUpdated` event

## Example Usage

### Basic Lifecycle Management

```csharp
public class LifecycleAwareEntity : SceneEntity
{
    [Header("Lifecycle Configuration")]
    [SerializeField] private bool spawnOnStart = true;
    [SerializeField] private bool activateOnSpawn = true;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Subscribe to lifecycle events
        this.OnSpawned += OnEntitySpawned;
        this.OnDespawned += OnEntityDespawned;
        this.OnActivated += OnEntityActivated;
        this.OnDeactivated += OnEntityDeactivated;
        this.OnUpdated += OnEntityUpdated;
        
        // Add lifecycle-aware behaviours
        this.AddBehaviour(new LifecycleLoggerBehaviour());
        this.AddBehaviour(new HealthBehaviour());
        this.AddBehaviour(new MovementBehaviour());
    }
    
    void Start()
    {
        if (spawnOnStart)
        {
            this.Spawn();
            
            if (activateOnSpawn)
            {
                this.Activate();
            }
        }
    }
    
    private void OnEntitySpawned()
    {
        Debug.Log($"Entity {Name} spawned at {Time.time}");
        
        // Initialize spawn-time values
        this.SetValue(EntityNames.SPAWN_TIME, Time.time);
        this.SetValue(EntityNames.SPAWN_POSITION, transform.position);
    }
    
    private void OnEntityDespawned()
    {
        Debug.Log($"Entity {Name} despawned after {Time.time - this.GetValue<float>(EntityNames.SPAWN_TIME)} seconds");
        
        // Cleanup spawn-time data
        this.DelValue(EntityNames.SPAWN_TIME);
        this.DelValue(EntityNames.SPAWN_POSITION);
    }
    
    private void OnEntityActivated()
    {
        Debug.Log($"Entity {Name} activated");
        
        // Setup active-state resources
        var renderer = GetComponent<Renderer>();
        if (renderer != null)
            renderer.enabled = true;
    }
    
    private void OnEntityDeactivated()
    {
        Debug.Log($"Entity {Name} deactivated");
        
        // Cleanup active-state resources
        var renderer = GetComponent<Renderer>();
        if (renderer != null)
            renderer.enabled = false;
    }
    
    private void OnEntityUpdated(float deltaTime)
    {
        // Track update statistics
        float totalUpdateTime = this.GetValue<float>(EntityNames.TOTAL_UPDATE_TIME, 0f) + deltaTime;
        this.SetValue(EntityNames.TOTAL_UPDATE_TIME, totalUpdateTime);
        
        int updateCount = this.GetValue<int>(EntityNames.UPDATE_COUNT, 0) + 1;
        this.SetValue(EntityNames.UPDATE_COUNT, updateCount);
    }
}

public class LifecycleLoggerBehaviour : IEntityBehaviour, IEntitySpawn, IEntityDespawn, 
    IEntityActivate, IEntityDeactivate, IEntityUpdate, IEntityFixedUpdate, IEntityLateUpdate
{
    public void OnInstall(IEntity entity)
    {
        Debug.Log($"[Lifecycle] {entity.Name}: Behaviour installed");
    }
    
    public void OnSpawn(IEntity entity)
    {
        Debug.Log($"[Lifecycle] {entity.Name}: Spawned");
    }
    
    public void OnDespawn(IEntity entity)
    {
        Debug.Log($"[Lifecycle] {entity.Name}: Despawned");
    }
    
    public void OnActivate(IEntity entity)
    {
        Debug.Log($"[Lifecycle] {entity.Name}: Activated");
    }
    
    public void OnDeactivate(IEntity entity)
    {
        Debug.Log($"[Lifecycle] {entity.Name}: Deactivated");
    }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Log periodic update info (every 60 frames)
        if (Time.frameCount % 60 == 0)
        {
            Debug.Log($"[Lifecycle] {entity.Name}: Update - Frame {Time.frameCount}, Delta {deltaTime:F3}");
        }
    }
    
    public void OnFixedUpdate(IEntity entity, float deltaTime)
    {
        // Log fixed update info occasionally
        if (Time.fixedTime % 1f < deltaTime)
        {
            Debug.Log($"[Lifecycle] {entity.Name}: FixedUpdate - Time {Time.fixedTime:F1}, Delta {deltaTime:F3}");
        }
    }
    
    public void OnLateUpdate(IEntity entity, float deltaTime)
    {
        // Log late update info very rarely
        if (Time.frameCount % 300 == 0)
        {
            Debug.Log($"[Lifecycle] {entity.Name}: LateUpdate - Frame {Time.frameCount}");
        }
    }
}
```

### Advanced State Management

```csharp
public class StateMachineEntity : SceneEntity
{
    public enum EntityState
    {
        Uninitialized,
        Spawning,
        Idle,
        Active,
        Paused,
        Dying,
        Dead
    }
    
    [Header("State Management")]
    [SerializeField] private EntityState currentState = EntityState.Uninitialized;
    [SerializeField] private float stateTransitionDelay = 0.1f;
    
    public EntityState CurrentState => currentState;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Setup state-based lifecycle handling
        this.OnSpawned += () => TransitionToState(EntityState.Idle);
        this.OnActivated += () => TransitionToState(EntityState.Active);
        this.OnDeactivated += () => TransitionToState(EntityState.Paused);
        this.OnDespawned += () => TransitionToState(EntityState.Dead);
        
        // Add state management behaviours
        this.AddBehaviour(new StateManagerBehaviour());
        this.AddBehaviour(new StateTransitionBehaviour());
    }
    
    public void TransitionToState(EntityState newState)
    {
        if (currentState == newState) return;
        
        var previousState = currentState;
        currentState = newState;
        
        Debug.Log($"State transition: {previousState} -> {newState}");
        
        // Store state information
        this.SetValue(EntityNames.CURRENT_STATE, newState);
        this.SetValue(EntityNames.PREVIOUS_STATE, previousState);
        this.SetValue(EntityNames.STATE_CHANGE_TIME, Time.time);
        
        // Handle state-specific logic
        OnStateChanged(previousState, newState);
    }
    
    private void OnStateChanged(EntityState from, EntityState to)
    {
        switch (to)
        {
            case EntityState.Spawning:
                StartCoroutine(DelayedSpawn());
                break;
                
            case EntityState.Idle:
                // Setup idle behaviors
                this.AddBehaviour(new IdleBehaviour());
                break;
                
            case EntityState.Active:
                // Remove idle behaviors, add active behaviors
                this.DelBehaviour<IdleBehaviour>();
                this.AddBehaviour(new ActiveBehaviour());
                break;
                
            case EntityState.Paused:
                // Pause without full deactivation
                this.GetBehaviour<ActiveBehaviour>()?.Pause();
                break;
                
            case EntityState.Dying:
                StartCoroutine(DeathSequence());
                break;
                
            case EntityState.Dead:
                // Final cleanup
                this.ClearBehaviours();
                break;
        }
    }
    
    private System.Collections.IEnumerator DelayedSpawn()
    {
        yield return new WaitForSeconds(stateTransitionDelay);
        this.Spawn();
    }
    
    private System.Collections.IEnumerator DeathSequence()
    {
        // Play death animation, effects, etc.
        yield return new WaitForSeconds(2f);
        
        this.Deactivate();
        this.Despawn();
    }
    
    // Manual state control methods
    public void StartSpawning() => TransitionToState(EntityState.Spawning);
    public void Pause() => this.Deactivate();
    public void Resume() => this.Activate();
    public void Kill() => TransitionToState(EntityState.Dying);
}
```

### Performance-Optimized Lifecycle

```csharp
public class OptimizedEntity : SceneEntity
{
    [Header("Performance Settings")]
    [SerializeField] private int maxUpdatesPerFrame = 10;
    [SerializeField] private float updateThrottleInterval = 0.1f;
    
    private float lastUpdateTime;
    private int updatesThisFrame;
    
    protected override void ProcessUpdate(float deltaTime)
    {
        // Throttle updates based on performance settings
        updatesThisFrame++;
        
        if (updatesThisFrame > maxUpdatesPerFrame)
            return;
            
        if (Time.time - lastUpdateTime < updateThrottleInterval)
            return;
        
        base.ProcessUpdate(deltaTime);
        lastUpdateTime = Time.time;
    }
    
    protected override void ProcessFixedUpdate(float deltaTime)
    {
        // FixedUpdate is already throttled by Unity's fixed timestep
        base.ProcessFixedUpdate(deltaTime);
    }
    
    protected override void ProcessLateUpdate(float deltaTime)
    {
        // Reset frame counters in LateUpdate
        updatesThisFrame = 0;
        base.ProcessLateUpdate(deltaTime);
    }
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Add performance monitoring
        this.AddBehaviour(new PerformanceMonitorBehaviour());
        this.AddBehaviour(new UpdateThrottleBehaviour(maxUpdatesPerFrame));
    }
}

public class PerformanceMonitorBehaviour : IEntityBehaviour, IEntityUpdate
{
    private float frameTime;
    private int frameCount;
    private float averageFrameTime;
    
    public void OnInstall(IEntity entity) { }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        frameTime += deltaTime;
        frameCount++;
        
        if (frameCount >= 60) // Update average every 60 frames
        {
            averageFrameTime = frameTime / frameCount;
            entity.SetValue(EntityNames.AVERAGE_FRAME_TIME, averageFrameTime);
            entity.SetValue(EntityNames.AVERAGE_FPS, 1f / averageFrameTime);
            
            frameTime = 0;
            frameCount = 0;
        }
    }
}
```

### Lifecycle-Based Component Management

```csharp
public class ComponentManagedEntity : SceneEntity
{
    [Header("Component Management")]
    [SerializeField] private List<MonoBehaviour> spawnComponents = new List<MonoBehaviour>();
    [SerializeField] private List<MonoBehaviour> activeComponents = new List<MonoBehaviour>();
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Disable all managed components initially
        DisableAllManagedComponents();
        
        // Setup lifecycle-based component management
        this.OnSpawned += EnableSpawnComponents;
        this.OnDespawned += DisableSpawnComponents;
        this.OnActivated += EnableActiveComponents;
        this.OnDeactivated += DisableActiveComponents;
    }
    
    private void DisableAllManagedComponents()
    {
        foreach (var component in spawnComponents)
        {
            if (component != null)
                component.enabled = false;
        }
        
        foreach (var component in activeComponents)
        {
            if (component != null)
                component.enabled = false;
        }
    }
    
    private void EnableSpawnComponents()
    {
        foreach (var component in spawnComponents)
        {
            if (component != null)
                component.enabled = true;
        }
    }
    
    private void DisableSpawnComponents()
    {
        foreach (var component in spawnComponents)
        {
            if (component != null)
                component.enabled = false;
        }
    }
    
    private void EnableActiveComponents()
    {
        foreach (var component in activeComponents)
        {
            if (component != null)
                component.enabled = true;
        }
    }
    
    private void DisableActiveComponents()
    {
        foreach (var component in activeComponents)
        {
            if (component != null)
                component.enabled = false;
        }
    }
}
```

## Lifecycle State Diagram

```
Uninitialized
     ↓
   Install()
     ↓
  Installed
     ↓
   Spawn() ← ← ← ← ← ← ←
     ↓                 ↑
   Spawned             ↑
     ↓                 ↑
  Activate()    Deactivate()
     ↓                 ↑
   Active → → → → → → → ↑
     ↓                 
  Despawn()
     ↓
  Despawned
```

## Best Practices

1. **Proper State Transitions** – Always follow the correct lifecycle order
2. **Event Handling** – Subscribe to lifecycle events for reactive behavior
3. **Resource Management** – Acquire resources on spawn/activate, release on deactivate/despawn
4. **Performance Awareness** – Be mindful of update frequency and behaviour count
5. **Error Handling** – Handle exceptions in lifecycle methods gracefully
6. **State Validation** – Check entity state before performing operations

## Performance Considerations

### Update Loop Efficiency
- Update arrays are dynamically managed for optimal iteration
- Behaviours are registered/unregistered efficiently during state changes
- Update methods include active state checks to prevent unnecessary processing

### Memory Management
- Update arrays are resized as needed to minimize allocations
- Equality comparers are cached for efficient behaviour lookup
- State changes trigger minimal garbage generation

### Unity Integration
- Compatible with Unity's MonoBehaviour lifecycle
- Proper integration with Unity's update loops and timing
- Support for both manual and automatic lifecycle management

## Common Patterns

### Delayed Activation
```csharp
IEnumerator DelayedActivation(float delay)
{
    this.Spawn();
    yield return new WaitForSeconds(delay);
    this.Activate();
}
```

### Conditional Lifecycle
```csharp
public override void Activate()
{
    if (ShouldActivate())
        base.Activate();
}

private bool ShouldActivate()
{
    return HasValue(EntityNames.ACTIVATION_CONDITION) && 
           GetValue<bool>(EntityNames.ACTIVATION_CONDITION);
}
```

### Lifecycle Chaining
```csharp
protected override void ProcessSpawn()
{
    base.ProcessSpawn();
    
    // Spawn child entities
    foreach (var child in children)
    {
        child.Spawn();
    }
}
```

## Notes

- Lifecycle methods are virtual and can be overridden for custom behavior
- State transitions are atomic and thread-safe within Unity's main thread
- Update registration/unregistration is automatically handled during state changes
- Events are fired after state changes are complete
- Supports both manual lifecycle control and automatic Unity integration
- Performance optimized with aggressive inlining for hot paths