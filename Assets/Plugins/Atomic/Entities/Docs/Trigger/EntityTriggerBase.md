# 🧩 EntityTriggerBase

An abstract base class implementation of `IEntityTrigger` that provides common functionality for entity trigger systems, including action management and optimized callback invocation. Serves as the foundation for building custom entity triggers with shared infrastructure.

## Overview

`EntityTriggerBase` provides a standardized base implementation for entity triggers, handling action storage, null safety, and callback optimization. Designed to simplify custom trigger development while maintaining high performance and consistent behavior patterns.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic base for IEntity triggers.
    /// </summary>
    public abstract class EntityTriggerBase : EntityTriggerBase<IEntity>
    {
    }

    /// <summary>
    /// Abstract base implementation for entity triggers providing common infrastructure.
    /// </summary>
    /// <typeparam name="E">The type of entity being tracked</typeparam>
    public abstract class EntityTriggerBase<E> : IEntityTrigger<E> where E : IEntity
    {
        private protected Action<E> _action;

        public void SetAction(Action<E> action) => 
            _action = action ?? throw new ArgumentNullException(nameof(action));

        public abstract void Track(E entity);
        public abstract void Untrack(E entity);

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        protected void InvokeAction(E entity) => _action?.Invoke(entity);
    }
}
```

## Key Features

### Common Infrastructure
- Standardized action storage and management
- Null safety validation for callback actions
- Performance-optimized callback invocation

### Abstract Template Pattern
- Template methods for Track and Untrack operations
- Shared callback management across implementations
- Consistent behavior patterns for all derived triggers

### Performance Optimization
- Aggressive inlining for callback invocation
- Minimal overhead for trigger operations
- Efficient null-safe callback execution

## Usage Examples

### Custom Health Monitor Trigger

```csharp
public class HealthMonitorTrigger : EntityTriggerBase
{
    private readonly float _criticalThreshold;
    private readonly Dictionary<IEntity, bool> _criticalStates = new();
    
    public HealthMonitorTrigger(float criticalThreshold = 0.2f)
    {
        _criticalThreshold = criticalThreshold;
    }
    
    public override void Track(IEntity entity)
    {
        if (entity.HasValue("Health") && entity.HasValue("MaxHealth"))
        {
            // Initialize critical state
            float health = entity.Get<float>("Health");
            float maxHealth = entity.Get<float>("MaxHealth");
            bool isCritical = (health / maxHealth) <= _criticalThreshold;
            _criticalStates[entity] = isCritical;
            
            // Subscribe to health changes
            entity.OnValueChanged += OnHealthChanged;
        }
    }
    
    public override void Untrack(IEntity entity)
    {
        _criticalStates.Remove(entity);
        entity.OnValueChanged -= OnHealthChanged;
    }
    
    private void OnHealthChanged(IEntity entity, int key)
    {
        // Check if this is a health change
        string keyName = entity.GetKeyName(key);
        if (keyName != "Health") return;
        
        if (_criticalStates.TryGetValue(entity, out bool wasCritical))
        {
            float health = entity.Get<float>("Health");
            float maxHealth = entity.Get<float>("MaxHealth");
            bool isCritical = (health / maxHealth) <= _criticalThreshold;
            
            // Check for critical state change
            if (isCritical != wasCritical)
            {
                _criticalStates[entity] = isCritical;
                
                // Trigger callback using base class method
                InvokeAction(entity);
            }
        }
    }
}
```

### Area-Based Proximity Trigger

```csharp
public class ProximityTrigger : EntityTriggerBase
{
    private readonly Vector3 _center;
    private readonly float _radius;
    private readonly Dictionary<IEntity, bool> _proximityStates = new();
    
    public ProximityTrigger(Vector3 center, float radius)
    {
        _center = center;
        _radius = radius;
    }
    
    public override void Track(IEntity entity)
    {
        if (entity.HasValue("Position"))
        {
            // Initialize proximity state
            Vector3 position = entity.Get<Vector3>("Position");
            bool isInRange = Vector3.Distance(position, _center) <= _radius;
            _proximityStates[entity] = isInRange;
            
            // Subscribe to position changes
            entity.OnValueChanged += OnPositionChanged;
        }
    }
    
    public override void Untrack(IEntity entity)
    {
        _proximityStates.Remove(entity);
        entity.OnValueChanged -= OnPositionChanged;
    }
    
    private void OnPositionChanged(IEntity entity, int key)
    {
        string keyName = entity.GetKeyName(key);
        if (keyName != "Position") return;
        
        if (_proximityStates.TryGetValue(entity, out bool wasInRange))
        {
            Vector3 position = entity.Get<Vector3>("Position");
            bool isInRange = Vector3.Distance(position, _center) <= _radius;
            
            // Check for proximity state change
            if (isInRange != wasInRange)
            {
                _proximityStates[entity] = isInRange;
                
                // Use base class callback invocation
                InvokeAction(entity);
            }
        }
    }
    
    // Helper methods
    public bool IsEntityInRange(IEntity entity)
    {
        return _proximityStates.TryGetValue(entity, out bool inRange) && inRange;
    }
    
    public void UpdateCenter(Vector3 newCenter)
    {
        // Update center and re-evaluate all entities
        var center = newCenter;
        
        foreach (var kvp in _proximityStates.ToArray())
        {
            var entity = kvp.Key;
            bool wasInRange = kvp.Value;
            
            Vector3 position = entity.Get<Vector3>("Position");
            bool isInRange = Vector3.Distance(position, center) <= _radius;
            
            if (isInRange != wasInRange)
            {
                _proximityStates[entity] = isInRange;
                InvokeAction(entity);
            }
        }
    }
}
```

### Multi-Condition State Trigger

```csharp
public class StateConditionTrigger : EntityTriggerBase
{
    [System.Flags]
    public enum ConditionFlags
    {
        None = 0,
        HasRequiredHealth = 1,
        HasRequiredLevel = 2,
        HasRequiredTags = 4,
        HasRequiredEquipment = 8
    }
    
    private readonly Dictionary<IEntity, ConditionFlags> _entityStates = new();
    private readonly int _requiredLevel;
    private readonly float _requiredHealthPercent;
    private readonly string[] _requiredTags;
    
    public StateConditionTrigger(int requiredLevel = 5, float requiredHealthPercent = 0.5f, params string[] requiredTags)
    {
        _requiredLevel = requiredLevel;
        _requiredHealthPercent = requiredHealthPercent;
        _requiredTags = requiredTags ?? new string[0];
    }
    
    public override void Track(IEntity entity)
    {
        // Calculate initial state
        var initialState = EvaluateEntityConditions(entity);
        _entityStates[entity] = initialState;
        
        // Subscribe to relevant changes
        entity.OnValueChanged += OnEntityValueChanged;
        entity.OnTagAdded += OnEntityTagChanged;
        entity.OnTagDeleted += OnEntityTagChanged;
    }
    
    public override void Untrack(IEntity entity)
    {
        _entityStates.Remove(entity);
        
        entity.OnValueChanged -= OnEntityValueChanged;
        entity.OnTagAdded -= OnEntityTagChanged;
        entity.OnTagDeleted -= OnEntityTagChanged;
    }
    
    private void OnEntityValueChanged(IEntity entity, int key)
    {
        string keyName = entity.GetKeyName(key);
        
        // Only evaluate for relevant value changes
        if (keyName == "Health" || keyName == "MaxHealth" || keyName == "Level")
        {
            CheckEntityStateChange(entity);
        }
    }
    
    private void OnEntityTagChanged(IEntity entity, int tagId)
    {
        CheckEntityStateChange(entity);
    }
    
    private void CheckEntityStateChange(IEntity entity)
    {
        if (_entityStates.TryGetValue(entity, out var oldState))
        {
            var newState = EvaluateEntityConditions(entity);
            
            if (oldState != newState)
            {
                _entityStates[entity] = newState;
                InvokeAction(entity);
            }
        }
    }
    
    private ConditionFlags EvaluateEntityConditions(IEntity entity)
    {
        var conditions = ConditionFlags.None;
        
        // Check health condition
        if (entity.HasValue("Health") && entity.HasValue("MaxHealth"))
        {
            float health = entity.Get<float>("Health");
            float maxHealth = entity.Get<float>("MaxHealth");
            
            if (health / maxHealth >= _requiredHealthPercent)
            {
                conditions |= ConditionFlags.HasRequiredHealth;
            }
        }
        
        // Check level condition
        if (entity.HasValue("Level"))
        {
            int level = entity.Get<int>("Level");
            if (level >= _requiredLevel)
            {
                conditions |= ConditionFlags.HasRequiredLevel;
            }
        }
        
        // Check required tags
        bool hasAllTags = true;
        foreach (string tag in _requiredTags)
        {
            if (!entity.HasTag(tag))
            {
                hasAllTags = false;
                break;
            }
        }
        
        if (hasAllTags)
        {
            conditions |= ConditionFlags.HasRequiredTags;
        }
        
        // Check equipment condition (example)
        if (entity.HasValue("EquipmentCount"))
        {
            int equipmentCount = entity.Get<int>("EquipmentCount");
            if (equipmentCount >= 2)
            {
                conditions |= ConditionFlags.HasRequiredEquipment;
            }
        }
        
        return conditions;
    }
    
    // Public API for checking specific conditions
    public bool HasCondition(IEntity entity, ConditionFlags condition)
    {
        return _entityStates.TryGetValue(entity, out var state) && (state & condition) == condition;
    }
    
    public ConditionFlags GetEntityConditions(IEntity entity)
    {
        return _entityStates.TryGetValue(entity, out var state) ? state : ConditionFlags.None;
    }
}
```

## Integration with Atomic Framework

### Reactive System Integration

```csharp
public class ReactiveEntitySystem : MonoBehaviour
{
    [System.Serializable]
    public struct TriggerConfiguration
    {
        public string name;
        public TriggerType type;
        public float numericThreshold;
        public string[] stringParameters;
    }
    
    public enum TriggerType
    {
        Health,
        Level,
        Proximity,
        StateCondition
    }
    
    [SerializeField] private TriggerConfiguration[] _triggerConfigs;
    
    private readonly Dictionary<string, EntityTriggerBase> _activeTriggers = new();
    private readonly ReactiveCollection<IEntity> _trackedEntities = new();
    
    void Start()
    {
        SetupTriggers();
        
        // React to entity collection changes
        _trackedEntities.OnAdded += OnEntityAdded;
        _trackedEntities.OnRemoved += OnEntityRemoved;
    }
    
    private void SetupTriggers()
    {
        foreach (var config in _triggerConfigs)
        {
            var trigger = CreateTriggerFromConfig(config);
            if (trigger != null)
            {
                trigger.SetAction(entity => OnTriggerActivated(config.name, entity));
                _activeTriggers[config.name] = trigger;
            }
        }
    }
    
    private EntityTriggerBase CreateTriggerFromConfig(TriggerConfiguration config)
    {
        return config.type switch
        {
            TriggerType.Health => new HealthMonitorTrigger(config.numericThreshold),
            TriggerType.Level => new LevelTrigger((int)config.numericThreshold),
            TriggerType.Proximity => new ProximityTrigger(Vector3.zero, config.numericThreshold),
            TriggerType.StateCondition => new StateConditionTrigger(
                requiredLevel: (int)config.numericThreshold,
                requiredTags: config.stringParameters
            ),
            _ => null
        };
    }
    
    private void OnEntityAdded(IEntity entity)
    {
        // Start tracking with all triggers
        foreach (var trigger in _activeTriggers.Values)
        {
            trigger.Track(entity);
        }
    }
    
    private void OnEntityRemoved(IEntity entity)
    {
        // Stop tracking with all triggers
        foreach (var trigger in _activeTriggers.Values)
        {
            trigger.Untrack(entity);
        }
    }
    
    private void OnTriggerActivated(string triggerName, IEntity entity)
    {
        Debug.Log($"Trigger '{triggerName}' activated for entity: {entity.Name}");
        GameEvents.OnEntityTriggerActivated?.Invoke(triggerName, entity);
    }
    
    public void AddEntityToSystem(IEntity entity)
    {
        _trackedEntities.Add(entity);
    }
    
    public void RemoveEntityFromSystem(IEntity entity)
    {
        _trackedEntities.Remove(entity);
    }
}
```

### Performance-Optimized Trigger Pool

```csharp
public class TriggerPool<T> where T : EntityTriggerBase, new()
{
    private readonly Stack<T> _availableTriggers = new();
    private readonly HashSet<T> _activeTriggers = new();
    private readonly int _maxPoolSize;
    
    public TriggerPool(int maxPoolSize = 50)
    {
        _maxPoolSize = maxPoolSize;
        
        // Pre-populate pool
        for (int i = 0; i < Math.Min(10, maxPoolSize); i++)
        {
            _availableTriggers.Push(new T());
        }
    }
    
    public T RentTrigger(Action<IEntity> callback)
    {
        T trigger;
        
        if (_availableTriggers.Count > 0)
        {
            trigger = _availableTriggers.Pop();
        }
        else
        {
            trigger = new T();
        }
        
        trigger.SetAction(callback);
        _activeTriggers.Add(trigger);
        
        return trigger;
    }
    
    public void ReturnTrigger(T trigger)
    {
        if (_activeTriggers.Remove(trigger))
        {
            // Clear any tracked entities
            // This would need to be implemented in derived classes
            
            if (_availableTriggers.Count < _maxPoolSize)
            {
                _availableTriggers.Push(trigger);
            }
        }
    }
    
    public int AvailableCount => _availableTriggers.Count;
    public int ActiveCount => _activeTriggers.Count;
}
```

## Implementation Notes

### Action Management
- Protected action field accessible to derived classes
- Null safety validation on SetAction calls
- Aggressive inlining for optimal callback performance

### Template Pattern
- Abstract Track/Untrack methods enforce implementation
- Common infrastructure shared across all implementations
- Consistent behavior patterns for derived classes

### Performance Optimization
- Minimal virtual call overhead
- Optimized callback invocation with inlining
- Efficient null checking for action callbacks

## Best Practices

### Derived Class Implementation
- Always call base.InvokeAction for consistent callback handling
- Implement proper cleanup in Untrack methods
- Use efficient tracking mechanisms for entity state

### Resource Management
- Unsubscribe from all events in Untrack method
- Clear internal state when untracking entities
- Avoid memory leaks with proper event unsubscription

### Performance Considerations
- Keep track/untrack operations lightweight
- Use efficient data structures for entity state tracking
- Minimize allocations in trigger callback paths

## Common Patterns

### State Machine Trigger

```csharp
public class StateMachineTrigger : EntityTriggerBase
{
    public enum EntityState { Idle, Moving, Combat, Dead }
    
    private readonly Dictionary<IEntity, EntityState> _entityStates = new();
    
    public override void Track(IEntity entity)
    {
        var initialState = DetermineState(entity);
        _entityStates[entity] = initialState;
        
        entity.OnValueChanged += OnEntityValueChanged;
        entity.OnTagAdded += OnEntityTagChanged;
        entity.OnTagDeleted += OnEntityTagChanged;
    }
    
    public override void Untrack(IEntity entity)
    {
        _entityStates.Remove(entity);
        entity.OnValueChanged -= OnEntityValueChanged;
        entity.OnTagAdded -= OnEntityTagChanged;
        entity.OnTagDeleted -= OnEntityTagChanged;
    }
    
    private void OnEntityValueChanged(IEntity entity, int key) => CheckStateChange(entity);
    private void OnEntityTagChanged(IEntity entity, int tagId) => CheckStateChange(entity);
    
    private void CheckStateChange(IEntity entity)
    {
        if (_entityStates.TryGetValue(entity, out var oldState))
        {
            var newState = DetermineState(entity);
            if (oldState != newState)
            {
                _entityStates[entity] = newState;
                InvokeAction(entity);
            }
        }
    }
    
    private EntityState DetermineState(IEntity entity)
    {
        if (entity.HasTag("Dead")) return EntityState.Dead;
        if (entity.HasTag("InCombat")) return EntityState.Combat;
        if (entity.HasTag("Moving")) return EntityState.Moving;
        return EntityState.Idle;
    }
}
```

The `EntityTriggerBase` provides a robust foundation for implementing custom entity triggers with shared infrastructure, performance optimization, and consistent behavior patterns within the Atomic framework's reactive architecture.