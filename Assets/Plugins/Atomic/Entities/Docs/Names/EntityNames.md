# 🏷️ EntityNames (Global Constants)

`EntityNames` in this context refers to global entity name and key constants used throughout the Atomic.Entities framework. This documentation covers the standard naming conventions and constant definitions for entity identification, property keys, and tag names.

## Key Features

- **Global Constants** – Framework-wide entity name definitions
- **Consistent Naming** – Standardized naming conventions
- **Type Safety** – Compile-time constant definitions
- **Documentation** – Clear purpose for each name constant
- **Extensible** – Easy to add custom constants

---

## Standard Entity Names

### Core Entity Types

```csharp
public static class EntityTypes
{
    public const string ENTITY = "Entity";
    public const string SCENE_ENTITY = "SceneEntity";
    public const string PROXY_ENTITY = "ProxyEntity";
    public const string POOLED_ENTITY = "PooledEntity";
}
```

### Gameplay Entity Types

```csharp
public static class GameplayTypes
{
    public const string PLAYER = "Player";
    public const string ENEMY = "Enemy";
    public const string NPC = "NPC";
    public const string PROJECTILE = "Projectile";
    public const string POWERUP = "PowerUp";
    public const string WEAPON = "Weapon";
    public const string ITEM = "Item";
    public const string TRIGGER = "Trigger";
    public const string SPAWNER = "Spawner";
    public const string COLLECTIBLE = "Collectible";
}
```

## Standard Property Names

### Transform Properties

```csharp
public static class TransformProperties
{
    public const string POSITION = "Position";
    public const string ROTATION = "Rotation";
    public const string SCALE = "Scale";
    public const string LOCAL_POSITION = "LocalPosition";
    public const string LOCAL_ROTATION = "LocalRotation";
    public const string LOCAL_SCALE = "LocalScale";
    public const string WORLD_MATRIX = "WorldMatrix";
    public const string LOCAL_MATRIX = "LocalMatrix";
}
```

### Physics Properties

```csharp
public static class PhysicsProperties
{
    public const string VELOCITY = "Velocity";
    public const string ACCELERATION = "Acceleration";
    public const string FORCE = "Force";
    public const string MASS = "Mass";
    public const string DRAG = "Drag";
    public const string ANGULAR_VELOCITY = "AngularVelocity";
    public const string ANGULAR_DRAG = "AngularDrag";
    public const string GRAVITY_SCALE = "GravityScale";
    public const string IS_KINEMATIC = "IsKinematic";
}
```

### Gameplay Properties

```csharp
public static class GameplayProperties
{
    public const string HEALTH = "Health";
    public const string MAX_HEALTH = "MaxHealth";
    public const string ARMOR = "Armor";
    public const string SHIELD = "Shield";
    public const string DAMAGE = "Damage";
    public const string ATTACK_POWER = "AttackPower";
    public const string DEFENSE = "Defense";
    public const string SPEED = "Speed";
    public const string LEVEL = "Level";
    public const string EXPERIENCE = "Experience";
    public const string SCORE = "Score";
}
```

### Resource Properties

```csharp
public static class ResourceProperties
{
    public const string ENERGY = "Energy";
    public const string MAX_ENERGY = "MaxEnergy";
    public const string MANA = "Mana";
    public const string MAX_MANA = "MaxMana";
    public const string STAMINA = "Stamina";
    public const string MAX_STAMINA = "MaxStamina";
    public const string FUEL = "Fuel";
    public const string MAX_FUEL = "MaxFuel";
    public const string AMMO = "Ammo";
    public const string MAX_AMMO = "MaxAmmo";
}
```

## Standard Tag Names

### State Tags

```csharp
public static class StateTags
{
    public const string ALIVE = "Alive";
    public const string DEAD = "Dead";
    public const string ACTIVE = "Active";
    public const string INACTIVE = "Inactive";
    public const string ENABLED = "Enabled";
    public const string DISABLED = "Disabled";
    public const string VISIBLE = "Visible";
    public const string HIDDEN = "Hidden";
    public const string SPAWNED = "Spawned";
    public const string DESPAWNED = "Despawned";
}
```

### Gameplay Tags

```csharp
public static class GameplayTags
{
    public const string PLAYER_CONTROLLED = "PlayerControlled";
    public const string AI_CONTROLLED = "AIControlled";
    public const string HOSTILE = "Hostile";
    public const string FRIENDLY = "Friendly";
    public const string NEUTRAL = "Neutral";
    public const string DAMAGEABLE = "Damageable";
    public const string INVULNERABLE = "Invulnerable";
    public const string INTERACTIVE = "Interactive";
    public const string PICKUPABLE = "Pickupable";
    public const string DESTROYABLE = "Destroyable";
}
```

### System Tags

```csharp
public static class SystemTags
{
    public const string PERSISTENT = "Persistent";
    public const string TEMPORARY = "Temporary";
    public const string POOLED = "Pooled";
    public const string SINGLETON = "Singleton";
    public const string DEBUG = "Debug";
    public const string EDITOR_ONLY = "EditorOnly";
    public const string RUNTIME_ONLY = "RuntimeOnly";
    public const string NETWORK_SYNCED = "NetworkSynced";
    public const string SERIALIZABLE = "Serializable";
}
```

## Usage Patterns

### Creating Named Constants

```csharp
public static class CustomEntityNames
{
    // Custom entity types for your game
    public const string SPACESHIP = "Spaceship";
    public const string ASTEROID = "Asteroid";
    public const string SPACE_STATION = "SpaceStation";
    
    // Custom properties
    public const string HULL_INTEGRITY = "HullIntegrity";
    public const string SHIELD_POWER = "ShieldPower";
    public const string ENGINE_THRUST = "EngineThrust";
    
    // Custom tags
    public const string PILOTED = "Piloted";
    public const string DRIFTING = "Drifting";
    public const string MINING_CAPABLE = "MiningCapable";
}
```

### Using with EntityNames Utility

```csharp
public static class GameConstants
{
    // Convert string constants to integer IDs for runtime efficiency
    public static readonly int PLAYER_ID = EntityNames.NameToId(GameplayTypes.PLAYER);
    public static readonly int HEALTH_ID = EntityNames.NameToId(GameplayProperties.HEALTH);
    public static readonly int ALIVE_ID = EntityNames.NameToId(StateTags.ALIVE);
    
    // Cache all commonly used IDs during initialization
    public static void InitializeConstants()
    {
        // Pre-register all standard names
        foreach (var field in typeof(GameplayTypes).GetFields())
        {
            if (field.FieldType == typeof(string))
            {
                string value = (string)field.GetValue(null);
                EntityNames.NameToId(value);
            }
        }
        
        foreach (var field in typeof(GameplayProperties).GetFields())
        {
            if (field.FieldType == typeof(string))
            {
                string value = (string)field.GetValue(null);
                EntityNames.NameToId(value);
            }
        }
    }
}
```

### Configuration-Driven Names

```csharp
[System.Serializable]
public class EntityNameConfig
{
    [Header("Entity Types")]
    public string playerType = GameplayTypes.PLAYER;
    public string enemyType = GameplayTypes.ENEMY;
    public string npcType = GameplayTypes.NPC;
    
    [Header("Core Properties")]
    public string healthProperty = GameplayProperties.HEALTH;
    public string positionProperty = TransformProperties.POSITION;
    public string speedProperty = GameplayProperties.SPEED;
    
    [Header("Common Tags")]
    public string aliveTag = StateTags.ALIVE;
    public string hostileTag = GameplayTags.HOSTILE;
    public string interactiveTag = GameplayTags.INTERACTIVE;
}

public class ConfigurableEntityFactory
{
    private EntityNameConfig config;
    
    public Entity CreatePlayer(Vector3 position)
    {
        var entity = new Entity(config.playerType);
        entity.SetValue(EntityNames.NameToId(config.positionProperty), position);
        entity.SetValue(EntityNames.NameToId(config.healthProperty), 100);
        entity.AddTag(EntityNames.NameToId(config.aliveTag));
        return entity;
    }
}
```

### Validation and Debug Utils

```csharp
public static class EntityNameValidator
{
    public static void ValidateStandardNames()
    {
        var allTypes = new[]
        {
            typeof(EntityTypes),
            typeof(GameplayTypes),
            typeof(TransformProperties),
            typeof(PhysicsProperties),
            typeof(GameplayProperties),
            typeof(ResourceProperties),
            typeof(StateTags),
            typeof(GameplayTags),
            typeof(SystemTags)
        };
        
        var allNames = new HashSet<string>();
        var duplicates = new List<string>();
        
        foreach (var type in allTypes)
        {
            foreach (var field in type.GetFields(BindingFlags.Public | BindingFlags.Static))
            {
                if (field.FieldType == typeof(string))
                {
                    string value = (string)field.GetValue(null);
                    if (!allNames.Add(value))
                    {
                        duplicates.Add(value);
                    }
                }
            }
        }
        
        if (duplicates.Count > 0)
        {
            Debug.LogWarning($"Duplicate entity names found: {string.Join(", ", duplicates)}");
        }
    }
    
    public static void LogAllRegisteredNames()
    {
        Debug.Log("=== Registered Entity Names ===");
        
        // Log all standard categories
        LogNamesFromType(typeof(EntityTypes), "Entity Types");
        LogNamesFromType(typeof(GameplayTypes), "Gameplay Types");
        LogNamesFromType(typeof(TransformProperties), "Transform Properties");
        LogNamesFromType(typeof(GameplayProperties), "Gameplay Properties");
        LogNamesFromType(typeof(StateTags), "State Tags");
        LogNamesFromType(typeof(GameplayTags), "Gameplay Tags");
    }
    
    private static void LogNamesFromType(Type type, string category)
    {
        Debug.Log($"--- {category} ---");
        foreach (var field in type.GetFields(BindingFlags.Public | BindingFlags.Static))
        {
            if (field.FieldType == typeof(string))
            {
                string value = (string)field.GetValue(null);
                int id = EntityNames.NameToId(value);
                Debug.Log($"  {field.Name} = \"{value}\" (ID: {id})");
            }
        }
    }
}
```

## Integration with Entity System

### Entity Factory with Standard Names

```csharp
public static class StandardEntityFactory
{
    public static Entity CreatePlayer(string name = null)
    {
        var entity = new Entity(name ?? GameplayTypes.PLAYER);
        
        // Add standard player properties
        entity.SetValue(EntityNames.NameToId(GameplayProperties.HEALTH), 100);
        entity.SetValue(EntityNames.NameToId(GameplayProperties.MAX_HEALTH), 100);
        entity.SetValue(EntityNames.NameToId(GameplayProperties.SPEED), 5.0f);
        entity.SetValue(EntityNames.NameToId(GameplayProperties.LEVEL), 1);
        
        // Add standard player tags
        entity.AddTag(EntityNames.NameToId(StateTags.ALIVE));
        entity.AddTag(EntityNames.NameToId(GameplayTags.PLAYER_CONTROLLED));
        entity.AddTag(EntityNames.NameToId(GameplayTags.DAMAGEABLE));
        
        return entity;
    }
    
    public static Entity CreateEnemy(string enemyType, int health, float speed)
    {
        var entity = new Entity(enemyType);
        
        // Add standard enemy properties
        entity.SetValue(EntityNames.NameToId(GameplayProperties.HEALTH), health);
        entity.SetValue(EntityNames.NameToId(GameplayProperties.MAX_HEALTH), health);
        entity.SetValue(EntityNames.NameToId(GameplayProperties.SPEED), speed);
        
        // Add standard enemy tags
        entity.AddTag(EntityNames.NameToId(StateTags.ALIVE));
        entity.AddTag(EntityNames.NameToId(GameplayTags.AI_CONTROLLED));
        entity.AddTag(EntityNames.NameToId(GameplayTags.HOSTILE));
        entity.AddTag(EntityNames.NameToId(GameplayTags.DAMAGEABLE));
        
        return entity;
    }
}
```

### Query System with Standard Names

```csharp
public static class StandardEntityQueries
{
    private static readonly int AliveTagId = EntityNames.NameToId(StateTags.ALIVE);
    private static readonly int HealthPropertyId = EntityNames.NameToId(GameplayProperties.HEALTH);
    private static readonly int HostileTagId = EntityNames.NameToId(GameplayTags.HOSTILE);
    
    public static IEnumerable<Entity> GetAliveEntities(IEntityWorld world)
    {
        return world.Entities.Where(e => e.HasTag(AliveTagId));
    }
    
    public static IEnumerable<Entity> GetDamagedEntities(IEntityWorld world)
    {
        return world.Entities.Where(e => 
            e.HasTag(AliveTagId) && 
            e.TryGetValue(HealthPropertyId, out int health) && 
            e.TryGetValue(EntityNames.NameToId(GameplayProperties.MAX_HEALTH), out int maxHealth) &&
            health < maxHealth);
    }
    
    public static IEnumerable<Entity> GetHostileEntities(IEntityWorld world)
    {
        return world.Entities.Where(e => 
            e.HasTag(AliveTagId) && 
            e.HasTag(HostileTagId));
    }
}
```

## Best Practices

### Naming Conventions

1. **Use ALL_CAPS** for constant names
2. **Use PascalCase** for string values
3. **Be descriptive** but concise
4. **Group related constants** in nested classes
5. **Avoid abbreviations** unless widely understood

### Organization

1. **Separate by purpose** – Types, Properties, Tags
2. **Use nested classes** for logical grouping
3. **Document purpose** of each constant group
4. **Maintain alphabetical order** within groups
5. **Version changes** carefully to avoid breaking existing code

### Performance

1. **Cache integer IDs** for frequently used names
2. **Pre-register constants** during initialization
3. **Use const instead of static readonly** when possible
4. **Avoid runtime string concatenation** for names
5. **Profile name resolution** in performance-critical code

### Extensibility

1. **Create custom constant classes** for domain-specific names
2. **Follow existing patterns** for consistency
3. **Document custom names** thoroughly
4. **Validate uniqueness** to prevent conflicts
5. **Consider namespacing** for large projects

This documentation provides a foundation for consistent entity naming throughout the Atomic.Entities framework, enabling better code organization, improved debugging, and enhanced maintainability.