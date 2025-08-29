# 🧩 IEntitySpawn

The `IEntitySpawn` interface defines behaviours that execute initialization logic when an entity is spawned into the world. This interface is called automatically during the entity spawn process, making it ideal for setting up initial state, registering event listeners, or preparing resources.

## Key Features

- **Automatic Invocation** – Called automatically when `IEntity.Spawn()` is executed
- **Initialization Point** – Perfect for setting up initial entity state and values
- **Generic Support** – Includes strongly-typed generic version for specific entity types
- **Procedural Design** – Operates on entity data rather than storing internal state
- **Event-Driven** – Reactive to entity lifecycle events

---

## Interface Definition

```csharp
public interface IEntitySpawn : IEntityBehaviour
{
    void OnSpawn(IEntity entity);
}

public interface IEntitySpawn<in T> : IEntitySpawn where T : IEntity
{
    void OnSpawn(T entity);
}
```

## When It's Called

The `OnSpawn` method is automatically invoked when:
- An entity transitions from despawned to spawned state
- `IEntity.Spawn()` is called on a despawned entity
- An entity is created and immediately spawned through factory methods
- Pool-managed entities are retrieved and activated

The spawn behaviour is called **before** activate behaviours, ensuring proper initialization order.

## Example Implementations

### Basic Initialization Behaviour

```csharp
public class HealthInitializerBehaviour : IEntitySpawn
{
    private readonly int maxHealth;
    
    public HealthInitializerBehaviour(int maxHealth)
    {
        this.maxHealth = maxHealth;
    }
    
    public void OnSpawn(IEntity entity)
    {
        // Initialize health values when entity spawns
        entity.SetValue(EntityNames.MAX_HEALTH, maxHealth);
        entity.SetValue(EntityNames.CURRENT_HEALTH, maxHealth);
        entity.RemoveTag(EntityTags.DEAD);
        
        Debug.Log($"Entity {entity.Name} spawned with {maxHealth} health");
    }
}
```

### Position and Transform Setup

```csharp
public class SpawnPositionBehaviour : IEntitySpawn
{
    private readonly Vector3 spawnPosition;
    private readonly Quaternion spawnRotation;
    
    public SpawnPositionBehaviour(Vector3 position, Quaternion rotation)
    {
        spawnPosition = position;
        spawnRotation = rotation;
    }
    
    public void OnSpawn(IEntity entity)
    {
        // Set initial position and rotation
        entity.SetValue(EntityNames.POSITION, spawnPosition);
        entity.SetValue(EntityNames.ROTATION, spawnRotation);
        entity.SetValue(EntityNames.SCALE, Vector3.one);
        
        // Apply to Unity Transform if it exists
        if (entity.TryGetValue<Transform>(EntityNames.TRANSFORM, out var transform))
        {
            transform.position = spawnPosition;
            transform.rotation = spawnRotation;
        }
    }
}
```

### Resource Loading Behaviour

```csharp
public class AssetLoaderBehaviour : IEntitySpawn
{
    private readonly string assetPath;
    
    public AssetLoaderBehaviour(string assetPath)
    {
        this.assetPath = assetPath;
    }
    
    public void OnSpawn(IEntity entity)
    {
        // Load required assets when spawning
        var asset = Resources.Load(assetPath);
        if (asset != null)
        {
            entity.SetValue(EntityNames.LOADED_ASSET, asset);
            entity.AddTag(EntityTags.ASSETS_LOADED);
        }
        else
        {
            Debug.LogWarning($"Failed to load asset: {assetPath}");
            entity.AddTag(EntityTags.ASSETS_MISSING);
        }
    }
}
```

### Generic Typed Implementation

```csharp
public class PlayerSpawnBehaviour : IEntitySpawn<PlayerEntity>
{
    private readonly PlayerConfig config;
    
    public PlayerSpawnBehaviour(PlayerConfig config)
    {
        this.config = config;
    }
    
    public void OnSpawn(PlayerEntity entity)
    {
        // Strongly-typed entity access
        entity.SetPlayerLevel(config.StartingLevel);
        entity.SetExperience(0);
        entity.SetPlayerName(config.PlayerName);
        
        // Set initial stats
        entity.SetValue(EntityNames.MOVE_SPEED, config.BaseSpeed);
        entity.SetValue(EntityNames.JUMP_POWER, config.JumpPower);
        
        // Initialize inventory
        entity.AddBehaviour(new InventoryBehaviour(config.InventorySize));
        
        // Apply player-specific tags
        entity.AddTag(EntityTags.PLAYER);
        entity.AddTag(EntityTags.CONTROLLABLE);
    }
}
```

### Event Subscription Behaviour

```csharp
public class EventSubscriberBehaviour : IEntitySpawn
{
    public void OnSpawn(IEntity entity)
    {
        // Subscribe to entity events during spawn
        entity.OnValueChanged += HandleValueChanged;
        entity.OnTagAdded += HandleTagAdded;
        entity.OnTagRemoved += HandleTagRemoved;
        
        // Register with global systems
        GameEvents.OnLevelChanged += (level) => HandleLevelChange(entity, level);
        
        entity.AddTag(EntityTags.EVENT_SUBSCRIBED);
    }
    
    private void HandleValueChanged(IEntity entity, int key)
    {
        // React to value changes
        if (key == EntityNames.HEALTH)
        {
            var health = entity.GetValue<int>(key);
            if (health <= 0)
            {
                entity.AddTag(EntityTags.DEAD);
            }
        }
    }
    
    private void HandleTagAdded(IEntity entity, int tag)
    {
        // React to tag additions
        if (tag == EntityTags.POISONED)
        {
            entity.SetValue(EntityNames.POISON_START_TIME, Time.time);
        }
    }
}
```

## Best Practices

### 1. Initialize Entity State

```csharp
public void OnSpawn(IEntity entity)
{
    // Set required initial values
    entity.SetValue(EntityNames.HEALTH, maxHealth);
    entity.SetValue(EntityNames.CREATED_TIME, Time.time);
    
    // Apply initial tags
    entity.AddTag(EntityTags.ALIVE);
    entity.RemoveTag(EntityTags.DEAD);
}
```

### 2. Use Configuration Data

```csharp
public class ConfigurableSpawnBehaviour : IEntitySpawn
{
    private readonly EntityConfig config;
    
    public ConfigurableSpawnBehaviour(EntityConfig config)
    {
        this.config = config ?? throw new ArgumentNullException(nameof(config));
    }
    
    public void OnSpawn(IEntity entity)
    {
        // Apply configuration values
        foreach (var kvp in config.InitialValues)
        {
            entity.SetValue(kvp.Key, kvp.Value);
        }
        
        foreach (var tag in config.InitialTags)
        {
            entity.AddTag(tag);
        }
    }
}
```

### 3. Handle Dependencies Safely

```csharp
public void OnSpawn(IEntity entity)
{
    // Check for required dependencies
    if (!entity.HasValue(EntityNames.TRANSFORM))
    {
        Debug.LogError($"Entity {entity.Name} requires Transform component");
        return;
    }
    
    // Initialize dependent systems
    InitializePhysics(entity);
    InitializeRenderer(entity);
}
```

### 4. Avoid State Storage

```csharp
// GOOD: Stateless behaviour
public class SpawnEffectBehaviour : IEntitySpawn
{
    public void OnSpawn(IEntity entity)
    {
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        EffectsSystem.PlaySpawnEffect(position);
    }
}

// AVOID: Storing state in behaviour
public class BadSpawnBehaviour : IEntitySpawn
{
    private Vector3 lastSpawnPosition; // Avoid storing state
    
    public void OnSpawn(IEntity entity)
    {
        lastSpawnPosition = entity.GetValue<Vector3>(EntityNames.POSITION); // Store in entity instead
        entity.SetValue(EntityNames.LAST_SPAWN_POSITION, lastSpawnPosition);
    }
}
```

## Common Use Cases

### Entity Initialization
- Setting default values for health, position, scale
- Applying configuration data from ScriptableObjects
- Initializing component references

### Resource Management
- Loading required assets and textures
- Allocating memory buffers
- Creating temporary objects

### System Registration
- Registering entity with update loops
- Adding to spatial partitioning systems
- Subscribing to global events

### Validation and Setup
- Validating required components exist
- Setting up inter-entity relationships
- Initializing state machines

### Visual and Audio
- Playing spawn effects and sounds
- Setting up initial animations
- Configuring renderer properties

## Notes

- **Call Order**: OnSpawn is called before OnActivate in the entity lifecycle
- **Multiple Behaviours**: Multiple spawn behaviours can be attached to one entity
- **Generic Benefits**: Use `IEntitySpawn<T>` for strongly-typed entity access
- **Error Handling**: Always validate entity state before setting values
- **Performance**: Keep spawn logic lightweight for frequently spawned entities
- **Cleanup**: Pair with `IEntityDespawn` for proper resource management