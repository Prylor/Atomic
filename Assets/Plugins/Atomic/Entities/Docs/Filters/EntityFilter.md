# 🧩 EntityFilter

`EntityFilter` provides a dynamic, observable filtered view over an entity collection. It automatically maintains a subset of entities based on a predicate and optional triggers, updating in real-time as entities change.

## Key Features

- **Dynamic Filtering** – Automatically updates as entities change
- **Predicate-Based** – Filter entities using custom logic
- **Trigger System** – React to specific entity changes
- **Observable** – Events for tracking filter changes
- **Lazy Evaluation** – Only evaluates when accessed
- **Memory Efficient** – Doesn't duplicate entity storage

---

## Class Definition

```csharp
public class EntityFilter : EntityFilter<IEntity>, IReadOnlyEntityCollection
{
    public EntityFilter(
        IReadOnlyEntityCollection<IEntity> source,
        Predicate<IEntity> predicate,
        params IEntityTrigger<IEntity>[] triggers)
}

public class EntityFilter<E> : IReadOnlyEntityCollection<E>, IDisposable where E : IEntity
{
    public event Action OnStateChanged;
    public event Action<E> OnAdded;
    public event Action<E> OnRemoved;
    
    public int Count { get; }
}
```

## Constructor Parameters

### source
- **Type**: `IReadOnlyEntityCollection<E>`
- **Description**: The source collection to filter
- **Note**: Filter observes this collection for changes

### predicate
- **Type**: `Predicate<E>`
- **Description**: Function determining if entity passes filter
- **Returns**: `true` to include entity, `false` to exclude

### triggers
- **Type**: `IEntityTrigger<E>[]`
- **Description**: Optional triggers for re-evaluation
- **Purpose**: Detect when entities should be re-filtered

## How It Works

1. **Initial Filtering** – Evaluates all source entities with predicate
2. **Source Monitoring** – Watches source collection for additions/removals
3. **Trigger Monitoring** – Watches triggers for entity state changes
4. **Re-evaluation** – Re-runs predicate when triggers fire
5. **Event Notification** – Notifies subscribers of filter changes

## Example Usage

### Basic Filtering

```csharp
// Create a filter for alive enemies
var allEntities = new EntityCollection<IEntity>();
var aliveEnemies = new EntityFilter(
    source: allEntities,
    predicate: entity => 
        entity.HasTag(EntityTags.ENEMY) && 
        entity.HasTag(EntityTags.ALIVE)
);

// Subscribe to filter changes
aliveEnemies.OnAdded += enemy => Debug.Log($"Enemy entered combat: {enemy.Name}");
aliveEnemies.OnRemoved += enemy => Debug.Log($"Enemy left combat: {enemy.Name}");

// Filter automatically updates as entities change
foreach (var enemy in aliveEnemies)
{
    // Process only alive enemies
    ProcessEnemy(enemy);
}
```

### Filter with Triggers

```csharp
// Create triggers for dynamic re-evaluation
var healthTrigger = new ValueEntityTrigger(EntityNames.HEALTH);
var tagTrigger = new TagEntityTrigger(EntityTags.ALIVE);

// Filter low-health allies
var lowHealthAllies = new EntityFilter(
    source: alliedUnits,
    predicate: entity =>
    {
        if (!entity.HasTag(EntityTags.ALLY))
            return false;
            
        if (!entity.TryGetValue<int>(EntityNames.HEALTH, out int health))
            return false;
            
        return health < 30;
    },
    triggers: healthTrigger, tagTrigger
);

// React to filter changes
lowHealthAllies.OnAdded += ally =>
{
    // Ally needs healing
    PrioritizeHealing(ally);
};
```

### Complex Filtering Scenarios

```csharp
public class CombatSystem
{
    private EntityFilter visibleEnemies;
    private EntityFilter targetableEnemies;
    private EntityFilter threateningEnemies;
    
    public void Initialize(IReadOnlyEntityCollection<IEntity> enemies)
    {
        // Enemies in view range
        visibleEnemies = new EntityFilter(
            enemies,
            entity => IsInViewRange(entity) && !entity.HasTag(EntityTags.INVISIBLE)
        );
        
        // Enemies that can be targeted
        targetableEnemies = new EntityFilter(
            visibleEnemies,  // Chain filters!
            entity => !entity.HasTag(EntityTags.INVULNERABLE) && 
                     entity.HasTag(EntityTags.ALIVE)
        );
        
        // High-threat enemies
        threateningEnemies = new EntityFilter(
            targetableEnemies,
            entity =>
            {
                var threat = CalculateThreatLevel(entity);
                return threat > 0.7f;
            }
        );
    }
    
    private bool IsInViewRange(IEntity entity)
    {
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        var distance = Vector3.Distance(playerPosition, position);
        return distance <= viewRange;
    }
}
```

### Dynamic Filter Creation

```csharp
public static class FilterFactory
{
    public static EntityFilter CreateTagFilter(
        IReadOnlyEntityCollection<IEntity> source,
        params int[] requiredTags)
    {
        return new EntityFilter(
            source,
            entity =>
            {
                foreach (int tag in requiredTags)
                {
                    if (!entity.HasTag(tag))
                        return false;
                }
                return true;
            },
            requiredTags.Select(tag => new TagEntityTrigger(tag)).ToArray()
        );
    }
    
    public static EntityFilter CreateValueRangeFilter<T>(
        IReadOnlyEntityCollection<IEntity> source,
        int valueKey,
        T min,
        T max) where T : IComparable<T>
    {
        return new EntityFilter(
            source,
            entity =>
            {
                if (!entity.TryGetValue<T>(valueKey, out T value))
                    return false;
                    
                return value.CompareTo(min) >= 0 && value.CompareTo(max) <= 0;
            },
            new ValueEntityTrigger(valueKey)
        );
    }
    
    public static EntityFilter CreateRadiusFilter(
        IReadOnlyEntityCollection<IEntity> source,
        Vector3 center,
        float radius)
    {
        return new EntityFilter(
            source,
            entity =>
            {
                var position = entity.GetValue<Vector3>(EntityNames.POSITION);
                return Vector3.Distance(center, position) <= radius;
            },
            new ValueEntityTrigger(EntityNames.POSITION)
        );
    }
}
```

### Filter Composition

```csharp
public class FilterComposer
{
    // AND composition
    public static EntityFilter And(
        IReadOnlyEntityCollection<IEntity> source,
        params Predicate<IEntity>[] predicates)
    {
        return new EntityFilter(
            source,
            entity =>
            {
                foreach (var predicate in predicates)
                {
                    if (!predicate(entity))
                        return false;
                }
                return true;
            }
        );
    }
    
    // OR composition
    public static EntityFilter Or(
        IReadOnlyEntityCollection<IEntity> source,
        params Predicate<IEntity>[] predicates)
    {
        return new EntityFilter(
            source,
            entity =>
            {
                foreach (var predicate in predicates)
                {
                    if (predicate(entity))
                        return true;
                }
                return false;
            }
        );
    }
    
    // NOT composition
    public static EntityFilter Not(
        IReadOnlyEntityCollection<IEntity> source,
        Predicate<IEntity> predicate)
    {
        return new EntityFilter(source, entity => !predicate(entity));
    }
}
```

### Procedural Filter Usage

Following Atomic's procedural approach:

```csharp
public static class EntityFilterUtils
{
    private static Dictionary<string, EntityFilter> filterCache = 
        new Dictionary<string, EntityFilter>();
    
    public static EntityFilter GetOrCreateFilter(
        string key,
        IReadOnlyEntityCollection<IEntity> source,
        Predicate<IEntity> predicate)
    {
        if (!filterCache.TryGetValue(key, out var filter))
        {
            filter = new EntityFilter(source, predicate);
            filterCache[key] = filter;
        }
        return filter;
    }
    
    public static List<IEntity> GetFiltered(
        IReadOnlyEntityCollection<IEntity> source,
        Predicate<IEntity> predicate)
    {
        var result = new List<IEntity>();
        foreach (var entity in source)
        {
            if (predicate(entity))
            {
                result.Add(entity);
            }
        }
        return result;
    }
    
    public static void ProcessFiltered(
        IReadOnlyEntityCollection<IEntity> source,
        Predicate<IEntity> predicate,
        Action<IEntity> processor)
    {
        foreach (var entity in source)
        {
            if (predicate(entity))
            {
                processor(entity);
            }
        }
    }
}
```

## Trigger Types

### TagEntityTrigger
- Fires when specified tag is added/removed
- Use for state-based filtering

### ValueEntityTrigger
- Fires when specified value changes
- Use for data-based filtering

### SubscriptionEntityTrigger
- Custom trigger with manual control
- Use for complex conditions

## Best Practices

1. **Reuse Filters** – Create once, use multiple times
2. **Chain Filters** – Use filtered results as source for other filters
3. **Simple Predicates** – Keep predicate logic fast and simple
4. **Appropriate Triggers** – Only use triggers for values that affect filter
5. **Dispose Filters** – Call Dispose() to unsubscribe from events
6. **Cache Results** – Store filter results if used multiple times per frame

## Performance Considerations

- **Lazy Evaluation** – Filter only evaluates when accessed
- **Predicate Cost** – Called for each entity on evaluation
- **Trigger Overhead** – Each trigger adds event subscriptions
- **Memory Efficient** – Doesn't duplicate entity references
- **Re-evaluation Cost** – Full re-filter when triggers fire

## Common Use Cases

- **Combat Targeting** – Find valid targets
- **AI Decision Making** – Filter relevant entities
- **UI Display** – Show filtered entity lists
- **Spatial Queries** – Entities in range
- **State Queries** – Entities with specific states
- **Team Management** – Filter by allegiance