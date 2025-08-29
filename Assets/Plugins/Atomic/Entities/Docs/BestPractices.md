# 📚 Best Practices - Atomic.Entities

This guide outlines best practices for using the Atomic.Entities module effectively, following the reactive procedural principles of the Atomic framework.

## 🎯 Core Principles

### 1. Embrace Procedural Design

**✅ DO:** Use static methods and standalone functions for entity operations
```csharp
// Good - Procedural approach
public static class EntityHealth
{
    public static void SetHealth(IEntity entity, int value)
    {
        entity.SetValue(EntityNames.HEALTH, value);
    }
    
    public static void Damage(IEntity entity, int amount)
    {
        var current = entity.GetValue<int>(EntityNames.HEALTH);
        SetHealth(entity, Math.Max(0, current - amount));
    }
}
```

**❌ DON'T:** Embed logic in entity classes
```csharp
// Bad - OOP approach
public class PlayerEntity : Entity
{
    public void TakeDamage(int amount)
    {
        // Logic embedded in entity class
        this.health -= amount;
    }
}
```

### 2. Separate State from Behavior

**✅ DO:** Store state in entity values, logic in behaviours
```csharp
// Good - Clear separation
public class MovementBehaviour : IEntityBehaviour, IEntityUpdate
{
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Get state from entity
        var velocity = entity.GetValue<Vector3>(EntityNames.VELOCITY);
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        
        // Calculate new position
        position += velocity * deltaTime;
        
        // Store state back in entity
        entity.SetValue(EntityNames.POSITION, position);
    }
}
```

**❌ DON'T:** Store state in behaviours
```csharp
// Bad - State in behaviour
public class MovementBehaviour : IEntityBehaviour
{
    private Vector3 position; // Don't store state here!
    private Vector3 velocity; // This breaks the pattern
}
```

### 3. Use Composition Over Inheritance

**✅ DO:** Compose entities from behaviours
```csharp
// Good - Composition
public static IEntity CreateEnemy(string type)
{
    var entity = new Entity(type);
    
    // Compose from behaviours
    entity.AddBehaviour(new HealthBehaviour());
    entity.AddBehaviour(new MovementBehaviour());
    
    if (type == "flying")
        entity.AddBehaviour(new FlyingBehaviour());
    
    return entity;
}
```

**❌ DON'T:** Create deep inheritance hierarchies
```csharp
// Bad - Inheritance hierarchy
public class Enemy : Entity { }
public class FlyingEnemy : Enemy { }
public class BossEnemy : FlyingEnemy { }
```

## 🏗️ Entity Construction

### Use Factory Pattern

```csharp
public static class EntityFactory
{
    private static readonly Dictionary<string, Func<IEntity>> templates = new();
    
    public static void RegisterTemplate(string type, Func<IEntity> creator)
    {
        templates[type] = creator;
    }
    
    public static IEntity Create(string type)
    {
        if (templates.TryGetValue(type, out var creator))
        {
            return creator();
        }
        
        return new Entity(type);
    }
}

// Register templates
EntityFactory.RegisterTemplate("player", () =>
{
    var entity = new Entity("Player");
    entity.AddValue(EntityNames.HEALTH, 100);
    entity.AddValue(EntityNames.SPEED, 5f);
    entity.AddBehaviour(new PlayerInputBehaviour());
    return entity;
});
```

### Pre-allocate Collections

```csharp
// Good - Pre-allocate for known sizes
var entity = new Entity(
    name: "Boss",
    tagCapacity: 10,      // Know we'll have ~10 tags
    valueCapacity: 20,    // Know we'll have ~20 values
    behaviourCapacity: 5  // Know we'll have ~5 behaviours
);
```

## 🔄 Lifecycle Management

### Proper Initialization Order

```csharp
public static void InitializeEntity(IEntity entity)
{
    // 1. Add values first (data)
    entity.AddValue(EntityNames.HEALTH, 100);
    entity.AddValue(EntityNames.POSITION, Vector3.zero);
    
    // 2. Add tags (categorization)
    entity.AddTag(EntityTags.ALIVE);
    entity.AddTag(EntityTags.PLAYER);
    
    // 3. Add behaviours last (they may depend on values/tags)
    entity.AddBehaviour(new HealthBehaviour());
    entity.AddBehaviour(new MovementBehaviour());
    
    // 4. Spawn when ready
    entity.Spawn();
}
```

### Clean Disposal

```csharp
public static void DisposeEntity(IEntity entity)
{
    // 1. Despawn first
    if (entity.Spawned)
        entity.Despawn();
    
    // 2. Clear behaviours
    entity.ClearBehaviours();
    
    // 3. Clear values
    entity.ClearValues();
    
    // 4. Clear tags
    entity.ClearTags();
    
    // 5. Dispose
    if (entity is IDisposable disposable)
        disposable.Dispose();
}
```

## 💾 State Management

### Use Integer Keys for Performance

```csharp
// Good - Integer constants
public static class EntityNames
{
    public const int HEALTH = 1;
    public const int POSITION = 2;
    public const int VELOCITY = 3;
}

// Usage
entity.SetValue(EntityNames.HEALTH, 100);
```

### Avoid String Keys

```csharp
// Bad - String keys are slower
entity.SetValue("health", 100); // Avoid this
```

### Use TryGet Pattern

```csharp
// Good - Safe access
if (entity.TryGetValue<int>(EntityNames.HEALTH, out int health))
{
    // Use health value
}

// Bad - Can throw exceptions
int health = entity.GetValue<int>(EntityNames.HEALTH); // May throw
```

## 🎭 Behaviour Design

### Single Responsibility

```csharp
// Good - Each behaviour has one job
public class HealthBehaviour : IEntityBehaviour { /* health logic */ }
public class MovementBehaviour : IEntityBehaviour { /* movement logic */ }
public class AttackBehaviour : IEntityBehaviour { /* attack logic */ }

// Bad - God behaviour
public class PlayerBehaviour : IEntityBehaviour 
{
    // Handles health, movement, attack, inventory, etc.
}
```

### Stateless Behaviours

```csharp
// Good - Stateless behaviour
public class RegenerationBehaviour : IEntityBehaviour, IEntityUpdate
{
    private readonly float regenRate;
    
    public RegenerationBehaviour(float rate)
    {
        regenRate = rate; // Configuration only
    }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Get state from entity
        var health = entity.GetValue<int>(EntityNames.HEALTH);
        var maxHealth = entity.GetValue<int>(EntityNames.MAX_HEALTH);
        
        // Modify and store back
        if (health < maxHealth)
        {
            entity.SetValue(EntityNames.HEALTH, 
                Math.Min(health + (int)(regenRate * deltaTime), maxHealth));
        }
    }
}
```

## 🔍 Filtering and Queries

### Cache Filters

```csharp
public class CombatSystem
{
    private EntityFilter aliveEnemies;
    
    public void Initialize(IEntityCollection<IEntity> entities)
    {
        // Create filter once
        aliveEnemies = new EntityFilter(
            entities,
            e => e.HasTag(EntityTags.ENEMY) && e.HasTag(EntityTags.ALIVE)
        );
    }
    
    public void Update()
    {
        // Reuse filter
        foreach (var enemy in aliveEnemies)
        {
            ProcessEnemy(enemy);
        }
    }
}
```

### Use Appropriate Triggers

```csharp
// Good - Specific triggers
var lowHealthFilter = new EntityFilter(
    entities,
    e => e.GetValue<int>(EntityNames.HEALTH) < 30,
    new ValueEntityTrigger(EntityNames.HEALTH) // Only re-evaluate on health change
);

// Bad - No triggers when needed
var lowHealthFilter = new EntityFilter(
    entities,
    e => e.GetValue<int>(EntityNames.HEALTH) < 30
    // Missing trigger - won't update when health changes!
);
```

## 🏊 Pooling Strategy

### Always Pool Frequently Created Entities

```csharp
public static class ProjectileManager
{
    private static readonly EntityPool projectilePool = new EntityPool(
        capacity: 100,
        factory: () => CreateProjectile()
    );
    
    public static IEntity SpawnProjectile(Vector3 position)
    {
        var projectile = projectilePool.Get();
        projectile.SetValue(EntityNames.POSITION, position);
        projectile.Spawn();
        return projectile;
    }
    
    public static void DespawnProjectile(IEntity projectile)
    {
        projectile.Despawn();
        projectilePool.Return(projectile);
    }
}
```

## 🌍 World Organization

### Use Worlds for Logical Grouping

```csharp
public class GameManager
{
    private EntityWorld gameWorld;      // Gameplay entities
    private EntityWorld uiWorld;        // UI entities
    private EntityWorld effectsWorld;   // Visual effects
    
    public void Initialize()
    {
        gameWorld = new EntityWorld("Game");
        uiWorld = new EntityWorld("UI");
        effectsWorld = new EntityWorld("Effects");
    }
    
    public void Update(float deltaTime)
    {
        // Update in order
        gameWorld.Update(deltaTime);
        effectsWorld.Update(deltaTime);
        uiWorld.Update(deltaTime);
    }
}
```

## ⚡ Performance Tips

### 1. Minimize Boxing
```csharp
// Good - Use generic methods
entity.SetValue<int>(EntityNames.HEALTH, 100);

// Bad - Boxing occurs
entity.SetValue(EntityNames.HEALTH, (object)100);
```

### 2. Batch Operations
```csharp
// Good - Batch changes
entity.OnStateChanged -= handler; // Temporarily unsubscribe
entity.SetValue(EntityNames.HEALTH, 100);
entity.SetValue(EntityNames.MANA, 50);
entity.AddTag(EntityTags.BUFFED);
entity.OnStateChanged += handler; // Resubscribe
entity.OnStateChanged?.Invoke();  // Single notification
```

### 3. Use Unsafe Access for Structs
```csharp
// Good - Zero allocation for structs
ref Vector3 position = ref entity.GetValueUnsafe<Vector3>(EntityNames.POSITION);
position.x += 10; // Direct modification
```

## 🐛 Common Pitfalls

### 1. Memory Leaks
```csharp
// Bad - Forgot to unsubscribe
entity.OnValueChanged += HandleValueChange;
// Entity disposed but handler still referenced

// Good - Proper cleanup
public void Dispose()
{
    entity.OnValueChanged -= HandleValueChange;
}
```

### 2. Null Reference Exceptions
```csharp
// Bad - No null check
var behaviour = entity.GetBehaviour<HealthBehaviour>();
behaviour.Heal(10); // May be null!

// Good - Safe access
if (entity.TryGetBehaviour<HealthBehaviour>(out var behaviour))
{
    behaviour.Heal(10);
}
```

### 3. Circular Dependencies
```csharp
// Bad - Behaviours depend on each other
public class BehaviourA : IEntityBehaviour
{
    void Update(IEntity entity)
    {
        var b = entity.GetBehaviour<BehaviourB>();
        b.DoSomething(); // A depends on B
    }
}

// Good - Use entity values for communication
public class BehaviourA : IEntityBehaviour
{
    void Update(IEntity entity)
    {
        entity.SetValue(EntityNames.SIGNAL, true);
        // B reacts to value change
    }
}
```

## 📋 Checklist

Before deploying your entity system:

- [ ] All entities are properly disposed when no longer needed
- [ ] Event subscriptions are cleaned up
- [ ] Pools are used for frequently created entities
- [ ] Filters are cached and reused
- [ ] Collections are pre-allocated where possible
- [ ] State and behavior are properly separated
- [ ] Procedural patterns are followed
- [ ] Integer keys are used for values and tags
- [ ] Error handling uses Try patterns
- [ ] No circular dependencies between behaviours