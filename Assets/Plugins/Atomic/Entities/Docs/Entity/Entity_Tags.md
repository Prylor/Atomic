# 🏷️ Entity_Tags

`Entity_Tags` is a partial class implementation that manages tag storage for entities using a high-performance hash table optimized for integer keys. Tags provide lightweight boolean flags for entity classification and filtering.

## Key Features

- **High-Performance Hash Table** – Prime-sized hash tables with separate chaining
- **Integer-Only Storage** – Lightweight tag representation using integer keys
- **Collision Resolution** – Efficient separate chaining for hash conflicts
- **Memory Efficient** – Minimal overhead per tag with slot-based allocation
- **Event System** – Tag addition/removal notifications
- **Fast Lookups** – O(1) average case tag existence checks

---

## Tag Storage Architecture

### Internal Structure

```csharp
internal struct TagSlot
{
    public int key;          // Tag identifier
    public int next;         // Collision chain pointer  
    public bool exists;      // Slot validity flag
}
```

### Storage Fields

```csharp
private TagSlot[] _tagSlots;         // Hash table slots
private int[] _tagBuckets;           // Hash bucket array
private int _tagCapacity;            // Current capacity
private int _tagCount;               // Number of active tags
private int _tagFreeList;            // Free slot chain
private int _tagLastIndex;           // Last used slot index
private int _tagPrimeIndex;          // Current prime table index
```

---

## Public API

### Events

```csharp
public event Action<IEntity, int> OnTagAdded;      // Tag added event
public event Action<IEntity, int> OnTagDeleted;    // Tag deleted event
```

### Properties

```csharp
public int TagCount { get; }                       // Total tag count
```

### Tag Management

```csharp
public bool HasTag(int key)                        // Check tag existence
public bool AddTag(int key)                        // Add tag (returns success)
public bool DelTag(int key)                        // Remove tag (returns success)
public void ClearTags()                            // Remove all tags
```

### Bulk Operations

```csharp
public int[] GetTags()                             // Get all tag IDs
public int CopyTags(int[] results)                 // Copy tags to array
```

### Enumeration

```csharp
public TagEnumerator GetTagEnumerator()            // Struct-based enumerator
IEnumerator<int> IEntity.GetTagEnumerator()       // Interface implementation
```

---

## Usage Patterns

### Basic Tag Management

```csharp
public class EntityTagging
{
    // Define tag constants
    private static readonly int PLAYER_TAG = EntityNames.NameToId("Player");
    private static readonly int ENEMY_TAG = EntityNames.NameToId("Enemy");
    private static readonly int ALIVE_TAG = EntityNames.NameToId("Alive");
    private static readonly int HOSTILE_TAG = EntityNames.NameToId("Hostile");
    
    public void SetupPlayerEntity(Entity entity)
    {
        entity.AddTag(PLAYER_TAG);
        entity.AddTag(ALIVE_TAG);
        
        Debug.Log($"Player entity has {entity.TagCount} tags");
        Debug.Log($"Is player alive: {entity.HasTag(ALIVE_TAG)}");
    }
    
    public void SetupEnemyEntity(Entity entity)
    {
        entity.AddTag(ENEMY_TAG);
        entity.AddTag(ALIVE_TAG);
        entity.AddTag(HOSTILE_TAG);
        
        // Tag addition is idempotent
        bool added = entity.AddTag(ENEMY_TAG); // Returns false - already exists
        Debug.Log($"Enemy tag re-added: {added}");
    }
}
```

### Event-Driven Tag Monitoring

```csharp
public class TagEventSystem
{
    private readonly Dictionary<int, List<Action<Entity>>> tagAddedCallbacks = new();
    private readonly Dictionary<int, List<Action<Entity>>> tagRemovedCallbacks = new();
    
    public void MonitorEntity(Entity entity)
    {
        entity.OnTagAdded += OnTagAdded;
        entity.OnTagDeleted += OnTagDeleted;
    }
    
    public void RegisterTagCallback(int tagId, Action<Entity> onAdded, Action<Entity> onRemoved = null)
    {
        if (onAdded != null)
        {
            if (!tagAddedCallbacks.ContainsKey(tagId))
                tagAddedCallbacks[tagId] = new List<Action<Entity>>();
            tagAddedCallbacks[tagId].Add(onAdded);
        }
        
        if (onRemoved != null)
        {
            if (!tagRemovedCallbacks.ContainsKey(tagId))
                tagRemovedCallbacks[tagId] = new List<Action<Entity>>();
            tagRemovedCallbacks[tagId].Add(onRemoved);
        }
    }
    
    private void OnTagAdded(IEntity entity, int tagId)
    {
        if (tagAddedCallbacks.TryGetValue(tagId, out var callbacks))
        {
            foreach (var callback in callbacks)
                callback(entity as Entity);
        }
        
        string tagName = EntityNames.IdToName(tagId);
        Debug.Log($"Tag added: {tagName} to entity {entity.Name}");
    }
    
    private void OnTagDeleted(IEntity entity, int tagId)
    {
        if (tagRemovedCallbacks.TryGetValue(tagId, out var callbacks))
        {
            foreach (var callback in callbacks)
                callback(entity as Entity);
        }
        
        string tagName = EntityNames.IdToName(tagId);
        Debug.Log($"Tag removed: {tagName} from entity {entity.Name}");
    }
}
```

### Tag-Based Entity Filtering

```csharp
public static class EntityTagFilters
{
    public static IEnumerable<Entity> WithTag(this IEnumerable<Entity> entities, int tagId)
    {
        return entities.Where(entity => entity.HasTag(tagId));
    }
    
    public static IEnumerable<Entity> WithAllTags(this IEnumerable<Entity> entities, params int[] tagIds)
    {
        return entities.Where(entity => tagIds.All(tag => entity.HasTag(tag)));
    }
    
    public static IEnumerable<Entity> WithAnyTag(this IEnumerable<Entity> entities, params int[] tagIds)
    {
        return entities.Where(entity => tagIds.Any(tag => entity.HasTag(tag)));
    }
    
    public static IEnumerable<Entity> WithoutTag(this IEnumerable<Entity> entities, int tagId)
    {
        return entities.Where(entity => !entity.HasTag(tagId));
    }
}

// Usage example
public class EntityQuerySystem
{
    private static readonly int ALIVE_TAG = EntityNames.NameToId("Alive");
    private static readonly int ENEMY_TAG = EntityNames.NameToId("Enemy");
    private static readonly int PLAYER_TAG = EntityNames.NameToId("Player");
    
    public void ProcessLivingEnemies(IEnumerable<Entity> entities)
    {
        var livingEnemies = entities
            .WithTag(ALIVE_TAG)
            .WithTag(ENEMY_TAG)
            .ToList();
            
        Debug.Log($"Found {livingEnemies.Count} living enemies");
        
        foreach (var enemy in livingEnemies)
        {
            ProcessEnemyAI(enemy);
        }
    }
    
    public void ProcessPlayersAndEnemies(IEnumerable<Entity> entities)
    {
        var combatants = entities
            .WithAnyTag(PLAYER_TAG, ENEMY_TAG)
            .WithTag(ALIVE_TAG)
            .ToList();
            
        foreach (var combatant in combatants)
        {
            ProcessCombatLogic(combatant);
        }
    }
}
```

### Tag State Management

```csharp
public class EntityStateManager
{
    private static readonly int ALIVE_TAG = EntityNames.NameToId("Alive");
    private static readonly int DEAD_TAG = EntityNames.NameToId("Dead");
    private static readonly int ACTIVE_TAG = EntityNames.NameToId("Active");
    private static readonly int DISABLED_TAG = EntityNames.NameToId("Disabled");
    
    public void SetEntityAlive(Entity entity, bool alive)
    {
        if (alive)
        {
            entity.AddTag(ALIVE_TAG);
            entity.DelTag(DEAD_TAG);
        }
        else
        {
            entity.DelTag(ALIVE_TAG);
            entity.AddTag(DEAD_TAG);
        }
    }
    
    public void SetEntityActive(Entity entity, bool active)
    {
        if (active)
        {
            entity.AddTag(ACTIVE_TAG);
            entity.DelTag(DISABLED_TAG);
        }
        else
        {
            entity.DelTag(ACTIVE_TAG);
            entity.AddTag(DISABLED_TAG);
        }
    }
    
    public bool IsEntityInValidState(Entity entity)
    {
        // Entity cannot be both alive and dead
        bool isAlive = entity.HasTag(ALIVE_TAG);
        bool isDead = entity.HasTag(DEAD_TAG);
        
        if (isAlive && isDead)
        {
            Debug.LogError($"Entity {entity.Name} has conflicting alive/dead tags");
            return false;
        }
        
        // Entity cannot be both active and disabled
        bool isActive = entity.HasTag(ACTIVE_TAG);
        bool isDisabled = entity.HasTag(DISABLED_TAG);
        
        if (isActive && isDisabled)
        {
            Debug.LogError($"Entity {entity.Name} has conflicting active/disabled tags");
            return false;
        }
        
        return true;
    }
}
```

### Tag-Based Component Systems

```csharp
public abstract class TagBasedSystem
{
    protected abstract int[] RequiredTags { get; }
    protected abstract int[] ExcludedTags { get; }
    
    public virtual bool CanProcessEntity(Entity entity)
    {
        // Check required tags
        if (RequiredTags != null)
        {
            foreach (int tag in RequiredTags)
            {
                if (!entity.HasTag(tag))
                    return false;
            }
        }
        
        // Check excluded tags
        if (ExcludedTags != null)
        {
            foreach (int tag in ExcludedTags)
            {
                if (entity.HasTag(tag))
                    return false;
            }
        }
        
        return true;
    }
    
    public abstract void ProcessEntity(Entity entity, float deltaTime);
}

public class MovementSystem : TagBasedSystem
{
    private static readonly int MOVABLE_TAG = EntityNames.NameToId("Movable");
    private static readonly int FROZEN_TAG = EntityNames.NameToId("Frozen");
    
    protected override int[] RequiredTags => new[] { MOVABLE_TAG };
    protected override int[] ExcludedTags => new[] { FROZEN_TAG };
    
    public override void ProcessEntity(Entity entity, float deltaTime)
    {
        // Process movement for entities that are movable but not frozen
        UpdateEntityMovement(entity, deltaTime);
    }
}

public class RenderSystem : TagBasedSystem
{
    private static readonly int VISIBLE_TAG = EntityNames.NameToId("Visible");
    private static readonly int HIDDEN_TAG = EntityNames.NameToId("Hidden");
    
    protected override int[] RequiredTags => new[] { VISIBLE_TAG };
    protected override int[] ExcludedTags => new[] { HIDDEN_TAG };
    
    public override void ProcessEntity(Entity entity, float deltaTime)
    {
        // Render entities that are visible but not hidden
        RenderEntity(entity);
    }
}
```

### Performance-Optimized Tag Operations

```csharp
public class OptimizedTagOperations
{
    // Cache frequently used tag IDs
    private static readonly int PLAYER_TAG = EntityNames.NameToId("Player");
    private static readonly int ENEMY_TAG = EntityNames.NameToId("Enemy");
    private static readonly int ALIVE_TAG = EntityNames.NameToId("Alive");
    
    public void BulkTagOperations(Entity entity)
    {
        // Batch multiple tag operations to minimize event overhead
        var tagsToAdd = new[] { PLAYER_TAG, ALIVE_TAG };
        
        foreach (int tag in tagsToAdd)
        {
            entity.AddTag(tag);
        }
        
        // Use array-based tag checking for multiple tags
        CheckMultipleTags(entity, tagsToAdd);
    }
    
    public bool CheckMultipleTags(Entity entity, int[] tags)
    {
        // Efficient multiple tag checking
        foreach (int tag in tags)
        {
            if (!entity.HasTag(tag))
                return false;
        }
        return true;
    }
    
    public void OptimizedTagCopy(Entity source, Entity destination)
    {
        // Efficient tag copying
        var sourceTags = source.GetTags();
        
        // Clear destination tags in bulk
        destination.ClearTags();
        
        // Add all source tags
        foreach (int tag in sourceTags)
        {
            destination.AddTag(tag);
        }
    }
}
```

## Performance Characteristics

### Time Complexity
- **HasTag**: O(1) average, O(n) worst case (hash collision)
- **AddTag/DelTag**: O(1) average, O(n) worst case
- **ClearTags**: O(n) - must notify about all deletions
- **Resize**: O(n) - rehashing all existing tags

### Space Complexity
- **Memory overhead**: ~12 bytes per tag slot
- **Hash table**: Prime-sized for optimal distribution  
- **No boxing**: Integer keys stored directly
- **Free list**: Efficient slot reuse after deletion

## Best Practices

### Performance Optimization
1. **Cache tag IDs** – Pre-calculate IDs for frequently used tags
2. **Batch operations** when adding/removing multiple tags
3. **Use tag existence checks** before addition to avoid redundant operations
4. **Pre-allocate capacity** if tag count is known

### Memory Management
1. **Clear unused tags** to free memory slots
2. **Monitor tag proliferation** in large entity collections
3. **Use meaningful tag hierarchies** to minimize tag count
4. **Consider tag lifetime** when designing systems

### Design Patterns
1. **Tag as state** – Use mutually exclusive tags for states
2. **Tag as capability** – Use tags to indicate entity capabilities
3. **Tag as classification** – Use tags for entity type identification
4. **Tag as temporary markers** – Use tags for transient states

## Thread Safety

- **Not thread-safe** – All operations must be on main thread
- **Event callbacks** execute synchronously on calling thread
- **Hash table modifications** not atomic across operations
- **Enumeration** not safe during concurrent modifications