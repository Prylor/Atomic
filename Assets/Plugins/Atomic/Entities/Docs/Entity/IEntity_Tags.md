# 🧩 IEntity_Tags

`IEntity_Tags` is a partial interface of `IEntity` that provides tag management functionality. Tags are lightweight integer identifiers used for categorizing, filtering, and identifying entities.

## Key Features

- **Lightweight Categorization** – Integer-based tags for minimal memory overhead
- **Fast Lookups** – O(1) tag checking for efficient filtering
- **Dynamic Management** – Add/remove tags at runtime
- **Event Notifications** – Track tag additions and removals
- **Bulk Operations** – Methods for working with multiple tags

---

## Events

```csharp
event Action<IEntity, int> OnTagAdded;
event Action<IEntity, int> OnTagDeleted;
```

### OnTagAdded
- **Triggered**: When a new tag is added to the entity
- **Parameters**: The entity and the added tag identifier

### OnTagDeleted
- **Triggered**: When a tag is removed from the entity
- **Parameters**: The entity and the removed tag identifier

## Properties

```csharp
int TagCount { get; }
```
- **Description**: Returns the number of tags currently associated with the entity
- **Access**: Read-only

## Methods

### Tag Checking

```csharp
bool HasTag(int key)
```
- **Description**: Checks if the entity has the specified tag
- **Returns**: `true` if tag exists, `false` otherwise
- **Performance**: O(1) lookup time

### Tag Management

```csharp
bool AddTag(int key)
```
- **Description**: Adds a tag to the entity
- **Returns**: `true` if tag was added, `false` if already existed
- **Events**: Triggers `OnTagAdded` if successful

```csharp
bool DelTag(int key)
```
- **Description**: Removes a tag from the entity
- **Returns**: `true` if tag was removed, `false` if not found
- **Events**: Triggers `OnTagDeleted` if successful

```csharp
void ClearTags()
```
- **Description**: Removes all tags from the entity
- **Events**: Triggers `OnTagDeleted` for each removed tag

### Bulk Operations

```csharp
int[] GetTags()
```
- **Description**: Returns all tag identifiers as an array
- **Note**: Allocates a new array

```csharp
int CopyTags(int[] results)
```
- **Description**: Copies tag identifiers into provided array
- **Returns**: Number of tags copied
- **Performance**: No allocation if array is sufficient size

```csharp
IEnumerator<int> GetTagEnumerator()
```
- **Description**: Returns enumerator for iterating over tags
- **Usage**: For foreach loops and LINQ operations

## Example Usage

### Basic Tag Management

```csharp
// Add tags to categorize entity
entity.AddTag(EntityTags.PLAYER);
entity.AddTag(EntityTags.CONTROLLABLE);
entity.AddTag(EntityTags.DAMAGEABLE);

// Check for tags
if (entity.HasTag(EntityTags.PLAYER))
{
    // Player-specific logic
    EnablePlayerControls(entity);
}

// Remove temporary tags
entity.DelTag(EntityTags.INVULNERABLE);

// Clear all tags
entity.ClearTags();
```

### Tag-Based Filtering

```csharp
public static class EntityFilter
{
    public static List<IEntity> GetEntitiesWithTag(IEnumerable<IEntity> entities, int tag)
    {
        var result = new List<IEntity>();
        foreach (var entity in entities)
        {
            if (entity.HasTag(tag))
            {
                result.Add(entity);
            }
        }
        return result;
    }
    
    public static List<IEntity> GetEntitiesWithAllTags(
        IEnumerable<IEntity> entities, 
        params int[] requiredTags)
    {
        var result = new List<IEntity>();
        foreach (var entity in entities)
        {
            bool hasAll = true;
            foreach (int tag in requiredTags)
            {
                if (!entity.HasTag(tag))
                {
                    hasAll = false;
                    break;
                }
            }
            if (hasAll)
            {
                result.Add(entity);
            }
        }
        return result;
    }
}

// Usage
var players = EntityFilter.GetEntitiesWithTag(allEntities, EntityTags.PLAYER);
var enemies = EntityFilter.GetEntitiesWithAllTags(
    allEntities, 
    EntityTags.ENEMY, 
    EntityTags.ALIVE
);
```

### Reactive Tag System

```csharp
// Subscribe to tag changes
entity.OnTagAdded += (e, tag) =>
{
    switch (tag)
    {
        case EntityTags.BURNING:
            StartBurningEffect(e);
            break;
        case EntityTags.FROZEN:
            StartFreezeEffect(e);
            break;
        case EntityTags.INVISIBLE:
            SetEntityVisibility(e, false);
            break;
    }
};

entity.OnTagDeleted += (e, tag) =>
{
    switch (tag)
    {
        case EntityTags.BURNING:
            StopBurningEffect(e);
            break;
        case EntityTags.FROZEN:
            StopFreezeEffect(e);
            break;
        case EntityTags.INVISIBLE:
            SetEntityVisibility(e, true);
            break;
    }
};
```

### Tag Constants Definition

```csharp
public static class EntityTags
{
    // Entity types
    public const int PLAYER = 1;
    public const int ENEMY = 2;
    public const int NPC = 3;
    public const int PROJECTILE = 4;
    public const int PICKUP = 5;
    
    // States
    public const int ALIVE = 10;
    public const int DEAD = 11;
    public const int SPAWNING = 12;
    public const int DESPAWNING = 13;
    
    // Abilities
    public const int CONTROLLABLE = 20;
    public const int DAMAGEABLE = 21;
    public const int MOVEABLE = 22;
    public const int INTERACTABLE = 23;
    
    // Status effects
    public const int BURNING = 30;
    public const int FROZEN = 31;
    public const int POISONED = 32;
    public const int INVULNERABLE = 33;
    public const int INVISIBLE = 34;
    
    // AI states
    public const int IDLE = 40;
    public const int PATROLLING = 41;
    public const int CHASING = 42;
    public const int ATTACKING = 43;
    public const int FLEEING = 44;
}
```

### Procedural Tag Operations

Following Atomic's procedural approach:

```csharp
public static class EntityTagUtils
{
    public static void ApplyStatusEffect(IEntity entity, int effectTag, float duration)
    {
        if (!entity.HasTag(effectTag))
        {
            entity.AddTag(effectTag);
            ScheduleTagRemoval(entity, effectTag, duration);
        }
    }
    
    public static void TransitionState(IEntity entity, int fromTag, int toTag)
    {
        entity.DelTag(fromTag);
        entity.AddTag(toTag);
    }
    
    public static bool HasAnyTag(IEntity entity, params int[] tags)
    {
        foreach (int tag in tags)
        {
            if (entity.HasTag(tag))
                return true;
        }
        return false;
    }
    
    public static bool HasAllTags(IEntity entity, params int[] tags)
    {
        foreach (int tag in tags)
        {
            if (!entity.HasTag(tag))
                return false;
        }
        return true;
    }
    
    public static void CopyTags(IEntity source, IEntity target)
    {
        var tags = source.GetTags();
        foreach (int tag in tags)
        {
            target.AddTag(tag);
        }
    }
}
```

## Best Practices

1. **Define Tag Constants** – Use named constants instead of magic numbers
2. **Group Related Tags** – Organize tags by category (types, states, effects)
3. **Avoid Tag Explosion** – Don't use tags for data that should be values
4. **Use for Filtering** – Tags are ideal for entity queries and filtering
5. **Combine with Values** – Use tags for categories, values for data
6. **Performance First** – Tags are faster than string comparisons

## Performance Notes

- **HashSet Storage** – Tags stored in HashSet for O(1) operations
- **Integer Comparison** – Faster than string-based categorization
- **Memory Efficient** – Each tag is just 4 bytes
- **Cache-Friendly** – Small memory footprint improves cache performance

## Common Use Cases

- **Entity Types** – Player, Enemy, NPC, Projectile
- **State Tracking** – Alive, Dead, Active, Inactive
- **Capabilities** – CanMove, CanAttack, CanInteract
- **Status Effects** – Burning, Frozen, Poisoned, Buffed
- **AI States** – Idle, Patrolling, Attacking, Fleeing
- **Filtering** – Finding all enemies, all interactables, etc.