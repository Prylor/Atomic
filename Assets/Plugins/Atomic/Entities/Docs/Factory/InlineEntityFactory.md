# 🧩 InlineEntityFactory

A lightweight factory implementation that wraps entity creation delegates for simple, functional-style entity instantiation. Provides immediate factory creation without requiring separate factory classes.

## Overview

`InlineEntityFactory` enables rapid prototyping and dynamic entity creation by wrapping creation functions in a factory interface. Perfect for scenarios where entity creation logic is simple or needs to be defined inline without additional infrastructure.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic inline factory for IEntity creation.
    /// </summary>
    public class InlineEntityFactory : InlineEntityFactory<IEntity>, IEntityFactory
    {
        public InlineEntityFactory(Func<IEntity> creator) : base(creator) { }
    }

    /// <summary>
    /// Generic inline factory wrapping entity creation delegates.
    /// </summary>
    /// <typeparam name="T">The type of IEntity to produce</typeparam>
    public class InlineEntityFactory<T> : IEntityFactory<T> where T : IEntity
    {
        private readonly Func<T> _creator;

        public InlineEntityFactory(Func<T> creator) =>
            _creator = creator ?? throw new ArgumentNullException(nameof(creator));

        public T Create() => _creator.Invoke();
    }
}
```

## Key Features

### Delegate-Based Creation
- Wraps any `Func<T>` delegate as an entity factory
- Supports lambda expressions, method references, and anonymous functions
- No inheritance or complex factory setup required

### Minimal Overhead
- Lightweight wrapper with single delegate field
- Direct delegate invocation for optimal performance
- Null-safe constructor with validation

### Generic Type Support
- Generic version supports specific entity types
- Non-generic version provides convenience for `IEntity`
- Maintains type safety while enabling flexible creation patterns

## Usage Examples

### Basic Inline Factory Creation

```csharp
public class SimpleEntityCreation : MonoBehaviour
{
    void Start()
    {
        // Create inline factory with lambda expression
        var playerFactory = new InlineEntityFactory(() => CreatePlayerEntity());
        
        // Create inline factory with method reference
        var enemyFactory = new InlineEntityFactory(CreateEnemyEntity);
        
        // Create inline factory with anonymous function
        var itemFactory = new InlineEntityFactory(() => 
        {
            var item = new Entity("Item", 2, 3, 1);
            item.AddTag("Collectible");
            item.Set("Value", Random.Range(10, 100));
            return item;
        });
        
        // Use factories to create entities
        IEntity player = playerFactory.Create();
        IEntity enemy = enemyFactory.Create();
        IEntity item = itemFactory.Create();
    }
    
    private IEntity CreatePlayerEntity()
    {
        var player = new Entity("Player", 8, 15, 5);
        player.Set("Health", 100f);
        player.Set("MaxHealth", 100f);
        player.Set("Speed", 10f);
        player.AddTag("Player");
        player.AddTag("Controllable");
        return player;
    }
    
    private IEntity CreateEnemyEntity()
    {
        var enemy = new Entity("Enemy", 4, 8, 3);
        enemy.Set("Health", 50f);
        enemy.Set("Speed", 5f);
        enemy.Set("Damage", 10f);
        enemy.AddTag("Enemy");
        enemy.AddTag("Hostile");
        return enemy;
    }
}
```

### Procedural Entity Factories

```csharp
public class ProceduralEntitySystem : MonoBehaviour
{
    [System.Serializable]
    public struct WeaponTemplate
    {
        public string name;
        public float damage;
        public float range;
        public float fireRate;
    }
    
    [SerializeField] private WeaponTemplate[] _weaponTemplates;
    
    void Start()
    {
        // Create inline factories for each weapon template
        var weaponFactories = _weaponTemplates.Select(template =>
            new InlineEntityFactory(() => CreateWeapon(template))
        ).ToArray();
        
        // Create random weapons
        foreach (var factory in weaponFactories)
        {
            IEntity weapon = factory.Create();
            Debug.Log($"Created weapon: {weapon.Get<string>("Name")}");
        }
    }
    
    private IEntity CreateWeapon(WeaponTemplate template)
    {
        var weapon = new Entity($"Weapon_{template.name}", 3, 6, 2);
        weapon.Set("Name", template.name);
        weapon.Set("Damage", template.damage);
        weapon.Set("Range", template.range);
        weapon.Set("FireRate", template.fireRate);
        weapon.Set("Durability", Random.Range(50f, 100f));
        weapon.AddTag("Weapon");
        weapon.AddTag(template.name);
        return weapon;
    }
}
```

### Factory Registry with Inline Factories

```csharp
public class InlineFactoryRegistry : MonoBehaviour
{
    private readonly Dictionary<string, IEntityFactory> _factories = new();
    
    void Start()
    {
        RegisterInlineFactories();
    }
    
    private void RegisterInlineFactories()
    {
        // Register various entity types with inline factories
        _factories["FastEnemy"] = new InlineEntityFactory(() =>
        {
            var entity = new Entity("FastEnemy", 3, 5, 2);
            entity.Set("Health", 30f);
            entity.Set("Speed", 15f);
            entity.AddTag("Enemy");
            entity.AddTag("Fast");
            return entity;
        });
        
        _factories["TankEnemy"] = new InlineEntityFactory(() =>
        {
            var entity = new Entity("TankEnemy", 3, 5, 2);
            entity.Set("Health", 150f);
            entity.Set("Speed", 3f);
            entity.Set("Armor", 50f);
            entity.AddTag("Enemy");
            entity.AddTag("Tank");
            return entity;
        });
        
        _factories["HealthPotion"] = new InlineEntityFactory(() =>
        {
            var entity = new Entity("HealthPotion", 2, 3, 1);
            entity.Set("HealAmount", Random.Range(20f, 50f));
            entity.AddTag("Item");
            entity.AddTag("Consumable");
            return entity;
        });
    }
    
    public IEntity CreateEntity(string type)
    {
        return _factories.TryGetValue(type, out var factory) 
            ? factory.Create() 
            : null;
    }
    
    public void AddFactory(string key, Func<IEntity> creator)
    {
        _factories[key] = new InlineEntityFactory(creator);
    }
}
```

## Integration with Atomic Framework

### Reactive Entity Creation

```csharp
public class ReactiveEntityFactory : MonoBehaviour
{
    private readonly ReactiveValue<string> _selectedEntityType = new();
    private readonly Dictionary<string, IEntityFactory> _factories = new();
    
    void Start()
    {
        SetupFactories();
        
        // React to entity type changes
        _selectedEntityType.Subscribe(OnEntityTypeChanged);
    }
    
    private void SetupFactories()
    {
        // Setup inline factories with reactive values
        _factories["Player"] = new InlineEntityFactory(() =>
        {
            var player = new Entity("Player", 5, 8, 3);
            player.Set("CreatedAt", Time.time);
            player.Set("SessionId", System.Guid.NewGuid().ToString());
            return player;
        });
        
        _factories["NPC"] = new InlineEntityFactory(() =>
        {
            var npc = new Entity("NPC", 3, 6, 2);
            npc.Set("DialogueId", Random.Range(1, 100));
            npc.Set("Mood", Random.Range(0f, 1f));
            return npc;
        });
    }
    
    private void OnEntityTypeChanged(string entityType)
    {
        if (_factories.TryGetValue(entityType, out var factory))
        {
            IEntity entity = factory.Create();
            entity.AddTag("AutoCreated");
            Debug.Log($"Auto-created {entityType} entity");
        }
    }
    
    public void SelectEntityType(string type)
    {
        _selectedEntityType.Value = type;
    }
}
```

### Event-Driven Factory Creation

```csharp
public class EventDrivenFactories : MonoBehaviour
{
    private readonly Dictionary<GameEvent, IEntityFactory> _eventFactories = new();
    
    void Start()
    {
        RegisterEventFactories();
        
        // Subscribe to game events
        GameEvents.OnPlayerLevelUp += () => TriggerFactory(GameEvent.PlayerLevelUp);
        GameEvents.OnEnemyDefeated += () => TriggerFactory(GameEvent.EnemyDefeated);
        GameEvents.OnBossEncounter += () => TriggerFactory(GameEvent.BossEncounter);
    }
    
    private void RegisterEventFactories()
    {
        // Level up rewards
        _eventFactories[GameEvent.PlayerLevelUp] = new InlineEntityFactory(() =>
        {
            var reward = new Entity("LevelUpReward", 2, 4, 1);
            reward.Set("ExperienceBonus", 100);
            reward.Set("SkillPoints", 3);
            reward.AddTag("Reward");
            return reward;
        });
        
        // Enemy loot
        _eventFactories[GameEvent.EnemyDefeated] = new InlineEntityFactory(() =>
        {
            var loot = new Entity("EnemyLoot", 3, 5, 1);
            loot.Set("Gold", Random.Range(5, 20));
            loot.Set("Experience", Random.Range(10, 25));
            loot.AddTag("Loot");
            return loot;
        });
        
        // Boss encounter effects
        _eventFactories[GameEvent.BossEncounter] = new InlineEntityFactory(() =>
        {
            var effect = new Entity("BossAura", 4, 6, 2);
            effect.Set("IntensityMultiplier", 2.0f);
            effect.Set("Duration", 30f);
            effect.AddTag("Effect");
            effect.AddTag("Boss");
            return effect;
        });
    }
    
    private void TriggerFactory(GameEvent gameEvent)
    {
        if (_eventFactories.TryGetValue(gameEvent, out var factory))
        {
            IEntity entity = factory.Create();
            entity.Set("TriggeredBy", gameEvent.ToString());
            entity.Set("TriggeredAt", Time.time);
        }
    }
}

public enum GameEvent
{
    PlayerLevelUp,
    EnemyDefeated,
    BossEncounter
}
```

## Implementation Notes

### Delegate Handling
- Stores creation function as readonly field
- Validates delegate is not null during construction
- Direct invocation provides minimal call overhead

### Memory Considerations
- Each factory instance captures its creation delegate
- Be aware of closure capture in lambda expressions
- Consider delegate caching for frequently used patterns

### Exception Handling
- Constructor throws `ArgumentNullException` for null delegates
- Creation exceptions bubble up from wrapped delegate
- No additional error handling or validation provided

## Best Practices

### Delegate Design
- Keep creation logic simple and focused
- Avoid complex operations in creation delegates
- Consider factory methods for complex initialization

### Closure Awareness
- Be mindful of variable capture in lambda expressions
- Avoid capturing large objects unnecessarily
- Consider delegate caching for performance-critical scenarios

### Error Handling
- Validate inputs before creating inline factories
- Handle potential exceptions from creation delegates
- Provide meaningful error messages for debugging

## Common Patterns

### Factory Method Pattern

```csharp
public static class EntityFactories
{
    public static InlineEntityFactory CreatePlayerFactory(PlayerConfig config)
    {
        return new InlineEntityFactory(() =>
        {
            var player = new Entity("Player", 8, 12, 4);
            player.Set("Health", config.maxHealth);
            player.Set("Speed", config.speed);
            player.Set("Level", config.startingLevel);
            return player;
        });
    }
    
    public static InlineEntityFactory CreateEnemyFactory(EnemyTemplate template)
    {
        return new InlineEntityFactory(() =>
        {
            var enemy = new Entity(template.name, 4, 8, 3);
            enemy.Set("Health", template.health);
            enemy.Set("Damage", template.damage);
            enemy.AddTag("Enemy");
            return enemy;
        });
    }
}
```

### Composition Pattern

```csharp
public class CompositeEntityFactory
{
    private readonly List<IEntityFactory> _componentFactories = new();
    
    public CompositeEntityFactory AddComponent(Func<IEntity> componentCreator)
    {
        _componentFactories.Add(new InlineEntityFactory(componentCreator));
        return this;
    }
    
    public IEntity CreateComposite()
    {
        var composite = new Entity("Composite", 10, 20, 5);
        
        foreach (var factory in _componentFactories)
        {
            var component = factory.Create();
            // Merge component data into composite
        }
        
        return composite;
    }
}
```

The `InlineEntityFactory` provides a lightweight, flexible solution for wrapping entity creation logic in factory interfaces, enabling rapid prototyping and dynamic factory creation within the Atomic framework's procedural architecture.