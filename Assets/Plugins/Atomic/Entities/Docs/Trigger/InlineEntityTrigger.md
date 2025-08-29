# 🧩 InlineEntityTrigger

A flexible delegate-based entity trigger that allows custom tracking and untracking logic to be defined inline. Provides maximum flexibility for creating specialized trigger behaviors without implementing new trigger classes.

## Overview

`InlineEntityTrigger` enables developers to define custom entity monitoring logic using delegates passed during instantiation. Essential for creating ad-hoc triggers, prototyping trigger behaviors, or implementing simple monitoring scenarios without class inheritance.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// A non-generic shortcut for InlineEntityTrigger{IEntity}.
    /// Provides inline tracking logic for basic IEntity instances.
    /// </summary>
    public class InlineEntityTrigger : InlineEntityTrigger<IEntity>
    {
        public InlineEntityTrigger(
            Action<IEntity, Action<IEntity>> track,
            Action<IEntity, Action<IEntity>> untrack
        ) : base(track, untrack)
        {
        }
    }

    /// <summary>
    /// An inline-configurable entity trigger that allows custom tracking and untracking logic
    /// for a specific entity type.
    /// </summary>
    /// <typeparam name="E">The type of entity to track, constrained to IEntity.</typeparam>
    public class InlineEntityTrigger<E> : EntityTriggerBase<E> where E : IEntity
    {
        private readonly Action<E, Action<E>> _track;
        private readonly Action<E, Action<E>> _untrack;

        public InlineEntityTrigger(Action<E, Action<E>> track, Action<E, Action<E>> untrack)
        {
            _track = track ?? throw new ArgumentNullException(nameof(track));
            _untrack = untrack ?? throw new ArgumentNullException(nameof(untrack));
        }

        public override void Track(E entity) => _track.Invoke(entity, _action);
        public override void Untrack(E entity) => _untrack.Invoke(entity, _action);
    }
}
```

## Key Features

### Delegate-Based Logic
- Custom tracking logic defined through delegates
- Inline behavior definition without class inheritance
- Maximum flexibility for specialized trigger requirements

### Type Safety
- Generic version supports specific entity types
- Non-generic convenience version for basic scenarios
- Compile-time type checking for delegate parameters

### Resource Management
- Delegates responsible for proper subscription management
- Clean separation of tracking and untracking logic
- No built-in resource cleanup - delegated to custom logic

## Usage Examples

### Basic Health Monitoring Trigger

```csharp
var healthTrigger = new InlineEntityTrigger(
    track: (entity, callback) =>
    {
        // Subscribe to health value changes
        entity.OnValueChanged += (e, key) =>
        {
            if (key == "Health")
            {
                float health = e.Get<float>("Health");
                float maxHealth = e.Get<float>("MaxHealth");
                
                // Trigger when health drops below 25%
                if (health / maxHealth < 0.25f)
                {
                    callback(e);
                }
            }
        };
    },
    untrack: (entity, callback) =>
    {
        // Note: This simplified example doesn't handle proper unsubscription
        // In practice, you'd need to store event handlers for removal
        entity.OnValueChanged -= callback;
    }
);

// Configure the trigger callback
healthTrigger.SetAction(entity => 
{
    Debug.Log($"Entity {entity.Name} is critically low on health!");
    GameEvents.OnEntityCriticalHealth?.Invoke(entity);
});

// Start tracking entities
healthTrigger.Track(playerEntity);
healthTrigger.Track(enemyEntity);
```

### Complex Multi-Condition Trigger

```csharp
public class CustomTriggerManager : MonoBehaviour
{
    private InlineEntityTrigger _combatReadinessTrigger;
    private readonly Dictionary<IEntity, List<Action<IEntity, int>>> _eventHandlers = new();

    void Start()
    {
        _combatReadinessTrigger = new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                var handlers = new List<Action<IEntity, int>>();

                // Health threshold handler
                Action<IEntity, int> healthHandler = (e, key) =>
                {
                    if (key == "Health" && CheckCombatReadiness(e))
                        callback(e);
                };

                // Stamina threshold handler  
                Action<IEntity, int> staminaHandler = (e, key) =>
                {
                    if (key == "Stamina" && CheckCombatReadiness(e))
                        callback(e);
                };

                // Equipment change handler
                Action<IEntity, int> equipmentHandler = (e, key) =>
                {
                    if (key == "Weapon" || key == "Armor")
                    {
                        if (CheckCombatReadiness(e))
                            callback(e);
                    }
                };

                // Subscribe to events
                entity.OnValueChanged += healthHandler;
                entity.OnValueChanged += staminaHandler;
                entity.OnValueChanged += equipmentHandler;

                // Store handlers for cleanup
                handlers.AddRange(new[] { healthHandler, staminaHandler, equipmentHandler });
                _eventHandlers[entity] = handlers;
            },
            untrack: (entity, callback) =>
            {
                if (_eventHandlers.TryGetValue(entity, out var handlers))
                {
                    // Properly unsubscribe all stored handlers
                    foreach (var handler in handlers)
                    {
                        entity.OnValueChanged -= handler;
                    }
                    
                    _eventHandlers.Remove(entity);
                }
            }
        );

        _combatReadinessTrigger.SetAction(OnEntityCombatReadinessChanged);
    }

    private bool CheckCombatReadiness(IEntity entity)
    {
        if (!entity.HasValue("Health") || !entity.HasValue("Stamina"))
            return false;

        float healthPercent = entity.Get<float>("Health") / entity.Get<float>("MaxHealth");
        float staminaPercent = entity.Get<float>("Stamina") / entity.Get<float>("MaxStamina");
        bool hasWeapon = entity.HasValue("Weapon");

        return healthPercent > 0.5f && staminaPercent > 0.3f && hasWeapon;
    }

    private void OnEntityCombatReadinessChanged(IEntity entity)
    {
        bool isReady = CheckCombatReadiness(entity);
        Debug.Log($"Entity {entity.Name} combat readiness changed: {(isReady ? "Ready" : "Not Ready")}");
        
        if (isReady)
            GameEvents.OnEntityReadyForCombat?.Invoke(entity);
        else
            GameEvents.OnEntityNotReadyForCombat?.Invoke(entity);
    }
}
```

### Time-Based Monitoring Trigger

```csharp
public class TimerBasedTrigger
{
    public static InlineEntityTrigger CreateCooldownTrigger(string cooldownKey, float interval)
    {
        var cooldownTimers = new Dictionary<IEntity, float>();

        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                // Initialize cooldown timer
                cooldownTimers[entity] = 0f;

                // Start monitoring coroutine (simplified - would need proper implementation)
                void CheckCooldown()
                {
                    if (cooldownTimers.TryGetValue(entity, out float lastTime))
                    {
                        float currentTime = Time.time;
                        if (currentTime - lastTime >= interval)
                        {
                            if (entity.HasValue(cooldownKey) && entity.Get<bool>(cooldownKey))
                            {
                                callback(entity);
                                cooldownTimers[entity] = currentTime;
                            }
                        }
                    }
                }

                // In practice, you'd use a coroutine or update system
                // This is a simplified example
                entity.OnValueChanged += (e, key) =>
                {
                    if (key == cooldownKey)
                        CheckCooldown();
                };
            },
            untrack: (entity, callback) =>
            {
                cooldownTimers.Remove(entity);
                // Remove event subscriptions
            }
        );
    }
}

// Usage
var cooldownTrigger = TimerBasedTrigger.CreateCooldownTrigger("SpecialAbilityCooldown", 5.0f);
cooldownTrigger.SetAction(entity =>
{
    Debug.Log($"Entity {entity.Name} special ability is ready!");
    entity.Set("SpecialAbilityCooldown", false); // Reset cooldown
});
```

### Tag Combination Trigger

```csharp
public static class TagCombinationTriggers
{
    public static InlineEntityTrigger CreateRequiredTagsTrigger(params string[] requiredTags)
    {
        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                Action<IEntity, int> tagHandler = (e, tagId) =>
                {
                    // Check if all required tags are present
                    bool hasAllTags = true;
                    foreach (string tagName in requiredTags)
                    {
                        if (!e.HasTag(tagName))
                        {
                            hasAllTags = false;
                            break;
                        }
                    }

                    if (hasAllTags)
                    {
                        callback(e);
                    }
                };

                entity.OnTagAdded += tagHandler;
                entity.OnTagDeleted += tagHandler;
            },
            untrack: (entity, callback) =>
            {
                // Note: Proper implementation would store and remove specific handlers
                entity.OnTagAdded -= (e, t) => { };
                entity.OnTagDeleted -= (e, t) => { };
            }
        );
    }

    public static InlineEntityTrigger CreateForbiddenTagsTrigger(params string[] forbiddenTags)
    {
        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                Action<IEntity, int> tagHandler = (e, tagId) =>
                {
                    // Check if any forbidden tags are present
                    foreach (string tagName in forbiddenTags)
                    {
                        if (e.HasTag(tagName))
                        {
                            callback(e);
                            return;
                        }
                    }
                };

                entity.OnTagAdded += tagHandler;
            },
            untrack: (entity, callback) =>
            {
                // Unsubscribe from events
            }
        );
    }
}

// Usage
var playerReadyTrigger = TagCombinationTriggers.CreateRequiredTagsTrigger(
    "Player", "Active", "Healthy", "Armed"
);

playerReadyTrigger.SetAction(entity =>
{
    Debug.Log($"Player {entity.Name} is fully ready!");
    GameEvents.OnPlayerFullyReady?.Invoke(entity);
});

var corruptedTrigger = TagCombinationTriggers.CreateForbiddenTagsTrigger(
    "Corrupted", "Poisoned", "Cursed"
);

corruptedTrigger.SetAction(entity =>
{
    Debug.Log($"Entity {entity.Name} has become corrupted!");
    GameEvents.OnEntityCorrupted?.Invoke(entity);
});
```

### Factory Pattern for Common Triggers

```csharp
public static class InlineTriggerFactory
{
    public static InlineEntityTrigger CreateValueThresholdTrigger<T>(
        string valueKey, 
        T threshold, 
        Func<T, T, bool> comparison)
        where T : IComparable<T>
    {
        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                Action<IEntity, int> valueHandler = (e, key) =>
                {
                    if (key == valueKey && e.HasValue<T>(valueKey))
                    {
                        T currentValue = e.Get<T>(valueKey);
                        if (comparison(currentValue, threshold))
                        {
                            callback(e);
                        }
                    }
                };

                entity.OnValueChanged += valueHandler;
            },
            untrack: (entity, callback) =>
            {
                // Proper cleanup implementation needed
            }
        );
    }

    public static InlineEntityTrigger CreateDistanceTrigger(
        Vector3 targetPosition, 
        float triggerDistance)
    {
        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                Action<IEntity, int> positionHandler = (e, key) =>
                {
                    if (key == "Position" && e.HasValue<Vector3>("Position"))
                    {
                        Vector3 entityPosition = e.Get<Vector3>("Position");
                        float distance = Vector3.Distance(entityPosition, targetPosition);
                        
                        if (distance <= triggerDistance)
                        {
                            callback(e);
                        }
                    }
                };

                entity.OnValueChanged += positionHandler;
            },
            untrack: (entity, callback) =>
            {
                // Cleanup implementation
            }
        );
    }

    public static InlineEntityTrigger CreatePeriodicTrigger(float intervalSeconds)
    {
        var lastTriggerTimes = new Dictionary<IEntity, float>();

        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                lastTriggerTimes[entity] = Time.time;

                // This would need a proper update mechanism in practice
                void CheckInterval()
                {
                    if (lastTriggerTimes.TryGetValue(entity, out float lastTime))
                    {
                        if (Time.time - lastTime >= intervalSeconds)
                        {
                            callback(entity);
                            lastTriggerTimes[entity] = Time.time;
                        }
                    }
                }
            },
            untrack: (entity, callback) =>
            {
                lastTriggerTimes.Remove(entity);
            }
        );
    }
}

// Usage examples
var healthCriticalTrigger = InlineTriggerFactory.CreateValueThresholdTrigger(
    "Health", 25f, (current, threshold) => current <= threshold
);

var proximityTrigger = InlineTriggerFactory.CreateDistanceTrigger(
    Vector3.zero, 10f
);

var periodicSaveTrigger = InlineTriggerFactory.CreatePeriodicTrigger(30f);
```

## Integration with Atomic Framework

### Filter System Integration

```csharp
public class InlineFilterSystem : MonoBehaviour
{
    [System.Serializable]
    public struct TriggerConfig
    {
        public string name;
        public string[] watchedValues;
        public string[] watchedTags;
    }

    [SerializeField] private TriggerConfig[] _configs;
    
    private readonly Dictionary<string, InlineEntityTrigger> _triggers = new();
    private readonly Dictionary<IEntity, HashSet<string>> _entityTriggers = new();

    void Start()
    {
        foreach (var config in _configs)
        {
            var trigger = CreateTriggerFromConfig(config);
            _triggers[config.name] = trigger;
            
            trigger.SetAction(entity => OnEntityTriggered(config.name, entity));
        }
    }

    private InlineEntityTrigger CreateTriggerFromConfig(TriggerConfig config)
    {
        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                var handlers = new List<Delegate>();

                // Watch specified values
                if (config.watchedValues != null)
                {
                    Action<IEntity, int> valueHandler = (e, key) =>
                    {
                        string keyName = GetKeyName(key); // Implementation needed
                        if (config.watchedValues.Contains(keyName))
                        {
                            callback(e);
                        }
                    };
                    entity.OnValueChanged += valueHandler;
                    handlers.Add(valueHandler);
                }

                // Watch specified tags
                if (config.watchedTags != null)
                {
                    Action<IEntity, int> tagHandler = (e, tagId) =>
                    {
                        string tagName = GetTagName(tagId); // Implementation needed
                        if (config.watchedTags.Contains(tagName))
                        {
                            callback(e);
                        }
                    };
                    entity.OnTagAdded += tagHandler;
                    entity.OnTagDeleted += tagHandler;
                    handlers.Add(tagHandler);
                }

                // Store handlers for cleanup (implementation needed)
                StoreHandlers(entity, config.name, handlers);
            },
            untrack: (entity, callback) =>
            {
                CleanupHandlers(entity, config.name);
            }
        );
    }

    public void StartTracking(IEntity entity)
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

    public void StopTracking(IEntity entity)
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

    private void OnEntityTriggered(string triggerName, IEntity entity)
    {
        Debug.Log($"Entity {entity.Name} triggered: {triggerName}");
        GameEvents.OnEntityTriggered?.Invoke(triggerName, entity);
    }

    private void StoreHandlers(IEntity entity, string triggerName, List<Delegate> handlers)
    {
        // Implementation for proper handler storage and cleanup
    }

    private void CleanupHandlers(IEntity entity, string triggerName)
    {
        // Implementation for proper handler cleanup
    }

    private string GetKeyName(int key) { /* Implementation */ return ""; }
    private string GetTagName(int tagId) { /* Implementation */ return ""; }
}
```

## Implementation Notes

### Delegate Responsibility
- Track and untrack delegates must handle all subscription logic
- Proper event handler storage required for clean unsubscription
- No automatic resource management - delegates handle cleanup

### Performance Considerations
- Delegate invocation overhead for each tracked entity
- Memory overhead for storing custom logic closures
- Efficient for small numbers of entities with complex logic

### Error Handling
- Null delegate validation at construction time
- Exception handling within custom delegate logic
- No built-in error recovery mechanisms

## Best Practices

### Delegate Design
- Keep tracking logic focused and efficient
- Store event handlers properly for cleanup
- Avoid creating excessive closures or allocations

### Resource Management
- Always implement proper unsubscription in untrack delegates
- Use weak references for large object graphs
- Consider object pooling for frequently created triggers

### Testing and Debugging
- Test both track and untrack logic thoroughly
- Verify no memory leaks from improper cleanup
- Use profiler to monitor delegate performance impact

## Common Patterns

### Handler Storage Pattern

```csharp
public class SafeInlineTrigger
{
    private readonly Dictionary<IEntity, List<Delegate>> _storedHandlers = new();

    public InlineEntityTrigger CreateSafeTrigger(
        Func<IEntity, Action<IEntity>, List<Delegate>> trackFunc)
    {
        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                var handlers = trackFunc(entity, callback);
                _storedHandlers[entity] = handlers;
            },
            untrack: (entity, callback) =>
            {
                if (_storedHandlers.TryGetValue(entity, out var handlers))
                {
                    // Cleanup handlers properly
                    foreach (var handler in handlers)
                    {
                        // Remove from appropriate events
                    }
                    _storedHandlers.Remove(entity);
                }
            }
        );
    }
}
```

### Condition Chain Pattern

```csharp
public class ConditionalInlineTrigger
{
    public static InlineEntityTrigger CreateChainedConditionTrigger(
        params Func<IEntity, bool>[] conditions)
    {
        return new InlineEntityTrigger(
            track: (entity, callback) =>
            {
                Action<IEntity, int> handler = (e, _) =>
                {
                    bool allConditionsMet = true;
                    foreach (var condition in conditions)
                    {
                        if (!condition(e))
                        {
                            allConditionsMet = false;
                            break;
                        }
                    }

                    if (allConditionsMet)
                        callback(e);
                };

                entity.OnValueChanged += handler;
                entity.OnTagAdded += handler;
                entity.OnTagDeleted += handler;
            },
            untrack: (entity, callback) =>
            {
                // Proper cleanup implementation needed
            }
        );
    }
}
```

The `InlineEntityTrigger` provides maximum flexibility for creating custom entity monitoring behaviors through delegate-based configuration, making it ideal for prototyping, specialized scenarios, and situations where full class inheritance would be overkill.