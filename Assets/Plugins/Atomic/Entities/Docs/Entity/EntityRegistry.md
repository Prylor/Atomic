# 🧩 EntityRegistry

`EntityRegistry` is a global singleton registry that tracks and manages all `IEntity` instances in the application. It provides unique ID assignment, entity lookup, and serves as a central repository for all entities.

## Key Features

- **Singleton Pattern** – Single global instance for all entities
- **Unique ID Management** – Automatic ID assignment and recycling
- **Entity Tracking** – Central repository for all active entities
- **Lookup Utilities** – Find entities by ID
- **Event Notifications** – Track entity registration/unregistration
- **Memory Efficient** – ID recycling for long-running applications

---

## Class Definition

```csharp
public sealed class EntityRegistry : IReadOnlyEntityCollection<IEntity>
{
    public static EntityRegistry Instance { get; }
    
    public event Action OnStateChanged;
    public event Action<IEntity> OnAdded;
    public event Action<IEntity> OnRemoved;
    
    public int Count { get; }
    
    // Internal registration (called by Entity constructor)
    internal void Register(IEntity entity, out int id);
    internal void Unregister(ref int id);
    
    // Public lookup methods
    public bool Contains(int id);
    public bool Contains(IEntity entity);
    public bool TryGet(int id, out IEntity entity);
    public IEntity Get(int id);
}
```

## Singleton Access

```csharp
EntityRegistry Instance { get; }
```
- **Access**: Global singleton instance
- **Thread Safety**: Not thread-safe
- **Lifetime**: Application lifetime

## Events

### OnAdded
```csharp
event Action<IEntity> OnAdded;
```
- **Triggered**: When entity is registered
- **Usage**: Track new entities, update UI

### OnRemoved
```csharp
event Action<IEntity> OnRemoved;
```
- **Triggered**: When entity is unregistered
- **Usage**: Cleanup references, update counts

### OnStateChanged
```csharp
event Action OnStateChanged;
```
- **Triggered**: When registry state changes
- **Usage**: General state change notifications

## Core Methods

### Contains
```csharp
bool Contains(int id)
bool Contains(IEntity entity)
```
- **Description**: Check if entity/ID exists in registry
- **Returns**: `true` if found

### TryGet
```csharp
bool TryGet(int id, out IEntity entity)
```
- **Description**: Safe entity lookup by ID
- **Returns**: `true` if found, entity in out parameter

### Get
```csharp
IEntity Get(int id)
```
- **Description**: Direct entity lookup by ID
- **Throws**: Exception if ID not found

## Internal Methods

### Register
```csharp
internal void Register(IEntity entity, out int id)
```
- **Access**: Internal only (called by Entity constructor)
- **Description**: Registers entity and assigns unique ID
- **ID Recycling**: Reuses IDs from disposed entities

### Unregister
```csharp
internal void Unregister(ref int id)
```
- **Access**: Internal only (called by Entity.Dispose)
- **Description**: Removes entity from registry
- **ID Recycling**: Returns ID to pool for reuse

## Usage Examples

### Entity Lookup

```csharp
// Find entity by ID
int playerId = 42;
if (EntityRegistry.Instance.TryGet(playerId, out IEntity player))
{
    Debug.Log($"Found player: {player.Name}");
}

// Direct lookup (throws if not found)
try
{
    IEntity enemy = EntityRegistry.Instance.Get(enemyId);
    ProcessEnemy(enemy);
}
catch (KeyNotFoundException)
{
    Debug.LogError($"Entity {enemyId} not found");
}

// Check existence
if (EntityRegistry.Instance.Contains(entityId))
{
    // Entity exists
}
```

### Monitoring Registry

```csharp
public class EntityMonitor
{
    private int entityCount;
    private HashSet<int> activeIds = new();
    
    public void Initialize()
    {
        var registry = EntityRegistry.Instance;
        
        // Track additions
        registry.OnAdded += OnEntityAdded;
        registry.OnRemoved += OnEntityRemoved;
        
        // Initial count
        entityCount = registry.Count;
    }
    
    private void OnEntityAdded(IEntity entity)
    {
        entityCount++;
        activeIds.Add(entity.InstanceID);
        Debug.Log($"Entity registered: {entity.Name} (ID: {entity.InstanceID})");
        UpdateUI();
    }
    
    private void OnEntityRemoved(IEntity entity)
    {
        entityCount--;
        activeIds.Remove(entity.InstanceID);
        Debug.Log($"Entity unregistered: {entity.Name}");
        UpdateUI();
    }
    
    private void UpdateUI()
    {
        UIManager.SetEntityCount(entityCount);
    }
}
```

### Global Entity Queries

```csharp
public static class EntityQueries
{
    public static List<IEntity> FindByName(string name)
    {
        var results = new List<IEntity>();
        foreach (var entity in EntityRegistry.Instance)
        {
            if (entity.Name == name)
                results.Add(entity);
        }
        return results;
    }
    
    public static List<IEntity> FindWithTag(int tag)
    {
        var results = new List<IEntity>();
        foreach (var entity in EntityRegistry.Instance)
        {
            if (entity.HasTag(tag))
                results.Add(entity);
        }
        return results;
    }
    
    public static IEntity FindFirst(Predicate<IEntity> predicate)
    {
        foreach (var entity in EntityRegistry.Instance)
        {
            if (predicate(entity))
                return entity;
        }
        return null;
    }
    
    public static void ProcessAll(Action<IEntity> processor)
    {
        foreach (var entity in EntityRegistry.Instance)
        {
            processor(entity);
        }
    }
}
```

### ID Management

```csharp
public class EntityIdManager
{
    private Dictionary<string, int> namedEntities = new();
    
    public void RegisterNamed(string name, IEntity entity)
    {
        namedEntities[name] = entity.InstanceID;
    }
    
    public IEntity GetByName(string name)
    {
        if (namedEntities.TryGetValue(name, out int id))
        {
            if (EntityRegistry.Instance.TryGet(id, out IEntity entity))
                return entity;
        }
        return null;
    }
    
    public void ValidateIds()
    {
        var toRemove = new List<string>();
        
        foreach (var kvp in namedEntities)
        {
            if (!EntityRegistry.Instance.Contains(kvp.Value))
            {
                toRemove.Add(kvp.Key);
            }
        }
        
        foreach (var name in toRemove)
        {
            namedEntities.Remove(name);
            Debug.Log($"Removed invalid entity reference: {name}");
        }
    }
}
```

### Procedural Registry Operations

Following Atomic's procedural pattern:

```csharp
public static class RegistryUtils
{
    public static int GetEntityCount()
    {
        return EntityRegistry.Instance.Count;
    }
    
    public static List<IEntity> GetAllEntities()
    {
        var entities = new List<IEntity>();
        EntityRegistry.Instance.CopyTo(entities);
        return entities;
    }
    
    public static void DisposeAllEntities()
    {
        var entities = GetAllEntities();
        foreach (var entity in entities)
        {
            if (entity is IDisposable disposable)
            {
                disposable.Dispose();
            }
        }
    }
    
    public static Dictionary<int, string> CreateEntityMap()
    {
        var map = new Dictionary<int, string>();
        foreach (var entity in EntityRegistry.Instance)
        {
            map[entity.InstanceID] = entity.Name;
        }
        return map;
    }
    
    public static void LogRegistryState()
    {
        Debug.Log($"=== Entity Registry State ===");
        Debug.Log($"Total Entities: {EntityRegistry.Instance.Count}");
        
        foreach (var entity in EntityRegistry.Instance)
        {
            Debug.Log($"  [{entity.InstanceID}] {entity.Name} - " +
                     $"Tags: {entity.TagCount}, Values: {entity.ValueCount}, " +
                     $"Behaviours: {entity.BehaviourCount}");
        }
    }
}
```

## ID Recycling System

The registry implements ID recycling to prevent ID exhaustion:

```csharp
// How it works internally:
private readonly Stack<int> _recycledIds = new();
private int _lastId;

// When registering:
if (!_recycledIds.TryPop(out id)) 
    id = ++_lastId;  // New ID if no recycled ones

// When unregistering:
_recycledIds.Push(id);  // Return ID to pool
```

Benefits:
- Prevents integer overflow in long-running applications
- Maintains compact ID ranges
- Efficient memory usage

## Unity Editor Integration

```csharp
#if UNITY_EDITOR
[InitializeOnEnterPlayMode]
internal static void ResetAll()
{
    // Clears registry when entering play mode
    if (_instance != null)
        _instance.Clear();
}
#endif
```

This ensures clean state when testing in Unity Editor.

## Best Practices

1. **Don't Store IDs Long-term** – Entity IDs are recycled
2. **Use TryGet** – Safer than Get for lookups
3. **Unsubscribe Events** – Prevent memory leaks
4. **Validate References** – Check if IDs still exist
5. **Avoid Direct Access** – Registry is internal to Entity system

## Performance Considerations

- **O(1) Lookup** – Dictionary-based storage
- **O(n) Iteration** – Linear time for enumeration
- **Memory Overhead** – ~40 bytes per entity entry
- **ID Recycling** – Stack operations are O(1)

## Common Patterns

### Entity Cache
```csharp
public class EntityCache
{
    private Dictionary<int, WeakReference> cache = new();
    
    public IEntity GetCached(int id)
    {
        if (cache.TryGetValue(id, out var weakRef) && 
            weakRef.IsAlive)
        {
            return (IEntity)weakRef.Target;
        }
        
        if (EntityRegistry.Instance.TryGet(id, out var entity))
        {
            cache[id] = new WeakReference(entity);
            return entity;
        }
        
        return null;
    }
}
```

### Registry Statistics
```csharp
public class RegistryStats
{
    public int TotalCreated { get; private set; }
    public int TotalDestroyed { get; private set; }
    public int CurrentActive => EntityRegistry.Instance.Count;
    public float ChurnRate => (float)TotalDestroyed / TotalCreated;
    
    public void Track()
    {
        EntityRegistry.Instance.OnAdded += _ => TotalCreated++;
        EntityRegistry.Instance.OnRemoved += _ => TotalDestroyed++;
    }
}
```

## Notes

- Registry is automatically managed by Entity class
- Not meant for direct manipulation
- Singleton pattern ensures global access
- ID recycling prevents exhaustion
- Unity Editor resets registry on play mode entry