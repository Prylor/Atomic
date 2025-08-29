# 🧩 ValueEntityTrigger

A specialized entity trigger that monitors value-related changes on entities, responding to value additions, deletions, and modifications. Provides reactive monitoring for entity data changes, enabling automatic system responses to property updates.

## Overview

`ValueEntityTrigger` automatically tracks when values are added to, removed from, or changed on entities, triggering callbacks for reactive system updates. Essential for building systems that respond to entity data modifications, stat changes, or property updates.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// A non-generic shortcut for ValueEntityTrigger{IEntity}.
    /// Provides value-based tracking behavior for basic IEntity instances,
    /// including reactions to value additions, removals, and changes.
    /// </summary>
    public class ValueEntityTrigger : ValueEntityTrigger<IEntity>
    {
    }

    /// <summary>
    /// A trigger that responds to value changes on an entity of type E.
    /// It listens for value additions, removals, and modifications using corresponding entity events.
    /// </summary>
    /// <typeparam name="E">The type of entity to track, constrained to IEntity.</typeparam>
    public class ValueEntityTrigger<E> : EntityTriggerBase<E> where E : IEntity
    {
        private readonly bool _added;
        private readonly bool _deleted;
        private readonly bool _changed;

        public ValueEntityTrigger(bool added = true, bool deleted = true, bool changed = true)
        {
            _added = added;
            _deleted = deleted;
            _changed = changed;
        }

        public override void Track(E entity)
        {
            if (_added) entity.OnValueAdded += this.OnValueAdded;
            if (_deleted) entity.OnValueDeleted += this.OnValueDeleted;
            if (_changed) entity.OnValueChanged += this.OnValueChanged;
        }

        public override void Untrack(E entity)
        {
            if (_added) entity.OnValueAdded -= this.OnValueAdded;
            if (_deleted) entity.OnValueDeleted -= this.OnValueDeleted;
            if (_changed) entity.OnValueChanged -= this.OnValueChanged;
        }

        private void OnValueDeleted(IEntity entity, int key) => _action.Invoke((E) entity);
        private void OnValueAdded(IEntity entity, int key) => _action.Invoke((E) entity);
        private void OnValueChanged(IEntity entity, int key) => _action.Invoke((E) entity);
    }
}
```

## Key Features

### Selective Value Monitoring
- Configurable tracking of value additions only
- Configurable tracking of value deletions only
- Configurable tracking of value changes only
- Full monitoring of all operations by default

### Comprehensive Value Events
- Monitors OnValueAdded for new property additions
- Monitors OnValueDeleted for property removals
- Monitors OnValueChanged for property modifications

### Efficient Event Management
- Automatic subscription to relevant entity events
- Clean unsubscription for proper resource management
- Type-safe entity casting in event callbacks

## Usage Examples

### Basic Value Change Monitoring

```csharp
// Monitor all value operations (additions, deletions, and changes)
var allValueTrigger = new ValueEntityTrigger();
allValueTrigger.SetAction(entity =>
{
    Debug.Log($"Value change detected on entity: {entity.Name}");
    // Re-evaluate entity in all relevant systems
    EntityManager.Instance.ReEvaluateEntity(entity);
});

// Track entities
allValueTrigger.Track(playerEntity);
allValueTrigger.Track(itemEntity);

// When values change on entities, the trigger automatically responds
playerEntity.Set("Health", 75f);        // Triggers callback (added or changed)
playerEntity.Set("Health", 50f);        // Triggers callback (changed)
itemEntity.Remove("Durability");        // Triggers callback (deleted)
```

### Health and Stats Monitoring System

```csharp
public class HealthStatsMonitor : MonoBehaviour
{
    // Monitor only value changes (not additions/deletions)
    private readonly ValueEntityTrigger _healthTrigger = new ValueEntityTrigger(
        added: false, deleted: false, changed: true);
    
    private readonly Dictionary<IEntity, HealthState> _previousHealthStates = new();

    [System.Serializable]
    public struct HealthState
    {
        public float health;
        public float maxHealth;
        public float shield;
        public DateTime lastUpdate;
    }

    void Start()
    {
        _healthTrigger.SetAction(OnHealthRelatedValueChanged);
        
        // Track all entities with health values
        EntityRegistry.ForEach(entity =>
        {
            if (entity.HasValue("Health"))
            {
                _healthTrigger.Track(entity);
                CacheHealthState(entity);
            }
        });
    }

    private void OnHealthRelatedValueChanged(IEntity entity)
    {
        if (!IsHealthRelatedChange(entity))
            return;

        var currentState = GetCurrentHealthState(entity);
        var previousState = _previousHealthStates.GetValueOrDefault(entity);

        AnalyzeHealthChanges(entity, previousState, currentState);
        _previousHealthStates[entity] = currentState;
    }

    private bool IsHealthRelatedChange(IEntity entity)
    {
        return entity.HasValue("Health") || 
               entity.HasValue("MaxHealth") || 
               entity.HasValue("Shield") ||
               entity.HasValue("Regeneration");
    }

    private HealthState GetCurrentHealthState(IEntity entity)
    {
        return new HealthState
        {
            health = entity.Get<float>("Health"),
            maxHealth = entity.Get<float>("MaxHealth"),
            shield = entity.HasValue("Shield") ? entity.Get<float>("Shield") : 0f,
            lastUpdate = DateTime.Now
        };
    }

    private void AnalyzeHealthChanges(IEntity entity, HealthState previous, HealthState current)
    {
        float healthChange = current.health - previous.health;
        float healthPercent = current.health / current.maxHealth;
        float previousPercent = previous.health / previous.maxHealth;

        // Detect health loss
        if (healthChange < 0)
        {
            float damage = -healthChange;
            Debug.Log($"Entity {entity.Name} took {damage} damage");
            
            GameEvents.OnEntityTookDamage?.Invoke(entity, damage);

            // Check for critical health thresholds
            if (healthPercent <= 0.25f && previousPercent > 0.25f)
            {
                entity.AddTag("CriticalHealth");
                GameEvents.OnEntityCriticalHealth?.Invoke(entity);
            }
            else if (healthPercent <= 0.1f && previousPercent > 0.1f)
            {
                entity.AddTag("NearDeath");
                GameEvents.OnEntityNearDeath?.Invoke(entity);
            }
        }
        // Detect health gain
        else if (healthChange > 0)
        {
            float healing = healthChange;
            Debug.Log($"Entity {entity.Name} healed for {healing}");
            
            GameEvents.OnEntityHealed?.Invoke(entity, healing);

            // Remove critical health tags if appropriate
            if (healthPercent > 0.25f)
            {
                entity.RemoveTag("CriticalHealth");
            }
            if (healthPercent > 0.1f)
            {
                entity.RemoveTag("NearDeath");
            }
        }

        // Detect shield changes
        float shieldChange = current.shield - previous.shield;
        if (shieldChange != 0)
        {
            if (shieldChange > 0)
            {
                GameEvents.OnEntityShieldGained?.Invoke(entity, shieldChange);
            }
            else
            {
                GameEvents.OnEntityShieldLost?.Invoke(entity, -shieldChange);
            }
        }

        // Detect max health changes (level up, equipment, etc.)
        if (current.maxHealth != previous.maxHealth)
        {
            float maxHealthChange = current.maxHealth - previous.maxHealth;
            Debug.Log($"Entity {entity.Name} max health changed by {maxHealthChange}");
            GameEvents.OnEntityMaxHealthChanged?.Invoke(entity, maxHealthChange);
        }
    }

    private void CacheHealthState(IEntity entity)
    {
        if (entity.HasValue("Health"))
        {
            _previousHealthStates[entity] = GetCurrentHealthState(entity);
        }
    }

    void OnDestroy()
    {
        // Clean up all tracked entities
        foreach (var entity in _previousHealthStates.Keys)
        {
            _healthTrigger.Untrack(entity);
        }
    }
}
```

### Equipment and Inventory System

```csharp
public class EquipmentMonitor : MonoBehaviour
{
    // Monitor value additions and deletions for equipment changes
    private readonly ValueEntityTrigger _equipmentTrigger = new ValueEntityTrigger(
        added: true, deleted: true, changed: false);
    
    private readonly Dictionary<IEntity, Dictionary<string, object>> _previousEquipment = new();
    
    private readonly string[] _equipmentSlots = 
    {
        "MainHand", "OffHand", "Helmet", "Chest", "Legs", "Boots", "Ring1", "Ring2", "Amulet"
    };

    void Start()
    {
        _equipmentTrigger.SetAction(OnEquipmentChanged);
        
        // Track all entities that can have equipment
        EntityRegistry.ForEach(entity =>
        {
            if (entity.HasTag("CanEquipItems"))
            {
                _equipmentTrigger.Track(entity);
                CacheEquipmentState(entity);
            }
        });
    }

    private void OnEquipmentChanged(IEntity entity)
    {
        if (!entity.HasTag("CanEquipItems"))
            return;

        var currentEquipment = GetCurrentEquipment(entity);
        var previousEquipment = _previousEquipment.GetValueOrDefault(entity, new Dictionary<string, object>());

        AnalyzeEquipmentChanges(entity, previousEquipment, currentEquipment);
        _previousEquipment[entity] = currentEquipment;
    }

    private Dictionary<string, object> GetCurrentEquipment(IEntity entity)
    {
        var equipment = new Dictionary<string, object>();
        
        foreach (string slot in _equipmentSlots)
        {
            if (entity.HasValue(slot))
            {
                equipment[slot] = entity.Get<object>(slot);
            }
        }

        return equipment;
    }

    private void AnalyzeEquipmentChanges(IEntity entity, 
        Dictionary<string, object> previous, 
        Dictionary<string, object> current)
    {
        // Find equipped items (new entries)
        var equippedItems = current.Where(kvp => !previous.ContainsKey(kvp.Key))
                                  .ToDictionary(kvp => kvp.Key, kvp => kvp.Value);

        // Find unequipped items (removed entries)
        var unequippedItems = previous.Where(kvp => !current.ContainsKey(kvp.Key))
                                    .ToDictionary(kvp => kvp.Key, kvp => kvp.Value);

        // Find changed items (different values)
        var changedItems = current.Where(kvp => previous.ContainsKey(kvp.Key) && 
                                               !previous[kvp.Key].Equals(kvp.Value))
                                 .ToDictionary(kvp => kvp.Key, kvp => kvp.Value);

        // Process equipped items
        foreach (var item in equippedItems)
        {
            ProcessItemEquipped(entity, item.Key, item.Value);
        }

        // Process unequipped items
        foreach (var item in unequippedItems)
        {
            ProcessItemUnequipped(entity, item.Key, item.Value);
        }

        // Process changed items (item swapping)
        foreach (var item in changedItems)
        {
            var oldItem = previous[item.Key];
            ProcessItemUnequipped(entity, item.Key, oldItem);
            ProcessItemEquipped(entity, item.Key, item.Value);
        }

        // Update derived stats
        UpdateDerivedStats(entity);
    }

    private void ProcessItemEquipped(IEntity entity, string slot, object item)
    {
        Debug.Log($"Entity {entity.Name} equipped {item} in {slot}");
        
        // Add equipment-specific tags
        entity.AddTag($"Equipped_{slot}");
        
        // Apply item bonuses
        ApplyItemBonuses(entity, item, isEquipping: true);
        
        GameEvents.OnItemEquipped?.Invoke(entity, slot, item);
    }

    private void ProcessItemUnequipped(IEntity entity, string slot, object item)
    {
        Debug.Log($"Entity {entity.Name} unequipped {item} from {slot}");
        
        // Remove equipment-specific tags
        entity.RemoveTag($"Equipped_{slot}");
        
        // Remove item bonuses
        ApplyItemBonuses(entity, item, isEquipping: false);
        
        GameEvents.OnItemUnequipped?.Invoke(entity, slot, item);
    }

    private void ApplyItemBonuses(IEntity entity, object item, bool isEquipping)
    {
        // This would integrate with an item system to get item properties
        // For demonstration, assuming item has stat bonuses
        
        if (item is IDictionary<string, object> itemData)
        {
            float multiplier = isEquipping ? 1f : -1f;
            
            if (itemData.TryGetValue("AttackBonus", out object attackBonus))
            {
                float currentAttack = entity.Get<float>("Attack");
                entity.Set("Attack", currentAttack + (float)attackBonus * multiplier);
            }
            
            if (itemData.TryGetValue("DefenseBonus", out object defenseBonus))
            {
                float currentDefense = entity.Get<float>("Defense");
                entity.Set("Defense", currentDefense + (float)defenseBonus * multiplier);
            }
        }
    }

    private void UpdateDerivedStats(IEntity entity)
    {
        // Update combat rating based on equipment
        float combatRating = CalculateCombatRating(entity);
        entity.Set("CombatRating", combatRating);
        
        // Update equipment set bonuses
        CheckSetBonuses(entity);
        
        // Update equipment tags
        UpdateEquipmentTags(entity);
    }

    private float CalculateCombatRating(IEntity entity)
    {
        float rating = 0f;
        
        rating += entity.Get<float>("Attack") * 1.2f;
        rating += entity.Get<float>("Defense") * 1.0f;
        rating += entity.Get<float>("Health") * 0.1f;
        
        return rating;
    }

    private void CheckSetBonuses(IEntity entity)
    {
        // Implementation for checking equipment set bonuses
        // Remove old set bonus tags and apply new ones
    }

    private void UpdateEquipmentTags(IEntity entity)
    {
        // Update tags based on current equipment state
        bool isFullyEquipped = _equipmentSlots.All(slot => entity.HasValue(slot));
        bool hasWeapon = entity.HasValue("MainHand");
        bool hasArmor = entity.HasValue("Chest");

        entity.SetTagState("FullyEquipped", isFullyEquipped);
        entity.SetTagState("Armed", hasWeapon);
        entity.SetTagState("Armored", hasArmor);
    }

    private void CacheEquipmentState(IEntity entity)
    {
        _previousEquipment[entity] = GetCurrentEquipment(entity);
    }

    public Dictionary<string, object> GetEntityEquipment(IEntity entity)
    {
        return GetCurrentEquipment(entity);
    }

    public bool IsSlotEquipped(IEntity entity, string slot)
    {
        return entity.HasValue(slot);
    }

    public object GetEquippedItem(IEntity entity, string slot)
    {
        return entity.HasValue(slot) ? entity.Get<object>(slot) : null;
    }
}
```

### Configuration and Settings Monitor

```csharp
public class EntityConfigurationMonitor : MonoBehaviour
{
    // Monitor all value operations for configuration changes
    private readonly ValueEntityTrigger _configTrigger = new ValueEntityTrigger();
    
    private readonly Dictionary<IEntity, Dictionary<string, object>> _baseConfigurations = new();
    private readonly Dictionary<IEntity, DateTime> _lastConfigUpdate = new();

    private readonly string[] _configurationKeys = 
    {
        "MovementSpeed", "AttackSpeed", "ViewRange", "DetectionRadius", 
        "MaxHealth", "MaxMana", "Armor", "MagicResistance"
    };

    void Start()
    {
        _configTrigger.SetAction(OnConfigurationChanged);
        
        // Track all configurable entities
        EntityRegistry.ForEach(entity =>
        {
            if (entity.HasTag("Configurable"))
            {
                _configTrigger.Track(entity);
                CacheBaseConfiguration(entity);
            }
        });
    }

    private void OnConfigurationChanged(IEntity entity)
    {
        if (!IsConfigurationChange(entity))
            return;

        ProcessConfigurationUpdate(entity);
        ValidateConfiguration(entity);
        _lastConfigUpdate[entity] = DateTime.Now;
    }

    private bool IsConfigurationChange(IEntity entity)
    {
        // Check if any configuration values were modified
        return _configurationKeys.Any(key => entity.HasValue(key));
    }

    private void ProcessConfigurationUpdate(IEntity entity)
    {
        var currentConfig = GetCurrentConfiguration(entity);
        var baseConfig = _baseConfigurations.GetValueOrDefault(entity, new Dictionary<string, object>());

        foreach (var kvp in currentConfig)
        {
            string key = kvp.Key;
            object currentValue = kvp.Value;
            object baseValue = baseConfig.GetValueOrDefault(key, currentValue);

            ProcessConfigurationValueChange(entity, key, baseValue, currentValue);
        }

        // Check for removed configurations
        foreach (var kvp in baseConfig)
        {
            if (!currentConfig.ContainsKey(kvp.Key))
            {
                ProcessConfigurationValueRemoved(entity, kvp.Key, kvp.Value);
            }
        }
    }

    private void ProcessConfigurationValueChange(IEntity entity, string key, object oldValue, object newValue)
    {
        if (!oldValue.Equals(newValue))
        {
            Debug.Log($"Entity {entity.Name} configuration changed: {key} = {oldValue} -> {newValue}");
            
            // Apply configuration-specific logic
            switch (key)
            {
                case "MovementSpeed":
                    UpdateMovementTags(entity, (float)newValue);
                    break;
                case "ViewRange":
                    UpdateVisionTags(entity, (float)newValue);
                    break;
                case "MaxHealth":
                    UpdateHealthConfiguration(entity, (float)oldValue, (float)newValue);
                    break;
                case "Armor":
                    UpdateDefenseTags(entity, (float)newValue);
                    break;
            }
            
            GameEvents.OnEntityConfigurationChanged?.Invoke(entity, key, oldValue, newValue);
        }
    }

    private void ProcessConfigurationValueRemoved(IEntity entity, string key, object oldValue)
    {
        Debug.Log($"Entity {entity.Name} configuration removed: {key} (was {oldValue})");
        
        // Handle configuration removal
        switch (key)
        {
            case "MovementSpeed":
                entity.RemoveTag("FastMovement");
                entity.RemoveTag("SlowMovement");
                break;
            case "ViewRange":
                entity.RemoveTag("LongRange");
                entity.RemoveTag("ShortRange");
                break;
        }
        
        GameEvents.OnEntityConfigurationRemoved?.Invoke(entity, key, oldValue);
    }

    private Dictionary<string, object> GetCurrentConfiguration(IEntity entity)
    {
        var config = new Dictionary<string, object>();
        
        foreach (string key in _configurationKeys)
        {
            if (entity.HasValue(key))
            {
                config[key] = entity.Get<object>(key);
            }
        }

        return config;
    }

    private void UpdateMovementTags(IEntity entity, float speed)
    {
        entity.RemoveTag("FastMovement");
        entity.RemoveTag("SlowMovement");
        entity.RemoveTag("NormalMovement");

        if (speed > 8f)
        {
            entity.AddTag("FastMovement");
        }
        else if (speed < 3f)
        {
            entity.AddTag("SlowMovement");
        }
        else
        {
            entity.AddTag("NormalMovement");
        }
    }

    private void UpdateVisionTags(IEntity entity, float range)
    {
        entity.RemoveTag("LongRange");
        entity.RemoveTag("ShortRange");
        entity.RemoveTag("NormalRange");

        if (range > 15f)
        {
            entity.AddTag("LongRange");
        }
        else if (range < 5f)
        {
            entity.AddTag("ShortRange");
        }
        else
        {
            entity.AddTag("NormalRange");
        }
    }

    private void UpdateHealthConfiguration(IEntity entity, float oldMaxHealth, float newMaxHealth)
    {
        // Adjust current health proportionally if max health changed
        if (entity.HasValue("Health"))
        {
            float currentHealth = entity.Get<float>("Health");
            float healthRatio = oldMaxHealth > 0 ? currentHealth / oldMaxHealth : 1f;
            float newCurrentHealth = newMaxHealth * healthRatio;
            
            entity.Set("Health", newCurrentHealth);
        }
    }

    private void UpdateDefenseTags(IEntity entity, float armor)
    {
        entity.RemoveTag("HighDefense");
        entity.RemoveTag("LowDefense");
        entity.RemoveTag("NormalDefense");

        if (armor > 20f)
        {
            entity.AddTag("HighDefense");
        }
        else if (armor < 5f)
        {
            entity.AddTag("LowDefense");
        }
        else
        {
            entity.AddTag("NormalDefense");
        }
    }

    private void ValidateConfiguration(IEntity entity)
    {
        bool isValid = true;
        var validationErrors = new List<string>();

        // Validate movement speed
        if (entity.HasValue("MovementSpeed"))
        {
            float speed = entity.Get<float>("MovementSpeed");
            if (speed < 0f || speed > 50f)
            {
                validationErrors.Add($"Invalid MovementSpeed: {speed} (must be 0-50)");
                isValid = false;
            }
        }

        // Validate health values
        if (entity.HasValue("Health") && entity.HasValue("MaxHealth"))
        {
            float health = entity.Get<float>("Health");
            float maxHealth = entity.Get<float>("MaxHealth");
            
            if (health > maxHealth)
            {
                entity.Set("Health", maxHealth);
                Debug.Log($"Clamped health for {entity.Name} to max health value");
            }
        }

        entity.SetTagState("ValidConfiguration", isValid);
        
        if (!isValid)
        {
            Debug.LogWarning($"Entity {entity.Name} has configuration errors: {string.Join(", ", validationErrors)}");
            GameEvents.OnEntityConfigurationInvalid?.Invoke(entity, validationErrors.ToArray());
        }
    }

    private void CacheBaseConfiguration(IEntity entity)
    {
        _baseConfigurations[entity] = GetCurrentConfiguration(entity);
        _lastConfigUpdate[entity] = DateTime.Now;
    }

    public Dictionary<string, object> GetEntityConfiguration(IEntity entity)
    {
        return GetCurrentConfiguration(entity);
    }

    public DateTime GetLastConfigurationUpdate(IEntity entity)
    {
        return _lastConfigUpdate.GetValueOrDefault(entity, DateTime.MinValue);
    }

    public bool HasValidConfiguration(IEntity entity)
    {
        return entity.HasTag("ValidConfiguration");
    }

    void OnDestroy()
    {
        foreach (var entity in _baseConfigurations.Keys)
        {
            _configTrigger.Untrack(entity);
        }
    }
}
```

## Integration with Atomic Framework

### Data-Driven Reactive System

```csharp
public class DataReactiveSystem : MonoBehaviour
{
    [System.Serializable]
    public struct ValueTriggerConfig
    {
        public string triggerName;
        public string[] watchedKeys;
        public bool trackAdded;
        public bool trackDeleted;
        public bool trackChanged;
        public UnityEvent<IEntity> onTriggered;
    }

    [SerializeField] private ValueTriggerConfig[] _triggerConfigs;
    
    private readonly Dictionary<string, ValueEntityTrigger> _triggers = new();
    private readonly Dictionary<IEntity, HashSet<string>> _entityTriggers = new();

    void Start()
    {
        foreach (var config in _triggerConfigs)
        {
            var trigger = new ValueEntityTrigger(config.trackAdded, config.trackDeleted, config.trackChanged);
            trigger.SetAction(entity => OnValueTriggerFired(config, entity));
            
            _triggers[config.triggerName] = trigger;
        }
        
        // Start tracking all entities
        EntityRegistry.ForEach(StartTrackingEntity);
        EntityRegistry.OnEntityCreated += StartTrackingEntity;
    }

    private void StartTrackingEntity(IEntity entity)
    {
        if (!_entityTriggers.ContainsKey(entity))
        {
            _entityTriggers[entity] = new HashSet<string>();
        }

        foreach (var kvp in _triggers)
        {
            kvp.Value.Track(entity);
            _entityTriggers[entity].Add(kvp.Key);
        }
    }

    private void OnValueTriggerFired(ValueTriggerConfig config, IEntity entity)
    {
        // Check if this entity has any of the watched keys
        bool hasWatchedKey = config.watchedKeys.Any(key => entity.HasValue(key));
        
        if (hasWatchedKey)
        {
            Debug.Log($"Value trigger '{config.triggerName}' fired for entity {entity.Name}");
            config.onTriggered.Invoke(entity);
        }
    }

    public void StopTrackingEntity(IEntity entity)
    {
        if (_entityTriggers.TryGetValue(entity, out var triggerNames))
        {
            foreach (string triggerName in triggerNames)
            {
                if (_triggers.TryGetValue(triggerName, out var trigger))
                {
                    trigger.Untrack(entity);
                }
            }
            
            _entityTriggers.Remove(entity);
        }
    }

    void OnDestroy()
    {
        EntityRegistry.OnEntityCreated -= StartTrackingEntity;
        
        foreach (var entityTriggers in _entityTriggers.Keys.ToArray())
        {
            StopTrackingEntity(entityTriggers);
        }
    }
}
```

## Implementation Notes

### Event Differentiation
- Separate handling for value additions, deletions, and changes
- Key-based event parameters for specific value identification
- Type-safe entity casting for callback execution

### Configuration Flexibility  
- Independent control over each event type monitoring
- Default behavior monitors all value operations
- Efficient when monitoring subset of operations

### Performance Characteristics
- Event-driven operation eliminates polling overhead
- Optimal for entities with frequent value modifications
- Minimal memory footprint per tracked entity

## Best Practices

### Monitoring Scope
- Use specific event type monitoring when possible (added/deleted/changed only)
- Consider system requirements for update frequency
- Balance between reactivity and performance overhead

### Value Change Handling
- Keep value change callbacks lightweight and fast
- Avoid expensive computations in immediate callback execution
- Consider batching or queuing heavy operations

### Integration Patterns
- Combine with tag triggers for comprehensive entity monitoring
- Use with entity filters for complete reactive system architectures
- Implement proper cleanup and disposal patterns

## Common Patterns

### Value Threshold Monitor

```csharp
public class ValueThresholdMonitor
{
    private readonly ValueEntityTrigger _trigger = new ValueEntityTrigger(changed: true);
    private readonly Dictionary<string, float> _thresholds = new();
    private readonly Action<IEntity, string, float> _onThresholdCrossed;

    public ValueThresholdMonitor(Action<IEntity, string, float> onThresholdCrossed)
    {
        _onThresholdCrossed = onThresholdCrossed;
        _trigger.SetAction(CheckThresholds);
    }

    public void SetThreshold(string valueKey, float threshold)
    {
        _thresholds[valueKey] = threshold;
    }

    private void CheckThresholds(IEntity entity)
    {
        foreach (var threshold in _thresholds)
        {
            if (entity.HasValue<float>(threshold.Key))
            {
                float value = entity.Get<float>(threshold.Key);
                if (value <= threshold.Value)
                {
                    _onThresholdCrossed(entity, threshold.Key, value);
                }
            }
        }
    }

    public void Track(IEntity entity) => _trigger.Track(entity);
    public void Untrack(IEntity entity) => _trigger.Untrack(entity);
}
```

### Value Change Logger

```csharp
public class ValueChangeLogger
{
    private readonly ValueEntityTrigger _logger = new ValueEntityTrigger();
    private readonly Dictionary<IEntity, Dictionary<string, object>> _previousValues = new();

    public ValueChangeLogger()
    {
        _logger.SetAction(LogValueChanges);
    }

    private void LogValueChanges(IEntity entity)
    {
        var currentValues = GetAllEntityValues(entity);
        var previousValues = _previousValues.GetValueOrDefault(entity, new Dictionary<string, object>());

        foreach (var current in currentValues)
        {
            if (!previousValues.TryGetValue(current.Key, out object previousValue))
            {
                Debug.Log($"[VALUE ADDED] {entity.Name}.{current.Key} = {current.Value}");
            }
            else if (!previousValue.Equals(current.Value))
            {
                Debug.Log($"[VALUE CHANGED] {entity.Name}.{current.Key}: {previousValue} -> {current.Value}");
            }
        }

        foreach (var previous in previousValues)
        {
            if (!currentValues.ContainsKey(previous.Key))
            {
                Debug.Log($"[VALUE REMOVED] {entity.Name}.{previous.Key} (was {previous.Value})");
            }
        }

        _previousValues[entity] = currentValues;
    }

    private Dictionary<string, object> GetAllEntityValues(IEntity entity)
    {
        // Implementation would depend on entity interface
        // This is a simplified example
        return new Dictionary<string, object>();
    }
}
```

The `ValueEntityTrigger` provides comprehensive, efficient monitoring of entity value changes, enabling sophisticated data-driven reactive systems and automatic responses to property modifications within the Atomic framework.