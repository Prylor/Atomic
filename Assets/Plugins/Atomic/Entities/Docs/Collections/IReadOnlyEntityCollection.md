# 🧩 IReadOnlyEntityCollection

A read-only interface for observable entity collections that provides access to entity presence, enumeration, and change notifications without allowing direct modification. Essential for exposing entity collections safely while maintaining reactive capabilities.

## Overview

`IReadOnlyEntityCollection` defines the contract for read-only access to entity collections with event notification capabilities. Perfect for scenarios where you need to expose entity collections to consumers who should observe but not modify the collection directly.

## Interface Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// A non-generic version of IReadOnlyEntityCollection{E} for collections of IEntity.
    /// </summary>
    public interface IReadOnlyEntityCollection : IReadOnlyEntityCollection<IEntity>
    {
    }
    
    /// <summary>
    /// Represents a read-only, observable collection of entities of type E.
    /// Provides access to entity presence, enumeration, and change events.
    /// </summary>
    /// <typeparam name="E">The type of entity contained in the collection. Must implement IEntity.</typeparam>
    public interface IReadOnlyEntityCollection<E> : IReadOnlyCollection<E> where E : IEntity
    {
        /// <summary>
        /// Occurs when collection state changed.
        /// </summary>
        event Action OnStateChanged;
        
        /// <summary>
        /// Occurs when an entity is added to the collection.
        /// </summary>
        event Action<E> OnAdded;

        /// <summary>
        /// Occurs when an entity is removed from the collection.
        /// </summary>
        event Action<E> OnRemoved;
        
        /// <summary>
        /// Determines whether the specified entity is currently present in the collection.
        /// </summary>
        /// <param name="entity">The entity to check for presence.</param>
        /// <returns>true if the entity is in the collection; otherwise, false.</returns>
        bool Contains(E entity);

        /// <summary>
        /// Copies all entities into the specified collection.
        /// </summary>
        /// <param name="results">The collection to populate with entities.</param>
        void CopyTo(ICollection<E> results);
        
        /// <summary>
        /// Copies all entities into the specified array starting at the given index.
        /// </summary>
        void CopyTo(E[] array, int arrayIndex);
    }
}
```

## Key Features

### Read-Only Safety
- No direct modification methods exposed
- Safe to pass to consumers without modification risk
- Maintains collection integrity while allowing observation

### Event Notifications
- State change notifications for reactive systems
- Individual entity add/remove notifications
- Non-intrusive observation patterns

### Standard Integration
- Implements IReadOnlyCollection<E> for .NET compatibility
- Full enumeration support with foreach loops
- LINQ compatibility for querying

### Type Safety
- Generic interface for specific entity types
- Non-generic convenience interface for general use
- Compile-time type checking for entity operations

## Usage Examples

### Basic Read-Only Collection Usage

```csharp
public class EntityDisplayManager : MonoBehaviour
{
    private IReadOnlyEntityCollection<IEntity> _entities;
    
    [SerializeField] private GameObject _entityDisplayPrefab;
    [SerializeField] private Transform _displayParent;
    
    private readonly Dictionary<IEntity, GameObject> _entityDisplays = new();

    public void Initialize(IReadOnlyEntityCollection<IEntity> entities)
    {
        _entities = entities;
        
        // Subscribe to collection events
        _entities.OnAdded += OnEntityAdded;
        _entities.OnRemoved += OnEntityRemoved;
        _entities.OnStateChanged += OnCollectionStateChanged;
        
        // Create displays for existing entities
        foreach (var entity in _entities)
        {
            CreateEntityDisplay(entity);
        }
        
        UpdateDisplayCount();
    }

    private void OnEntityAdded(IEntity entity)
    {
        CreateEntityDisplay(entity);
        UpdateDisplayCount();
    }

    private void OnEntityRemoved(IEntity entity)
    {
        DestroyEntityDisplay(entity);
        UpdateDisplayCount();
    }

    private void OnCollectionStateChanged()
    {
        Debug.Log($"Collection state changed. Total entities: {_entities.Count}");
        UpdateDisplayCount();
    }

    private void CreateEntityDisplay(IEntity entity)
    {
        if (_entityDisplays.ContainsKey(entity))
            return;

        var display = Instantiate(_entityDisplayPrefab, _displayParent);
        var displayComponent = display.GetComponent<EntityDisplay>();
        
        if (displayComponent != null)
        {
            displayComponent.SetEntity(entity);
        }
        
        _entityDisplays[entity] = display;
    }

    private void DestroyEntityDisplay(IEntity entity)
    {
        if (_entityDisplays.TryGetValue(entity, out var display))
        {
            _entityDisplays.Remove(entity);
            Destroy(display);
        }
    }

    private void UpdateDisplayCount()
    {
        // Update UI with current count
        UIManager.SetEntityCount(_entities.Count);
    }

    void OnDestroy()
    {
        if (_entities != null)
        {
            _entities.OnAdded -= OnEntityAdded;
            _entities.OnRemoved -= OnEntityRemoved;
            _entities.OnStateChanged -= OnCollectionStateChanged;
        }
    }
}
```

### Entity Status Monitor System

```csharp
public class EntityStatusMonitor : MonoBehaviour
{
    private IReadOnlyEntityCollection<IEntity> _monitoredEntities;
    
    [SerializeField] private float _updateInterval = 1f;
    [SerializeField] private bool _logHealthChanges = true;
    [SerializeField] private bool _trackStatistics = true;
    
    private float _lastUpdateTime;
    private EntityStatistics _statistics = new();
    private readonly Dictionary<IEntity, EntitySnapshot> _lastSnapshots = new();

    public void StartMonitoring(IReadOnlyEntityCollection<IEntity> entities)
    {
        if (_monitoredEntities != null)
            StopMonitoring();

        _monitoredEntities = entities;
        
        // Subscribe to collection changes
        _monitoredEntities.OnAdded += OnEntityAddedToMonitoring;
        _monitoredEntities.OnRemoved += OnEntityRemovedFromMonitoring;
        
        // Take initial snapshots
        foreach (var entity in _monitoredEntities)
        {
            TakeEntitySnapshot(entity);
        }
        
        Debug.Log($"Started monitoring {_monitoredEntities.Count} entities");
    }

    public void StopMonitoring()
    {
        if (_monitoredEntities != null)
        {
            _monitoredEntities.OnAdded -= OnEntityAddedToMonitoring;
            _monitoredEntities.OnRemoved -= OnEntityRemovedFromMonitoring;
            _monitoredEntities = null;
        }
        
        _lastSnapshots.Clear();
        Debug.Log("Stopped entity monitoring");
    }

    void Update()
    {
        if (_monitoredEntities == null)
            return;

        if (Time.time - _lastUpdateTime >= _updateInterval)
        {
            UpdateMonitoring();
            _lastUpdateTime = Time.time;
        }
    }

    private void UpdateMonitoring()
    {
        if (_trackStatistics)
        {
            UpdateStatistics();
        }
        
        foreach (var entity in _monitoredEntities)
        {
            MonitorEntityChanges(entity);
        }
    }

    private void UpdateStatistics()
    {
        _statistics = new EntityStatistics
        {
            TotalEntities = _monitoredEntities.Count,
            HealthyEntities = 0,
            CriticalEntities = 0,
            DeadEntities = 0,
            AverageHealth = 0f
        };

        float totalHealth = 0f;
        int healthEntityCount = 0;

        foreach (var entity in _monitoredEntities)
        {
            if (entity.HasValue("Health") && entity.HasValue("MaxHealth"))
            {
                float health = entity.Get<float>("Health");
                float maxHealth = entity.Get<float>("MaxHealth");
                float healthPercent = health / maxHealth;
                
                totalHealth += health;
                healthEntityCount++;

                if (healthPercent <= 0f)
                    _statistics.DeadEntities++;
                else if (healthPercent <= 0.25f)
                    _statistics.CriticalEntities++;
                else
                    _statistics.HealthyEntities++;
            }
        }

        _statistics.AverageHealth = healthEntityCount > 0 ? totalHealth / healthEntityCount : 0f;
        
        // Broadcast statistics update
        GameEvents.OnEntityStatisticsUpdated?.Invoke(_statistics);
    }

    private void MonitorEntityChanges(IEntity entity)
    {
        var currentSnapshot = CreateEntitySnapshot(entity);
        
        if (_lastSnapshots.TryGetValue(entity, out var lastSnapshot))
        {
            CompareSnapshots(entity, lastSnapshot, currentSnapshot);
        }
        
        _lastSnapshots[entity] = currentSnapshot;
    }

    private void CompareSnapshots(IEntity entity, EntitySnapshot oldSnapshot, EntitySnapshot newSnapshot)
    {
        // Check health changes
        if (_logHealthChanges && oldSnapshot.Health != newSnapshot.Health)
        {
            float healthDelta = newSnapshot.Health - oldSnapshot.Health;
            string changeType = healthDelta > 0 ? "gained" : "lost";
            
            Debug.Log($"{entity.Name} {changeType} {Mathf.Abs(healthDelta)} health " +
                     $"({newSnapshot.Health}/{newSnapshot.MaxHealth})");
                     
            if (newSnapshot.Health <= 0 && oldSnapshot.Health > 0)
            {
                GameEvents.OnEntityDied?.Invoke(entity);
            }
        }

        // Check position changes
        if (Vector3.Distance(oldSnapshot.Position, newSnapshot.Position) > 0.1f)
        {
            GameEvents.OnEntityMoved?.Invoke(entity, oldSnapshot.Position, newSnapshot.Position);
        }

        // Check tag changes
        var addedTags = newSnapshot.Tags.Except(oldSnapshot.Tags).ToList();
        var removedTags = oldSnapshot.Tags.Except(newSnapshot.Tags).ToList();
        
        foreach (string tag in addedTags)
        {
            Debug.Log($"{entity.Name} gained tag: {tag}");
        }
        
        foreach (string tag in removedTags)
        {
            Debug.Log($"{entity.Name} lost tag: {tag}");
        }
    }

    private EntitySnapshot CreateEntitySnapshot(IEntity entity)
    {
        return new EntitySnapshot
        {
            Health = entity.HasValue("Health") ? entity.Get<float>("Health") : 0f,
            MaxHealth = entity.HasValue("MaxHealth") ? entity.Get<float>("MaxHealth") : 0f,
            Position = entity.HasValue("Position") ? entity.Get<Vector3>("Position") : Vector3.zero,
            Tags = GetEntityTags(entity),
            IsAlive = !entity.HasTag("Dead")
        };
    }

    private void TakeEntitySnapshot(IEntity entity)
    {
        _lastSnapshots[entity] = CreateEntitySnapshot(entity);
    }

    private HashSet<string> GetEntityTags(IEntity entity)
    {
        // This would need to be implemented based on entity's tag system
        // Placeholder implementation
        return new HashSet<string>();
    }

    private void OnEntityAddedToMonitoring(IEntity entity)
    {
        TakeEntitySnapshot(entity);
        Debug.Log($"Started monitoring entity: {entity.Name}");
    }

    private void OnEntityRemovedFromMonitoring(IEntity entity)
    {
        _lastSnapshots.Remove(entity);
        Debug.Log($"Stopped monitoring entity: {entity.Name}");
    }

    public EntityStatistics GetStatistics() => _statistics;
    
    public bool IsMonitoring(IEntity entity) => _monitoredEntities?.Contains(entity) ?? false;
    
    public int GetMonitoredCount() => _monitoredEntities?.Count ?? 0;

    public IEntity[] GetEntitiesInCriticalCondition()
    {
        if (_monitoredEntities == null)
            return new IEntity[0];

        var criticalEntities = new List<IEntity>();
        
        foreach (var entity in _monitoredEntities)
        {
            if (IsEntityInCriticalCondition(entity))
            {
                criticalEntities.Add(entity);
            }
        }
        
        return criticalEntities.ToArray();
    }

    private bool IsEntityInCriticalCondition(IEntity entity)
    {
        if (!entity.HasValue("Health") || !entity.HasValue("MaxHealth"))
            return false;
            
        float health = entity.Get<float>("Health");
        float maxHealth = entity.Get<float>("MaxHealth");
        
        return health / maxHealth <= 0.25f;
    }

    void OnDestroy()
    {
        StopMonitoring();
    }
}

[System.Serializable]
public struct EntityStatistics
{
    public int TotalEntities;
    public int HealthyEntities;
    public int CriticalEntities;
    public int DeadEntities;
    public float AverageHealth;
}

[System.Serializable]
public struct EntitySnapshot
{
    public float Health;
    public float MaxHealth;
    public Vector3 Position;
    public HashSet<string> Tags;
    public bool IsAlive;
}
```

### Collection Analytics System

```csharp
public class CollectionAnalytics : MonoBehaviour
{
    private readonly Dictionary<string, IReadOnlyEntityCollection<IEntity>> _trackedCollections = new();
    private readonly Dictionary<string, CollectionMetrics> _metrics = new();
    
    [SerializeField] private float _metricsUpdateInterval = 5f;
    [SerializeField] private int _historySize = 100;
    [SerializeField] private bool _enableDetailedLogging = false;
    
    private float _lastMetricsUpdate;

    public void RegisterCollection(string name, IReadOnlyEntityCollection<IEntity> collection)
    {
        if (_trackedCollections.ContainsKey(name))
        {
            UnregisterCollection(name);
        }

        _trackedCollections[name] = collection;
        _metrics[name] = new CollectionMetrics(name, _historySize);

        // Subscribe to collection events
        collection.OnAdded += entity => OnCollectionEntityAdded(name, entity);
        collection.OnRemoved += entity => OnCollectionEntityRemoved(name, entity);
        collection.OnStateChanged += () => OnCollectionStateChanged(name);

        Debug.Log($"Registered collection '{name}' for analytics (Initial count: {collection.Count})");
    }

    public void UnregisterCollection(string name)
    {
        if (_trackedCollections.TryGetValue(name, out var collection))
        {
            // Unsubscribe from events
            collection.OnAdded -= entity => OnCollectionEntityAdded(name, entity);
            collection.OnRemoved -= entity => OnCollectionEntityRemoved(name, entity);
            collection.OnStateChanged -= () => OnCollectionStateChanged(name);
            
            _trackedCollections.Remove(name);
            _metrics.Remove(name);
            
            Debug.Log($"Unregistered collection '{name}' from analytics");
        }
    }

    void Update()
    {
        if (Time.time - _lastMetricsUpdate >= _metricsUpdateInterval)
        {
            UpdateAllMetrics();
            _lastMetricsUpdate = Time.time;
        }
    }

    private void UpdateAllMetrics()
    {
        foreach (var kvp in _trackedCollections)
        {
            UpdateCollectionMetrics(kvp.Key, kvp.Value);
        }
        
        if (_enableDetailedLogging)
        {
            LogDetailedMetrics();
        }
    }

    private void UpdateCollectionMetrics(string collectionName, IReadOnlyEntityCollection<IEntity> collection)
    {
        if (!_metrics.TryGetValue(collectionName, out var metrics))
            return;

        // Record current count
        metrics.RecordCount(collection.Count);
        
        // Calculate additional metrics
        metrics.UpdateAverages();
        metrics.CalculateTrends();
        
        // Update peak values
        if (collection.Count > metrics.PeakCount)
        {
            metrics.PeakCount = collection.Count;
            metrics.PeakTime = Time.time;
        }
    }

    private void OnCollectionEntityAdded(string collectionName, IEntity entity)
    {
        if (_metrics.TryGetValue(collectionName, out var metrics))
        {
            metrics.TotalAdditions++;
            metrics.LastAdditionTime = Time.time;
            
            if (_enableDetailedLogging)
            {
                Debug.Log($"[{collectionName}] Entity added: {entity.Name} (Total: {_trackedCollections[collectionName].Count})");
            }
        }
    }

    private void OnCollectionEntityRemoved(string collectionName, IEntity entity)
    {
        if (_metrics.TryGetValue(collectionName, out var metrics))
        {
            metrics.TotalRemovals++;
            metrics.LastRemovalTime = Time.time;
            
            if (_enableDetailedLogging)
            {
                Debug.Log($"[{collectionName}] Entity removed: {entity.Name} (Total: {_trackedCollections[collectionName].Count})");
            }
        }
    }

    private void OnCollectionStateChanged(string collectionName)
    {
        if (_metrics.TryGetValue(collectionName, out var metrics))
        {
            metrics.StateChangeCount++;
            
            if (_enableDetailedLogging)
            {
                Debug.Log($"[{collectionName}] State changed (Count: {_trackedCollections[collectionName].Count})");
            }
        }
    }

    private void LogDetailedMetrics()
    {
        foreach (var kvp in _metrics)
        {
            var metrics = kvp.Value;
            Debug.Log($"Collection '{kvp.Key}' Metrics:\n" +
                     $"  Current: {_trackedCollections[kvp.Key].Count}\n" +
                     $"  Peak: {metrics.PeakCount} at {metrics.PeakTime:F1}s\n" +
                     $"  Average: {metrics.AverageCount:F1}\n" +
                     $"  Additions: {metrics.TotalAdditions}, Removals: {metrics.TotalRemovals}\n" +
                     $"  State Changes: {metrics.StateChangeCount}\n" +
                     $"  Trend: {metrics.Trend}");
        }
    }

    public CollectionMetrics GetMetrics(string collectionName)
    {
        return _metrics.GetValueOrDefault(collectionName, null);
    }

    public Dictionary<string, CollectionMetrics> GetAllMetrics()
    {
        return new Dictionary<string, CollectionMetrics>(_metrics);
    }

    public void ResetMetrics(string collectionName = null)
    {
        if (collectionName != null)
        {
            if (_metrics.TryGetValue(collectionName, out var metrics))
            {
                metrics.Reset();
            }
        }
        else
        {
            foreach (var metrics in _metrics.Values)
            {
                metrics.Reset();
            }
        }
    }

    void OnDestroy()
    {
        var collectionNames = _trackedCollections.Keys.ToArray();
        foreach (var name in collectionNames)
        {
            UnregisterCollection(name);
        }
    }
}

public class CollectionMetrics
{
    public string CollectionName { get; private set; }
    public int PeakCount { get; set; }
    public float PeakTime { get; set; }
    public int TotalAdditions { get; set; }
    public int TotalRemovals { get; set; }
    public int StateChangeCount { get; set; }
    public float LastAdditionTime { get; set; }
    public float LastRemovalTime { get; set; }
    public float AverageCount { get; private set; }
    public CollectionTrend Trend { get; private set; }

    private readonly Queue<int> _countHistory;
    private readonly int _maxHistorySize;

    public CollectionMetrics(string name, int historySize = 100)
    {
        CollectionName = name;
        _maxHistorySize = historySize;
        _countHistory = new Queue<int>(historySize);
        Reset();
    }

    public void RecordCount(int count)
    {
        _countHistory.Enqueue(count);
        
        while _countHistory.Count > _maxHistorySize)
        {
            _countHistory.Dequeue();
        }
    }

    public void UpdateAverages()
    {
        if (_countHistory.Count == 0)
        {
            AverageCount = 0f;
            return;
        }

        AverageCount = _countHistory.Average();
    }

    public void CalculateTrends()
    {
        if (_countHistory.Count < 2)
        {
            Trend = CollectionTrend.Stable;
            return;
        }

        var recent = _countHistory.TakeLast(Math.Min(10, _countHistory.Count)).ToArray();
        
        if (recent.Length < 2)
        {
            Trend = CollectionTrend.Stable;
            return;
        }

        float slope = CalculateSlope(recent);
        
        if (slope > 0.5f)
            Trend = CollectionTrend.Growing;
        else if (slope < -0.5f)
            Trend = CollectionTrend.Declining;
        else
            Trend = CollectionTrend.Stable;
    }

    private float CalculateSlope(int[] values)
    {
        if (values.Length < 2) return 0f;
        
        float n = values.Length;
        float sumX = 0f, sumY = 0f, sumXY = 0f, sumXX = 0f;
        
        for (int i = 0; i < values.Length; i++)
        {
            float x = i;
            float y = values[i];
            
            sumX += x;
            sumY += y;
            sumXY += x * y;
            sumXX += x * x;
        }
        
        return (n * sumXY - sumX * sumY) / (n * sumXX - sumX * sumX);
    }

    public void Reset()
    {
        PeakCount = 0;
        PeakTime = 0f;
        TotalAdditions = 0;
        TotalRemovals = 0;
        StateChangeCount = 0;
        LastAdditionTime = 0f;
        LastRemovalTime = 0f;
        AverageCount = 0f;
        Trend = CollectionTrend.Stable;
        _countHistory.Clear();
    }
}

public enum CollectionTrend
{
    Declining,
    Stable,
    Growing
}
```

## Implementation Notes

### Event Safety
- Events provide read-only access to collection changes
- No risk of external modification through event handlers
- Events fired synchronously on the modifying thread

### Performance Characteristics
- Contains(): O(1) for hash-based implementations
- Enumeration: O(n) with no modification overhead
- CopyTo(): O(n) with potential allocation for target collections

### Thread Safety
- Interface itself doesn't guarantee thread safety
- Implementations may or may not be thread-safe
- Check specific implementation documentation

## Best Practices

### Event Subscription
- Always unsubscribe from events to prevent memory leaks
- Use weak event patterns for long-lived subscribers
- Consider event aggregation for high-frequency updates

### Collection Usage
- Use Contains() instead of enumeration for membership tests
- Cache results when possible to avoid repeated operations
- Consider copying to arrays/lists for expensive operations

### Performance Considerations
- CopyTo() methods may allocate memory
- Event handlers should be lightweight
- Avoid expensive operations in event callbacks

## Common Patterns

### Collection Wrapper Pattern

```csharp
public class ReadOnlyCollectionWrapper<E> : IReadOnlyEntityCollection<E> where E : IEntity
{
    private readonly IEntityCollection<E> _innerCollection;

    public ReadOnlyCollectionWrapper(IEntityCollection<E> collection)
    {
        _innerCollection = collection ?? throw new ArgumentNullException(nameof(collection));
    }

    public event Action OnStateChanged
    {
        add => _innerCollection.OnStateChanged += value;
        remove => _innerCollection.OnStateChanged -= value;
    }

    public event Action<E> OnAdded
    {
        add => _innerCollection.OnAdded += value;
        remove => _innerCollection.OnAdded -= value;
    }

    public event Action<E> OnRemoved
    {
        add => _innerCollection.OnRemoved += value;
        remove => _innerCollection.OnRemoved -= value;
    }

    public int Count => _innerCollection.Count;
    public bool Contains(E entity) => _innerCollection.Contains(entity);
    public void CopyTo(ICollection<E> results) => _innerCollection.CopyTo(results);
    public void CopyTo(E[] array, int arrayIndex) => _innerCollection.CopyTo(array, arrayIndex);
    public IEnumerator<E> GetEnumerator() => _innerCollection.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

### Event Aggregation Pattern

```csharp
public class CollectionEventAggregator
{
    private readonly List<IReadOnlyEntityCollection<IEntity>> _collections = new();
    
    public event Action<IEntity> OnAnyEntityAdded;
    public event Action<IEntity> OnAnyEntityRemoved;
    public event Action OnAnyCollectionStateChanged;

    public void RegisterCollection(IReadOnlyEntityCollection<IEntity> collection)
    {
        if (_collections.Contains(collection))
            return;

        _collections.Add(collection);
        
        collection.OnAdded += OnAnyEntityAdded;
        collection.OnRemoved += OnAnyEntityRemoved;
        collection.OnStateChanged += OnAnyCollectionStateChanged;
    }

    public void UnregisterCollection(IReadOnlyEntityCollection<IEntity> collection)
    {
        if (_collections.Remove(collection))
        {
            collection.OnAdded -= OnAnyEntityAdded;
            collection.OnRemoved -= OnAnyEntityRemoved;
            collection.OnStateChanged -= OnAnyCollectionStateChanged;
        }
    }
}
```

The `IReadOnlyEntityCollection` interface provides safe, observable access to entity collections, enabling reactive systems and monitoring capabilities without compromising collection integrity within the Atomic framework.