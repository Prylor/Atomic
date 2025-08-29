# 🧩 IActivatable

`IActivatable` represents an object that can be enabled or disabled during runtime. It provides a standardized way to control whether an object is actively processing or dormant.

## Key Features

- **Enable/Disable Control** – Runtime activation management
- **State Tracking** – Check if object is currently active
- **Event Notifications** – React to activation state changes
- **Performance Optimization** – Disable inactive objects to save resources

---

## Interface Definition

```csharp
public interface IActivatable
{
    event Action OnActivated;
    event Action OnDeactivated;
    bool IsActive { get; }
    void Activate();
    void Deactivate();
}
```

## Events

### OnActivated
```csharp
event Action OnActivated;
```
- **Triggered**: After object successfully activates
- **Usage**: Resume processing, enable renderers, start coroutines

### OnDeactivated
```csharp
event Action OnDeactivated;
```
- **Triggered**: After object successfully deactivates
- **Usage**: Pause processing, disable effects, stop updates

## Properties

### IsActive
```csharp
bool IsActive { get; }
```
- **Description**: Indicates if object is currently active
- **Returns**: `true` if active, `false` otherwise

## Methods

### Activate
```csharp
void Activate();
```
- **Description**: Enables the object
- **Behavior**:
  - Enables processing/updates
  - Sets IsActive to true
  - Triggers OnActivated event
- **Idempotent**: Should handle multiple calls safely

### Deactivate
```csharp
void Deactivate();
```
- **Description**: Disables the object
- **Behavior**:
  - Stops processing/updates
  - Sets IsActive to false
  - Triggers OnDeactivated event
- **Idempotent**: Should handle multiple calls safely

## Implementation Examples

### Basic Implementation

```csharp
public class ActivatableObject : IActivatable
{
    public event Action OnActivated;
    public event Action OnDeactivated;
    
    public bool IsActive { get; private set; }
    
    public void Activate()
    {
        if (IsActive) return; // Idempotent
        
        IsActive = true;
        OnActivate();
        OnActivated?.Invoke();
    }
    
    public void Deactivate()
    {
        if (!IsActive) return; // Idempotent
        
        IsActive = false;
        OnDeactivate();
        OnDeactivated?.Invoke();
    }
    
    protected virtual void OnActivate()
    {
        // Override for activation logic
    }
    
    protected virtual void OnDeactivate()
    {
        // Override for deactivation logic
    }
}
```

### Update System

```csharp
public class UpdateableSystem : IActivatable, IUpdatable
{
    private bool isActive;
    private List<Action<float>> updateCallbacks = new();
    
    public event Action OnActivated;
    public event Action OnDeactivated;
    public event Action<float> OnUpdated;
    
    public bool IsActive => isActive;
    
    public void Activate()
    {
        if (isActive) return;
        
        isActive = true;
        UpdateManager.Register(this);
        OnActivated?.Invoke();
    }
    
    public void Deactivate()
    {
        if (!isActive) return;
        
        isActive = false;
        UpdateManager.Unregister(this);
        OnDeactivated?.Invoke();
    }
    
    public void OnUpdate(float deltaTime)
    {
        if (!isActive) return;
        
        OnUpdated?.Invoke(deltaTime);
        
        foreach (var callback in updateCallbacks)
        {
            callback(deltaTime);
        }
    }
}
```

### Component System

```csharp
public class AIController : IActivatable
{
    private NavMeshAgent agent;
    private Coroutine thinkCoroutine;
    
    public event Action OnActivated;
    public event Action OnDeactivated;
    public bool IsActive { get; private set; }
    
    public AIController(NavMeshAgent navAgent)
    {
        agent = navAgent;
    }
    
    public void Activate()
    {
        if (IsActive) return;
        
        IsActive = true;
        
        // Enable AI components
        agent.enabled = true;
        thinkCoroutine = StartCoroutine(ThinkLoop());
        
        OnActivated?.Invoke();
    }
    
    public void Deactivate()
    {
        if (!IsActive) return;
        
        IsActive = false;
        
        // Disable AI components
        agent.enabled = false;
        if (thinkCoroutine != null)
        {
            StopCoroutine(thinkCoroutine);
            thinkCoroutine = null;
        }
        
        OnDeactivated?.Invoke();
    }
    
    private IEnumerator ThinkLoop()
    {
        while (IsActive)
        {
            Think();
            yield return new WaitForSeconds(0.5f);
        }
    }
}
```

## Usage Patterns

### Activation Manager

```csharp
public class ActivationManager
{
    private HashSet<IActivatable> activeObjects = new();
    private HashSet<IActivatable> inactiveObjects = new();
    
    public void Register(IActivatable obj)
    {
        if (obj.IsActive)
            activeObjects.Add(obj);
        else
            inactiveObjects.Add(obj);
            
        obj.OnActivated += () => OnObjectActivated(obj);
        obj.OnDeactivated += () => OnObjectDeactivated(obj);
    }
    
    private void OnObjectActivated(IActivatable obj)
    {
        inactiveObjects.Remove(obj);
        activeObjects.Add(obj);
    }
    
    private void OnObjectDeactivated(IActivatable obj)
    {
        activeObjects.Remove(obj);
        inactiveObjects.Add(obj);
    }
    
    public void ActivateAll()
    {
        foreach (var obj in inactiveObjects.ToArray())
        {
            obj.Activate();
        }
    }
    
    public void DeactivateAll()
    {
        foreach (var obj in activeObjects.ToArray())
        {
            obj.Deactivate();
        }
    }
}
```

### Conditional Activation

```csharp
public class ConditionalActivator
{
    private IActivatable target;
    private Func<bool> condition;
    private bool wasActive;
    
    public ConditionalActivator(IActivatable activatable, Func<bool> activationCondition)
    {
        target = activatable;
        condition = activationCondition;
    }
    
    public void Update()
    {
        bool shouldBeActive = condition();
        
        if (shouldBeActive != wasActive)
        {
            if (shouldBeActive)
                target.Activate();
            else
                target.Deactivate();
                
            wasActive = shouldBeActive;
        }
    }
}

// Usage
var activator = new ConditionalActivator(
    enemy,
    () => Vector3.Distance(player.position, enemy.position) < 50f
);
```

### Procedural Activation Operations

Following Atomic's procedural pattern:

```csharp
public static class ActivationUtils
{
    public static void ToggleActivation(IActivatable obj)
    {
        if (obj.IsActive)
            obj.Deactivate();
        else
            obj.Activate();
    }
    
    public static void ActivateForDuration(IActivatable obj, float duration)
    {
        obj.Activate();
        Timer.Schedule(duration, () => obj.Deactivate());
    }
    
    public static void PulseActivation(IActivatable obj, float onTime, float offTime, int cycles)
    {
        IEnumerator Pulse()
        {
            for (int i = 0; i < cycles; i++)
            {
                obj.Activate();
                yield return new WaitForSeconds(onTime);
                obj.Deactivate();
                yield return new WaitForSeconds(offTime);
            }
        }
        CoroutineRunner.Start(Pulse());
    }
    
    public static void ActivateWhenConditionMet(
        IActivatable obj, 
        Func<bool> condition, 
        float checkInterval = 0.1f)
    {
        IEnumerator CheckCondition()
        {
            while (!condition())
            {
                yield return new WaitForSeconds(checkInterval);
            }
            obj.Activate();
        }
        CoroutineRunner.Start(CheckCondition());
    }
    
    public static void SynchronizeActivation(
        IActivatable source, 
        params IActivatable[] targets)
    {
        source.OnActivated += () =>
        {
            foreach (var target in targets)
                target.Activate();
        };
        
        source.OnDeactivated += () =>
        {
            foreach (var target in targets)
                target.Deactivate();
        };
    }
}
```

## Combined Lifecycle

### With ISpawnable

```csharp
public class GameObject : ISpawnable, IActivatable
{
    public bool IsSpawned { get; private set; }
    public bool IsActive { get; private set; }
    
    public void Spawn()
    {
        if (IsSpawned) return;
        
        IsSpawned = true;
        // Spawning doesn't automatically activate
        OnSpawned?.Invoke();
    }
    
    public void Activate()
    {
        if (!IsSpawned)
        {
            Debug.LogWarning("Cannot activate unspawned object");
            return;
        }
        
        if (IsActive) return;
        
        IsActive = true;
        OnActivated?.Invoke();
    }
    
    public void Deactivate()
    {
        if (!IsActive) return;
        
        IsActive = false;
        OnDeactivated?.Invoke();
    }
    
    public void Despawn()
    {
        if (!IsSpawned) return;
        
        // Deactivate before despawn
        if (IsActive)
            Deactivate();
            
        IsSpawned = false;
        OnDespawned?.Invoke();
    }
}
```

## Best Practices

1. **Check State First** – Always check IsActive before operations
2. **Idempotent Methods** – Handle multiple activation calls safely
3. **Event Order** – Trigger events after state changes
4. **Nested Activation** – Deactivate children when parent deactivates
5. **Performance** – Use activation to skip expensive operations

## Common Patterns

### LOD Activation
```csharp
// Activate based on distance
public void UpdateLOD(float distance)
{
    if (distance < nearDistance)
        highDetailObject.Activate();
    else
        highDetailObject.Deactivate();
}
```

### Pause System
```csharp
// Global pause using activation
public static void PauseGame()
{
    foreach (var obj in pausableObjects)
        obj.Deactivate();
}

public static void ResumeGame()
{
    foreach (var obj in pausableObjects)
        obj.Activate();
}
```

### Performance Optimization
```csharp
// Only process active objects
foreach (var obj in objects)
{
    if (obj.IsActive)
    {
        ProcessObject(obj);
    }
}
```

## Notes

- IActivatable is often combined with ISpawnable for full lifecycle
- Activation state is independent of spawn state
- Useful for pause systems, LOD, and culling
- Consider the relationship between Unity's GameObject.SetActive and IActivatable