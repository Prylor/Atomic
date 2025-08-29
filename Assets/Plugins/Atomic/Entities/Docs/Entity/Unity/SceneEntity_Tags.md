# 🏷️ SceneEntity_Tags

The `SceneEntity_Tags` partial class provides efficient tag management functionality for `SceneEntity` instances. It implements a high-performance hash table-based system for storing, querying, and managing integer-based tags that categorize and identify entity characteristics.

## Key Features

- **High-Performance Storage** – Hash table implementation for O(1) operations
- **Integer-Based Tags** – Efficient integer key storage and lookup
- **Event-Driven** – Events for tag addition and removal
- **Memory Efficient** – Prime-sized buckets for optimal memory usage
- **Enumeration Support** – Custom enumerator for efficient iteration
- **Batch Operations** – Support for adding/removing multiple tags

---

## Core Properties and Events

### Properties
```csharp
// Tag count
public int TagCount { get; }

// Events
public event Action<IEntity, int> OnTagAdded;
public event Action<IEntity, int> OnTagDeleted;
```

### Internal Storage Structure
```csharp
internal struct TagSlot
{
    public int key;      // Tag identifier
    public int next;     // Next slot in hash chain
    public bool exists;  // Slot validity flag
}
```

## Core Methods

### Tag Query Operations
```csharp
// Check if entity has specific tag
public bool HasTag(int key)

// Count total tags
public int TagCount { get; }
```

### Tag Modification Operations
```csharp
// Add single tag
public bool AddTag(int key)

// Remove single tag  
public bool DelTag(int key)

// Clear all tags
public void ClearTags()
```

### Tag Retrieval Operations
```csharp
// Get all tags as array
public int[] GetTags()

// Copy tags to existing array
public int CopyTags(int[] results)

// Get tag enumerator
public TagEnumerator GetTagEnumerator()
```

### Batch Operations
```csharp
// Add multiple tags
public void AddTags(IEnumerable<int> tags)

// Remove multiple tags
public void DelTags(IEnumerable<int> tags)
```

## Example Usage

### Basic Tag Management

```csharp
public class TaggedEntity : SceneEntity
{
    [Header("Initial Tags")]
    [SerializeField] private string[] initialTagNames = new string[]
    {
        "Player",
        "Controllable", 
        "Damageable"
    };
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Subscribe to tag events
        this.OnTagAdded += OnTagAdded;
        this.OnTagDeleted += OnTagDeleted;
        
        // Add initial tags
        foreach (var tagName in initialTagNames)
        {
            int tagId = EntityNames.NameToId(tagName);
            this.AddTag(tagId);
        }
        
        Debug.Log($"Entity initialized with {TagCount} tags");
    }
    
    private void OnTagAdded(IEntity entity, int tag)
    {
        string tagName = EntityNames.IdToName(tag);
        Debug.Log($"Tag added: {tagName} ({tag})");
        
        // React to specific tag additions
        switch (tag)
        {
            case var _ when tag == EntityTags.BURNING:
                StartBurningEffect();
                break;
                
            case var _ when tag == EntityTags.FROZEN:
                StartFreezeEffect();
                break;
                
            case var _ when tag == EntityTags.INVISIBLE:
                StartInvisibilityEffect();
                break;
        }
    }
    
    private void OnTagDeleted(IEntity entity, int tag)
    {
        string tagName = EntityNames.IdToName(tag);
        Debug.Log($"Tag removed: {tagName} ({tag})");
        
        // React to specific tag removals
        switch (tag)
        {
            case var _ when tag == EntityTags.BURNING:
                StopBurningEffect();
                break;
                
            case var _ when tag == EntityTags.FROZEN:
                StopFreezeEffect();
                break;
                
            case var _ when tag == EntityTags.INVISIBLE:
                StopInvisibilityEffect();
                break;
        }
    }
    
    // Effect management methods
    private void StartBurningEffect() { /* Implementation */ }
    private void StopBurningEffect() { /* Implementation */ }
    private void StartFreezeEffect() { /* Implementation */ }
    private void StopFreezeEffect() { /* Implementation */ }
    private void StartInvisibilityEffect() { /* Implementation */ }
    private void StopInvisibilityEffect() { /* Implementation */ }
}
```

### Advanced Tag System

```csharp
public class AdvancedTagSystem : SceneEntity
{
    [Header("Tag Configuration")]
    [SerializeField] private TagCategory[] tagCategories;
    [SerializeField] private bool logTagChanges = true;
    [SerializeField] private int maxTagsPerCategory = 5;
    
    [System.Serializable]
    public class TagCategory
    {
        public string categoryName;
        public List<string> allowedTags;
        public bool exclusive; // Only one tag per category
        public Color debugColor = Color.white;
    }
    
    private Dictionary<string, TagCategory> categoryMap;
    private Dictionary<int, string> tagToCategory;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Initialize tag system
        InitializeTagSystem();
        
        // Subscribe to events
        this.OnTagAdded += ValidateTagAddition;
        this.OnTagDeleted += HandleTagRemoval;
    }
    
    private void InitializeTagSystem()
    {
        categoryMap = new Dictionary<string, TagCategory>();
        tagToCategory = new Dictionary<int, string>();
        
        // Build category mappings
        foreach (var category in tagCategories)
        {
            categoryMap[category.categoryName] = category;
            
            foreach (var tagName in category.allowedTags)
            {
                int tagId = EntityNames.NameToId(tagName);
                tagToCategory[tagId] = category.categoryName;
            }
        }
    }
    
    public bool AddTagToCategory(string tagName, string categoryName)
    {
        if (!categoryMap.ContainsKey(categoryName))
        {
            Debug.LogWarning($"Category '{categoryName}' not found");
            return false;
        }
        
        var category = categoryMap[categoryName];
        int tagId = EntityNames.NameToId(tagName);
        
        // Check if category is exclusive
        if (category.exclusive)
        {
            // Remove other tags from this category first
            var existingTags = GetTagsInCategory(categoryName);
            foreach (var existingTag in existingTags)
            {
                this.DelTag(existingTag);
            }
        }
        
        // Check category limits
        var categoryTags = GetTagsInCategory(categoryName);
        if (categoryTags.Count >= maxTagsPerCategory)
        {
            Debug.LogWarning($"Category '{categoryName}' already has maximum tags ({maxTagsPerCategory})");
            return false;
        }
        
        return this.AddTag(tagId);
    }
    
    public List<int> GetTagsInCategory(string categoryName)
    {
        var result = new List<int>();
        
        using (var enumerator = this.GetTagEnumerator())
        {
            while (enumerator.MoveNext())
            {
                int tag = enumerator.Current;
                if (tagToCategory.TryGetValue(tag, out string cat) && cat == categoryName)
                {
                    result.Add(tag);
                }
            }
        }
        
        return result;
    }
    
    public bool HasTagInCategory(string categoryName)
    {
        return GetTagsInCategory(categoryName).Count > 0;
    }
    
    public void ClearCategory(string categoryName)
    {
        var tagsToRemove = GetTagsInCategory(categoryName);
        foreach (var tag in tagsToRemove)
        {
            this.DelTag(tag);
        }
    }
    
    private void ValidateTagAddition(IEntity entity, int tag)
    {
        if (logTagChanges)
        {
            string tagName = EntityNames.IdToName(tag);
            string category = tagToCategory.TryGetValue(tag, out string cat) ? cat : "Uncategorized";
            Debug.Log($"Added tag '{tagName}' to category '{category}'");
        }
    }
    
    private void HandleTagRemoval(IEntity entity, int tag)
    {
        if (logTagChanges)
        {
            string tagName = EntityNames.IdToName(tag);
            string category = tagToCategory.TryGetValue(tag, out string cat) ? cat : "Uncategorized";
            Debug.Log($"Removed tag '{tagName}' from category '{category}'");
        }
    }
    
    // Utility methods for common tag patterns
    public bool IsPlayer() => this.HasTag(EntityTags.PLAYER);
    public bool IsEnemy() => this.HasTag(EntityTags.ENEMY);
    public bool IsNPC() => this.HasTag(EntityTags.NPC);
    public bool IsAlive() => !this.HasTag(EntityTags.DEAD);
    public bool CanTakeDamage() => this.HasTag(EntityTags.DAMAGEABLE) && IsAlive();
    public bool IsControllable() => this.HasTag(EntityTags.CONTROLLABLE) && IsAlive();
}
```

### Dynamic Tag System

```csharp
public class DynamicTaggedEntity : SceneEntity
{
    [Header("Dynamic Tagging")]
    [SerializeField] private float tagUpdateInterval = 1.0f;
    [SerializeField] private bool enableAutoTagging = true;
    
    private float lastTagUpdate;
    private Dictionary<int, float> temporaryTags = new Dictionary<int, float>();
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Add dynamic tag behaviors
        this.AddBehaviour(new DynamicTagBehaviour());
        this.AddBehaviour(new TagExpirationBehaviour());
    }
    
    protected override void OnUpdate(float deltaTime)
    {
        base.OnUpdate(deltaTime);
        
        if (enableAutoTagging && Time.time - lastTagUpdate >= tagUpdateInterval)
        {
            UpdateDynamicTags();
            lastTagUpdate = Time.time;
        }
        
        UpdateTemporaryTags(deltaTime);
    }
    
    private void UpdateDynamicTags()
    {
        // Auto-tag based on current state
        if (this.HasValue(EntityNames.HEALTH))
        {
            float health = this.GetValue<float>(EntityNames.HEALTH);
            float maxHealth = this.GetValue<float>(EntityNames.MAX_HEALTH, 100f);
            float healthPercent = health / maxHealth;
            
            // Health-based tags
            if (healthPercent < 0.25f)
                this.AddTag(EntityTags.CRITICAL_HEALTH);
            else
                this.DelTag(EntityTags.CRITICAL_HEALTH);
                
            if (healthPercent < 0.5f)
                this.AddTag(EntityTags.LOW_HEALTH);
            else
                this.DelTag(EntityTags.LOW_HEALTH);
        }
        
        // Movement-based tags
        if (this.HasValue(EntityNames.VELOCITY))
        {
            Vector3 velocity = this.GetValue<Vector3>(EntityNames.VELOCITY);
            
            if (velocity.magnitude > 0.1f)
                this.AddTag(EntityTags.MOVING);
            else
                this.DelTag(EntityTags.MOVING);
                
            if (velocity.magnitude > 10f)
                this.AddTag(EntityTags.FAST_MOVING);
            else
                this.DelTag(EntityTags.FAST_MOVING);
        }
        
        // Position-based tags
        Vector3 position = transform.position;
        if (position.y > 10f)
            this.AddTag(EntityTags.AIRBORNE);
        else
            this.DelTag(EntityTags.AIRBORNE);
            
        if (position.y < -5f)
            this.AddTag(EntityTags.UNDERGROUND);
        else
            this.DelTag(EntityTags.UNDERGROUND);
    }
    
    public void AddTemporaryTag(int tag, float duration)
    {
        this.AddTag(tag);
        temporaryTags[tag] = Time.time + duration;
        
        Debug.Log($"Added temporary tag {EntityNames.IdToName(tag)} for {duration} seconds");
    }
    
    private void UpdateTemporaryTags(float deltaTime)
    {
        var tagsToRemove = new List<int>();
        
        foreach (var kvp in temporaryTags)
        {
            if (Time.time >= kvp.Value)
            {
                tagsToRemove.Add(kvp.Key);
            }
        }
        
        foreach (var tag in tagsToRemove)
        {
            temporaryTags.Remove(tag);
            this.DelTag(tag);
            Debug.Log($"Removed expired temporary tag {EntityNames.IdToName(tag)}");
        }
    }
}

public class DynamicTagBehaviour : IEntityBehaviour, IEntityUpdate
{
    public void OnInstall(IEntity entity) { }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Additional dynamic tag logic based on entity state
        UpdateCombatTags(entity);
        UpdateEnvironmentalTags(entity);
        UpdateInteractionTags(entity);
    }
    
    private void UpdateCombatTags(IEntity entity)
    {
        // Example: Add combat tags based on recent damage
        if (entity.HasValue(EntityNames.LAST_DAMAGE_TIME))
        {
            float lastDamageTime = entity.GetValue<float>(EntityNames.LAST_DAMAGE_TIME);
            bool recentlyDamaged = Time.time - lastDamageTime < 5f;
            
            if (recentlyDamaged && !entity.HasTag(EntityTags.RECENTLY_DAMAGED))
                entity.AddTag(EntityTags.RECENTLY_DAMAGED);
            else if (!recentlyDamaged && entity.HasTag(EntityTags.RECENTLY_DAMAGED))
                entity.DelTag(EntityTags.RECENTLY_DAMAGED);
        }
    }
    
    private void UpdateEnvironmentalTags(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        // Example: Environmental detection
        Vector3 position = sceneEntity.transform.position;
        
        // Water detection
        if (Physics.CheckSphere(position, 0.5f, LayerMask.GetMask("Water")))
            entity.AddTag(EntityTags.IN_WATER);
        else
            entity.DelTag(EntityTags.IN_WATER);
    }
    
    private void UpdateInteractionTags(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        // Example: Interaction range detection
        var nearbyEntities = Physics.OverlapSphere(sceneEntity.transform.position, 2f);
        bool hasNearbyPlayer = false;
        
        foreach (var collider in nearbyEntities)
        {
            var otherEntity = collider.GetComponent<SceneEntity>();
            if (otherEntity != null && otherEntity.HasTag(EntityTags.PLAYER))
            {
                hasNearbyPlayer = true;
                break;
            }
        }
        
        if (hasNearbyPlayer && !entity.HasTag(EntityTags.NEAR_PLAYER))
            entity.AddTag(EntityTags.NEAR_PLAYER);
        else if (!hasNearbyPlayer && entity.HasTag(EntityTags.NEAR_PLAYER))
            entity.DelTag(EntityTags.NEAR_PLAYER);
    }
}
```

### Tag-Based Entity Filtering

```csharp
public class TagBasedEntityFilter : MonoBehaviour
{
    [Header("Filter Configuration")]
    [SerializeField] private int[] requiredTags;
    [SerializeField] private int[] excludedTags;
    [SerializeField] private bool requireAllTags = false;
    [SerializeField] private float updateInterval = 0.5f;
    
    private List<SceneEntity> filteredEntities = new List<SceneEntity>();
    private float lastUpdateTime;
    
    public IReadOnlyList<SceneEntity> FilteredEntities => filteredEntities;
    
    void Update()
    {
        if (Time.time - lastUpdateTime >= updateInterval)
        {
            UpdateFilteredEntities();
            lastUpdateTime = Time.time;
        }
    }
    
    private void UpdateFilteredEntities()
    {
        filteredEntities.Clear();
        
        var allEntities = FindObjectsOfType<SceneEntity>();
        foreach (var entity in allEntities)
        {
            if (PassesFilter(entity))
            {
                filteredEntities.Add(entity);
            }
        }
        
        Debug.Log($"Found {filteredEntities.Count} entities matching filter criteria");
    }
    
    private bool PassesFilter(SceneEntity entity)
    {
        // Check excluded tags first
        foreach (var excludedTag in excludedTags)
        {
            if (entity.HasTag(excludedTag))
                return false;
        }
        
        // Check required tags
        if (requireAllTags)
        {
            // Entity must have ALL required tags
            foreach (var requiredTag in requiredTags)
            {
                if (!entity.HasTag(requiredTag))
                    return false;
            }
        }
        else
        {
            // Entity must have at least ONE required tag
            bool hasRequiredTag = false;
            foreach (var requiredTag in requiredTags)
            {
                if (entity.HasTag(requiredTag))
                {
                    hasRequiredTag = true;
                    break;
                }
            }
            
            if (requiredTags.Length > 0 && !hasRequiredTag)
                return false;
        }
        
        return true;
    }
    
    public List<SceneEntity> GetEntitiesWithAllTags(params int[] tags)
    {
        var result = new List<SceneEntity>();
        
        foreach (var entity in filteredEntities)
        {
            bool hasAllTags = true;
            foreach (var tag in tags)
            {
                if (!entity.HasTag(tag))
                {
                    hasAllTags = false;
                    break;
                }
            }
            
            if (hasAllTags)
                result.Add(entity);
        }
        
        return result;
    }
    
    public List<SceneEntity> GetEntitiesWithAnyTag(params int[] tags)
    {
        var result = new List<SceneEntity>();
        
        foreach (var entity in filteredEntities)
        {
            foreach (var tag in tags)
            {
                if (entity.HasTag(tag))
                {
                    result.Add(entity);
                    break;
                }
            }
        }
        
        return result;
    }
}
```

## Performance Characteristics

### Time Complexity
- **HasTag**: O(1) average case
- **AddTag**: O(1) average case  
- **DelTag**: O(1) average case
- **Enumeration**: O(n) where n is tag count

### Memory Usage
- Hash table with prime-sized buckets
- Efficient slot-based storage
- Minimal memory overhead per tag

### Hash Table Implementation
- Uses prime numbers for bucket sizing
- Chaining for collision resolution
- Dynamic resizing maintains load factor

## Best Practices

1. **Tag Design** – Use meaningful tag constants rather than magic numbers
2. **Event Handling** – Subscribe to tag events for reactive behavior
3. **Performance** – Leverage O(1) lookup for frequent tag checks
4. **Enumeration** – Use custom enumerator for efficient iteration
5. **Batch Operations** – Add/remove multiple tags efficiently
6. **Memory Management** – Consider tag system capacity during entity design

## Common Patterns

### State-Based Tagging
```csharp
public void UpdateHealthTags(float health, float maxHealth)
{
    float healthPercent = health / maxHealth;
    
    this.DelTag(EntityTags.FULL_HEALTH);
    this.DelTag(EntityTags.LOW_HEALTH);
    this.DelTag(EntityTags.CRITICAL_HEALTH);
    
    if (healthPercent >= 1.0f)
        this.AddTag(EntityTags.FULL_HEALTH);
    else if (healthPercent < 0.25f)
        this.AddTag(EntityTags.CRITICAL_HEALTH);
    else if (healthPercent < 0.5f)
        this.AddTag(EntityTags.LOW_HEALTH);
}
```

### Tag-Based Queries
```csharp
public bool CanInteract()
{
    return this.HasTag(EntityTags.INTERACTABLE) && 
           !this.HasTag(EntityTags.BUSY) && 
           !this.HasTag(EntityTags.DEAD);
}

public bool IsHostileToPlayer()
{
    return this.HasTag(EntityTags.ENEMY) || 
           this.HasTag(EntityTags.HOSTILE);
}
```

### Conditional Tag Management
```csharp
public void AddTagIfNotPresent(int tag)
{
    if (!this.HasTag(tag))
        this.AddTag(tag);
}

public void ToggleTag(int tag)
{
    if (this.HasTag(tag))
        this.DelTag(tag);
    else
        this.AddTag(tag);
}
```

## Notes

- Tags are stored as integer identifiers for maximum performance
- Hash table implementation provides consistent O(1) performance
- Events are fired synchronously after tag operations complete
- Custom enumerator avoids garbage allocation during iteration
- Supports dynamic resizing to maintain optimal performance
- Integration with EntityNames provides string-to-int mapping for convenience