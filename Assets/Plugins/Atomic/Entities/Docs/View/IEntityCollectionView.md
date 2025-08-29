# 🧩 IEntityCollectionView

`IEntityCollectionView` is the interface for managing collections of entity views in the Atomic framework. It extends the read-only collection view with modification capabilities, enabling dynamic addition, removal, and clearing of entity views while maintaining reactive notifications for view lifecycle events.

## Key Features

- **Collection Management** – Add, remove, and clear entity views dynamically
- **Event Notifications** – Reactive events for view addition and removal
- **Entity-View Mapping** – Maintains associations between entities and their visual representations
- **View Lifecycle** – Manages view spawning and despawning automatically
- **Type Safety** – Strongly typed collection operations
- **Performance Optimized** – Efficient collection operations and event handling

---

## Interface Definition

```csharp
public interface IEntityCollectionView : IReadOnlyEntityCollectionView
{
    /// <summary>
    /// Adds a view for the specified entity to the collection.
    /// </summary>
    /// <param name="entity">The entity for which a view should be added.</param>
    void AddView(IEntity entity);

    /// <summary>
    /// Removes the view associated with the specified entity from the collection.
    /// </summary>
    /// <param name="entity">The entity whose view should be removed.</param>
    void RemoveView(IEntity entity);

    /// <summary>
    /// Removes all views from the collection, clearing all associated entity views.
    /// </summary>
    void ClearViews();
}
```

## Inheritance

- **IReadOnlyEntityCollectionView** – Provides read-only access and event notifications
- **IReadOnlyCollection<KeyValuePair<IEntity, IReadOnlyEntityView>>** – Collection interface support

## Inherited Properties and Events

### Events

#### OnAdded
```csharp
event Action<IEntity, IReadOnlyEntityView> OnAdded;
```
- **Description**: Fired when a new view is added for an entity
- **Parameters**: 
  - `entity` – The entity that got a new view
  - `view` – The view instance that was created
- **Usage**: React to view spawning, setup additional components, logging

#### OnRemoved
```csharp
event Action<IEntity, IReadOnlyEntityView> OnRemoved;
```
- **Description**: Fired when a view is removed from the collection
- **Parameters**: 
  - `entity` – The entity whose view was removed
  - `view` – The view instance that was removed
- **Usage**: Cleanup operations, statistics tracking, notifications

### Properties

#### Count
```csharp
int Count { get; }
```
- **Description**: Number of active entity-view pairs in the collection
- **Usage**: Performance monitoring, UI updates, collection state checking

### Methods

#### GetView
```csharp
IReadOnlyEntityView GetView(IEntity entity);
```
- **Description**: Retrieves the view associated with a specific entity
- **Returns**: The view instance, or null if no view exists
- **Usage**: Access existing views for updates or modifications

## Core Methods

### AddView
```csharp
void AddView(IEntity entity);
```
- **Description**: Creates and adds a view for the specified entity
- **Parameters**: `entity` – The entity to visualize
- **Behavior**:
  - Checks if view already exists (prevents duplicates)
  - Creates new view instance from pool or factory
  - Associates view with entity
  - Shows the view
  - Triggers `OnAdded` event

### RemoveView
```csharp
void RemoveView(IEntity entity);
```
- **Description**: Removes and cleans up the view for the specified entity
- **Parameters**: `entity` – The entity whose view should be removed
- **Behavior**:
  - Locates existing view for entity
  - Hides the view
  - Triggers `OnRemoved` event
  - Returns view to pool or destroys it
  - Removes entity-view association

### ClearViews
```csharp
void ClearViews();
```
- **Description**: Removes all entity views from the collection
- **Behavior**:
  - Iterates through all active views
  - Calls `RemoveView` for each entity
  - Triggers `OnRemoved` events for all removals
  - Results in empty collection

## Example Usage

### Basic Collection Management

```csharp
using UnityEngine;
using System.Collections.Generic;

public class EntityDisplayManager : MonoBehaviour
{
    [SerializeField] private EntityCollectionView collectionView;
    
    private List<IEntity> entities = new();
    
    private void Start()
    {
        // Subscribe to collection events
        collectionView.OnAdded += OnEntityViewAdded;
        collectionView.OnRemoved += OnEntityViewRemoved;
    }
    
    public void ShowEntity(IEntity entity)
    {
        entities.Add(entity);
        collectionView.AddView(entity);
    }
    
    public void HideEntity(IEntity entity)
    {
        entities.Remove(entity);
        collectionView.RemoveView(entity);
    }
    
    public void ShowAllEntities(IEnumerable<IEntity> entitiesToShow)
    {
        foreach (var entity in entitiesToShow)
        {
            ShowEntity(entity);
        }
    }
    
    public void HideAllEntities()
    {
        collectionView.ClearViews();
        entities.Clear();
    }
    
    private void OnEntityViewAdded(IEntity entity, IReadOnlyEntityView view)
    {
        Debug.Log($"View added for entity: {entity.Name} (Total views: {collectionView.Count})");
    }
    
    private void OnEntityViewRemoved(IEntity entity, IReadOnlyEntityView view)
    {
        Debug.Log($"View removed for entity: {entity.Name} (Remaining views: {collectionView.Count})");
    }
}
```

### Procedural Collection Operations

Following Atomic's procedural approach:

```csharp
// Static utility methods for collection view management
public static class CollectionViewUtils
{
    public static void PopulateCollection(IEntityCollectionView collection, 
        IEnumerable<IEntity> entities)
    {
        if (collection == null || entities == null) return;
        
        foreach (var entity in entities)
        {
            collection.AddView(entity);
        }
        
        Debug.Log($"Populated collection with {collection.Count} views");
    }
    
    public static void FilterCollection(IEntityCollectionView collection, 
        System.Func<IEntity, bool> predicate)
    {
        if (collection == null || predicate == null) return;
        
        var toRemove = new List<IEntity>();
        
        // Find entities that don't match the filter
        foreach (var pair in collection)
        {
            if (!predicate(pair.Key))
            {
                toRemove.Add(pair.Key);
            }
        }
        
        // Remove filtered entities
        foreach (var entity in toRemove)
        {
            collection.RemoveView(entity);
        }
        
        Debug.Log($"Filtered collection, removed {toRemove.Count} views");
    }
    
    public static void UpdateAllViews(IEntityCollectionView collection, 
        System.Action<IEntity, IReadOnlyEntityView> updateAction)
    {
        if (collection == null || updateAction == null) return;
        
        foreach (var pair in collection)
        {
            updateAction(pair.Key, pair.Value);
        }
    }
    
    public static bool HasViewFor(IEntityCollectionView collection, IEntity entity)
    {
        if (collection == null || entity == null) return false;
        
        return collection.GetView(entity) != null;
    }
    
    public static IReadOnlyEntityView FindViewByName(IEntityCollectionView collection, 
        string viewName)
    {
        if (collection == null || string.IsNullOrEmpty(viewName)) return null;
        
        foreach (var pair in collection)
        {
            if (pair.Value.Name == viewName)
            {
                return pair.Value;
            }
        }
        
        return null;
    }
}

// Usage examples
public class InventoryDisplayManager : MonoBehaviour
{
    [SerializeField] private EntityCollectionView itemViews;
    
    public void DisplayInventory(IEnumerable<IEntity> items)
    {
        // Clear existing views
        itemViews.ClearViews();
        
        // Populate with new items
        CollectionViewUtils.PopulateCollection(itemViews, items);
    }
    
    public void FilterByCategory(string category)
    {
        CollectionViewUtils.FilterCollection(itemViews, 
            entity => entity.GetValue<string>("Category") == category);
    }
    
    public void UpdateAllItemQuantities()
    {
        CollectionViewUtils.UpdateAllViews(itemViews, 
            (entity, view) =>
            {
                int quantity = entity.GetValue<int>("Quantity");
                // Update view display logic would go here
            });
    }
}
```

### Event-Driven Architecture

```csharp
public class GameWorldManager : MonoBehaviour
{
    [SerializeField] private EntityCollectionView playerViews;
    [SerializeField] private EntityCollectionView enemyViews;
    [SerializeField] private EntityCollectionView itemViews;
    
    // Statistics tracking
    private int totalSpawnedViews;
    private int totalDespawnedViews;
    
    private void Start()
    {
        SetupEventHandlers();
    }
    
    private void SetupEventHandlers()
    {
        // Player views
        playerViews.OnAdded += (entity, view) =>
        {
            OnPlayerSpawned(entity, view);
            UpdatePlayerCount();
        };
        
        playerViews.OnRemoved += (entity, view) =>
        {
            OnPlayerDespawned(entity, view);
            UpdatePlayerCount();
        };
        
        // Enemy views
        enemyViews.OnAdded += (entity, view) =>
        {
            OnEnemySpawned(entity, view);
            UpdateEnemyCount();
        };
        
        enemyViews.OnRemoved += (entity, view) =>
        {
            OnEnemyDespawned(entity, view);
            UpdateEnemyCount();
        };
        
        // Item views
        itemViews.OnAdded += (entity, view) => OnItemSpawned(entity, view);
        itemViews.OnRemoved += (entity, view) => OnItemDespawned(entity, view);
    }
    
    private void OnPlayerSpawned(IEntity player, IReadOnlyEntityView view)
    {
        totalSpawnedViews++;
        Debug.Log($"Player {player.Name} spawned at {view.Name}");
        
        // Setup player-specific systems
        SetupPlayerSystems(player);
    }
    
    private void OnPlayerDespawned(IEntity player, IReadOnlyEntityView view)
    {
        totalDespawnedViews++;
        Debug.Log($"Player {player.Name} despawned");
        
        // Cleanup player-specific systems
        CleanupPlayerSystems(player);
    }
    
    private void OnEnemySpawned(IEntity enemy, IReadOnlyEntityView view)
    {
        totalSpawnedViews++;
        
        // Add enemy to AI system
        AISystem.RegisterEnemy(enemy);
        
        // Update difficulty if too many enemies
        if (enemyViews.Count > 10)
        {
            AdjustDifficulty();
        }
    }
    
    private void OnEnemyDespawned(IEntity enemy, IReadOnlyEntityView view)
    {
        totalDespawnedViews++;
        
        // Remove enemy from AI system
        AISystem.UnregisterEnemy(enemy);
        
        // Award experience to nearby players
        AwardExperienceToNearbyPlayers(enemy);
    }
    
    private void OnItemSpawned(IEntity item, IReadOnlyEntityView view)
    {
        // Setup item interaction systems
        InteractionSystem.RegisterItem(item);
        
        // Start despawn timer for temporary items
        if (item.HasTag("Temporary"))
        {
            StartCoroutine(DespawnItemAfterDelay(item, 30f));
        }
    }
    
    private void OnItemDespawned(IEntity item, IReadOnlyEntityView view)
    {
        InteractionSystem.UnregisterItem(item);
    }
}
```

### Dynamic Collection Management

```csharp
public class DynamicSpawner : MonoBehaviour
{
    [SerializeField] private EntityCollectionView unitViews;
    [SerializeField] private Transform spawnArea;
    
    private readonly Queue<IEntity> spawnQueue = new();
    private readonly HashSet<IEntity> activeUnits = new();
    
    public void QueueUnitForSpawn(IEntity unit)
    {
        spawnQueue.Enqueue(unit);
    }
    
    public void ProcessSpawnQueue()
    {
        int spawnCount = Mathf.Min(spawnQueue.Count, GetMaxSpawnCount());
        
        for (int i = 0; i < spawnCount; i++)
        {
            if (spawnQueue.TryDequeue(out IEntity unit))
            {
                SpawnUnit(unit);
            }
        }
    }
    
    private void SpawnUnit(IEntity unit)
    {
        // Set spawn position
        Vector3 spawnPos = GetRandomSpawnPosition();
        unit.SetValue("Position", spawnPos);
        
        // Add to collection view
        unitViews.AddView(unit);
        activeUnits.Add(unit);
        
        // Setup unit lifetime
        StartCoroutine(ManageUnitLifetime(unit));
    }
    
    private IEnumerator ManageUnitLifetime(IEntity unit)
    {
        // Wait for unit lifetime
        float lifetime = unit.GetValue<float>("Lifetime");
        yield return new WaitForSeconds(lifetime);
        
        // Check if unit is still active
        if (activeUnits.Contains(unit))
        {
            DespawnUnit(unit);
        }
    }
    
    private void DespawnUnit(IEntity unit)
    {
        unitViews.RemoveView(unit);
        activeUnits.Remove(unit);
    }
    
    public void DespawnAllUnits()
    {
        unitViews.ClearViews();
        activeUnits.Clear();
        spawnQueue.Clear();
    }
    
    private int GetMaxSpawnCount()
    {
        // Limit based on performance
        int maxUnits = 50;
        return Mathf.Max(0, maxUnits - unitViews.Count);
    }
    
    private Vector3 GetRandomSpawnPosition()
    {
        Bounds bounds = spawnArea.GetComponent<Collider>().bounds;
        return new Vector3(
            Random.Range(bounds.min.x, bounds.max.x),
            bounds.center.y,
            Random.Range(bounds.min.z, bounds.max.z)
        );
    }
}
```

## Best Practices

### Collection Management
- **Event Handling** – Always subscribe to OnAdded/OnRemoved for proper system integration
- **Null Checking** – Validate entities and views before operations
- **Batch Operations** – Use ClearViews() instead of individual RemoveView() calls when clearing all
- **Memory Management** – Unsubscribe from events to prevent memory leaks

```csharp
// ✅ Good: Proper event management
private void Start()
{
    collectionView.OnAdded += HandleViewAdded;
    collectionView.OnRemoved += HandleViewRemoved;
}

private void OnDestroy()
{
    if (collectionView != null)
    {
        collectionView.OnAdded -= HandleViewAdded;
        collectionView.OnRemoved -= HandleViewRemoved;
    }
}

// ❌ Bad: No event cleanup
private void Start()
{
    collectionView.OnAdded += HandleViewAdded; // Memory leak potential
}
```

### Performance Optimization
- **Batch Updates** – Group multiple view operations together
- **Conditional Operations** – Check if view exists before removing
- **Event Throttling** – Limit event frequency for high-frequency operations

```csharp
// ✅ Good: Batch operations
public void UpdateEntityCollection(IEnumerable<IEntity> newEntities)
{
    // Clear all at once
    collectionView.ClearViews();
    
    // Add all new entities
    foreach (var entity in newEntities)
    {
        collectionView.AddView(entity);
    }
}

// ❌ Bad: Inefficient individual operations
public void UpdateEntityCollection(IEnumerable<IEntity> newEntities)
{
    foreach (var pair in collectionView.ToList()) // Creates unnecessary list
    {
        collectionView.RemoveView(pair.Key); // Individual removals
    }
}
```

### State Management
- **Consistent State** – Ensure collection state matches game state
- **Error Handling** – Handle cases where entities or views might be null
- **Thread Safety** – Perform collection operations on main thread only

## Performance Considerations

### Collection Operations
- **Add/Remove Complexity** – O(1) for hash-based implementations
- **Iteration Performance** – Efficient enumeration through entity-view pairs
- **Memory Usage** – Views are pooled and reused when possible

### Event System
- **Event Frequency** – High-frequency spawning/despawning can impact performance
- **Handler Complexity** – Keep event handlers lightweight
- **Event Cleanup** – Always unsubscribe to prevent memory leaks

### View Management
- **Pool Usage** – Collection views typically use pooling for view instances
- **Batch Processing** – Process multiple changes together when possible
- **Selective Updates** – Only update views that actually need changes

## Integration Patterns

### World Management
```csharp
public class WorldEntityManager : MonoBehaviour
{
    [SerializeField] private EntityCollectionView worldViews;
    
    public void LoadWorldEntities(WorldData worldData)
    {
        worldViews.ClearViews();
        
        foreach (var entityData in worldData.entities)
        {
            var entity = CreateEntityFromData(entityData);
            worldViews.AddView(entity);
        }
    }
}
```

### UI Collections
```csharp
public class InventoryUI : MonoBehaviour
{
    [SerializeField] private EntityCollectionView inventoryViews;
    
    public void RefreshInventory(IEnumerable<IEntity> items)
    {
        inventoryViews.ClearViews();
        
        foreach (var item in items)
        {
            inventoryViews.AddView(item);
        }
    }
}
```

## Notes

- **Reactive Architecture** – Events enable reactive programming patterns
- **Type Safety** – Strongly typed operations prevent runtime errors
- **Performance Optimized** – Efficient collection operations and memory management
- **Unity Integration** – Works seamlessly with Unity's component system
- **Scalable Design** – Handles collections from small inventories to large worlds