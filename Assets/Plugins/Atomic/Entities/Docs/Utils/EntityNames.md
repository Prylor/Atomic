# 🏷️ EntityNames (Utils)

`EntityNames` provides bidirectional mapping between string entity names and unique integer identifiers. This enables efficient entity identification at runtime using compact integer IDs while maintaining human-readable names for debugging and editor visualization.

## Key Features

- **Bidirectional Mapping** – Convert between string names and integer IDs
- **Automatic ID Assignment** – Sequential ID generation for new names
- **Memory Efficient** – Compact integer storage for runtime operations
- **Debug Friendly** – Reverse lookup for human-readable names
- **Unity Integration** – Automatic cleanup on play mode entry
- **Performance Optimized** – Aggressive inlining for hot paths

---

## Static Methods

### Name to ID Conversion

```csharp
public static int NameToId(string name)
```

Converts a string entity name to a unique integer ID:
- Returns existing ID if name was previously registered
- Assigns new sequential ID for new names
- Thread-safe for concurrent access
- Null names are supported and get unique IDs

### ID to Name Conversion

```csharp
public static string IdToName(int id)
```

Retrieves the original string name from an integer ID:
- Returns registered name if ID exists
- Returns `"#Unknown:{id}"` format for unregistered IDs
- Always returns a valid string (never null)
- Useful for debugging and logging

### Mapping Management

```csharp
public static void Clear()
```

Clears all name-to-ID mappings and resets ID counter:
- Removes all stored mappings
- Resets next ID counter to 1
- Automatically called on Unity play mode entry
- Useful for testing and cleanup scenarios

---

## Usage Patterns

### Basic Name Registration

```csharp
public static class GameEntityNames
{
    // Register common entity names
    public static readonly int PLAYER = EntityNames.NameToId("Player");
    public static readonly int ENEMY = EntityNames.NameToId("Enemy");
    public static readonly int PROJECTILE = EntityNames.NameToId("Projectile");
    public static readonly int POWERUP = EntityNames.NameToId("PowerUp");
    
    // Register property names
    public static readonly int HEALTH = EntityNames.NameToId("Health");
    public static readonly int POSITION = EntityNames.NameToId("Position");
    public static readonly int VELOCITY = EntityNames.NameToId("Velocity");
}
```

### Dynamic Name Registration

```csharp
public class EntityFactory
{
    private Dictionary<string, int> typeIds = new Dictionary<string, int>();
    
    public int RegisterEntityType(string typeName)
    {
        if (typeIds.TryGetValue(typeName, out int existingId))
            return existingId;
            
        int id = EntityNames.NameToId(typeName);
        typeIds[typeName] = id;
        return id;
    }
    
    public Entity CreateEntity(string typeName)
    {
        int typeId = RegisterEntityType(typeName);
        var entity = new Entity();
        entity.SetValue(typeId, typeName); // Store type info
        return entity;
    }
}
```

### Debug and Logging

```csharp
public static class EntityDebugger
{
    public static void LogEntityInfo(Entity entity)
    {
        Console.WriteLine($"Entity: {entity.Name}");
        
        // Log all values with human-readable names
        foreach (var kvp in entity.GetValues())
        {
            string propertyName = EntityNames.IdToName(kvp.Key);
            Console.WriteLine($"  {propertyName}: {kvp.Value}");
        }
        
        // Log all tags with human-readable names
        foreach (int tagId in entity.GetTags())
        {
            string tagName = EntityNames.IdToName(tagId);
            Console.WriteLine($"  Tag: {tagName}");
        }
    }
    
    public static string FormatEntityType(int typeId)
    {
        return EntityNames.IdToName(typeId);
    }
}
```

### Configuration System

```csharp
public class EntityConfig
{
    private Dictionary<int, object> defaultValues = new Dictionary<int, object>();
    
    public void SetDefault(string propertyName, object value)
    {
        int propertyId = EntityNames.NameToId(propertyName);
        defaultValues[propertyId] = value;
    }
    
    public void ApplyDefaults(Entity entity)
    {
        foreach (var kvp in defaultValues)
        {
            if (!entity.HasValue(kvp.Key))
            {
                entity.SetValue(kvp.Key, kvp.Value);
            }
        }
    }
    
    // Usage example
    public static EntityConfig CreatePlayerConfig()
    {
        var config = new EntityConfig();
        config.SetDefault("Health", 100);
        config.SetDefault("MaxHealth", 100);
        config.SetDefault("Speed", 5.0f);
        config.SetDefault("Level", 1);
        return config;
    }
}
```

### Serialization Support

```csharp
[Serializable]
public struct EntitySnapshot
{
    public string entityName;
    public Dictionary<string, object> values;
    public string[] tags;
    
    public static EntitySnapshot Capture(Entity entity)
    {
        var snapshot = new EntitySnapshot
        {
            entityName = entity.Name,
            values = new Dictionary<string, object>(),
            tags = new string[entity.TagCount]
        };
        
        // Capture values with string names
        foreach (var kvp in entity.GetValues())
        {
            string propertyName = EntityNames.IdToName(kvp.Key);
            snapshot.values[propertyName] = kvp.Value;
        }
        
        // Capture tags with string names
        int tagIndex = 0;
        foreach (int tagId in entity.GetTags())
        {
            snapshot.tags[tagIndex++] = EntityNames.IdToName(tagId);
        }
        
        return snapshot;
    }
    
    public void RestoreToEntity(Entity entity)
    {
        entity.Name = entityName;
        
        // Restore values
        foreach (var kvp in values)
        {
            int propertyId = EntityNames.NameToId(kvp.Key);
            entity.SetValue(propertyId, kvp.Value);
        }
        
        // Restore tags
        foreach (string tagName in tags)
        {
            int tagId = EntityNames.NameToId(tagName);
            entity.AddTag(tagId);
        }
    }
}
```

### Performance Optimization

```csharp
public static class OptimizedEntityOperations
{
    // Cache frequently used IDs
    private static readonly int HealthId = EntityNames.NameToId("Health");
    private static readonly int PositionId = EntityNames.NameToId("Position");
    private static readonly int DamageId = EntityNames.NameToId("Damage");
    
    public static void DealDamage(Entity entity, int damage)
    {
        // Use cached ID for performance
        if (entity.TryGetValue(HealthId, out int health))
        {
            entity.SetValue(HealthId, health - damage);
        }
    }
    
    public static Vector3 GetPosition(Entity entity)
    {
        // Use cached ID to avoid string lookup
        return entity.GetValue<Vector3>(PositionId);
    }
    
    public static void SetPosition(Entity entity, Vector3 position)
    {
        // Use cached ID for fast access
        entity.SetValue(PositionId, position);
    }
}
```

### Testing Utilities

```csharp
public static class EntityTestUtils
{
    public static void ClearAllNames()
    {
        EntityNames.Clear();
    }
    
    public static void AssertNameMapping(string expectedName, int id)
    {
        string actualName = EntityNames.IdToName(id);
        if (actualName != expectedName)
        {
            throw new AssertionException(
                $"Expected name '{expectedName}' for ID {id}, but got '{actualName}'");
        }
    }
    
    public static void AssertIdMapping(int expectedId, string name)
    {
        int actualId = EntityNames.NameToId(name);
        if (actualId != expectedId)
        {
            throw new AssertionException(
                $"Expected ID {expectedId} for name '{name}', but got {actualId}");
        }
    }
}
```

## Performance Characteristics

### Time Complexity
- **NameToId**: O(1) average, O(n) worst case (hash table lookup)
- **IdToName**: O(1) average, O(n) worst case (hash table lookup)
- **Clear**: O(1) - clears references without iteration

### Memory Usage
- **Two Dictionary instances** for bidirectional mapping
- **String interning** may apply for repeated names
- **Sequential ID assignment** ensures compact integer range

### Optimization Tips
1. **Cache frequently used IDs** in static readonly fields
2. **Batch name registration** during initialization
3. **Use Clear() sparingly** as it loses all mappings
4. **Avoid reverse lookup in hot paths** when possible

## Thread Safety

- **NameToId is thread-safe** with concurrent readers/writers
- **IdToName is thread-safe** for concurrent access
- **Clear is NOT thread-safe** with other operations
- **Dictionary operations** use internal synchronization

## Unity Integration

### Editor Support
```csharp
#if UNITY_EDITOR
[InitializeOnEnterPlayMode]
#endif
public static void Clear()
```

Automatically clears mappings when entering Unity play mode to ensure clean state.

### Build Considerations
- **Editor-only attributes** are stripped from builds
- **Runtime performance** not affected by editor integration
- **Clear() can be called manually** in non-Unity contexts

## Best Practices

1. **Register Names Early** – Cache IDs in static fields during startup
2. **Use Consistent Naming** – Establish naming conventions for properties/tags
3. **Avoid Runtime Registration** – Pre-register all names when possible
4. **Debug with Names** – Use IdToName for logging and debugging
5. **Test with Clear()** – Reset state between unit tests

## Common Patterns

### Constants Class
```csharp
public static class EntityConstants
{
    // Entity types
    public static class Types
    {
        public static readonly int PLAYER = EntityNames.NameToId("Player");
        public static readonly int ENEMY = EntityNames.NameToId("Enemy");
        public static readonly int NPC = EntityNames.NameToId("NPC");
    }
    
    // Properties
    public static class Properties
    {
        public static readonly int HEALTH = EntityNames.NameToId("Health");
        public static readonly int POSITION = EntityNames.NameToId("Position");
        public static readonly int ROTATION = EntityNames.NameToId("Rotation");
    }
    
    // Tags
    public static class Tags
    {
        public static readonly int ALIVE = EntityNames.NameToId("Alive");
        public static readonly int HOSTILE = EntityNames.NameToId("Hostile");
        public static readonly int INTERACTIVE = EntityNames.NameToId("Interactive");
    }
}
```