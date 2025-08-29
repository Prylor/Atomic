# 🧩 IEntityFactory

`IEntityFactory` is the core interface for entity creation in the Atomic framework. It provides a standardized way to instantiate and configure entities, supporting both generic and non-generic implementations for flexible entity production patterns.

## Key Features

- **Generic Type Safety** – Strongly typed entity creation with `IEntityFactory<T>`
- **Non-Generic Flexibility** – Base `IEntityFactory` for heterogeneous scenarios
- **Procedural Integration** – Seamlessly works with Atomic's procedural patterns
- **Configuration Support** – Allows preconfiguration during entity creation
- **Factory Chaining** – Can be composed with other factory patterns
- **Pool Integration** – Designed to work with entity pooling systems

---

## Interface Definition

### Generic Factory Interface
```csharp
public interface IEntityFactory<out T> where T : IEntity
{
    T Create();
}
```

### Non-Generic Factory Interface
```csharp
public interface IEntityFactory : IEntityFactory<IEntity>
{
}
```

## Core Methods

### Create
```csharp
T Create()
```
- Creates and returns a new entity instance of type `T`
- May optionally preconfigure the entity with default values
- Should return a ready-to-use entity instance
- Implementation-specific configuration is allowed

## Implementation Examples

### Basic Entity Factory
```csharp
public class BasicEntityFactory : IEntityFactory<Entity>
{
    public Entity Create()
    {
        return new Entity();
    }
}
```

### Configured Entity Factory
```csharp
public class PlayerEntityFactory : IEntityFactory<Entity>
{
    private readonly int initialHealth;
    private readonly float initialSpeed;
    
    public PlayerEntityFactory(int health = 100, float speed = 5.0f)
    {
        initialHealth = health;
        initialSpeed = speed;
    }
    
    public Entity Create()
    {
        var entity = new Entity("Player");
        
        // Preconfigure with default values
        entity.AddTag(EntityTags.PLAYER);
        entity.AddValue(EntityNames.HEALTH, initialHealth);
        entity.AddValue(EntityNames.MAX_HEALTH, initialHealth);
        entity.AddValue(EntityNames.SPEED, initialSpeed);
        
        // Add default behaviours
        entity.AddBehaviour(new PlayerInputBehaviour());
        entity.AddBehaviour(new HealthBehaviour());
        
        return entity;
    }
}
```

### Template-Based Factory
```csharp
public class TemplateEntityFactory : IEntityFactory<Entity>
{
    private readonly EntityTemplate template;
    
    public TemplateEntityFactory(EntityTemplate template)
    {
        this.template = template ?? throw new ArgumentNullException(nameof(template));
    }
    
    public Entity Create()
    {
        var entity = new Entity(template.Name);
        
        // Apply template configuration
        foreach (var tag in template.Tags)
        {
            entity.AddTag(tag);
        }
        
        foreach (var kvp in template.Values)
        {
            entity.AddValue(kvp.Key, kvp.Value);
        }
        
        foreach (var behaviourType in template.BehaviourTypes)
        {
            var behaviour = Activator.CreateInstance(behaviourType) as IEntityBehaviour;
            entity.AddBehaviour(behaviour);
        }
        
        return entity;
    }
}
```

### Factory with Dependency Injection
```csharp
public class ServiceEntityFactory : IEntityFactory<Entity>
{
    private readonly IServiceProvider serviceProvider;
    
    public ServiceEntityFactory(IServiceProvider serviceProvider)
    {
        this.serviceProvider = serviceProvider;
    }
    
    public Entity Create()
    {
        var entity = new Entity();
        
        // Inject services into behaviours
        var movementBehaviour = serviceProvider.GetService<MovementBehaviour>();
        var combatBehaviour = serviceProvider.GetService<CombatBehaviour>();
        
        entity.AddBehaviour(movementBehaviour);
        entity.AddBehaviour(combatBehaviour);
        
        return entity;
    }
}
```

## Usage Patterns

### Direct Factory Usage
```csharp
public class GameManager
{
    private readonly IEntityFactory<Entity> playerFactory;
    private readonly IEntityFactory<Entity> enemyFactory;
    
    public GameManager()
    {
        playerFactory = new PlayerEntityFactory(health: 150, speed: 6.0f);
        enemyFactory = new EnemyEntityFactory();
    }
    
    public Entity SpawnPlayer(Vector3 position)
    {
        var player = playerFactory.Create();
        player.SetValue(EntityNames.POSITION, position);
        player.Spawn();
        return player;
    }
    
    public Entity SpawnEnemy(Vector3 position, int enemyType)
    {
        var enemy = enemyFactory.Create();
        enemy.SetValue(EntityNames.POSITION, position);
        enemy.SetValue(EntityNames.ENEMY_TYPE, enemyType);
        enemy.Spawn();
        return enemy;
    }
}
```

### Factory Registry Pattern
```csharp
public class EntityFactoryRegistry
{
    private readonly Dictionary<string, IEntityFactory> factories = new();
    
    public void RegisterFactory(string key, IEntityFactory factory)
    {
        factories[key] = factory;
    }
    
    public IEntity CreateEntity(string factoryKey)
    {
        if (factories.TryGetValue(factoryKey, out var factory))
        {
            return factory.Create();
        }
        
        throw new ArgumentException($"No factory registered for key: {factoryKey}");
    }
}

// Usage
public static class FactorySetup
{
    public static EntityFactoryRegistry SetupFactories()
    {
        var registry = new EntityFactoryRegistry();
        
        registry.RegisterFactory("player", new PlayerEntityFactory());
        registry.RegisterFactory("enemy", new EnemyEntityFactory());
        registry.RegisterFactory("projectile", new ProjectileEntityFactory());
        registry.RegisterFactory("pickup", new PickupEntityFactory());
        
        return registry;
    }
}
```

### Pool Integration
```csharp
public class PooledEntityFactory : IEntityFactory<Entity>
{
    private readonly EntityPool pool;
    private readonly IEntityFactory<Entity> baseFactory;
    
    public PooledEntityFactory(IEntityFactory<Entity> baseFactory)
    {
        this.baseFactory = baseFactory;
        this.pool = new EntityPool(baseFactory);
        
        // Pre-populate pool
        this.pool.Init(10);
    }
    
    public Entity Create()
    {
        // Rent from pool instead of creating new
        return pool.Rent();
    }
    
    public void Return(Entity entity)
    {
        // Reset entity state before returning to pool
        entity.Despawn();
        entity.ClearTags();
        entity.ClearValues();
        
        pool.Return(entity);
    }
}
```

### Procedural Factory Operations
```csharp
public static class EntityFactoryOperations
{
    public static Entity CreateCharacter(IEntityFactory<Entity> factory, string name, int health)
    {
        var entity = factory.Create();
        entity.Name = name;
        
        InitializeCharacter(entity, health);
        return entity;
    }
    
    public static void InitializeCharacter(Entity entity, int health)
    {
        entity.AddTag(EntityTags.CHARACTER);
        entity.AddValue(EntityNames.HEALTH, health);
        entity.AddValue(EntityNames.MAX_HEALTH, health);
        entity.AddValue(EntityNames.ALIVE, true);
    }
    
    public static Entity CreateProjectile(IEntityFactory<Entity> factory, Vector3 direction, float speed)
    {
        var projectile = factory.Create();
        
        projectile.AddTag(EntityTags.PROJECTILE);
        projectile.SetValue(EntityNames.DIRECTION, direction);
        projectile.SetValue(EntityNames.SPEED, speed);
        projectile.SetValue(EntityNames.LIFETIME, 5.0f);
        
        return projectile;
    }
    
    public static void ConfigureForLevel(Entity entity, int level)
    {
        // Scale entity stats based on level
        var baseHealth = entity.GetValue<int>(EntityNames.HEALTH);
        var scaledHealth = baseHealth + (level * 20);
        entity.SetValue(EntityNames.HEALTH, scaledHealth);
        entity.SetValue(EntityNames.MAX_HEALTH, scaledHealth);
        
        entity.SetValue(EntityNames.LEVEL, level);
    }
}
```

### Multi-Type Factory
```csharp
public class MultiEntityFactory : IEntityFactory
{
    private readonly Dictionary<Type, Func<IEntity>> factoryMethods;
    
    public MultiEntityFactory()
    {
        factoryMethods = new Dictionary<Type, Func<IEntity>>();
    }
    
    public void RegisterFactory<T>(Func<T> factory) where T : IEntity
    {
        factoryMethods[typeof(T)] = () => factory();
    }
    
    public T Create<T>() where T : IEntity
    {
        if (factoryMethods.TryGetValue(typeof(T), out var factory))
        {
            return (T)factory();
        }
        
        throw new InvalidOperationException($"No factory registered for type {typeof(T)}");
    }
    
    public IEntity Create()
    {
        // Default to base Entity
        return new Entity();
    }
}
```

### Configuration-Driven Factory
```csharp
[Serializable]
public class EntityFactoryConfig
{
    public string entityName;
    public int[] tags;
    public EntityValueConfig[] values;
    public string[] behaviourTypes;
}

public class ConfigurableEntityFactory : IEntityFactory<Entity>
{
    private readonly EntityFactoryConfig config;
    
    public ConfigurableEntityFactory(EntityFactoryConfig config)
    {
        this.config = config;
    }
    
    public Entity Create()
    {
        var entity = new Entity(config.entityName);
        
        // Apply tags
        foreach (var tag in config.tags)
        {
            entity.AddTag(tag);
        }
        
        // Apply values
        foreach (var valueConfig in config.values)
        {
            entity.SetValue(valueConfig.key, valueConfig.value);
        }
        
        // Add behaviours
        foreach (var behaviourTypeName in config.behaviourTypes)
        {
            var type = Type.GetType(behaviourTypeName);
            if (type != null && typeof(IEntityBehaviour).IsAssignableFrom(type))
            {
                var behaviour = Activator.CreateInstance(type) as IEntityBehaviour;
                entity.AddBehaviour(behaviour);
            }
        }
        
        return entity;
    }
}
```

## Best Practices

1. **Keep Factories Stateless** – Factories should not maintain mutable state
2. **Preconfigure Entities** – Set up default values and behaviours in Create()
3. **Use Generic Versions** – Prefer `IEntityFactory<T>` for type safety
4. **Consider Pooling** – Combine factories with pools for performance
5. **Validate Dependencies** – Check for null dependencies in constructor
6. **Document Configuration** – Clearly document what Create() produces

## Performance Considerations

- **Object Allocation** – Each Create() call allocates a new entity
- **Configuration Overhead** – Complex setup may impact creation time
- **Dependency Resolution** – Service injection can add creation cost
- **Memory Pressure** – Consider pooling for frequently created entities
- **Factory Caching** – Cache expensive factory instances when possible

## Integration with Atomic Systems

### With Entity Collections
```csharp
public static void PopulateCollection(IEntityCollection<Entity> collection, 
                                    IEntityFactory<Entity> factory, 
                                    int count)
{
    for (int i = 0; i < count; i++)
    {
        var entity = factory.Create();
        collection.Add(entity);
    }
}
```

### With Entity Worlds
```csharp
public class WorldFactory
{
    public static EntityWorld CreateWorld(string worldName, 
                                        IEntityFactory<Entity> factory, 
                                        int entityCount)
    {
        var world = new EntityWorld(worldName);
        
        for (int i = 0; i < entityCount; i++)
        {
            var entity = factory.Create();
            world.Add(entity);
        }
        
        return world;
    }
}
```

The `IEntityFactory` interface provides the foundation for flexible, reusable entity creation patterns that integrate seamlessly with Atomic's procedural architecture and performance-oriented design.