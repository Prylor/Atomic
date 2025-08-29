# 🧩 IEntityBehaviour

`IEntityBehaviour` is the base interface for all entity behaviours in the Atomic framework. It serves as a marker interface that identifies classes as behaviours that can be attached to entities.

## Key Features

- **Marker Interface** – Identifies classes as entity behaviours
- **Composition Root** – Base for all behaviour interfaces
- **Type Safety** – Enables type-safe behaviour management
- **Lifecycle Integration** – Behaviours implementing lifecycle interfaces are automatically called

---

## Interface Definition

```csharp
public interface IEntityBehaviour
{
}
```

## Purpose

The `IEntityBehaviour` interface serves several important purposes:

1. **Type Identification** – Marks classes as behaviours for the entity system
2. **Constraint Definition** – Used as a generic constraint in entity methods
3. **Polymorphism** – Allows storing different behaviour types in collections
4. **Extension Point** – Other interfaces extend this to add specific functionality

## Behaviour Lifecycle Interfaces

Behaviours can implement additional interfaces to participate in the entity lifecycle:

| Interface | Called When | Purpose |
|-----------|------------|---------|
| `IEntitySpawn` | Entity spawns | Initialize behaviour state |
| `IEntityDespawn` | Entity despawns | Cleanup behaviour resources |
| `IEntityActivate` | Entity activates | Resume behaviour operation |
| `IEntityDeactivate` | Entity deactivates | Pause behaviour operation |
| `IEntityUpdate` | Every frame | Update behaviour logic |
| `IEntityFixedUpdate` | Fixed timestep | Physics/deterministic updates |
| `IEntityLateUpdate` | After Update | Post-processing updates |
| `IEntityGizmos` | Gizmo drawing | Debug visualization |

## Creating Custom Behaviours

### Basic Behaviour

```csharp
public class SimpleBehaviour : IEntityBehaviour
{
    // Basic behaviour with no lifecycle participation
    public void DoSomething()
    {
        // Custom logic
    }
}
```

### Behaviour with Lifecycle

```csharp
public class HealthBehaviour : IEntityBehaviour, IEntitySpawn, IEntityUpdate
{
    private int maxHealth;
    private int currentHealth;
    
    public HealthBehaviour(int maxHealth)
    {
        this.maxHealth = maxHealth;
    }
    
    public void OnSpawn(IEntity entity)
    {
        // Initialize health when entity spawns
        currentHealth = maxHealth;
        entity.SetValue(EntityNames.HEALTH, currentHealth);
        entity.SetValue(EntityNames.MAX_HEALTH, maxHealth);
    }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Check health status each frame
        if (currentHealth <= 0)
        {
            entity.AddTag(EntityTags.DEAD);
            entity.Despawn();
        }
    }
    
    public void TakeDamage(IEntity entity, int damage)
    {
        currentHealth = Math.Max(0, currentHealth - damage);
        entity.SetValue(EntityNames.HEALTH, currentHealth);
    }
}
```

### Composite Behaviour

```csharp
public class PlayerControllerBehaviour : IEntityBehaviour, 
    IEntityActivate, IEntityDeactivate, IEntityUpdate
{
    private InputSystem input;
    private bool isActive;
    
    public void OnActivate(IEntity entity)
    {
        isActive = true;
        input = new InputSystem();
        input.Enable();
    }
    
    public void OnDeactivate(IEntity entity)
    {
        isActive = false;
        input?.Disable();
    }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        if (!isActive) return;
        
        var moveInput = input.GetMovement();
        if (moveInput != Vector2.zero)
        {
            MoveEntity(entity, moveInput, deltaTime);
        }
        
        if (input.GetJumpPressed())
        {
            Jump(entity);
        }
    }
    
    private void MoveEntity(IEntity entity, Vector2 input, float deltaTime)
    {
        var speed = entity.GetValue<float>(EntityNames.MOVE_SPEED);
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        position += new Vector3(input.x, 0, input.y) * speed * deltaTime;
        entity.SetValue(EntityNames.POSITION, position);
    }
    
    private void Jump(IEntity entity)
    {
        if (entity.HasTag(EntityTags.GROUNDED))
        {
            // Apply jump force
            var jumpForce = entity.GetValue<float>(EntityNames.JUMP_FORCE);
            ApplyForce(entity, Vector3.up * jumpForce);
        }
    }
}
```

## Behaviour Patterns

### State Machine Behaviour

```csharp
public abstract class StateMachineBehaviour : IEntityBehaviour, IEntityUpdate
{
    private Dictionary<int, IState> states = new Dictionary<int, IState>();
    private IState currentState;
    private int currentStateId;
    
    protected void AddState(int id, IState state)
    {
        states[id] = state;
    }
    
    protected void ChangeState(IEntity entity, int newStateId)
    {
        if (states.TryGetValue(newStateId, out var newState))
        {
            currentState?.Exit(entity);
            currentState = newState;
            currentStateId = newStateId;
            currentState.Enter(entity);
        }
    }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        currentState?.Update(entity, deltaTime);
    }
}
```

### Reactive Behaviour

```csharp
public class ReactiveBehaviour : IEntityBehaviour, IEntitySpawn, IEntityDespawn
{
    private Action<IEntity, int> onValueChanged;
    
    public void OnSpawn(IEntity entity)
    {
        // Subscribe to entity events
        onValueChanged = (e, key) => HandleValueChange(e, key);
        entity.OnValueChanged += onValueChanged;
    }
    
    public void OnDespawn(IEntity entity)
    {
        // Unsubscribe from events
        if (onValueChanged != null)
        {
            entity.OnValueChanged -= onValueChanged;
        }
    }
    
    private void HandleValueChange(IEntity entity, int key)
    {
        if (key == EntityNames.HEALTH)
        {
            var health = entity.GetValue<int>(key);
            if (health < 20)
            {
                entity.AddTag(EntityTags.CRITICAL_HEALTH);
            }
        }
    }
}
```

### Data-Driven Behaviour

```csharp
[Serializable]
public class ConfigurableBehaviour : IEntityBehaviour, IEntitySpawn
{
    [SerializeField] private float speed = 5f;
    [SerializeField] private int damage = 10;
    [SerializeField] private bool canFly = false;
    
    public void OnSpawn(IEntity entity)
    {
        // Apply configuration to entity
        entity.SetValue(EntityNames.SPEED, speed);
        entity.SetValue(EntityNames.DAMAGE, damage);
        
        if (canFly)
        {
            entity.AddTag(EntityTags.CAN_FLY);
        }
    }
}
```

## Best Practices

1. **Single Responsibility** – Each behaviour should have one clear purpose
2. **Stateless When Possible** – Prefer storing state in entity values
3. **Interface Segregation** – Only implement needed lifecycle interfaces
4. **Composition** – Build complex behaviours from simple ones
5. **Null Safety** – Check entity state before accessing values
6. **Event Cleanup** – Unsubscribe from events in OnDespawn

## Common Behaviour Categories

- **Input** – Handle player or AI input
- **Movement** – Control entity position and velocity
- **Combat** – Attack, defense, damage handling
- **Health** – Health management and regeneration
- **Inventory** – Item and equipment management
- **AI** – Decision making and pathfinding
- **Animation** – Control visual representations
- **Audio** – Sound effects and music
- **Effects** – Particles, trails, visual effects
- **Physics** – Collision, forces, constraints

## Notes

- Behaviours are the primary way to add functionality to entities
- The empty interface allows maximum flexibility in implementation
- Behaviours can interact with entity state through the IEntity interface
- Multiple behaviours of the same type can be attached to one entity
- Behaviours are processed in the order they were added