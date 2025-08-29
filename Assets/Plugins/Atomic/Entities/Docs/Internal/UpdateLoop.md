# ⚙️ UpdateLoop

`UpdateLoop` is an internal MonoBehaviour singleton that manages and dispatches Unity update callbacks (Update, FixedUpdate, LateUpdate) to all registered `IUpdatable` instances. It provides centralized update handling across the entire application lifecycle.

## Key Features

- **Singleton Pattern** – Single instance manages all updates
- **Hidden GameObject** – Invisible in hierarchy and persistent across scenes
- **Unity Integration** – Seamlessly integrates with Unity's update system
- **Automatic Lifecycle** – Self-instantiates and manages registration
- **Performance Optimized** – Direct array iteration for minimal overhead
- **Editor Safe** – Proper handling of play mode transitions

---

## Internal Architecture

### Instance Management

```csharp
internal static UpdateLoop Instance { get; }
```

Singleton instance that:
- Auto-creates hidden GameObject when first accessed
- Persists across scene loads with DontDestroyOnLoad
- Handles editor play mode transitions
- Uses lazy initialization for optimal startup

### Registration System

```csharp
internal void Add(IUpdatable updatable)
internal void Del(IUpdatable updatable)
internal bool Contains(IUpdatable updatable)
```

Manages IUpdatable instances:
- **Add**: Registers updatable (no duplicates allowed)
- **Del**: Unregisters updatable with safe removal
- **Contains**: Checks if updatable is currently registered

### Update Dispatch

```csharp
private void Update()        // Calls IUpdatable.OnUpdate
private void FixedUpdate()   // Calls IUpdatable.OnFixedUpdate  
private void LateUpdate()    // Calls IUpdatable.OnLateUpdate
```

Unity callback methods that dispatch to registered updatables using appropriate delta time.

---

## Implementation Details

### GameObject Creation

```csharp
private static UpdateLoop CreateInstance()
{
    GameObject go = new GameObject("Update Manager");
    go.hideFlags = HideFlags.HideAndDontSave;
    DontDestroyOnLoad(go);
    return go.AddComponent<UpdateLoop>();
}
```

Creates a hidden, persistent GameObject:
- **Name**: "Update Manager" for debugging
- **HideFlags**: Hidden from hierarchy and not saved
- **DontDestroyOnLoad**: Persists across scene changes
- **Component**: Adds UpdateLoop MonoBehaviour

### Array Management

Uses `EntityUtils` for efficient array operations:
- Dynamic array expansion as needed
- Duplicate prevention with equality comparer
- Safe removal with element shifting
- Optimized iteration without bounds checking

### Editor Integration

```csharp
#if UNITY_EDITOR
[InitializeOnEnterPlayMode]
private static void OnEnterPlayMode()
{
    _spawned = false;
}
#endif
```

Handles Unity Editor play mode transitions:
- Resets spawn flag when entering play mode
- Prevents stale singleton instances
- Only active during play mode execution

---

## Usage by Framework Components

### EntityWorld Registration

```csharp
public class EntityWorld : IUpdatable
{
    private bool _registered;
    
    public void Initialize()
    {
        if (!_registered)
        {
            UpdateLoop.Instance.Add(this);
            _registered = true;
        }
    }
    
    public void OnUpdate(float deltaTime)
    {
        // Update all entities in this world
        UpdateEntities(deltaTime);
    }
    
    public void OnFixedUpdate(float deltaTime)
    {
        // Fixed timestep physics updates
        FixedUpdateEntities(deltaTime);
    }
    
    public void OnLateUpdate(float deltaTime)
    {
        // Post-processing updates
        LateUpdateEntities(deltaTime);
    }
    
    public void Dispose()
    {
        if (_registered)
        {
            UpdateLoop.Instance.Del(this);
            _registered = false;
        }
    }
}
```

### Custom Update Systems

```csharp
public class ParticleSystem : IUpdatable
{
    private List<Particle> particles = new List<Particle>();
    private bool active;
    
    public void Start()
    {
        active = true;
        UpdateLoop.Instance.Add(this);
    }
    
    public void Stop()
    {
        if (active)
        {
            UpdateLoop.Instance.Del(this);
            active = false;
        }
    }
    
    public void OnUpdate(float deltaTime)
    {
        if (!active) return;
        
        // Update particle positions
        for (int i = particles.Count - 1; i >= 0; i--)
        {
            var particle = particles[i];
            particle.lifetime -= deltaTime;
            
            if (particle.lifetime <= 0)
            {
                particles.RemoveAt(i);
            }
            else
            {
                particle.position += particle.velocity * deltaTime;
                particles[i] = particle;
            }
        }
    }
    
    public void OnFixedUpdate(float deltaTime)
    {
        // Particle physics updates
    }
    
    public void OnLateUpdate(float deltaTime)
    {
        // Render particle effects
    }
}
```

### Entity Behaviour Updates

```csharp
public class MovementBehaviour : IEntityBehaviour, IUpdatable
{
    private Entity entity;
    private Vector3 velocity;
    private bool registered;
    
    public void Initialize(Entity owner)
    {
        entity = owner;
        if (!registered)
        {
            UpdateLoop.Instance.Add(this);
            registered = true;
        }
    }
    
    public void Deinitialize()
    {
        if (registered)
        {
            UpdateLoop.Instance.Del(this);
            registered = false;
        }
    }
    
    public void OnUpdate(float deltaTime)
    {
        if (entity?.Spawned == true && entity.Enabled)
        {
            var position = entity.GetValue<Vector3>("Position");
            position += velocity * deltaTime;
            entity.SetValue("Position", position);
        }
    }
    
    public void OnFixedUpdate(float deltaTime) { }
    public void OnLateUpdate(float deltaTime) { }
}
```

## Performance Characteristics

### Update Dispatch Performance
- **Direct array iteration** – No LINQ or collection overhead
- **Single loop per frame** – Minimal function call overhead  
- **No boxing/unboxing** – Direct interface calls
- **Cache friendly** – Sequential memory access pattern

### Registration Performance
- **O(n) for Add/Remove** – Linear search for duplicates/removal
- **O(1) for Contains** – Simple linear search with early exit
- **Minimal allocations** – Array expansion only when needed

### Memory Footprint
- **Single GameObject** – Minimal Unity object overhead
- **Dynamic array** – Grows as needed, no pre-allocation
- **Interface references** – No boxing of value types

## Thread Safety

- **Main thread only** – All operations must occur on Unity main thread
- **Not thread-safe** – No synchronization for concurrent access
- **Unity callbacks** – Update methods called from Unity's main thread

## Lifecycle Management

### Automatic Startup
1. First `IUpdatable` registration triggers instance creation
2. Hidden GameObject created with UpdateLoop component
3. GameObject marked as DontDestroyOnLoad
4. Update callbacks begin immediately

### Play Mode Transitions
1. Editor tracks play mode state changes
2. Spawn flag reset on play mode entry
3. Previous instances cleaned up automatically
4. New instance created on first access

### Shutdown Behavior
- GameObject persists until application exit
- No explicit cleanup required
- Editor properly resets state between sessions

## Best Practices

### Registration Management
1. **Register once** – Check registration status before adding
2. **Unregister on cleanup** – Always remove when no longer needed
3. **Null checks** – Verify instance validity in update methods
4. **Exception handling** – Wrap update logic to prevent crashes

### Performance Optimization
1. **Minimal work in updates** – Keep update methods lightweight
2. **State checks** – Skip expensive operations when inactive
3. **Batch operations** – Group multiple updates when possible
4. **Profile regularly** – Monitor update performance impact

## Common Patterns

### Conditional Registration

```csharp
public class ConditionalUpdater : IUpdatable
{
    private bool shouldUpdate;
    private bool registered;
    
    public void SetUpdateEnabled(bool enabled)
    {
        if (enabled && !registered)
        {
            UpdateLoop.Instance.Add(this);
            registered = true;
        }
        else if (!enabled && registered)
        {
            UpdateLoop.Instance.Del(this);
            registered = false;
        }
        shouldUpdate = enabled;
    }
    
    public void OnUpdate(float deltaTime)
    {
        if (shouldUpdate)
        {
            // Perform updates
        }
    }
}
```

### Pooled Updates

```csharp
public class UpdatePool<T> : IUpdatable where T : IPooledUpdatable
{
    private List<T> activeItems = new List<T>();
    private bool registered;
    
    public void AddItem(T item)
    {
        activeItems.Add(item);
        
        if (!registered && activeItems.Count > 0)
        {
            UpdateLoop.Instance.Add(this);
            registered = true;
        }
    }
    
    public void RemoveItem(T item)
    {
        activeItems.Remove(item);
        
        if (registered && activeItems.Count == 0)
        {
            UpdateLoop.Instance.Del(this);
            registered = false;
        }
    }
    
    public void OnUpdate(float deltaTime)
    {
        for (int i = activeItems.Count - 1; i >= 0; i--)
        {
            if (!activeItems[i].UpdateItem(deltaTime))
            {
                // Item finished, remove from active list
                RemoveItem(activeItems[i]);
            }
        }
    }
}
```

## Internal Implementation Notes

- Uses `#if UNITY_5_3_OR_NEWER` compilation guards
- Leverages `EntityUtils` for array management
- Implements singleton pattern with lazy initialization
- Hidden from AddComponentMenu with empty string
- DisallowMultipleComponent ensures single instance per GameObject