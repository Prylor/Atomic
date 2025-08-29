# 🧩 IEntityWorld

`IEntityWorld` is the core interface for managing collections of entities with comprehensive lifecycle support in the Atomic framework. It combines entity collection management with spawning, activation, and update coordination, providing the foundation for sophisticated entity management systems.

## Key Features

- **Entity Collection Management** – Inherits full `IEntityCollection<E>` functionality
- **Lifecycle Coordination** – Implements `ISpawnable`, `IActivatable`, `IUpdatable`
- **Generic Type Safety** – Strongly typed with `IEntityWorld<E>`
- **Non-Generic Support** – Base `IEntityWorld` for heterogeneous scenarios  
- **World Identification** – Named worlds for organization and debugging
- **Procedural Integration** – Designed for Atomic's procedural patterns
- **Event-Driven Architecture** – Reactive lifecycle notifications

---

## Interface Definition

### Generic World Interface
```csharp
public interface IEntityWorld<E> : IEntityCollection<E>, ISpawnable, IActivatable, IUpdatable
    where E : IEntity
{
    string Name { get; set; }
}
```

### Non-Generic World Interface
```csharp
public interface IEntityWorld : IEntityWorld<IEntity>
{
}
```

## Inherited Interfaces

### IEntityCollection<E>
```csharp
// Entity management
int Count { get; }
bool Add(E entity);
bool Remove(E entity);
bool Contains(E entity);
void Clear();

// Events
event Action<E> OnAdded;
event Action<E> OnRemoved;
event Action OnStateChanged;

// Enumeration
IEnumerator<E> GetEnumerator();
```

### ISpawnable
```csharp
bool IsSpawned { get; }
event Action OnSpawned;
event Action OnDespawned;

void Spawn();
void Despawn();
```

### IActivatable
```csharp
bool IsActive { get; }
event Action OnActivated;
event Action OnDeactivated;

void Activate();
void Deactivate();
```

### IUpdatable
```csharp
event Action<float> OnUpdated;
event Action<float> OnFixedUpdated;
event Action<float> OnLateUpdated;

void OnUpdate(float deltaTime);
void OnFixedUpdate(float deltaTime);
void OnLateUpdate(float deltaTime);
```

## Properties

### Name
```csharp
string Name { get; set; }
```
- Identifies the world for debugging and organization
- Can be used for world lookup and management
- Mutable property that can be changed at runtime

## Implementation Requirements

When implementing `IEntityWorld<E>`, ensure:

1. **Lifecycle Propagation** – All lifecycle methods propagate to contained entities
2. **State Coordination** – World state changes affect all entities appropriately
3. **Event Aggregation** – World events aggregate entity-level events
4. **Collection Synchronization** – Entity additions/removals respect world state
5. **Proper Disposal** – Clean up all resources and references

## Usage Patterns

### World Management System
```csharp
public class WorldManager
{
    private readonly Dictionary<string, IEntityWorld> worlds;
    
    public WorldManager()
    {
        worlds = new Dictionary<string, IEntityWorld>();
    }
    
    public void RegisterWorld(IEntityWorld world)
    {
        if (string.IsNullOrEmpty(world.Name))
        {
            world.Name = $"World_{worlds.Count}";
        }
        
        worlds[world.Name] = world;
        
        // Subscribe to world events
        world.OnSpawned += () => OnWorldSpawned(world);
        world.OnDespawned += () => OnWorldDespawned(world);
    }
    
    public IEntityWorld GetWorld(string name)
    {
        return worlds.TryGetValue(name, out var world) ? world : null;
    }
    
    public void UpdateAll(float deltaTime)
    {
        foreach (var world in worlds.Values)
        {
            if (world.IsActive)
            {
                world.OnUpdate(deltaTime);
            }
        }
    }
    
    public void SpawnAll()
    {
        foreach (var world in worlds.Values)
        {
            world.Spawn();
        }
    }
    
    private void OnWorldSpawned(IEntityWorld world)
    {
        Debug.Log($"World '{world.Name}' spawned with {world.Count} entities");
    }
    
    private void OnWorldDespawned(IEntityWorld world)
    {
        Debug.Log($"World '{world.Name}' despawned");
    }
}
```

### Layered World System
```csharp
public class GameLayerManager
{
    private readonly IEntityWorld backgroundWorld;
    private readonly IEntityWorld gameplayWorld;
    private readonly IEntityWorld uiWorld;
    
    public GameLayerManager(
        IEntityWorld background,
        IEntityWorld gameplay,
        IEntityWorld ui)
    {
        backgroundWorld = background;
        gameplayWorld = gameplay;
        uiWorld = ui;
        
        // Set up layer names
        backgroundWorld.Name = "Background";
        gameplayWorld.Name = "Gameplay";
        uiWorld.Name = "UI";
    }
    
    public void StartGame()
    {
        // Spawn worlds in order
        backgroundWorld.Spawn();
        gameplayWorld.Spawn();
        uiWorld.Spawn();
    }
    
    public void PauseGame()
    {
        // Deactivate gameplay but keep UI active
        gameplayWorld.Deactivate();
    }
    
    public void ResumeGame()
    {
        gameplayWorld.Activate();
    }
    
    public void UpdateGame(float deltaTime)
    {
        // Update all active worlds
        if (backgroundWorld.IsActive)
            backgroundWorld.OnUpdate(deltaTime);
            
        if (gameplayWorld.IsActive)
            gameplayWorld.OnUpdate(deltaTime);
            
        if (uiWorld.IsActive)
            uiWorld.OnUpdate(deltaTime);
    }
    
    public void StopGame()
    {
        // Despawn in reverse order
        uiWorld.Despawn();
        gameplayWorld.Despawn();
        backgroundWorld.Despawn();
    }
}
```

### Dynamic World Creation
```csharp
public class DynamicWorldSystem
{
    private readonly IEntityFactory<Entity> entityFactory;
    private readonly Dictionary<string, IEntityWorld<Entity>> activeWorlds;
    
    public DynamicWorldSystem(IEntityFactory<Entity> factory)
    {
        entityFactory = factory;
        activeWorlds = new Dictionary<string, IEntityWorld<Entity>>();
    }
    
    public IEntityWorld<Entity> CreateWorld(string worldName, int entityCount = 0)
    {
        // Create world implementation (could be injected)
        var world = new EntityWorld<Entity>(worldName);
        
        // Populate with entities if requested
        for (int i = 0; i < entityCount; i++)
        {
            var entity = entityFactory.Create();
            world.Add(entity);
        }
        
        // Register and activate
        activeWorlds[worldName] = world;
        world.Spawn();
        
        return world;
    }
    
    public void DestroyWorld(string worldName)
    {
        if (activeWorlds.TryGetValue(worldName, out var world))
        {
            world.Despawn();
            world.Dispose();
            activeWorlds.Remove(worldName);
        }
    }
    
    public void TransferEntity(Entity entity, string fromWorld, string toWorld)
    {
        var source = activeWorlds.GetValueOrDefault(fromWorld);
        var target = activeWorlds.GetValueOrDefault(toWorld);
        
        if (source != null && target != null && source.Contains(entity))
        {
            source.Remove(entity);
            target.Add(entity);
        }
    }
}
```

### World State Machine
```csharp
public class StatefulWorld : IEntityWorld<Entity>
{
    public enum WorldState
    {
        Uninitialized,
        Initialized,
        Spawned,
        Active,
        Paused,
        Despawned
    }
    
    private readonly IEntityWorld<Entity> implementation;
    private WorldState currentState;
    
    public WorldState State => currentState;
    public string Name { get; set; }
    
    // Delegate all IEntityWorld implementation
    public int Count => implementation.Count;
    public bool IsSpawned => implementation.IsSpawned;
    public bool IsActive => implementation.IsActive;
    
    public event Action<E> OnAdded
    {
        add => implementation.OnAdded += value;
        remove => implementation.OnAdded -= value;
    }
    
    // ... other event delegations
    
    public StatefulWorld(IEntityWorld<Entity> impl, string name)
    {
        implementation = impl;
        Name = name;
        currentState = WorldState.Initialized;
        
        // Subscribe to state changes
        implementation.OnSpawned += () => ChangeState(WorldState.Spawned);
        implementation.OnActivated += () => ChangeState(WorldState.Active);
        implementation.OnDeactivated += () => ChangeState(WorldState.Paused);
        implementation.OnDespawned += () => ChangeState(WorldState.Despawned);
    }
    
    public void Spawn()
    {
        if (currentState == WorldState.Initialized || currentState == WorldState.Despawned)
        {
            implementation.Spawn();
        }
    }
    
    public void Activate()
    {
        if (currentState == WorldState.Spawned || currentState == WorldState.Paused)
        {
            implementation.Activate();
        }
    }
    
    // ... other method delegations with state validation
    
    private void ChangeState(WorldState newState)
    {
        var oldState = currentState;
        currentState = newState;
        OnStateChanged?.Invoke(oldState, newState);
    }
    
    public event Action<WorldState, WorldState> OnStateChanged;
}
```

### Procedural World Operations
```csharp
public static class WorldOperations
{
    public static void PopulateWorld<E>(IEntityWorld<E> world, 
                                      IEntityFactory<E> factory, 
                                      int count) 
        where E : IEntity
    {
        for (int i = 0; i < count; i++)
        {
            var entity = factory.Create();
            world.Add(entity);
        }
    }
    
    public static void ClearAndRespawn<E>(IEntityWorld<E> world) 
        where E : IEntity
    {
        var wasActive = world.IsActive;
        
        if (world.IsSpawned)
        {
            world.Despawn();
        }
        
        world.Clear();
        world.Spawn();
        
        if (wasActive)
        {
            world.Activate();
        }
    }
    
    public static void TransferEntities<E>(IEntityWorld<E> source, 
                                         IEntityWorld<E> target,
                                         Predicate<E> predicate = null) 
        where E : IEntity
    {
        var entitiesToTransfer = new List<E>();
        
        foreach (var entity in source)
        {
            if (predicate == null || predicate(entity))
            {
                entitiesToTransfer.Add(entity);
            }
        }
        
        foreach (var entity in entitiesToTransfer)
        {
            source.Remove(entity);
            target.Add(entity);
        }
    }
    
    public static void SynchronizeWorlds<E>(IEntityWorld<E> world1, 
                                          IEntityWorld<E> world2) 
        where E : IEntity
    {
        // Synchronize spawn state
        if (world1.IsSpawned && !world2.IsSpawned)
            world2.Spawn();
        else if (!world1.IsSpawned && world2.IsSpawned)
            world2.Despawn();
        
        // Synchronize active state
        if (world1.IsActive && !world2.IsActive)
            world2.Activate();
        else if (!world1.IsActive && world2.IsActive)
            world2.Deactivate();
    }
    
    public static IEntityWorld<E> CreateMirrorWorld<E>(IEntityWorld<E> source,
                                                     string mirrorName) 
        where E : IEntity
    {
        var mirror = new EntityWorld<E>(mirrorName);
        
        // Copy all entities
        foreach (var entity in source)
        {
            mirror.Add(entity);
        }
        
        // Synchronize state
        SynchronizeWorlds(source, mirror);
        
        return mirror;
    }
    
    public static void BatchUpdate(IEnumerable<IEntityWorld> worlds, float deltaTime)
    {
        foreach (var world in worlds)
        {
            if (world.IsActive)
            {
                world.OnUpdate(deltaTime);
            }
        }
    }
    
    public static int GetTotalEntityCount(IEnumerable<IEntityWorld> worlds)
    {
        return worlds.Sum(world => world.Count);
    }
}
```

### World Composition Pattern
```csharp
public class CompositeWorld : IEntityWorld<Entity>
{
    private readonly List<IEntityWorld<Entity>> childWorlds;
    
    public string Name { get; set; }
    public int Count => childWorlds.Sum(w => w.Count);
    public bool IsSpawned => childWorlds.Any(w => w.IsSpawned);
    public bool IsActive => childWorlds.Any(w => w.IsActive);
    
    public event Action<Entity> OnAdded;
    public event Action<Entity> OnRemoved;
    public event Action OnStateChanged;
    public event Action OnSpawned;
    public event Action OnDespawned;
    public event Action OnActivated;
    public event Action OnDeactivated;
    public event Action<float> OnUpdated;
    public event Action<float> OnFixedUpdated;
    public event Action<float> OnLateUpdated;
    
    public CompositeWorld(string name, params IEntityWorld<Entity>[] worlds)
    {
        Name = name;
        childWorlds = new List<IEntityWorld<Entity>>(worlds);
        
        // Subscribe to child events
        foreach (var world in childWorlds)
        {
            world.OnAdded += entity => OnAdded?.Invoke(entity);
            world.OnRemoved += entity => OnRemoved?.Invoke(entity);
            world.OnStateChanged += () => OnStateChanged?.Invoke();
        }
    }
    
    public void AddChildWorld(IEntityWorld<Entity> world)
    {
        childWorlds.Add(world);
        // Subscribe to events...
    }
    
    public void RemoveChildWorld(IEntityWorld<Entity> world)
    {
        childWorlds.Remove(world);
        // Unsubscribe from events...
    }
    
    public void Spawn()
    {
        foreach (var world in childWorlds)
        {
            world.Spawn();
        }
        OnSpawned?.Invoke();
    }
    
    public void Despawn()
    {
        foreach (var world in childWorlds)
        {
            world.Despawn();
        }
        OnDespawned?.Invoke();
    }
    
    public void Activate()
    {
        foreach (var world in childWorlds)
        {
            world.Activate();
        }
        OnActivated?.Invoke();
    }
    
    public void Deactivate()
    {
        foreach (var world in childWorlds)
        {
            world.Deactivate();
        }
        OnDeactivated?.Invoke();
    }
    
    public void OnUpdate(float deltaTime)
    {
        foreach (var world in childWorlds)
        {
            if (world.IsActive)
            {
                world.OnUpdate(deltaTime);
            }
        }
        OnUpdated?.Invoke(deltaTime);
    }
    
    // Implement other IEntityCollection methods by delegating to first appropriate world
    // or throwing NotSupportedException for operations that don't make sense on composite
}
```

## Integration with Atomic Systems

### With Entity Filters
```csharp
public class FilteredWorldSystem
{
    private readonly IEntityWorld<Entity> sourceWorld;
    private readonly Dictionary<string, EntityFilter> filters;
    
    public FilteredWorldSystem(IEntityWorld<Entity> world)
    {
        sourceWorld = world;
        filters = new Dictionary<string, EntityFilter>();
    }
    
    public EntityFilter CreateFilter(string name, Predicate<Entity> predicate)
    {
        var filter = new EntityFilter(sourceWorld, predicate);
        filters[name] = filter;
        return filter;
    }
    
    public void UpdateFilters()
    {
        foreach (var filter in filters.Values)
        {
            filter.Refresh();
        }
    }
}
```

### With Entity Pools
```csharp
public class PooledWorldSystem
{
    private readonly IEntityWorld<Entity> world;
    private readonly Dictionary<string, EntityPool<Entity>> pools;
    
    public PooledWorldSystem(IEntityWorld<Entity> world)
    {
        this.world = world;
        pools = new Dictionary<string, EntityPool<Entity>>();
    }
    
    public Entity SpawnFromPool(string poolKey)
    {
        if (pools.TryGetValue(poolKey, out var pool))
        {
            var entity = pool.Rent();
            world.Add(entity);
            return entity;
        }
        return null;
    }
    
    public void ReturnToPool(string poolKey, Entity entity)
    {
        if (pools.TryGetValue(poolKey, out var pool))
        {
            world.Remove(entity);
            pool.Return(entity);
        }
    }
}
```

## Best Practices

1. **Name Your Worlds** – Always set meaningful names for debugging
2. **Manage Lifecycle Properly** – Ensure spawn/despawn and activate/deactivate are balanced
3. **Use Generic Versions** – Prefer `IEntityWorld<T>` for type safety
4. **Monitor Performance** – Large worlds can impact update performance
5. **Handle State Changes** – React appropriately to world lifecycle events
6. **Proper Disposal** – Clean up worlds when no longer needed

## Performance Considerations

- **Update Overhead** – All entities updated during world update cycles
- **Event Propagation** – Many entities firing events can impact performance
- **Collection Size** – Large entity collections affect iteration performance
- **State Synchronization** – Coordinating entity states adds overhead
- **Memory Usage** – Worlds maintain references to all contained entities

The `IEntityWorld` interface provides comprehensive entity management with lifecycle coordination, making it the cornerstone of sophisticated entity systems in the Atomic framework.