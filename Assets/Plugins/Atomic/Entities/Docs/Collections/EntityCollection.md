# 🧩 EntityCollection

A high-performance, unique collection designed to store entity instances with fast lookup, insertion, and deletion capabilities. Combines hash table and linked list semantics for both efficient access and ordered enumeration while providing comprehensive event notifications.

## Overview

`EntityCollection` provides a sophisticated storage solution for entities that requires uniqueness guarantees, fast operations, and ordered enumeration. It implements a hybrid data structure combining hash table performance with linked list ordering, making it ideal for entity management systems that need both quick lookups and deterministic iteration order.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// A non-generic version of EntityCollection{E} that operates specifically on IEntity instances.
    /// Provides the same functionality as the generic base class but simplifies usage when generic typing is unnecessary.
    /// </summary>
    public class EntityCollection : EntityCollection<IEntity>
    {
        public EntityCollection() { }
        public EntityCollection(int capacity) : base(capacity) { }
        public EntityCollection(params IEntity[] entities) : base(entities) { }
        public EntityCollection(IReadOnlyCollection<IEntity> elements) : base(elements) { }
        public EntityCollection(IEnumerable<IEntity> elements) : base(elements) { }
    }

    /// <summary>
    /// A performant and flexible collection designed to store unique IEntity elements
    /// with fast lookup, insertion, and deletion. Combines hash table and linked list semantics
    /// for both efficient access and ordered enumeration.
    /// </summary>
    /// <typeparam name="E">The type of the entity. Must implement IEntity.</typeparam>
    public class EntityCollection<E> : IEntityCollection<E> where E : IEntity
    {
        public event Action OnStateChanged;
        public event Action<E> OnAdded;
        public event Action<E> OnRemoved;

        public int Count { get; }
        public bool IsReadOnly => false;

        public bool Add(E item);
        public bool Remove(E item);
        public bool Contains(E item);
        public void Clear();
        public void CopyTo(E[] array, int arrayIndex);
        public void CopyTo(ICollection<E> results);
        public void Dispose();
        public void UnsubscribeAll();
        
        public Enumerator GetEnumerator();
    }
}
```

## Key Features

### Hybrid Data Structure
- Hash table for O(1) average lookup, insertion, and deletion
- Doubly-linked list for ordered enumeration and insertion order preservation
- Unique element guarantee with efficient duplicate detection

### High Performance
- Custom slot-based allocation with free list management
- ArrayPool usage for temporary allocations during bulk operations
- Aggressive inlining for critical path operations

### Event System
- Comprehensive event notifications for collection changes
- State change tracking for reactive system integration
- Individual entity add/remove notifications

### Memory Efficiency
- Dynamic resizing with prime-number-based capacity growth
- Efficient memory reuse through free list management
- Minimal per-element overhead with struct-based storage

## Usage Examples

### Basic Collection Operations

```csharp
// Create a collection with initial capacity
var playerParty = new EntityCollection<IEntity>(capacity: 4);

// Subscribe to events for reactive behavior
playerParty.OnAdded += entity =>
{
    Debug.Log($"Party member joined: {entity.Name}");
    entity.AddTag("PartyMember");
    UpdatePartyStats();
};

playerParty.OnRemoved += entity =>
{
    Debug.Log($"Party member left: {entity.Name}");
    entity.RemoveTag("PartyMember");
    UpdatePartyStats();
};

// Add entities
var hero = new Entity("Hero");
var warrior = new Entity("Warrior");
var mage = new Entity("Mage");

playerParty.Add(hero);      // Returns true, triggers OnAdded
playerParty.Add(warrior);   // Returns true, triggers OnAdded
playerParty.Add(mage);      // Returns true, triggers OnAdded
playerParty.Add(hero);      // Returns false, no duplicate allowed

// Check membership and remove
if (playerParty.Contains(warrior))
{
    playerParty.Remove(warrior);  // Returns true, triggers OnRemoved
}

Debug.Log($"Party size: {playerParty.Count}");  // Output: 2
```

### Enemy Wave Management System

```csharp
public class EnemyWaveManager : MonoBehaviour
{
    private readonly EntityCollection<IEntity> _activeEnemies = new();
    private readonly EntityCollection<IEntity> _enemyPool = new(capacity: 50);
    
    [SerializeField] private int _maxActiveEnemies = 10;
    [SerializeField] private float _spawnInterval = 2f;
    
    private float _lastSpawnTime;

    void Start()
    {
        // Setup event handlers for active enemies
        _activeEnemies.OnAdded += OnEnemySpawned;
        _activeEnemies.OnRemoved += OnEnemyDefeated;
        _activeEnemies.OnStateChanged += UpdateWaveProgress;

        // Pre-populate enemy pool
        InitializeEnemyPool();
        
        // Start spawning enemies
        StartWaveSpawning();
    }

    private void InitializeEnemyPool()
    {
        for (int i = 0; i < 20; i++)
        {
            var enemy = CreateEnemy($"Enemy_{i}");
            enemy.AddTag("Pooled");
            enemy.Despawn();  // Start despawned
            _enemyPool.Add(enemy);
        }
        
        Debug.Log($"Enemy pool initialized with {_enemyPool.Count} enemies");
    }

    private void StartWaveSpawning()
    {
        _lastSpawnTime = Time.time;
    }

    void Update()
    {
        // Spawn enemies at intervals
        if (Time.time - _lastSpawnTime >= _spawnInterval)
        {
            if (_activeEnemies.Count < _maxActiveEnemies)
            {
                SpawnNextEnemy();
            }
            _lastSpawnTime = Time.time;
        }

        // Check for wave completion
        CheckWaveCompletion();
    }

    private void SpawnNextEnemy()
    {
        // Try to get enemy from pool first
        IEntity enemy = GetPooledEnemy() ?? CreateEnemy($"Enemy_{Time.time}");
        
        if (enemy != null)
        {
            PrepareEnemyForSpawn(enemy);
            _activeEnemies.Add(enemy);  // Triggers OnEnemySpawned
        }
    }

    private IEntity GetPooledEnemy()
    {
        foreach (var pooledEnemy in _enemyPool)
        {
            if (pooledEnemy.HasTag("Pooled") && !pooledEnemy.Spawned)
            {
                _enemyPool.Remove(pooledEnemy);
                return pooledEnemy;
            }
        }
        return null;
    }

    private void PrepareEnemyForSpawn(IEntity enemy)
    {
        enemy.RemoveTag("Pooled");
        enemy.RemoveTag("Defeated");
        enemy.AddTag("Active");
        
        // Reset enemy stats
        enemy.Set("Health", 100f);
        entity.Set("Position", GetRandomSpawnPosition());
        
        enemy.Spawn();
    }

    private void OnEnemySpawned(IEntity enemy)
    {
        Debug.Log($"Enemy spawned: {enemy.Name} (Active: {_activeEnemies.Count})");
        
        // Setup enemy behavior
        enemy.AddTag("Hostile");
        enemy.Set("SpawnTime", Time.time);
        enemy.Set("TargetPlayer", FindPlayerTarget());
        
        // Subscribe to enemy death
        enemy.OnDeath += () => HandleEnemyDeath(enemy);
        
        GameEvents.OnEnemySpawned?.Invoke(enemy);
    }

    private void OnEnemyDefeated(IEntity enemy)
    {
        Debug.Log($"Enemy defeated: {enemy.Name} (Active: {_activeEnemies.Count})");
        
        enemy.RemoveTag("Active");
        enemy.RemoveTag("Hostile");
        enemy.AddTag("Defeated");
        
        // Return to pool if possible
        if (_enemyPool.Count < 20)
        {
            enemy.AddTag("Pooled");
            enemy.Despawn();
            _enemyPool.Add(enemy);
        }
        else
        {
            enemy.Dispose();  // Pool full, dispose entirely
        }
        
        GameEvents.OnEnemyDefeated?.Invoke(enemy);
    }

    private void HandleEnemyDeath(IEntity enemy)
    {
        if (_activeEnemies.Contains(enemy))
        {
            _activeEnemies.Remove(enemy);  // Triggers OnEnemyDefeated
        }
    }

    private void UpdateWaveProgress()
    {
        float progress = (float)GetDefeatedEnemyCount() / GetTotalWaveEnemies();
        GameEvents.OnWaveProgressChanged?.Invoke(progress);
        
        // Update UI
        UIManager.UpdateWaveProgress(_activeEnemies.Count, _maxActiveEnemies);
    }

    private void CheckWaveCompletion()
    {
        if (_activeEnemies.Count == 0 && AllEnemiesSpawned())
        {
            CompleteWave();
        }
    }

    private IEntity CreateEnemy(string name)
    {
        var enemy = new Entity(name);
        enemy.Set("Health", 100f);
        enemy.Set("MaxHealth", 100f);
        enemy.Set("Damage", 25f);
        enemy.Set("MovementSpeed", 3f);
        return enemy;
    }

    private Vector3 GetRandomSpawnPosition()
    {
        // Implementation for random spawn positioning
        return Vector3.zero;
    }

    private IEntity FindPlayerTarget()
    {
        // Implementation for finding player target
        return null;
    }

    void OnDestroy()
    {
        _activeEnemies.Dispose();
        _enemyPool.Dispose();
    }
}
```

### Inventory System with Categorization

```csharp
public class AdvancedInventorySystem : MonoBehaviour
{
    private readonly Dictionary<string, EntityCollection<IEntity>> _itemCategories = new();
    private readonly EntityCollection<IEntity> _allItems = new(capacity: 100);
    
    [SerializeField] private int _maxItemsPerCategory = 20;
    [SerializeField] private string[] _itemCategories = { "Weapons", "Armor", "Consumables", "Materials", "Quest" };

    void Start()
    {
        InitializeCategories();
        SetupGlobalItemTracking();
    }

    private void InitializeCategories()
    {
        foreach (string category in _itemCategories)
        {
            var collection = new EntityCollection<IEntity>(capacity: _maxItemsPerCategory);
            
            // Setup category-specific event handlers
            collection.OnAdded += item => OnItemAddedToCategory(category, item);
            collection.OnRemoved += item => OnItemRemovedFromCategory(category, item);
            collection.OnStateChanged += () => OnCategoryStateChanged(category);
            
            _itemCategories[category] = collection;
        }
    }

    private void SetupGlobalItemTracking()
    {
        _allItems.OnAdded += item =>
        {
            Debug.Log($"Item added to inventory: {item.Name}");
            item.AddTag("InInventory");
            GameEvents.OnInventoryChanged?.Invoke();
        };

        _allItems.OnRemoved += item =>
        {
            Debug.Log($"Item removed from inventory: {item.Name}");
            item.RemoveTag("InInventory");
            GameEvents.OnInventoryChanged?.Invoke();
        };
    }

    public bool AddItem(IEntity item, string category = null)
    {
        if (item == null)
            return false;

        // Determine category if not provided
        category = category ?? DetermineItemCategory(item);
        
        if (!_itemCategories.TryGetValue(category, out var categoryCollection))
        {
            Debug.LogWarning($"Unknown item category: {category}");
            return false;
        }

        // Check category capacity
        if (categoryCollection.Count >= _maxItemsPerCategory)
        {
            Debug.LogWarning($"Category '{category}' is full ({_maxItemsPerCategory} items)");
            return false;
        }

        // Add to both global and category collections
        bool addedToGlobal = _allItems.Add(item);
        if (addedToGlobal)
        {
            categoryCollection.Add(item);
            item.Set("InventoryCategory", category);
            return true;
        }

        return false;
    }

    public bool RemoveItem(IEntity item)
    {
        if (item == null || !_allItems.Contains(item))
            return false;

        string category = item.Get<string>("InventoryCategory");
        if (category != null && _itemCategories.TryGetValue(category, out var categoryCollection))
        {
            categoryCollection.Remove(item);
        }

        item.Remove("InventoryCategory");
        return _allItems.Remove(item);
    }

    public IEntity FindItem(string itemName)
    {
        foreach (var item in _allItems)
        {
            if (item.Name == itemName)
                return item;
        }
        return null;
    }

    public IReadOnlyCollection<IEntity> GetItemsInCategory(string category)
    {
        if (_itemCategories.TryGetValue(category, out var collection))
        {
            var items = new List<IEntity>();
            collection.CopyTo(items);
            return items;
        }
        return new List<IEntity>();
    }

    public IEntity[] GetItemsWithTag(string tag)
    {
        var results = new List<IEntity>();
        
        foreach (var item in _allItems)
        {
            if (item.HasTag(tag))
            {
                results.Add(item);
            }
        }
        
        return results.ToArray();
    }

    public int GetTotalValue()
    {
        int totalValue = 0;
        foreach (var item in _allItems)
        {
            if (item.HasValue("Value"))
            {
                totalValue += item.Get<int>("Value");
            }
        }
        return totalValue;
    }

    public float GetTotalWeight()
    {
        float totalWeight = 0f;
        foreach (var item in _allItems)
        {
            if (item.HasValue("Weight"))
            {
                totalWeight += item.Get<float>("Weight");
            }
        }
        return totalWeight;
    }

    public void SortCategory(string category, Func<IEntity, IEntity, int> comparer)
    {
        if (!_itemCategories.TryGetValue(category, out var collection))
            return;

        // Extract items and sort
        var items = new List<IEntity>();
        collection.CopyTo(items);
        items.Sort((a, b) => comparer(a, b));

        // Rebuild collection in sorted order
        collection.Clear();
        foreach (var item in items)
        {
            collection.Add(item);
        }
    }

    public InventoryStatistics GetStatistics()
    {
        var stats = new InventoryStatistics
        {
            TotalItems = _allItems.Count,
            TotalValue = GetTotalValue(),
            TotalWeight = GetTotalWeight(),
            CategoryCounts = new Dictionary<string, int>()
        };

        foreach (var kvp in _itemCategories)
        {
            stats.CategoryCounts[kvp.Key] = kvp.Value.Count;
        }

        return stats;
    }

    private string DetermineItemCategory(IEntity item)
    {
        if (item.HasTag("Weapon")) return "Weapons";
        if (item.HasTag("Armor")) return "Armor";
        if (item.HasTag("Consumable")) return "Consumables";
        if (item.HasTag("Material")) return "Materials";
        if (item.HasTag("Quest")) return "Quest";
        return "Materials"; // Default category
    }

    private void OnItemAddedToCategory(string category, IEntity item)
    {
        Debug.Log($"Item added to {category}: {item.Name} ({_itemCategories[category].Count}/{_maxItemsPerCategory})");
        GameEvents.OnCategoryChanged?.Invoke(category, _itemCategories[category].Count);
    }

    private void OnItemRemovedFromCategory(string category, IEntity item)
    {
        Debug.Log($"Item removed from {category}: {item.Name} ({_itemCategories[category].Count}/{_maxItemsPerCategory})");
        GameEvents.OnCategoryChanged?.Invoke(category, _itemCategories[category].Count);
    }

    private void OnCategoryStateChanged(string category)
    {
        // Update UI for this category
        UIManager.UpdateCategoryDisplay(category, _itemCategories[category]);
    }

    void OnDestroy()
    {
        _allItems.Dispose();
        foreach (var collection in _itemCategories.Values)
        {
            collection.Dispose();
        }
    }
}

[System.Serializable]
public struct InventoryStatistics
{
    public int TotalItems;
    public int TotalValue;
    public float TotalWeight;
    public Dictionary<string, int> CategoryCounts;
}
```

### Entity State Management System

```csharp
public class EntityStateManager : MonoBehaviour
{
    private readonly Dictionary<EntityState, EntityCollection<IEntity>> _stateCollections = new();
    private readonly Dictionary<IEntity, EntityState> _entityStates = new();
    
    private enum EntityState
    {
        Spawned,
        Active,
        Inactive,
        Paused,
        Destroyed
    }

    void Start()
    {
        InitializeStateCollections();
        SetupGlobalEntityTracking();
    }

    private void InitializeStateCollections()
    {
        foreach (EntityState state in System.Enum.GetValues(typeof(EntityState)))
        {
            var collection = new EntityCollection<IEntity>();
            collection.OnAdded += entity => OnEntityStateChanged(entity, state);
            collection.OnRemoved += entity => OnEntityLeftState(entity, state);
            
            _stateCollections[state] = collection;
        }
    }

    private void SetupGlobalEntityTracking()
    {
        // Listen for global entity events
        EntityRegistry.OnEntityCreated += entity => TransitionEntityToState(entity, EntityState.Spawned);
        EntityRegistry.OnEntityDestroyed += entity => TransitionEntityToState(entity, EntityState.Destroyed);
    }

    public void TransitionEntityToState(IEntity entity, EntityState newState)
    {
        if (entity == null)
            return;

        // Get current state
        EntityState currentState = _entityStates.GetValueOrDefault(entity, EntityState.Spawned);
        
        if (currentState == newState)
            return; // Already in target state

        // Remove from current state collection
        if (_stateCollections.TryGetValue(currentState, out var currentCollection))
        {
            currentCollection.Remove(entity);
        }

        // Add to new state collection
        if (_stateCollections.TryGetValue(newState, out var newCollection))
        {
            newCollection.Add(entity);
        }

        // Update tracking
        _entityStates[entity] = newState;

        Debug.Log($"Entity {entity.Name} transitioned from {currentState} to {newState}");
    }

    public void ActivateEntity(IEntity entity)
    {
        TransitionEntityToState(entity, EntityState.Active);
    }

    public void DeactivateEntity(IEntity entity)
    {
        TransitionEntityToState(entity, EntityState.Inactive);
    }

    public void PauseEntity(IEntity entity)
    {
        TransitionEntityToState(entity, EntityState.Paused);
    }

    public void ResumeEntity(IEntity entity)
    {
        // Resume to active state
        TransitionEntityToState(entity, EntityState.Active);
    }

    public IReadOnlyCollection<IEntity> GetEntitiesInState(EntityState state)
    {
        if (_stateCollections.TryGetValue(state, out var collection))
        {
            var entities = new List<IEntity>();
            collection.CopyTo(entities);
            return entities;
        }
        return new List<IEntity>();
    }

    public EntityState GetEntityState(IEntity entity)
    {
        return _entityStates.GetValueOrDefault(entity, EntityState.Spawned);
    }

    public void ProcessActiveEntities(Action<IEntity> processor)
    {
        var activeCollection = _stateCollections[EntityState.Active];
        foreach (var entity in activeCollection)
        {
            try
            {
                processor(entity);
            }
            catch (System.Exception ex)
            {
                Debug.LogError($"Error processing active entity {entity.Name}: {ex.Message}");
            }
        }
    }

    public void PauseAllEntities()
    {
        var activeEntities = new List<IEntity>();
        _stateCollections[EntityState.Active].CopyTo(activeEntities);
        
        foreach (var entity in activeEntities)
        {
            PauseEntity(entity);
        }
    }

    public void ResumeAllEntities()
    {
        var pausedEntities = new List<IEntity>();
        _stateCollections[EntityState.Paused].CopyTo(pausedEntities);
        
        foreach (var entity in pausedEntities)
        {
            ResumeEntity(entity);
        }
    }

    private void OnEntityStateChanged(IEntity entity, EntityState newState)
    {
        Debug.Log($"Entity {entity.Name} entered state: {newState}");
        
        // Apply state-specific behavior
        switch (newState)
        {
            case EntityState.Active:
                entity.AddTag("Active");
                entity.Spawn();
                break;
            case EntityState.Inactive:
                entity.RemoveTag("Active");
                entity.Despawn();
                break;
            case EntityState.Paused:
                entity.AddTag("Paused");
                break;
            case EntityState.Destroyed:
                entity.AddTag("Destroyed");
                entity.Dispose();
                break;
        }
        
        GameEvents.OnEntityStateChanged?.Invoke(entity, newState.ToString());
    }

    private void OnEntityLeftState(IEntity entity, EntityState oldState)
    {
        Debug.Log($"Entity {entity.Name} left state: {oldState}");
        
        // Remove state-specific tags
        switch (oldState)
        {
            case EntityState.Active:
                entity.RemoveTag("Active");
                break;
            case EntityState.Paused:
                entity.RemoveTag("Paused");
                break;
        }
    }

    public StateStatistics GetStatistics()
    {
        var stats = new StateStatistics();
        
        foreach (var kvp in _stateCollections)
        {
            stats.StateCounts[kvp.Key.ToString()] = kvp.Value.Count;
        }
        
        stats.TotalEntities = _entityStates.Count;
        return stats;
    }

    void OnDestroy()
    {
        foreach (var collection in _stateCollections.Values)
        {
            collection.Dispose();
        }
    }
}

[System.Serializable]
public struct StateStatistics
{
    public int TotalEntities;
    public Dictionary<string, int> StateCounts;
    
    public StateStatistics()
    {
        TotalEntities = 0;
        StateCounts = new Dictionary<string, int>();
    }
}
```

## Integration with Atomic Framework

### Collection-Based Entity Filter

```csharp
public class CollectionEntityFilter : MonoBehaviour
{
    private readonly EntityCollection<IEntity> _filteredEntities = new();
    private readonly Dictionary<IEntity, bool> _lastEvaluationResults = new();
    
    [SerializeField] private string[] _requiredTags;
    [SerializeField] private string[] _forbiddenTags;
    [SerializeField] private bool _autoReEvaluate = true;
    [SerializeField] private float _reEvaluationInterval = 1f;
    
    private float _lastReEvaluationTime;

    void Start()
    {
        _filteredEntities.OnAdded += OnEntityPassedFilter;
        _filteredEntities.OnRemoved += OnEntityFailedFilter;
        
        // Initial evaluation
        ReEvaluateAllEntities();
    }

    void Update()
    {
        if (_autoReEvaluate && Time.time - _lastReEvaluationTime >= _reEvaluationInterval)
        {
            ReEvaluateAllEntities();
            _lastReEvaluationTime = Time.time;
        }
    }

    public void ReEvaluateAllEntities()
    {
        EntityRegistry.ForEach(entity =>
        {
            EvaluateEntity(entity);
        });
    }

    public void EvaluateEntity(IEntity entity)
    {
        bool shouldInclude = PassesFilter(entity);
        bool currentlyIncluded = _filteredEntities.Contains(entity);
        bool wasIncluded = _lastEvaluationResults.GetValueOrDefault(entity, false);

        if (shouldInclude && !currentlyIncluded)
        {
            _filteredEntities.Add(entity);
        }
        else if (!shouldInclude && currentlyIncluded)
        {
            _filteredEntities.Remove(entity);
        }

        _lastEvaluationResults[entity] = shouldInclude;
    }

    private bool PassesFilter(IEntity entity)
    {
        // Check required tags
        foreach (string requiredTag in _requiredTags)
        {
            if (!entity.HasTag(requiredTag))
                return false;
        }

        // Check forbidden tags
        foreach (string forbiddenTag in _forbiddenTags)
        {
            if (entity.HasTag(forbiddenTag))
                return false;
        }

        return true;
    }

    private void OnEntityPassedFilter(IEntity entity)
    {
        Debug.Log($"Entity {entity.Name} passed filter");
        GameEvents.OnEntityFilterPassed?.Invoke(entity);
    }

    private void OnEntityFailedFilter(IEntity entity)
    {
        Debug.Log($"Entity {entity.Name} failed filter");
        GameEvents.OnEntityFilterFailed?.Invoke(entity);
    }

    public IReadOnlyCollection<IEntity> GetFilteredEntities()
    {
        var entities = new List<IEntity>();
        _filteredEntities.CopyTo(entities);
        return entities;
    }

    public int GetFilteredCount() => _filteredEntities.Count;

    void OnDestroy()
    {
        _filteredEntities.Dispose();
    }
}
```

## Implementation Notes

### Performance Characteristics
- **Add/Remove/Contains**: O(1) average time complexity through hash table implementation
- **Enumeration**: O(n) with guaranteed insertion order through linked list structure
- **Memory**: Efficient slot-based storage with minimal per-element overhead

### Thread Safety
- Collection is NOT thread-safe by default
- Events fired synchronously on the calling thread
- Consider external synchronization for multi-threaded scenarios

### Event Ordering
- Events fired in order: OnAdd/OnRemove → specific event → OnStateChanged
- Virtual methods OnAdd/OnRemove called before public events
- All events fired synchronously during the operation

## Best Practices

### Initialization
- Pre-allocate capacity when expected size is known
- Use appropriate constructor overloads for initial population
- Subscribe to events immediately after creation

### Event Management
- Always unsubscribe from events to prevent memory leaks
- Use weak references for long-lived event handlers
- Consider batching operations to reduce event frequency

### Performance Optimization
- Batch multiple operations when possible
- Use appropriate capacity to minimize resizing
- Consider disabling events during bulk operations

## Common Patterns

### Batch Operations Pattern

```csharp
public static class BatchOperations
{
    public static void BatchAdd<E>(EntityCollection<E> collection, IEnumerable<E> entities) 
        where E : IEntity
    {
        // Temporarily disable events for performance
        var onAdded = collection.OnAdded;
        var onStateChanged = collection.OnStateChanged;
        
        collection.OnAdded = null;
        collection.OnStateChanged = null;
        
        try
        {
            foreach (var entity in entities)
            {
                collection.Add(entity);
            }
        }
        finally
        {
            // Restore events and fire state changed once
            collection.OnAdded = onAdded;
            collection.OnStateChanged = onStateChanged;
            collection.OnStateChanged?.Invoke();
        }
    }
}
```

### Collection Synchronization Pattern

```csharp
public class SynchronizedCollections<E> where E : IEntity
{
    private readonly EntityCollection<E> _primary;
    private readonly EntityCollection<E> _secondary;
    
    public SynchronizedCollections(EntityCollection<E> primary, EntityCollection<E> secondary)
    {
        _primary = primary;
        _secondary = secondary;
        
        _primary.OnAdded += entity => _secondary.Add(entity);
        _primary.OnRemoved += entity => _secondary.Remove(entity);
    }
    
    public void Sync()
    {
        _secondary.Clear();
        foreach (var entity in _primary)
        {
            _secondary.Add(entity);
        }
    }
}
```

The `EntityCollection` provides a robust, high-performance foundation for entity storage and management within the Atomic framework, combining the benefits of hash table performance with ordered enumeration and comprehensive event notifications.