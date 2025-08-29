# 🧩 IUpdatable

`IUpdatable` represents an object that supports update callbacks during the game loop, including regular, fixed, and late update phases. It provides a standardized interface for objects that need frame-based processing.

## Key Features

- **Three Update Phases** – Regular, Fixed, and Late update support
- **Delta Time Support** – Frame-independent movement and timing
- **Event Notifications** – Optional events for update completion
- **Unity Compatibility** – Matches Unity's update cycle

---

## Interface Definition

```csharp
public interface IUpdatable
{
    event Action<float> OnUpdated;
    event Action<float> OnFixedUpdated;
    event Action<float> OnLateUpdated;
    
    void OnUpdate(float deltaTime);
    void OnFixedUpdate(float deltaTime);
    void OnLateUpdate(float deltaTime);
}
```

## Events

### OnUpdated
```csharp
event Action<float> OnUpdated;
```
- **Triggered**: After OnUpdate completes
- **Parameter**: Delta time since last frame
- **Usage**: Chain dependent updates

### OnFixedUpdated
```csharp
event Action<float> OnFixedUpdated;
```
- **Triggered**: After OnFixedUpdate completes
- **Parameter**: Fixed timestep value
- **Usage**: Physics response handling

### OnLateUpdated
```csharp
event Action<float> OnLateUpdated;
```
- **Triggered**: After OnLateUpdate completes
- **Parameter**: Delta time since last frame
- **Usage**: Post-processing notifications

## Methods

### OnUpdate
```csharp
void OnUpdate(float deltaTime);
```
- **Description**: Called once per frame
- **Parameter**: Time elapsed since last frame
- **Usage**: General game logic, input handling, animations
- **Timing**: Variable framerate

### OnFixedUpdate
```csharp
void OnFixedUpdate(float deltaTime);
```
- **Description**: Called at fixed intervals
- **Parameter**: Fixed timestep (typically 0.02s)
- **Usage**: Physics calculations, deterministic logic
- **Timing**: Fixed framerate (50Hz default in Unity)

### OnLateUpdate
```csharp
void OnLateUpdate(float deltaTime);
```
- **Description**: Called after all OnUpdate calls
- **Parameter**: Time elapsed since last frame
- **Usage**: Camera follow, UI updates, post-processing
- **Timing**: Variable framerate, after Update

## Implementation Examples

### Basic Implementation

```csharp
public class UpdateableObject : IUpdatable
{
    public event Action<float> OnUpdated;
    public event Action<float> OnFixedUpdated;
    public event Action<float> OnLateUpdated;
    
    public void OnUpdate(float deltaTime)
    {
        // Regular update logic
        ProcessInput(deltaTime);
        UpdateAnimation(deltaTime);
        
        OnUpdated?.Invoke(deltaTime);
    }
    
    public void OnFixedUpdate(float deltaTime)
    {
        // Physics update logic
        ApplyPhysics(deltaTime);
        
        OnFixedUpdated?.Invoke(deltaTime);
    }
    
    public void OnLateUpdate(float deltaTime)
    {
        // Post-update logic
        UpdateCamera(deltaTime);
        
        OnLateUpdated?.Invoke(deltaTime);
    }
}
```

### Movement Controller

```csharp
public class MovementController : IUpdatable, IActivatable
{
    private Vector3 position;
    private Vector3 velocity;
    private bool isActive;
    
    public event Action<float> OnUpdated;
    public event Action<float> OnFixedUpdated;
    public event Action<float> OnLateUpdated;
    
    public bool IsActive => isActive;
    
    public void OnUpdate(float deltaTime)
    {
        if (!isActive) return;
        
        // Handle input
        var input = Input.GetAxis("Horizontal");
        velocity.x = input * speed;
        
        OnUpdated?.Invoke(deltaTime);
    }
    
    public void OnFixedUpdate(float deltaTime)
    {
        if (!isActive) return;
        
        // Apply movement physics
        position += velocity * deltaTime;
        
        // Apply gravity
        velocity.y -= gravity * deltaTime;
        
        OnFixedUpdated?.Invoke(deltaTime);
    }
    
    public void OnLateUpdate(float deltaTime)
    {
        if (!isActive) return;
        
        // Update visual position
        transform.position = position;
        
        OnLateUpdated?.Invoke(deltaTime);
    }
}
```

### Animation System

```csharp
public class AnimationSystem : IUpdatable
{
    private Dictionary<string, AnimationClip> animations = new();
    private AnimationClip currentClip;
    private float animationTime;
    
    public event Action<float> OnUpdated;
    public event Action<float> OnFixedUpdated;
    public event Action<float> OnLateUpdated;
    
    public void OnUpdate(float deltaTime)
    {
        if (currentClip == null) return;
        
        // Advance animation
        animationTime += deltaTime * animationSpeed;
        
        if (animationTime >= currentClip.Duration)
        {
            if (currentClip.IsLooping)
                animationTime %= currentClip.Duration;
            else
                OnAnimationComplete();
        }
        
        // Sample animation
        currentClip.Sample(animationTime);
        
        OnUpdated?.Invoke(deltaTime);
    }
    
    public void OnFixedUpdate(float deltaTime)
    {
        // Not used for animation
        OnFixedUpdated?.Invoke(deltaTime);
    }
    
    public void OnLateUpdate(float deltaTime)
    {
        // Apply animation transforms after physics
        ApplyBoneTransforms();
        
        OnLateUpdated?.Invoke(deltaTime);
    }
}
```

## Usage Patterns

### Update Manager

```csharp
public class UpdateManager : MonoBehaviour
{
    private static List<IUpdatable> updateables = new();
    private static List<IUpdatable> fixedUpdateables = new();
    private static List<IUpdatable> lateUpdateables = new();
    
    public static void Register(IUpdatable obj)
    {
        updateables.Add(obj);
        fixedUpdateables.Add(obj);
        lateUpdateables.Add(obj);
    }
    
    public static void Unregister(IUpdatable obj)
    {
        updateables.Remove(obj);
        fixedUpdateables.Remove(obj);
        lateUpdateables.Remove(obj);
    }
    
    void Update()
    {
        float deltaTime = Time.deltaTime;
        for (int i = 0; i < updateables.Count; i++)
        {
            updateables[i].OnUpdate(deltaTime);
        }
    }
    
    void FixedUpdate()
    {
        float fixedDeltaTime = Time.fixedDeltaTime;
        for (int i = 0; i < fixedUpdateables.Count; i++)
        {
            fixedUpdateables[i].OnFixedUpdate(fixedDeltaTime);
        }
    }
    
    void LateUpdate()
    {
        float deltaTime = Time.deltaTime;
        for (int i = 0; i < lateUpdateables.Count; i++)
        {
            lateUpdateables[i].OnLateUpdate(deltaTime);
        }
    }
}
```

### Conditional Updates

```csharp
public class ConditionalUpdater : IUpdatable
{
    private IUpdatable target;
    private Func<bool> condition;
    
    public ConditionalUpdater(IUpdatable updatable, Func<bool> updateCondition)
    {
        target = updatable;
        condition = updateCondition;
    }
    
    public void OnUpdate(float deltaTime)
    {
        if (condition())
            target.OnUpdate(deltaTime);
    }
    
    public void OnFixedUpdate(float deltaTime)
    {
        if (condition())
            target.OnFixedUpdate(deltaTime);
    }
    
    public void OnLateUpdate(float deltaTime)
    {
        if (condition())
            target.OnLateUpdate(deltaTime);
    }
}
```

### Procedural Update Operations

Following Atomic's procedural pattern:

```csharp
public static class UpdateUtils
{
    public static void UpdateWithTimescale(IUpdatable obj, float deltaTime, float timescale)
    {
        obj.OnUpdate(deltaTime * timescale);
    }
    
    public static void UpdateAtInterval(IUpdatable obj, float interval, ref float accumulator, float deltaTime)
    {
        accumulator += deltaTime;
        while (accumulator >= interval)
        {
            obj.OnUpdate(interval);
            accumulator -= interval;
        }
    }
    
    public static void UpdateWithSmoothing(IUpdatable obj, float deltaTime, float smoothing)
    {
        float smoothedDelta = Mathf.Lerp(deltaTime, Time.smoothDeltaTime, smoothing);
        obj.OnUpdate(smoothedDelta);
    }
    
    public static IEnumerator UpdateCoroutine(IUpdatable obj, float duration)
    {
        float elapsed = 0;
        while (elapsed < duration)
        {
            float deltaTime = Time.deltaTime;
            obj.OnUpdate(deltaTime);
            elapsed += deltaTime;
            yield return null;
        }
    }
    
    public static void BatchUpdate(IEnumerable<IUpdatable> objects, float deltaTime)
    {
        foreach (var obj in objects)
        {
            obj.OnUpdate(deltaTime);
        }
    }
}
```

## Update Phase Guidelines

### When to Use Update
- Input handling
- Game logic
- AI decisions
- Animation state machines
- Non-physics movement

### When to Use FixedUpdate
- Physics calculations
- Rigidbody manipulation
- Force application
- Collision detection
- Deterministic simulation

### When to Use LateUpdate
- Camera follow
- UI positioning
- Procedural animation
- Transform adjustments
- Post-processing

## Performance Optimization

```csharp
public class OptimizedUpdater : IUpdatable
{
    private float updateInterval = 0.1f;
    private float accumulator = 0;
    
    public void OnUpdate(float deltaTime)
    {
        // Throttle expensive updates
        accumulator += deltaTime;
        if (accumulator >= updateInterval)
        {
            PerformExpensiveUpdate();
            accumulator = 0;
        }
        
        // Always do cheap updates
        PerformCheapUpdate(deltaTime);
    }
    
    public void OnFixedUpdate(float deltaTime)
    {
        // Only if physics needed
        if (hasPhysics)
        {
            UpdatePhysics(deltaTime);
        }
    }
    
    public void OnLateUpdate(float deltaTime)
    {
        // Only if visible
        if (IsVisible())
        {
            UpdateVisuals(deltaTime);
        }
    }
}
```

## Best Practices

1. **Check Active State** – Skip updates for inactive objects
2. **Use Appropriate Phase** – Put logic in correct update method
3. **Frame Independence** – Always use deltaTime for movement
4. **Minimize Work** – Keep update methods lightweight
5. **Profile Performance** – Monitor update times

## Common Patterns

### Update Priority
```csharp
public class PriorityUpdater
{
    private SortedList<int, IUpdatable> updateables;
    
    public void Update(float deltaTime)
    {
        foreach (var kvp in updateables)
        {
            kvp.Value.OnUpdate(deltaTime);
        }
    }
}
```

### Update Groups
```csharp
public enum UpdateGroup
{
    PreUpdate,
    Update,
    PostUpdate
}

public class GroupedUpdater
{
    private Dictionary<UpdateGroup, List<IUpdatable>> groups;
    
    public void UpdateAll(float deltaTime)
    {
        groups[UpdateGroup.PreUpdate].ForEach(u => u.OnUpdate(deltaTime));
        groups[UpdateGroup.Update].ForEach(u => u.OnUpdate(deltaTime));
        groups[UpdateGroup.PostUpdate].ForEach(u => u.OnUpdate(deltaTime));
    }
}
```

## Notes

- IUpdatable doesn't require implementation of all three methods
- Empty implementations are valid for unused update phases
- Consider combining with IActivatable to control update execution
- Unity's execution order affects when updates are called
- Fixed timestep can be configured in Unity's Time settings