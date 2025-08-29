# 🧩 EntityWorld

`EntityWorld` is the concrete implementation of `IEntityWorld` in the Atomic framework. It provides comprehensive entity collection management with full lifecycle support, making it the primary choice for managing groups of entities with coordinated spawning, activation, and update cycles.

## Key Features

- **Complete Implementation** – Full `IEntityWorld<E>` interface implementation
- **Entity Collection Base** – Extends `EntityCollection<E>` for efficient storage
- **Lifecycle Coordination** – Automatic entity state synchronization
- **Generic Type Safety** – Strongly typed with `EntityWorld<E>`
- **Non-Generic Convenience** – Base `EntityWorld` for general use
- **Event Aggregation** – Comprehensive event system for reactive programming
- **Performance Optimized** – Aggressive inlining and efficient iteration
- **Pre-allocation Support** – Constructor overloads for capacity planning

---

## Class Definition

### Generic EntityWorld
```csharp
public class EntityWorld<E> : EntityCollection<E>, IEntityWorld<E> where E : IEntity
{
    // Lifecycle events
    public event Action OnSpawned;
    public event Action OnDespawned;
    public event Action OnActivated;
    public event Action OnDeactivated;
    public event Action<float> OnUpdated;
    public event Action<float> OnFixedUpdated;
    public event Action<float> OnLateUpdated;
    
    // State properties
    public bool IsSpawned { get; }
    public bool IsActive { get; }
    public string Name { get; set; }
    
    // Lifecycle methods
    public void Spawn();
    public void Despawn();
    public void Activate();
    public void Deactivate();
    public void OnUpdate(float deltaTime);
    public void OnFixedUpdate(float deltaTime);
    public void OnLateUpdate(float deltaTime);
}
```

### Non-Generic EntityWorld
```csharp
public class EntityWorld : EntityWorld<IEntity>, IEntityWorld
{
    public EntityWorld();
    public EntityWorld(params IEntity[] entities);
    public EntityWorld(string name = null, params IEntity[] entities);
    public EntityWorld(string name, IEnumerable<IEntity> entities);
}
```

## Constructors

### Default Constructor
```csharp
public EntityWorld()
```
- Creates empty world with no name
- No entities pre-allocated
- Ready for dynamic entity addition

### Entity Array Constructor
```csharp
public EntityWorld(params E[] entities)
public EntityWorld(string name = null, params E[] entities)
```
- **entities**: Initial entities to add to world
- **name**: Optional world name (empty string if null)
- Entities added during construction

### Entity Collection Constructor
```csharp
public EntityWorld(string name, IEnumerable<E> entities)
```
- **name**: World identifier name
- **entities**: Collection of entities to add
- Most flexible constructor for large collections

## Lifecycle Methods

### Spawn
```csharp
public void Spawn()
```
- Transitions world to spawned state
- Calls `Spawn()` on all contained entities
- Sets `IsSpawned` to true
- Triggers `OnSpawned` event
- Logs warning if already spawned

### Despawn
```csharp
public void Despawn()
```
- Deactivates world if currently active
- Calls `Despawn()` on all contained entities
- Sets `IsSpawned` to false
- Triggers `OnDespawned` event
- Safe to call if not spawned

### Activate
```csharp
public void Activate()
```
- Automatically spawns if not already spawned
- Calls `Activate()` on all contained entities
- Sets `IsActive` to true
- Triggers `OnActivated` event
- Logs warning if already active

### Deactivate
```csharp
public void Deactivate()
```
- Calls `Deactivate()` on all contained entities
- Sets `IsActive` to false
- Triggers `OnDeactivated` event
- Logs warning if not active

### Update Methods
```csharp
public void OnUpdate(float deltaTime)
public void OnFixedUpdate(float deltaTime)
public void OnLateUpdate(float deltaTime)
```
- Only processes if world is active
- Calls corresponding method on all entities
- Triggers respective `On*Updated` events
- Logs warning if world not enabled

## Protected Virtual Methods

### ProcessSpawn
```csharp
protected virtual void ProcessSpawn()
```
- Handles entity spawning logic
- Can be overridden for custom spawn behavior
- Called during `Spawn()`

### ProcessDespawn
```csharp
protected virtual void ProcessDespawn()
```
- Handles entity despawning logic
- Can be overridden for custom despawn behavior
- Called during `Despawn()`

### ProcessActivate
```csharp
protected virtual void ProcessActivate()
```
- Handles entity activation logic
- Can be overridden for custom activation behavior
- Called during `Activate()`

### ProcessDeactivate
```csharp
protected virtual void ProcessDeactivate()
```
- Handles entity deactivation logic
- Can be overridden for custom deactivation behavior
- Called during `Deactivate()`

### Process Update Methods
```csharp
protected virtual void ProcessUpdate(float deltaTime)
protected virtual void ProcessFixedUpdate(float deltaTime)
protected virtual void ProcessLateUpdate(float deltaTime)
```
- Handle entity update logic
- Can be overridden for custom update behavior
- Called during respective update methods

## Entity Addition/Removal Behavior

### OnAdd Override
```csharp
protected override void OnAdd(E entity)
```
- If world is spawned, automatically spawns the entity
- If world is active, automatically activates the entity
- Ensures new entities match world state

### OnRemove Override
```csharp
protected override void OnRemove(E entity)
```
- If world is active, automatically deactivates the entity
- If world is spawned, automatically despawns the entity
- Ensures removed entities are properly cleaned up

## Implementation Examples

### Basic Game World
```csharp
public class GameWorld : EntityWorld<Entity>
{
    private readonly IEntityFactory<Entity> playerFactory;
    private readonly IEntityFactory<Entity> enemyFactory;
    
    public GameWorld(string name, 
                    IEntityFactory<Entity> playerFactory, 
                    IEntityFactory<Entity> enemyFactory) 
        : base(name)
    {
        this.playerFactory = playerFactory;
        this.enemyFactory = enemyFactory;
    }
    
    public Entity SpawnPlayer(Vector3 position)
    {
        var player = playerFactory.Create();
        player.SetValue(EntityNames.POSITION, position);
        Add(player);
        return player;
    }
    
    public Entity SpawnEnemy(Vector3 position)
    {
        var enemy = enemyFactory.Create();
        enemy.SetValue(EntityNames.POSITION, position);
        Add(enemy);
        return enemy;
    }
    
    protected override void ProcessUpdate(float deltaTime)
    {
        base.ProcessUpdate(deltaTime);
        
        // Additional world-specific update logic
        CheckCollisions();
        CleanupDeadEntities();
    }
    
    private void CheckCollisions()
    {
        // Custom collision detection logic
    }
    
    private void CleanupDeadEntities()
    {
        var deadEntities = new List<Entity>();
        
        foreach (var entity in this)
        {
            if (entity.HasTag(EntityTags.DEAD))
            {
                deadEntities.Add(entity);
            }
        }
        
        foreach (var deadEntity in deadEntities)
        {
            Remove(deadEntity);
            deadEntity.Dispose();
        }
    }
}
```

### Layered World System
```csharp
public class LayeredGameWorld
{
    private readonly EntityWorld<Entity> backgroundWorld;
    private readonly EntityWorld<Entity> gameplayWorld;
    private readonly EntityWorld<Entity> effectsWorld;
    private readonly EntityWorld<Entity> uiWorld;
    
    public LayeredGameWorld()
    {
        backgroundWorld = new EntityWorld<Entity>("Background");
        gameplayWorld = new EntityWorld<Entity>("Gameplay");
        effectsWorld = new EntityWorld<Entity>("Effects");
        uiWorld = new EntityWorld<Entity>("UI");
    }
    
    public void StartGame()
    {
        // Spawn worlds in layer order
        backgroundWorld.Spawn();
        gameplayWorld.Spawn();
        effectsWorld.Spawn();
        uiWorld.Spawn();
    }
    
    public void UpdateGame(float deltaTime)
    {
        // Update in layer order
        backgroundWorld.OnUpdate(deltaTime);
        gameplayWorld.OnUpdate(deltaTime);
        effectsWorld.OnUpdate(deltaTime);
        uiWorld.OnUpdate(deltaTime);
    }
    
    public void PauseGame()
    {
        // Only pause gameplay, keep UI active
        gameplayWorld.Deactivate();
        effectsWorld.Deactivate();
    }
    
    public void ResumeGame()
    {
        gameplayWorld.Activate();
        effectsWorld.Activate();
    }
    
    public void AddToLayer(LayerType layer, Entity entity)
    {
        GetWorldForLayer(layer).Add(entity);
    }
    
    private EntityWorld<Entity> GetWorldForLayer(LayerType layer)
    {
        return layer switch
        {
            LayerType.Background => backgroundWorld,
            LayerType.Gameplay => gameplayWorld,
            LayerType.Effects => effectsWorld,
            LayerType.UI => uiWorld,
            _ => gameplayWorld
        };
    }
}
```

### Level-Based World
```csharp
public class LevelWorld : EntityWorld<Entity>
{
    private readonly LevelData levelData;
    private readonly Dictionary<string, IEntityFactory<Entity>> factories;
    
    public int LevelNumber { get; }
    public bool IsComplete { get; private set; }
    
    public event Action<LevelWorld> OnLevelComplete;
    
    public LevelWorld(int levelNumber, LevelData data, 
                     Dictionary<string, IEntityFactory<Entity>> factories) 
        : base($"Level_{levelNumber}")
    {
        LevelNumber = levelNumber;
        levelData = data;
        this.factories = factories;
        
        LoadLevel();
    }
    
    private void LoadLevel()
    {
        // Spawn level entities from data
        foreach (var entityData in levelData.Entities)
        {
            if (factories.TryGetValue(entityData.Type, out var factory))
            {
                var entity = factory.Create();
                ConfigureEntity(entity, entityData);
                Add(entity);
            }
        }
    }
    
    private void ConfigureEntity(Entity entity, EntityData data)
    {
        entity.SetValue(EntityNames.POSITION, data.Position);
        
        foreach (var tag in data.Tags)
        {
            entity.AddTag(tag);
        }
        
        foreach (var kvp in data.Values)
        {
            entity.SetValue(kvp.Key, kvp.Value);
        }
    }
    
    protected override void ProcessUpdate(float deltaTime)
    {
        base.ProcessUpdate(deltaTime);
        
        CheckLevelCompletion();
    }
    
    private void CheckLevelCompletion()
    {
        if (!IsComplete && CheckCompletionConditions())
        {
            IsComplete = true;
            OnLevelComplete?.Invoke(this);
        }
    }
    
    private bool CheckCompletionConditions()
    {
        // Example: level complete when no enemies remain
        foreach (var entity in this)
        {
            if (entity.HasTag(EntityTags.ENEMY) && !entity.HasTag(EntityTags.DEAD))
            {
                return false;
            }
        }
        return true;
    }
}
```

### Pooled World System
```csharp
public class PooledEntityWorld : EntityWorld<Entity>
{
    private readonly Dictionary<string, EntityPool<Entity>> pools;
    private readonly Dictionary<Entity, string> entityPools;
    
    public PooledEntityWorld(string name) : base(name)
    {
        pools = new Dictionary<string, EntityPool<Entity>>();
        entityPools = new Dictionary<Entity, string>();
    }
    
    public void RegisterPool(string key, EntityPool<Entity> pool)
    {
        pools[key] = pool;
    }
    
    public Entity SpawnFromPool(string poolKey)
    {
        if (pools.TryGetValue(poolKey, out var pool))
        {
            var entity = pool.Rent();
            entityPools[entity] = poolKey;
            Add(entity);
            return entity;
        }
        return null;
    }
    
    public void ReturnToPool(Entity entity)
    {
        if (entityPools.TryGetValue(entity, out var poolKey) && 
            pools.TryGetValue(poolKey, out var pool))
        {
            Remove(entity);
            pool.Return(entity);
            entityPools.Remove(entity);
        }
    }
    
    protected override void OnRemove(Entity entity)
    {
        base.OnRemove(entity);
        
        // Automatically return to pool if removed
        if (entityPools.ContainsKey(entity))
        {
            ReturnToPool(entity);
        }
    }
    
    public override void Dispose()
    {
        // Return all entities to their pools before disposal
        var entitiesToReturn = new List<Entity>(entityPools.Keys);
        
        foreach (var entity in entitiesToReturn)
        {
            ReturnToPool(entity);
        }
        
        base.Dispose();
    }
}
```

### Performance-Optimized World
```csharp
public class HighPerformanceWorld : EntityWorld<Entity>
{
    private Entity[] cachedEntities;
    private int cachedCount;
    private bool cacheInvalid = true;
    
    public HighPerformanceWorld(string name, int capacity = 1000) : base(name)
    {
        cachedEntities = new Entity[capacity];
    }
    
    protected override void OnAdd(Entity entity)
    {
        base.OnAdd(entity);
        cacheInvalid = true;
    }
    
    protected override void OnRemove(Entity entity)
    {
        base.OnRemove(entity);
        cacheInvalid = true;
    }
    
    protected override void ProcessUpdate(float deltaTime)
    {
        // Use cached array for better performance
        UpdateCache();
        
        for (int i = 0; i < cachedCount; i++)
        {
            cachedEntities[i].OnUpdate(deltaTime);
        }
    }
    
    protected override void ProcessFixedUpdate(float deltaTime)
    {
        UpdateCache();
        
        for (int i = 0; i < cachedCount; i++)
        {
            cachedEntities[i].OnFixedUpdate(deltaTime);
        }
    }
    
    private void UpdateCache()
    {
        if (!cacheInvalid) return;
        
        cachedCount = 0;
        foreach (var entity in this)
        {
            if (cachedCount < cachedEntities.Length)
            {
                cachedEntities[cachedCount++] = entity;
            }
        }
        
        cacheInvalid = false;
    }
}
```

## Usage Patterns

### Game State Management
```csharp
public class GameStateManager
{
    private EntityWorld<Entity> currentWorld;
    private readonly Dictionary<GameState, EntityWorld<Entity>> stateWorlds;
    
    public GameStateManager()
    {
        stateWorlds = new Dictionary<GameState, EntityWorld<Entity>>();
    }
    
    public void RegisterState(GameState state, EntityWorld<Entity> world)
    {
        stateWorlds[state] = world;
    }
    
    public void TransitionToState(GameState newState)
    {
        // Deactivate current world
        if (currentWorld != null)
        {
            currentWorld.Deactivate();
        }
        
        // Activate new world
        if (stateWorlds.TryGetValue(newState, out var newWorld))
        {
            currentWorld = newWorld;
            currentWorld.Activate();
        }
    }
    
    public void UpdateCurrentWorld(float deltaTime)
    {
        currentWorld?.OnUpdate(deltaTime);
    }
}
```

### Procedural World Operations
```csharp
public static class WorldOperations
{
    public static EntityWorld<Entity> CreateDemoWorld(string name, int entityCount)
    {
        var world = new EntityWorld<Entity>(name);
        var factory = new BasicEntityFactory();
        
        for (int i = 0; i < entityCount; i++)
        {
            var entity = factory.Create();
            entity.Name = $"Entity_{i}";
            world.Add(entity);
        }
        
        return world;
    }
    
    public static void PopulateWorldWithFactory<T>(EntityWorld<T> world, 
                                                 IEntityFactory<T> factory, 
                                                 int count) where T : IEntity
    {
        for (int i = 0; i < count; i++)
        {
            var entity = factory.Create();
            world.Add(entity);
        }
    }
    
    public static void CloneWorldContents<T>(EntityWorld<T> source, 
                                           EntityWorld<T> target) where T : IEntity
    {
        foreach (var entity in source)
        {
            target.Add(entity);
        }
        
        // Synchronize states
        if (source.IsSpawned && !target.IsSpawned)
            target.Spawn();
        if (source.IsActive && !target.IsActive)
            target.Activate();
    }
    
    public static void MergeWorlds<T>(EntityWorld<T> world1, EntityWorld<T> world2) 
        where T : IEntity
    {
        // Move all entities from world2 to world1
        var entities = new List<T>(world2);
        
        world2.Clear();
        
        foreach (var entity in entities)
        {
            world1.Add(entity);
        }
        
        world2.Despawn();
        world2.Dispose();
    }
    
    public static int CountEntitiesWithTag<T>(EntityWorld<T> world, int tag) 
        where T : IEntity
    {
        int count = 0;
        foreach (var entity in world)
        {
            if (entity.HasTag(tag))
                count++;
        }
        return count;
    }
    
    public static void RemoveEntitiesWithTag<T>(EntityWorld<T> world, int tag) 
        where T : IEntity
    {
        var toRemove = new List<T>();
        
        foreach (var entity in world)
        {
            if (entity.HasTag(tag))
                toRemove.Add(entity);
        }
        
        foreach (var entity in toRemove)
        {
            world.Remove(entity);
        }
    }
}
```

### World Event System
```csharp
public class WorldEventSystem
{
    private readonly List<EntityWorld<Entity>> managedWorlds;
    private readonly Dictionary<string, Action<EntityWorld<Entity>>> eventHandlers;
    
    public WorldEventSystem()
    {
        managedWorlds = new List<EntityWorld<Entity>>();
        eventHandlers = new Dictionary<string, Action<EntityWorld<Entity>>>();
    }
    
    public void RegisterWorld(EntityWorld<Entity> world)
    {
        managedWorlds.Add(world);
        
        // Subscribe to world events
        world.OnSpawned += () => HandleWorldEvent("spawned", world);
        world.OnDespawned += () => HandleWorldEvent("despawned", world);
        world.OnActivated += () => HandleWorldEvent("activated", world);
        world.OnDeactivated += () => HandleWorldEvent("deactivated", world);
        world.OnStateChanged += () => HandleWorldEvent("state_changed", world);
    }
    
    public void RegisterEventHandler(string eventName, Action<EntityWorld<Entity>> handler)
    {
        if (eventHandlers.ContainsKey(eventName))
        {
            eventHandlers[eventName] += handler;
        }
        else
        {
            eventHandlers[eventName] = handler;
        }
    }
    
    private void HandleWorldEvent(string eventName, EntityWorld<Entity> world)
    {
        if (eventHandlers.TryGetValue(eventName, out var handler))
        {
            handler(world);
        }
    }
}
```

## Best Practices

1. **Set Meaningful Names** – Always provide descriptive world names
2. **Manage Lifecycle Properly** – Balance spawn/despawn and activate/deactivate calls
3. **Override Process Methods** – Customize behavior by overriding virtual process methods
4. **Monitor Performance** – Large worlds with many entities can impact performance
5. **Handle Events Appropriately** – Subscribe and unsubscribe from events properly
6. **Use Appropriate Constructor** – Choose constructor based on initialization needs

## Performance Considerations

- **Entity Count** – Performance scales with number of entities
- **Update Frequency** – Each update calls methods on all entities
- **Event Overhead** – Many event subscriptions can impact performance
- **Memory Usage** – Worlds maintain references to all entities
- **Garbage Collection** – Frequent entity addition/removal creates GC pressure

## Thread Safety

- EntityWorld is **NOT thread-safe**
- All operations should be performed on the main thread
- Use synchronization if accessing from multiple threads

## Integration with Atomic Systems

EntityWorld seamlessly integrates with all other Atomic systems:
- **Entity Factories** – For entity creation
- **Entity Pools** – For performance optimization
- **Entity Filters** – For entity querying
- **Entity Collections** – As base functionality
- **Scene Entities** – For Unity integration

The `EntityWorld` class provides a robust, high-performance foundation for entity management in Atomic's procedural architecture.