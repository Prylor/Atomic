# 🧩 ISpawnable

`ISpawnable` represents an object that can be spawned and despawned within a system. It defines the fundamental lifecycle contract for objects that can enter and exit the game world or simulation environment.

## Key Features

- **Spawn/Despawn Lifecycle** – Core object activation pattern
- **State Tracking** – Check if object is currently spawned
- **Event Notifications** – React to spawn/despawn transitions
- **Foundation Interface** – Base for entity lifecycle management

---

## Interface Definition

```csharp
public interface ISpawnable
{
    event Action OnSpawned;
    event Action OnDespawned;
    bool IsSpawned { get; }
    void Spawn();
    void Despawn();
}
```

## Events

### OnSpawned
```csharp
event Action OnSpawned;
```
- **Triggered**: After object successfully spawns
- **Usage**: Initialize dependent systems, start animations, play sounds

### OnDespawned
```csharp
event Action OnDespawned;
```
- **Triggered**: After object successfully despawns
- **Usage**: Cleanup resources, stop effects, notify managers

## Properties

### IsSpawned
```csharp
bool IsSpawned { get; }
```
- **Description**: Indicates if object is currently spawned
- **Returns**: `true` if spawned, `false` otherwise

## Methods

### Spawn
```csharp
void Spawn();
```
- **Description**: Spawns the object into the world
- **Behavior**: 
  - Prepares object for active use
  - Sets IsSpawned to true
  - Triggers OnSpawned event
- **Idempotent**: Should handle multiple calls safely

### Despawn
```csharp
void Despawn();
```
- **Description**: Removes object from active world
- **Behavior**:
  - Deactivates object
  - Cleans up state
  - Sets IsSpawned to false
  - Triggers OnDespawned event
- **Idempotent**: Should handle multiple calls safely

## Implementation Example

### Basic Implementation

```csharp
public class SpawnableObject : ISpawnable
{
    public event Action OnSpawned;
    public event Action OnDespawned;
    
    public bool IsSpawned { get; private set; }
    
    public void Spawn()
    {
        if (IsSpawned) return; // Idempotent
        
        IsSpawned = true;
        
        // Perform spawn logic
        Initialize();
        
        OnSpawned?.Invoke();
    }
    
    public void Despawn()
    {
        if (!IsSpawned) return; // Idempotent
        
        IsSpawned = false;
        
        // Perform despawn logic
        Cleanup();
        
        OnDespawned?.Invoke();
    }
    
    protected virtual void Initialize()
    {
        // Override in derived classes
    }
    
    protected virtual void Cleanup()
    {
        // Override in derived classes
    }
}
```

### Game Object Implementation

```csharp
public class Enemy : ISpawnable
{
    private GameObject gameObject;
    private Vector3 spawnPosition;
    
    public event Action OnSpawned;
    public event Action OnDespawned;
    public bool IsSpawned { get; private set; }
    
    public Enemy(GameObject prefab, Vector3 position)
    {
        gameObject = Object.Instantiate(prefab);
        gameObject.SetActive(false);
        spawnPosition = position;
    }
    
    public void Spawn()
    {
        if (IsSpawned) return;
        
        // Position and activate
        gameObject.transform.position = spawnPosition;
        gameObject.SetActive(true);
        
        // Reset state
        var health = gameObject.GetComponent<Health>();
        health?.Reset();
        
        IsSpawned = true;
        OnSpawned?.Invoke();
    }
    
    public void Despawn()
    {
        if (!IsSpawned) return;
        
        // Deactivate
        gameObject.SetActive(false);
        
        // Cleanup
        StopAllEffects();
        
        IsSpawned = false;
        OnDespawned?.Invoke();
    }
}
```

### Pooled Object

```csharp
public class PooledProjectile : ISpawnable
{
    private IObjectPool<PooledProjectile> pool;
    private Transform transform;
    private Rigidbody rigidbody;
    
    public event Action OnSpawned;
    public event Action OnDespawned;
    public bool IsSpawned { get; private set; }
    
    public void Initialize(IObjectPool<PooledProjectile> objectPool)
    {
        pool = objectPool;
    }
    
    public void Spawn()
    {
        if (IsSpawned) return;
        
        IsSpawned = true;
        
        // Reset physics
        rigidbody.velocity = Vector3.zero;
        rigidbody.angularVelocity = Vector3.zero;
        
        // Enable components
        transform.gameObject.SetActive(true);
        
        OnSpawned?.Invoke();
    }
    
    public void Despawn()
    {
        if (!IsSpawned) return;
        
        IsSpawned = false;
        
        // Disable
        transform.gameObject.SetActive(false);
        
        OnDespawned?.Invoke();
        
        // Return to pool
        pool?.Return(this);
    }
}
```

## Usage Patterns

### Spawn Manager

```csharp
public class SpawnManager
{
    private List<ISpawnable> activeObjects = new List<ISpawnable>();
    private Queue<ISpawnable> spawnQueue = new Queue<ISpawnable>();
    
    public void QueueSpawn(ISpawnable obj)
    {
        spawnQueue.Enqueue(obj);
    }
    
    public void ProcessSpawnQueue()
    {
        while (spawnQueue.Count > 0)
        {
            var obj = spawnQueue.Dequeue();
            SpawnObject(obj);
        }
    }
    
    private void SpawnObject(ISpawnable obj)
    {
        obj.Spawn();
        activeObjects.Add(obj);
        
        // Subscribe to despawn
        obj.OnDespawned += () => activeObjects.Remove(obj);
    }
    
    public void DespawnAll()
    {
        // Copy list to avoid modification during iteration
        var objects = activeObjects.ToArray();
        foreach (var obj in objects)
        {
            obj.Despawn();
        }
    }
}
```

### Lifecycle Coordination

```csharp
public class LifecycleCoordinator
{
    public static void TransitionToSpawned(ISpawnable spawnable, IActivatable activatable)
    {
        if (!spawnable.IsSpawned)
        {
            spawnable.Spawn();
        }
        
        if (!activatable.IsActive)
        {
            activatable.Activate();
        }
    }
    
    public static void TransitionToDespawned(ISpawnable spawnable, IActivatable activatable)
    {
        if (activatable.IsActive)
        {
            activatable.Deactivate();
        }
        
        if (spawnable.IsSpawned)
        {
            spawnable.Despawn();
        }
    }
}
```

### Procedural Spawn Operations

Following Atomic's procedural pattern:

```csharp
public static class SpawnOperations
{
    public static void SpawnWithDelay(ISpawnable obj, float delay)
    {
        Timer.Schedule(delay, () => obj.Spawn());
    }
    
    public static void SpawnAtPosition(ISpawnable obj, Vector3 position)
    {
        // Set position before spawning
        if (obj is IEntity entity)
        {
            entity.SetValue(EntityNames.SPAWN_POSITION, position);
        }
        obj.Spawn();
    }
    
    public static void SpawnBatch(IEnumerable<ISpawnable> objects)
    {
        foreach (var obj in objects)
        {
            obj.Spawn();
        }
    }
    
    public static void DespawnAfterTime(ISpawnable obj, float lifetime)
    {
        obj.Spawn();
        Timer.Schedule(lifetime, () => obj.Despawn());
    }
    
    public static void RespawnCycle(ISpawnable obj, float spawnTime, float despawnTime)
    {
        void Cycle()
        {
            obj.Spawn();
            Timer.Schedule(spawnTime, () =>
            {
                obj.Despawn();
                Timer.Schedule(despawnTime, Cycle);
            });
        }
        Cycle();
    }
}
```

## Best Practices

1. **Idempotency** – Spawn/Despawn should handle multiple calls safely
2. **State Consistency** – Always update IsSpawned correctly
3. **Event Order** – Trigger events after state changes complete
4. **Resource Management** – Clean up resources in Despawn
5. **Null Safety** – Check object validity before operations

## Common Patterns

### Spawn-Activate Pattern
```csharp
// Common lifecycle order
obj.Spawn();    // Enter world
obj.Activate();  // Enable updates
// ... active use ...
obj.Deactivate(); // Disable updates
obj.Despawn();    // Exit world
```

### Pooling Pattern
```csharp
// Objects return to pool on despawn
public void Despawn()
{
    // ... cleanup ...
    OnDespawned?.Invoke();
    ObjectPool.Return(this);
}
```

### Auto-Despawn Pattern
```csharp
// Automatic despawn after condition
public void Spawn()
{
    IsSpawned = true;
    StartCoroutine(AutoDespawnCoroutine());
    OnSpawned?.Invoke();
}

private IEnumerator AutoDespawnCoroutine()
{
    yield return new WaitForSeconds(lifetime);
    Despawn();
}
```

## Notes

- ISpawnable is often combined with IActivatable for full lifecycle
- Spawn typically happens once, while Activate/Deactivate may cycle
- Consider using object pools for frequently spawned objects
- Events allow external systems to react to lifecycle changes