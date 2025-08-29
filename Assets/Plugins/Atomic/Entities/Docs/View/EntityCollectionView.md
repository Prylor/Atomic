# 🧩 EntityCollectionView

`EntityCollectionView` is the concrete MonoBehaviour implementation of `IEntityCollectionView` in the Atomic framework. It provides a complete Unity-integrated solution for managing collections of entity views, featuring view pooling, viewport management, and efficient entity-to-view mapping with automatic lifecycle management.

## Key Features

- **MonoBehaviour Implementation** – Full Unity integration with scene management
- **View Pooling** – Efficient view instance reuse through `EntityViewPoolBase`
- **Viewport Management** – Organized view hierarchy under a specified viewport transform
- **Entity-View Mapping** – Maintains dictionary-based associations for fast lookups
- **Pool-Friendly Names** – Configurable entity name resolution for view pool operations
- **Memory Efficient** – Uses `ArrayPool` for temporary allocations
- **Event System** – Reactive notifications for view lifecycle events

---

## Class Definition

```csharp
[AddComponentMenu("Atomic/Entities/Entity Collection View")]
[DisallowMultipleComponent]
public class EntityCollectionView : MonoBehaviour, IEntityCollectionView
{
    [SerializeField] internal Transform _viewport;
    [SerializeField] internal EntityViewPoolBase _viewPool;
    
    private readonly Dictionary<IEntity, EntityViewBase> _views = new();
    
    public event Action<IEntity, IReadOnlyEntityView> OnAdded;
    public event Action<IEntity, IReadOnlyEntityView> OnRemoved;
}
```

## Key Components

### Viewport
```csharp
[SerializeField] internal Transform _viewport;
```
- **Description**: Transform under which all spawned views will be parented
- **Usage**: Scene organization, culling optimization, UI layout management
- **Configuration**: Set in inspector or via script

### View Pool
```csharp
[SerializeField] internal EntityViewPoolBase _viewPool;
```
- **Description**: Pool responsible for providing and recycling entity view instances
- **Usage**: Performance optimization, memory management
- **Types**: Supports different pool implementations for various view types

### Internal Mapping
```csharp
private readonly Dictionary<IEntity, EntityViewBase> _views = new();
```
- **Description**: Internal dictionary maintaining entity-to-view associations
- **Performance**: O(1) lookups, additions, and removals
- **Thread Safety**: Designed for main thread usage only

## Core Properties

### Count
```csharp
public int Count => _views.Count;
```
- **Description**: Number of active entity-view pairs
- **Usage**: Performance monitoring, UI updates, collection state checking

## Core Methods

### AddView
```csharp
public void AddView(IEntity entity);
```
- **Description**: Creates and displays a view for the specified entity
- **Process**:
  1. Checks for existing view (prevents duplicates)
  2. Gets entity name for pool lookup
  3. Rents view from pool
  4. Parents view to viewport
  5. Shows view with entity
  6. Adds to internal dictionary
  7. Triggers `OnAdded` event

### RemoveView
```csharp
public void RemoveView(IEntity entity);
```
- **Description**: Hides and returns view to pool
- **Process**:
  1. Locates view in internal dictionary
  2. Hides the view
  3. Triggers `OnRemoved` event
  4. Gets entity name for pool return
  5. Returns view to pool
  6. Removes from internal dictionary

### ClearViews
```csharp
public void ClearViews();
```
- **Description**: Removes all active views efficiently
- **Implementation**: Uses `ArrayPool` to avoid allocations during bulk operations

### GetView
```csharp
public IReadOnlyEntityView GetView(IEntity entity);
```
- **Description**: Retrieves the view associated with a specific entity
- **Returns**: View instance or throws exception if not found
- **Usage**: Direct view access for updates or modifications

### GetEntityName (Virtual)
```csharp
protected virtual string GetEntityName(IEntity entity) => entity.Name;
```
- **Description**: Determines pool name for entity view lookup
- **Extensibility**: Override to customize naming logic
- **Usage**: Pool key resolution, view type determination

## Example Usage

### Basic Collection Setup

```csharp
using UnityEngine;
using Atomic.Entities;

public class GameWorldDisplay : MonoBehaviour
{
    [SerializeField] private EntityCollectionView unitCollection;
    [SerializeField] private Transform unitViewport;
    [SerializeField] private EntityViewPool unitPool;
    
    private void Start()
    {
        // Configure collection
        unitCollection._viewport = unitViewport;
        unitCollection._viewPool = unitPool;
        
        // Subscribe to events
        unitCollection.OnAdded += OnUnitViewAdded;
        unitCollection.OnRemoved += OnUnitViewRemoved;
        
        // Spawn some test units
        SpawnTestUnits();
    }
    
    private void SpawnTestUnits()
    {
        for (int i = 0; i < 5; i++)
        {
            var unit = new Entity($"Unit_{i}");
            unit.SetValue("Health", 100);
            unit.SetValue("Position", new Vector3(i * 2, 0, 0));
            
            unitCollection.AddView(unit);
        }
    }
    
    private void OnUnitViewAdded(IEntity entity, IReadOnlyEntityView view)
    {
        Debug.Log($"Unit view added: {entity.Name} -> {view.Name}");
    }
    
    private void OnUnitViewRemoved(IEntity entity, IReadOnlyEntityView view)
    {
        Debug.Log($"Unit view removed: {entity.Name}");
    }
    
    private void OnDestroy()
    {
        // Cleanup event subscriptions
        if (unitCollection != null)
        {
            unitCollection.OnAdded -= OnUnitViewAdded;
            unitCollection.OnRemoved -= OnUnitViewRemoved;
        }
    }
}
```

### Procedural Collection Management

Following Atomic's procedural approach:

```csharp
// Static utility methods for EntityCollectionView
public static class EntityCollectionUtils
{
    public static void ConfigureCollection(EntityCollectionView collection,
        Transform viewport, EntityViewPoolBase pool)
    {
        if (collection == null) return;
        
        collection._viewport = viewport;
        collection._viewPool = pool;
        
        Debug.Log($"Configured collection with viewport: {viewport?.name} and pool: {pool?.name}");
    }
    
    public static void PopulateFromArray(EntityCollectionView collection, 
        IEntity[] entities)
    {
        if (collection == null || entities == null) return;
        
        foreach (var entity in entities)
        {
            if (entity != null)
            {
                collection.AddView(entity);
            }
        }
        
        Debug.Log($"Populated collection with {entities.Length} entities");
    }
    
    public static void SynchronizeWithEntityList(EntityCollectionView collection,
        IReadOnlyList<IEntity> targetEntities)
    {
        if (collection == null || targetEntities == null) return;
        
        // Get current entities
        var currentEntities = new HashSet<IEntity>();
        foreach (var pair in collection)
        {
            currentEntities.Add(pair.Key);
        }
        
        // Add missing entities
        foreach (var entity in targetEntities)
        {
            if (!currentEntities.Contains(entity))
            {
                collection.AddView(entity);
            }
        }
        
        // Remove entities not in target list
        var targetSet = new HashSet<IEntity>(targetEntities);
        var toRemove = currentEntities.Where(e => !targetSet.Contains(e)).ToList();
        
        foreach (var entity in toRemove)
        {
            collection.RemoveView(entity);
        }
    }
    
    public static void BatchAddViews(EntityCollectionView collection,
        IEnumerable<IEntity> entities, int batchSize = 10)
    {
        if (collection == null || entities == null) return;
        
        StartCoroutine(BatchAddCoroutine(collection, entities, batchSize));
    }
    
    private static IEnumerator BatchAddCoroutine(EntityCollectionView collection,
        IEnumerable<IEntity> entities, int batchSize)
    {
        int count = 0;
        foreach (var entity in entities)
        {
            collection.AddView(entity);
            count++;
            
            if (count >= batchSize)
            {
                count = 0;
                yield return null; // Wait one frame
            }
        }
    }
    
    public static void FilterViews(EntityCollectionView collection,
        System.Func<IEntity, bool> predicate)
    {
        if (collection == null || predicate == null) return;
        
        var toRemove = new List<IEntity>();
        foreach (var pair in collection)
        {
            if (!predicate(pair.Key))
            {
                toRemove.Add(pair.Key);
            }
        }
        
        foreach (var entity in toRemove)
        {
            collection.RemoveView(entity);
        }
        
        Debug.Log($"Filtered collection, removed {toRemove.Count} views");
    }
}

// Usage example
public class ArmyDisplayManager : MonoBehaviour
{
    [SerializeField] private EntityCollectionView unitCollection;
    [SerializeField] private Transform unitsViewport;
    [SerializeField] private EntityViewPool unitViewPool;
    
    private List<IEntity> armyUnits = new();
    
    private void Start()
    {
        // Configure collection
        EntityCollectionUtils.ConfigureCollection(unitCollection, unitsViewport, unitViewPool);
        
        // Create army
        CreateArmy();
        
        // Display army
        EntityCollectionUtils.PopulateFromArray(unitCollection, armyUnits.ToArray());
    }
    
    public void ReorganizeArmy(UnitType filterType)
    {
        EntityCollectionUtils.FilterViews(unitCollection,
            entity => entity.GetValue<UnitType>("Type") == filterType);
    }
    
    public void ReinforceArmy(IEnumerable<IEntity> reinforcements)
    {
        armyUnits.AddRange(reinforcements);
        EntityCollectionUtils.BatchAddViews(unitCollection, reinforcements);
    }
    
    public void SynchronizeWithCurrentArmy()
    {
        EntityCollectionUtils.SynchronizeWithEntityList(unitCollection, armyUnits);
    }
}
```

### Custom Entity Naming Strategy

```csharp
public class CustomEntityCollectionView : EntityCollectionView
{
    [Header("Custom Naming")]
    [SerializeField] private bool useTypeBasedNaming;
    [SerializeField] private bool includeLevelInName;
    
    protected override string GetEntityName(IEntity entity)
    {
        if (entity == null) return "Unknown";
        
        string baseName = entity.Name;
        
        // Use entity type for pool lookup
        if (useTypeBasedNaming && entity.TryGetValue("Type", out string entityType))
        {
            baseName = entityType;
        }
        
        // Include level for pool variants
        if (includeLevelInName && entity.TryGetValue("Level", out int level))
        {
            baseName = $"{baseName}_Level{level}";
        }
        
        return baseName;
    }
}

// Usage with custom naming
public class RPGCharacterDisplay : MonoBehaviour
{
    [SerializeField] private CustomEntityCollectionView characterViews;
    
    public void DisplayCharacter(IEntity character)
    {
        // Entity name: "Hero_Warrior"
        character.SetValue("Type", "Warrior");
        character.SetValue("Level", 5);
        
        // Pool lookup will use: "Warrior_Level5"
        characterViews.AddView(character);
    }
}
```

### Event-Driven Systems Integration

```csharp
public class BattlefieldManager : MonoBehaviour
{
    [SerializeField] private EntityCollectionView allyViews;
    [SerializeField] private EntityCollectionView enemyViews;
    [SerializeField] private EntityCollectionView projectileViews;
    
    // Systems
    private CombatSystem combatSystem;
    private AudioSystem audioSystem;
    private EffectsSystem effectsSystem;
    
    private void Start()
    {
        SetupEventHandlers();
        InitializeSystems();
    }
    
    private void SetupEventHandlers()
    {
        // Ally spawning
        allyViews.OnAdded += (entity, view) =>
        {
            combatSystem.RegisterAlly(entity);
            audioSystem.PlaySpawnSound("AllySpawn");
            effectsSystem.PlaySpawnEffect(view.transform.position, "Ally");
            
            Debug.Log($"Ally spawned: {entity.Name} (Total allies: {allyViews.Count})");
        };
        
        allyViews.OnRemoved += (entity, view) =>
        {
            combatSystem.UnregisterAlly(entity);
            effectsSystem.PlayDeathEffect(view.transform.position, "Ally");
            
            CheckBattleEndCondition();
        };
        
        // Enemy spawning
        enemyViews.OnAdded += (entity, view) =>
        {
            combatSystem.RegisterEnemy(entity);
            audioSystem.PlaySpawnSound("EnemySpawn");
            effectsSystem.PlaySpawnEffect(view.transform.position, "Enemy");
            
            // Scale difficulty based on enemy count
            if (enemyViews.Count > 10)
            {
                combatSystem.IncreaseDifficulty();
            }
        };
        
        enemyViews.OnRemoved += (entity, view) =>
        {
            combatSystem.UnregisterEnemy(entity);
            AwardExperience(entity);
            
            CheckBattleEndCondition();
        };
        
        // Projectile management
        projectileViews.OnAdded += (entity, view) =>
        {
            StartCoroutine(ManageProjectileLifetime(entity));
        };
    }
    
    private IEnumerator ManageProjectileLifetime(IEntity projectile)
    {
        float lifetime = projectile.GetValue<float>("Lifetime");
        yield return new WaitForSeconds(lifetime);
        
        // Remove projectile if still active
        if (projectileViews.GetView(projectile) != null)
        {
            projectileViews.RemoveView(projectile);
        }
    }
    
    private void CheckBattleEndCondition()
    {
        if (allyViews.Count == 0)
        {
            OnBattleDefeated();
        }
        else if (enemyViews.Count == 0)
        {
            OnBattleVictory();
        }
    }
    
    private void OnBattleVictory()
    {
        Debug.Log("Battle Victory!");
        projectileViews.ClearViews(); // Clear remaining projectiles
        effectsSystem.PlayVictoryEffect();
    }
    
    private void OnBattleDefeated()
    {
        Debug.Log("Battle Defeated!");
        projectileViews.ClearViews();
        effectsSystem.PlayDefeatEffect();
    }
}
```

### Performance-Optimized Loading

```csharp
public class WorldLoader : MonoBehaviour
{
    [SerializeField] private EntityCollectionView worldEntities;
    [SerializeField] private int maxEntitiesPerFrame = 5;
    
    public void LoadWorldAsync(WorldData worldData)
    {
        StartCoroutine(LoadWorldCoroutine(worldData));
    }
    
    private IEnumerator LoadWorldCoroutine(WorldData worldData)
    {
        // Clear existing world
        worldEntities.ClearViews();
        yield return null;
        
        // Load entities in batches
        var entities = CreateEntitiesFromWorldData(worldData);
        
        int loadedCount = 0;
        foreach (var entity in entities)
        {
            worldEntities.AddView(entity);
            loadedCount++;
            
            // Yield control after batch
            if (loadedCount >= maxEntitiesPerFrame)
            {
                loadedCount = 0;
                yield return null;
            }
        }
        
        Debug.Log($"World loaded with {worldEntities.Count} entities");
    }
    
    public void UnloadWorld()
    {
        worldEntities.ClearViews();
        
        // Force garbage collection after large unload
        System.GC.Collect();
    }
    
    private IEnumerable<IEntity> CreateEntitiesFromWorldData(WorldData worldData)
    {
        foreach (var entityData in worldData.entities)
        {
            var entity = new Entity(entityData.name);
            
            // Set entity properties from data
            foreach (var property in entityData.properties)
            {
                entity.SetValue(property.key, property.value);
            }
            
            yield return entity;
        }
    }
}
```

## Best Practices

### Collection Configuration
- **Viewport Setup** – Always assign appropriate viewport for organization
- **Pool Configuration** – Ensure view pool contains required prefabs
- **Event Management** – Subscribe and unsubscribe from events properly

```csharp
// ✅ Good: Proper configuration
private void Start()
{
    collectionView._viewport = GetComponent<Transform>();
    collectionView._viewPool = FindObjectOfType<EntityViewPool>();
    
    collectionView.OnAdded += HandleViewAdded;
    collectionView.OnRemoved += HandleViewRemoved;
}

// ❌ Bad: Missing configuration
private void Start()
{
    // No viewport or pool setup
    collectionView.OnAdded += HandleViewAdded; // Only event handling
}
```

### Performance Optimization
- **Batch Operations** – Use `ClearViews()` instead of individual removals
- **Pool Management** – Ensure pool has sufficient view instances
- **Event Handler Efficiency** – Keep event handlers lightweight

```csharp
// ✅ Good: Efficient batch clearing
public void ResetCollection(IEnumerable<IEntity> newEntities)
{
    collectionView.ClearViews(); // Batch clear
    
    foreach (var entity in newEntities)
    {
        collectionView.AddView(entity);
    }
}

// ❌ Bad: Inefficient individual removals
public void ResetCollection(IEnumerable<IEntity> newEntities)
{
    foreach (var pair in collectionView.ToList())
    {
        collectionView.RemoveView(pair.Key); // Individual removals
    }
}
```

### Memory Management
- **Event Cleanup** – Always unsubscribe from events
- **Pool Returns** – Views are automatically returned to pool
- **Array Pool Usage** – Internal ArrayPool usage minimizes allocations

## Performance Considerations

### Collection Operations
- **Dictionary Performance** – O(1) lookups, additions, removals
- **Pool Efficiency** – View reuse minimizes instantiation costs
- **Event Overhead** – Lightweight event system with minimal allocations

### Memory Management
- **Pool-Based Views** – Views are pooled and reused
- **ArrayPool Usage** – Temporary arrays use ArrayPool to avoid allocations
- **Dictionary Capacity** – Internal dictionary grows as needed

### Scalability
- **Large Collections** – Handles hundreds of entities efficiently
- **Batch Loading** – Supports async loading for large worlds
- **Selective Updates** – Only views that change are updated

## Integration Patterns

### World Systems
```csharp
// World entity management
public void LoadChunk(ChunkData chunk)
{
    foreach (var entityData in chunk.entities)
    {
        var entity = CreateEntity(entityData);
        worldViews.AddView(entity);
    }
}
```

### UI Systems
```csharp
// Inventory display
public void RefreshInventory(List<IEntity> items)
{
    inventoryViews.ClearViews();
    
    foreach (var item in items)
    {
        inventoryViews.AddView(item);
    }
}
```

### Game Systems
```csharp
// Unit management
public void SpawnSquad(SquadData squad)
{
    foreach (var unitData in squad.units)
    {
        var unit = CreateUnit(unitData);
        unitViews.AddView(unit);
    }
}
```

## Notes

- **Unity Integration** – Full MonoBehaviour support with scene serialization
- **Pool-Based Architecture** – Efficient view management through pooling systems
- **Event-Driven Design** – Reactive programming support through lifecycle events
- **Performance Optimized** – Dictionary-based lookups and ArrayPool usage
- **Extensible Design** – Virtual methods allow customization of behavior
- **Memory Efficient** – Minimal allocations through pooling and ArrayPool