# 🧩 IEntityTrigger

A base interface for entity trigger systems that monitor specific aspects of entity state and signal when entities should be re-evaluated by filters or other systems. Provides the foundation for reactive entity monitoring and automatic system updates.

## Overview

`IEntityTrigger` defines the contract for trigger systems that monitor entity changes and trigger re-evaluation callbacks. Essential for building reactive systems that automatically respond to entity state changes without manual polling or update loops.

## Interface Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic shorthand for IEntityTrigger&lt;IEntity&gt;.
    /// </summary>
    public interface IEntityTrigger : IEntityTrigger<IEntity>
    {
    }

    /// <summary>
    /// Represents a trigger that monitors entity state aspects and signals re-evaluation.
    /// </summary>
    /// <typeparam name="E">The entity type being tracked</typeparam>
    public interface IEntityTrigger<E> where E : IEntity
    {
        /// <summary>
        /// Sets the callback to invoke when tracked entities should be re-evaluated.
        /// </summary>
        void SetAction(Action<E> action);

        /// <summary>
        /// Starts tracking the specified entity for relevant changes.
        /// </summary>
        void Track(E entity);

        /// <summary>
        /// Stops tracking the specified entity.
        /// </summary>
        void Untrack(E entity);
    }
}
```

## Key Features

### Action-Based Callbacks
- Configurable callback system for re-evaluation notifications
- Generic action support for type-safe entity handling
- Flexible trigger response patterns

### Entity Tracking
- Track/untrack lifecycle for entity monitoring
- Multiple entity support per trigger instance
- Clean resource management patterns

### Type Safety
- Generic interface for specific entity types
- Compile-time type checking for callbacks
- Non-generic convenience interface available

## Usage Examples

### Basic Trigger Implementation

```csharp
public class BasicEntityTrigger : IEntityTrigger
{
    private Action<IEntity> _callback;
    private readonly HashSet<IEntity> _trackedEntities = new();
    
    public void SetAction(Action<IEntity> action)
    {
        _callback = action ?? throw new ArgumentNullException(nameof(action));
    }
    
    public void Track(IEntity entity)
    {
        if (_trackedEntities.Add(entity))
        {
            // Subscribe to entity events
            entity.OnHealthChanged += OnEntityHealthChanged;
            entity.OnTagAdded += OnEntityTagAdded;
        }
    }
    
    public void Untrack(IEntity entity)
    {
        if (_trackedEntities.Remove(entity))
        {
            // Unsubscribe from entity events
            entity.OnHealthChanged -= OnEntityHealthChanged;
            entity.OnTagAdded -= OnEntityTagAdded;
        }
    }
    
    private void OnEntityHealthChanged(IEntity entity, float newHealth)
    {
        // Trigger re-evaluation when health changes
        _callback?.Invoke(entity);
    }
    
    private void OnEntityTagAdded(IEntity entity, string tag)
    {
        // Trigger re-evaluation when tags are added
        _callback?.Invoke(entity);
    }
}
```

### Filter Integration System

```csharp
public class ReactiveEntityFilter : IDisposable
{
    private readonly List<IEntity> _matchedEntities = new();
    private readonly List<IEntityTrigger> _triggers = new();
    private readonly Predicate<IEntity> _filterPredicate;
    
    public ReactiveEntityFilter(Predicate<IEntity> predicate)
    {
        _filterPredicate = predicate ?? throw new ArgumentNullException(nameof(predicate));
        
        // Setup triggers for automatic re-evaluation
        SetupTriggers();
    }
    
    private void SetupTriggers()
    {
        // Create triggers for different entity change types
        var tagTrigger = new TagEntityTrigger(added: true, deleted: true);
        var valueTrigger = new ValueEntityTrigger(added: true, deleted: true, changed: true);
        
        // Set callback for entity re-evaluation
        tagTrigger.SetAction(OnEntityChanged);
        valueTrigger.SetAction(OnEntityChanged);
        
        _triggers.Add(tagTrigger);
        _triggers.Add(valueTrigger);
    }
    
    public void AddEntity(IEntity entity)
    {
        if (_filterPredicate(entity) && !_matchedEntities.Contains(entity))
        {
            _matchedEntities.Add(entity);
            
            // Start tracking with all triggers
            foreach (var trigger in _triggers)
            {
                trigger.Track(entity);
            }
            
            OnEntityAdded?.Invoke(entity);
        }
    }
    
    public void RemoveEntity(IEntity entity)
    {
        if (_matchedEntities.Remove(entity))
        {
            // Stop tracking with all triggers
            foreach (var trigger in _triggers)
            {
                trigger.Untrack(entity);
            }
            
            OnEntityRemoved?.Invoke(entity);
        }
    }
    
    private void OnEntityChanged(IEntity entity)
    {
        bool currentlyMatches = _filterPredicate(entity);
        bool wasMatching = _matchedEntities.Contains(entity);
        
        if (currentlyMatches && !wasMatching)
        {
            // Entity now matches - add to filter
            _matchedEntities.Add(entity);
            OnEntityAdded?.Invoke(entity);
        }
        else if (!currentlyMatches && wasMatching)
        {
            // Entity no longer matches - remove from filter
            _matchedEntities.Remove(entity);
            OnEntityRemoved?.Invoke(entity);
        }
    }
    
    public IReadOnlyList<IEntity> MatchedEntities => _matchedEntities;
    
    public event Action<IEntity> OnEntityAdded;
    public event Action<IEntity> OnEntityRemoved;
    
    public void Dispose()
    {
        // Clean up all tracked entities
        foreach (var entity in _matchedEntities.ToArray())
        {
            RemoveEntity(entity);
        }
        
        _matchedEntities.Clear();
        _triggers.Clear();
    }
}
```

### Custom Health Monitor Trigger

```csharp
public class HealthMonitorTrigger : IEntityTrigger
{
    private Action<IEntity> _callback;
    private readonly Dictionary<IEntity, float> _lastHealthValues = new();
    private readonly float _healthThreshold;
    
    public HealthMonitorTrigger(float healthThreshold = 0.25f)
    {
        _healthThreshold = healthThreshold;
    }
    
    public void SetAction(Action<IEntity> action)
    {
        _callback = action ?? throw new ArgumentNullException(nameof(action));
    }
    
    public void Track(IEntity entity)
    {
        if (entity.HasValue("Health"))
        {
            float currentHealth = entity.Get<float>("Health");
            _lastHealthValues[entity] = currentHealth;
            
            // Subscribe to health changes
            entity.OnValueChanged += OnEntityValueChanged;
        }
    }
    
    public void Untrack(IEntity entity)
    {
        _lastHealthValues.Remove(entity);
        entity.OnValueChanged -= OnEntityValueChanged;
    }
    
    private void OnEntityValueChanged(IEntity entity, string key)
    {
        if (key == "Health" && _lastHealthValues.TryGetValue(entity, out float lastHealth))
        {
            float currentHealth = entity.Get<float>("Health");
            float maxHealth = entity.Get<float>("MaxHealth");
            
            float lastHealthPercent = lastHealth / maxHealth;
            float currentHealthPercent = currentHealth / maxHealth;
            
            // Trigger if health crossed the threshold
            bool wasAboveThreshold = lastHealthPercent > _healthThreshold;
            bool isAboveThreshold = currentHealthPercent > _healthThreshold;
            
            if (wasAboveThreshold != isAboveThreshold)
            {
                _callback?.Invoke(entity);
            }
            
            _lastHealthValues[entity] = currentHealth;
        }
    }
}
```

### Multi-Condition Trigger System

```csharp
public class MultiConditionTrigger : IEntityTrigger
{
    private Action<IEntity> _callback;
    private readonly Dictionary<IEntity, EntityState> _entityStates = new();
    
    public struct EntityState
    {
        public bool hasRequiredTags;
        public bool hasMinimumHealth;
        public bool hasRequiredLevel;
        public float lastCheckTime;
    }
    
    public void SetAction(Action<IEntity> action)
    {
        _callback = action ?? throw new ArgumentNullException(nameof(action));
    }
    
    public void Track(IEntity entity)
    {
        if (!_entityStates.ContainsKey(entity))
        {
            // Initialize entity state
            var state = new EntityState
            {
                hasRequiredTags = CheckRequiredTags(entity),
                hasMinimumHealth = CheckMinimumHealth(entity),
                hasRequiredLevel = CheckRequiredLevel(entity),
                lastCheckTime = Time.time
            };
            
            _entityStates[entity] = state;
            
            // Subscribe to relevant events
            entity.OnTagAdded += OnEntityChanged;
            entity.OnTagDeleted += OnEntityChanged;
            entity.OnValueChanged += OnEntityValueChanged;
        }
    }
    
    public void Untrack(IEntity entity)
    {
        if (_entityStates.Remove(entity))
        {
            entity.OnTagAdded -= OnEntityChanged;
            entity.OnTagDeleted -= OnEntityChanged;
            entity.OnValueChanged -= OnEntityValueChanged;
        }
    }
    
    private void OnEntityChanged(IEntity entity, int tagId)
    {
        EvaluateEntity(entity);
    }
    
    private void OnEntityValueChanged(IEntity entity, int key)
    {
        // Only evaluate for specific value changes
        string keyName = entity.GetValueKeyName(key);
        if (keyName == "Health" || keyName == "Level")
        {
            EvaluateEntity(entity);
        }
    }
    
    private void EvaluateEntity(IEntity entity)
    {
        if (!_entityStates.TryGetValue(entity, out var state))
            return;
        
        // Check all conditions
        bool newRequiredTags = CheckRequiredTags(entity);
        bool newMinimumHealth = CheckMinimumHealth(entity);
        bool newRequiredLevel = CheckRequiredLevel(entity);
        
        // Check if any condition changed
        if (newRequiredTags != state.hasRequiredTags ||
            newMinimumHealth != state.hasMinimumHealth ||
            newRequiredLevel != state.hasRequiredLevel)
        {
            // Update state
            state.hasRequiredTags = newRequiredTags;
            state.hasMinimumHealth = newMinimumHealth;
            state.hasRequiredLevel = newRequiredLevel;
            state.lastCheckTime = Time.time;
            
            _entityStates[entity] = state;
            
            // Trigger callback
            _callback?.Invoke(entity);
        }
    }
    
    private bool CheckRequiredTags(IEntity entity)
    {
        return entity.HasTag("Active") && entity.HasTag("Player");
    }
    
    private bool CheckMinimumHealth(IEntity entity)
    {
        if (!entity.HasValue("Health") || !entity.HasValue("MaxHealth"))
            return false;
        
        float health = entity.Get<float>("Health");
        float maxHealth = entity.Get<float>("MaxHealth");
        return health >= (maxHealth * 0.5f);
    }
    
    private bool CheckRequiredLevel(IEntity entity)
    {
        if (!entity.HasValue("Level"))
            return false;
        
        int level = entity.Get<int>("Level");
        return level >= 5;
    }
}
```

## Integration with Atomic Framework

### Reactive Filter System

```csharp
public class ReactiveEntitySystem : MonoBehaviour
{
    [System.Serializable]
    public struct FilterConfiguration
    {
        public string filterName;
        public string[] requiredTags;
        public string[] forbiddenTags;
        public KeyValuePair<string, object>[] requiredValues;
    }
    
    [SerializeField] private FilterConfiguration[] _filterConfigs;
    
    private readonly Dictionary<string, ReactiveEntityFilter> _filters = new();
    private readonly ReactiveCollection<IEntity> _allEntities = new();
    
    void Start()
    {
        SetupFilters();
        
        // React to global entity changes
        _allEntities.OnAdded += OnEntityAddedToSystem;
        _allEntities.OnRemoved += OnEntityRemovedFromSystem;
        
        GameEvents.OnEntityCreated += _allEntities.Add;
        GameEvents.OnEntityDestroyed += _allEntities.Remove;
    }
    
    private void SetupFilters()
    {
        foreach (var config in _filterConfigs)
        {
            var predicate = CreatePredicateFromConfig(config);
            var filter = new ReactiveEntityFilter(predicate);
            
            filter.OnEntityAdded += entity => OnEntityMatchedFilter(config.filterName, entity);
            filter.OnEntityRemoved += entity => OnEntityUnmatchedFilter(config.filterName, entity);
            
            _filters[config.filterName] = filter;
        }
    }
    
    private Predicate<IEntity> CreatePredicateFromConfig(FilterConfiguration config)
    {
        return entity =>
        {
            // Check required tags
            foreach (string tag in config.requiredTags)
            {
                if (!entity.HasTag(tag)) return false;
            }
            
            // Check forbidden tags
            foreach (string tag in config.forbiddenTags)
            {
                if (entity.HasTag(tag)) return false;
            }
            
            // Check required values
            foreach (var kvp in config.requiredValues)
            {
                if (!entity.HasValue(kvp.Key)) return false;
                
                var entityValue = entity.Get<object>(kvp.Key);
                if (!entityValue.Equals(kvp.Value)) return false;
            }
            
            return true;
        };
    }
    
    private void OnEntityAddedToSystem(IEntity entity)
    {
        // Add entity to all applicable filters
        foreach (var filter in _filters.Values)
        {
            filter.AddEntity(entity);
        }
    }
    
    private void OnEntityRemovedFromSystem(IEntity entity)
    {
        // Remove entity from all filters
        foreach (var filter in _filters.Values)
        {
            filter.RemoveEntity(entity);
        }
    }
    
    private void OnEntityMatchedFilter(string filterName, IEntity entity)
    {
        Debug.Log($"Entity {entity.Name} matched filter: {filterName}");
        GameEvents.OnEntityMatchedFilter?.Invoke(filterName, entity);
    }
    
    private void OnEntityUnmatchedFilter(string filterName, IEntity entity)
    {
        Debug.Log($"Entity {entity.Name} unmatched filter: {filterName}");
        GameEvents.OnEntityUnmatchedFilter?.Invoke(filterName, entity);
    }
    
    public IReadOnlyList<IEntity> GetFilteredEntities(string filterName)
    {
        return _filters.TryGetValue(filterName, out var filter) 
            ? filter.MatchedEntities 
            : new List<IEntity>();
    }
    
    public void AddEntityToSystem(IEntity entity)
    {
        _allEntities.Add(entity);
    }
    
    public void RemoveEntityFromSystem(IEntity entity)
    {
        _allEntities.Remove(entity);
    }
    
    void OnDestroy()
    {
        foreach (var filter in _filters.Values)
        {
            filter.Dispose();
        }
        _filters.Clear();
    }
}
```

### Performance-Optimized Trigger Manager

```csharp
public class TriggerManager : MonoBehaviour
{
    private readonly Dictionary<Type, List<IEntityTrigger>> _triggersByType = new();
    private readonly Dictionary<IEntity, List<IEntityTrigger>> _triggersByEntity = new();
    
    [Header("Performance Settings")]
    [SerializeField] private int _maxTriggersPerFrame = 50;
    [SerializeField] private float _triggerProcessingInterval = 0.016f; // ~60 FPS
    
    private readonly Queue<(IEntity entity, Action action)> _triggerQueue = new();
    
    void Start()
    {
        // Start trigger processing coroutine
        StartCoroutine(ProcessTriggerQueue());
    }
    
    public void RegisterTrigger<T>(IEntityTrigger<T> trigger) where T : IEntity
    {
        var triggerType = typeof(T);
        
        if (!_triggersByType.TryGetValue(triggerType, out var triggers))
        {
            triggers = new List<IEntityTrigger>();
            _triggersByType[triggerType] = triggers;
        }
        
        triggers.Add(trigger as IEntityTrigger);
        
        // Set up batched callback
        trigger.SetAction(entity => QueueTriggerCallback(entity, () => OnEntityTriggered(entity)));
    }
    
    public void TrackEntity(IEntity entity)
    {
        var entityType = entity.GetType();
        var applicableTriggers = new List<IEntityTrigger>();
        
        // Find all applicable triggers for this entity type
        foreach (var kvp in _triggersByType)
        {
            if (kvp.Key.IsAssignableFrom(entityType))
            {
                foreach (var trigger in kvp.Value)
                {
                    trigger.Track(entity);
                    applicableTriggers.Add(trigger);
                }
            }
        }
        
        if (applicableTriggers.Count > 0)
        {
            _triggersByEntity[entity] = applicableTriggers;
        }
    }
    
    public void UntrackEntity(IEntity entity)
    {
        if (_triggersByEntity.TryGetValue(entity, out var triggers))
        {
            foreach (var trigger in triggers)
            {
                trigger.Untrack(entity);
            }
            
            _triggersByEntity.Remove(entity);
        }
    }
    
    private void QueueTriggerCallback(IEntity entity, Action action)
    {
        _triggerQueue.Enqueue((entity, action));
    }
    
    private IEnumerator ProcessTriggerQueue()
    {
        while (true)
        {
            int processedCount = 0;
            
            while (_triggerQueue.Count > 0 && processedCount < _maxTriggersPerFrame)
            {
                var (entity, action) = _triggerQueue.Dequeue();
                
                try
                {
                    action.Invoke();
                }
                catch (Exception ex)
                {
                    Debug.LogError($"Error processing trigger for entity {entity?.Name}: {ex.Message}");
                }
                
                processedCount++;
            }
            
            yield return new WaitForSeconds(_triggerProcessingInterval);
        }
    }
    
    private void OnEntityTriggered(IEntity entity)
    {
        // Custom trigger processing logic
        Debug.Log($"Entity triggered: {entity.Name}");
        GameEvents.OnEntityTriggered?.Invoke(entity);
    }
    
    // Statistics and monitoring
    public int GetQueuedTriggerCount() => _triggerQueue.Count;
    public int GetTrackedEntityCount() => _triggersByEntity.Count;
    public int GetRegisteredTriggerCount() => _triggersByType.Values.Sum(list => list.Count);
}
```

## Implementation Notes

### Callback Management
- Action callbacks provide flexible response mechanisms
- Null safety checks prevent callback errors
- Generic actions enable type-safe entity handling

### Resource Management
- Track/untrack methods manage subscription lifecycles
- Clean unsubscription prevents memory leaks
- Multiple entity support per trigger instance

### Performance Considerations
- Efficient entity tracking with minimal overhead
- Event-driven architecture avoids polling
- Batched processing for high-frequency triggers

## Best Practices

### Trigger Design
- Keep trigger logic focused and specific
- Implement proper cleanup in untrack methods
- Use appropriate trigger types for different scenarios

### Callback Implementation
- Handle null callbacks gracefully
- Keep callback execution lightweight
- Consider batching for high-frequency triggers

### Resource Management
- Always pair track calls with untrack calls
- Implement proper disposal patterns
- Monitor trigger performance in complex systems

## Common Patterns

### Trigger Factory Pattern

```csharp
public static class TriggerFactory
{
    public static IEntityTrigger CreateHealthTrigger(float threshold)
    {
        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                entity.OnValueChanged += (e, key) =>
                {
                    if (key == "Health")
                    {
                        float health = e.Get<float>("Health");
                        float maxHealth = e.Get<float>("MaxHealth");
                        if (health / maxHealth <= threshold)
                        {
                            callback(e);
                        }
                    }
                };
            },
            untrack: (entity, callback) =>
            {
                entity.OnValueChanged -= (e, key) => callback(e);
            }
        );
    }
}
```

### Composite Trigger

```csharp
public class CompositeTrigger : IEntityTrigger
{
    private readonly List<IEntityTrigger> _triggers = new();
    private Action<IEntity> _callback;
    
    public void AddTrigger(IEntityTrigger trigger)
    {
        _triggers.Add(trigger);
        if (_callback != null)
        {
            trigger.SetAction(_callback);
        }
    }
    
    public void SetAction(Action<IEntity> action)
    {
        _callback = action;
        foreach (var trigger in _triggers)
        {
            trigger.SetAction(action);
        }
    }
    
    public void Track(IEntity entity)
    {
        foreach (var trigger in _triggers)
        {
            trigger.Track(entity);
        }
    }
    
    public void Untrack(IEntity entity)
    {
        foreach (var trigger in _triggers)
        {
            trigger.Untrack(entity);
        }
    }
}
```

The `IEntityTrigger` interface provides the foundation for building sophisticated reactive entity monitoring systems, enabling automatic and efficient responses to entity state changes within the Atomic framework's reactive architecture.