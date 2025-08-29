# 🧩 IEntity_Behaviours

`IEntity_Behaviours` is a partial interface of `IEntity` that provides behaviour management functionality. Behaviours are modular units of logic that can be dynamically attached to entities, implementing the composition pattern for flexible entity functionality.

## Key Features

- **Modular Logic** – Attach/detach behaviours at runtime
- **Composition Pattern** – Build complex entities from simple behaviours
- **Type-Safe Access** – Generic methods for retrieving specific behaviour types
- **Lifecycle Integration** – Behaviours automatically participate in entity lifecycle
- **Event Notifications** – Track behaviour additions and removals

---

## Events

```csharp
event Action<IEntity, IEntityBehaviour> OnBehaviourAdded;
event Action<IEntity, IEntityBehaviour> OnBehaviourDeleted;
```

### OnBehaviourAdded
- **Triggered**: When a behaviour is attached to the entity
- **Parameters**: The entity and the added behaviour

### OnBehaviourDeleted
- **Triggered**: When a behaviour is removed from the entity
- **Parameters**: The entity and the removed behaviour

## Properties

```csharp
int BehaviourCount { get; }
```
- **Description**: Returns the number of behaviours currently attached to the entity
- **Access**: Read-only

## Methods

### Adding Behaviours

```csharp
void AddBehaviour(IEntityBehaviour behaviour)
```
- **Description**: Attaches a behaviour to the entity
- **Events**: Triggers `OnBehaviourAdded`
- **Lifecycle**: If entity is spawned, behaviour's spawn methods are called

### Getting Behaviours

```csharp
T GetBehaviour<T>() where T : IEntityBehaviour
```
- **Description**: Gets the first behaviour of the specified type
- **Returns**: The behaviour instance or null if not found
- **Performance**: O(n) where n is number of behaviours

```csharp
bool TryGetBehaviour<T>(out T behaviour) where T : IEntityBehaviour
```
- **Description**: Attempts to get a behaviour of the specified type
- **Returns**: `true` if behaviour found, `false` otherwise

```csharp
T[] GetBehaviours<T>() where T : IEntityBehaviour
```
- **Description**: Gets all behaviours of the specified type
- **Returns**: Array of matching behaviours (empty if none found)

```csharp
IEntityBehaviour[] GetBehaviours()
```
- **Description**: Gets all behaviours attached to the entity
- **Returns**: Array of all behaviours

### Checking Behaviours

```csharp
bool HasBehaviour(IEntityBehaviour behaviour)
```
- **Description**: Checks if specific behaviour instance is attached
- **Returns**: `true` if behaviour is attached

```csharp
bool HasBehaviour<T>() where T : IEntityBehaviour
```
- **Description**: Checks if any behaviour of type T is attached
- **Returns**: `true` if at least one behaviour of type T exists

### Removing Behaviours

```csharp
bool DelBehaviour(IEntityBehaviour behaviour)
```
- **Description**: Removes specific behaviour instance
- **Returns**: `true` if behaviour was removed
- **Events**: Triggers `OnBehaviourDeleted` if successful

```csharp
bool DelBehaviour<T>() where T : IEntityBehaviour
```
- **Description**: Removes first behaviour of type T
- **Returns**: `true` if behaviour was removed

```csharp
void DelBehaviours<T>() where T : IEntityBehaviour
```
- **Description**: Removes all behaviours of type T
- **Events**: Triggers `OnBehaviourDeleted` for each removed behaviour

```csharp
void ClearBehaviours()
```
- **Description**: Removes all behaviours from the entity
- **Events**: Triggers `OnBehaviourDeleted` for each behaviour

### Bulk Operations

```csharp
int CopyBehaviours(IEntityBehaviour[] results)
```
- **Description**: Copies behaviours into provided array
- **Returns**: Number of behaviours copied

```csharp
int CopyBehaviours<T>(T[] results) where T : IEntityBehaviour
```
- **Description**: Copies behaviours of type T into provided array
- **Returns**: Number of behaviours copied

```csharp
IEnumerator<IEntityBehaviour> GetBehaviourEnumerator()
```
- **Description**: Returns enumerator for iterating over behaviours

## Example Usage

### Basic Behaviour Management

```csharp
// Create and add behaviours
var movementBehaviour = new MovementBehaviour();
var healthBehaviour = new HealthBehaviour(maxHealth: 100);
var attackBehaviour = new AttackBehaviour(damage: 10);

entity.AddBehaviour(movementBehaviour);
entity.AddBehaviour(healthBehaviour);
entity.AddBehaviour(attackBehaviour);

// Get specific behaviour
var health = entity.GetBehaviour<HealthBehaviour>();
if (health != null)
{
    health.Heal(20);
}

// Check for behaviour
if (entity.HasBehaviour<AttackBehaviour>())
{
    // Entity can attack
    PerformAttack(entity);
}

// Remove behaviour
entity.DelBehaviour<AttackBehaviour>();
```

### Creating Custom Behaviours

```csharp
public class RegenerationBehaviour : IEntityBehaviour, IEntityUpdate
{
    private float regenRate;
    private float regenTimer;
    
    public RegenerationBehaviour(float healthPerSecond)
    {
        this.regenRate = healthPerSecond;
    }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        regenTimer += deltaTime;
        if (regenTimer >= 1f)
        {
            var health = entity.GetValue<int>(EntityNames.HEALTH);
            var maxHealth = entity.GetValue<int>(EntityNames.MAX_HEALTH);
            
            if (health < maxHealth)
            {
                entity.SetValue(EntityNames.HEALTH, 
                    Math.Min(health + (int)regenRate, maxHealth));
            }
            
            regenTimer = 0f;
        }
    }
}

// Usage
entity.AddBehaviour(new RegenerationBehaviour(5f)); // 5 HP per second
```

### Behaviour Composition

```csharp
public static class EntityBuilder
{
    public static IEntity CreatePlayer(string name)
    {
        var player = new Entity(name);
        
        // Add core behaviours
        player.AddBehaviour(new PlayerInputBehaviour());
        player.AddBehaviour(new MovementBehaviour(speed: 5f));
        player.AddBehaviour(new HealthBehaviour(maxHealth: 100));
        player.AddBehaviour(new InventoryBehaviour());
        
        // Add abilities
        player.AddBehaviour(new JumpBehaviour(jumpHeight: 2f));
        player.AddBehaviour(new AttackBehaviour(damage: 10));
        
        return player;
    }
    
    public static IEntity CreateEnemy(string name, int difficulty)
    {
        var enemy = new Entity(name);
        
        // Base behaviours
        enemy.AddBehaviour(new AIBehaviour(difficulty));
        enemy.AddBehaviour(new MovementBehaviour(speed: 3f));
        enemy.AddBehaviour(new HealthBehaviour(maxHealth: 50 * difficulty));
        
        // Conditional behaviours based on difficulty
        if (difficulty > 1)
        {
            enemy.AddBehaviour(new AttackBehaviour(damage: 5 * difficulty));
        }
        
        if (difficulty > 2)
        {
            enemy.AddBehaviour(new ShieldBehaviour());
        }
        
        return enemy;
    }
}
```

### Dynamic Behaviour Modification

```csharp
public static class PowerUpSystem
{
    public static void ApplySpeedBoost(IEntity entity, float duration)
    {
        // Add temporary behaviour
        var boost = new SpeedBoostBehaviour(multiplier: 2f);
        entity.AddBehaviour(boost);
        
        // Schedule removal
        ScheduleBehaviourRemoval(entity, boost, duration);
    }
    
    public static void UpgradeEntity(IEntity entity)
    {
        // Replace basic behaviour with advanced version
        if (entity.HasBehaviour<BasicAttackBehaviour>())
        {
            entity.DelBehaviour<BasicAttackBehaviour>();
            entity.AddBehaviour(new AdvancedAttackBehaviour());
        }
        
        // Add new capabilities
        if (!entity.HasBehaviour<DashBehaviour>())
        {
            entity.AddBehaviour(new DashBehaviour());
        }
    }
}
```

### Behaviour Events

```csharp
// Track behaviour changes
entity.OnBehaviourAdded += (e, behaviour) =>
{
    Debug.Log($"Behaviour added: {behaviour.GetType().Name}");
    
    // Special handling for specific behaviours
    if (behaviour is IWeaponBehaviour weapon)
    {
        UpdateWeaponUI(weapon);
    }
};

entity.OnBehaviourDeleted += (e, behaviour) =>
{
    Debug.Log($"Behaviour removed: {behaviour.GetType().Name}");
    
    // Cleanup
    if (behaviour is IDisposable disposable)
    {
        disposable.Dispose();
    }
};
```

### Procedural Behaviour Management

Following Atomic's procedural approach:

```csharp
public static class BehaviourUtils
{
    public static void ReplaceBehaviour<TOld, TNew>(IEntity entity, TNew newBehaviour) 
        where TOld : IEntityBehaviour
        where TNew : IEntityBehaviour
    {
        entity.DelBehaviour<TOld>();
        entity.AddBehaviour(newBehaviour);
    }
    
    public static void ToggleBehaviour<T>(IEntity entity) where T : IEntityBehaviour, new()
    {
        if (entity.HasBehaviour<T>())
        {
            entity.DelBehaviour<T>();
        }
        else
        {
            entity.AddBehaviour(new T());
        }
    }
    
    public static List<T> GetBehavioursOfType<T>(IEnumerable<IEntity> entities) 
        where T : IEntityBehaviour
    {
        var behaviours = new List<T>();
        foreach (var entity in entities)
        {
            behaviours.AddRange(entity.GetBehaviours<T>());
        }
        return behaviours;
    }
}
```

## Best Practices

1. **Single Responsibility** – Each behaviour should have one clear purpose
2. **Composition Over Inheritance** – Combine simple behaviours for complex functionality
3. **Interface Segregation** – Behaviours implement only needed lifecycle interfaces
4. **Avoid Inter-Dependencies** – Behaviours should be independent when possible
5. **Use Type Checking** – Check for behaviour existence before accessing
6. **Clean Disposal** – Implement IDisposable for behaviours with resources

## Performance Notes

- **List Storage** – Behaviours stored in a list, O(n) for type searches
- **Type Filtering** – Generic methods iterate through all behaviours
- **Lifecycle Overhead** – Each behaviour's lifecycle methods are called
- **Memory Allocation** – GetBehaviours() allocates arrays; use CopyBehaviours() to reuse

## Common Behaviour Types

- **Input Handling** – PlayerInput, AIInput
- **Movement** – Movement, Flying, Swimming
- **Combat** – Attack, Defense, Special Abilities
- **Health System** – Health, Regeneration, Damage
- **Inventory** – Item Management, Equipment
- **AI** – Pathfinding, Decision Making, State Machines
- **Effects** – Buffs, Debuffs, Status Effects
- **Physics** – Collision, Gravity, Forces