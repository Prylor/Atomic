# 🧩 IEntityUpdate

The `IEntityUpdate` interface defines behaviours that execute during the main update cycle of an entity. This interface is called automatically every frame during the game loop, making it perfect for real-time logic, input processing, animation updates, and frame-dependent calculations.

## Key Features

- **Automatic Invocation** – Called automatically during `IEntity.OnUpdate()` every frame
- **Real-Time Processing** – Perfect for frame-dependent logic and real-time interactions
- **Delta Time Support** – Receives delta time for frame-rate independent calculations
- **Generic Support** – Includes strongly-typed generic version for specific entity types
- **Performance Critical** – Optimized for frequent execution without frame drops
- **Reactive Design** – Operates on entity state rather than storing internal data

---

## Interface Definition

```csharp
public interface IEntityUpdate : IEntityBehaviour
{
    void OnUpdate(IEntity entity, float deltaTime);
}

public interface IEntityUpdate<in T> : IEntityUpdate where T : IEntity
{
    void OnUpdate(T entity, float deltaTime);
}
```

## When It's Called

The `OnUpdate` method is automatically invoked:
- Every frame during the main game loop
- When `IEntity.OnUpdate(float deltaTime)` is called
- After `OnActivate` and before `OnLateUpdate` in the frame cycle
- Only for active and spawned entities
- With delta time representing elapsed time since last frame

The update frequency depends on the frame rate and is typically 60+ times per second.

## Example Implementations

### Movement Controller Behaviour

```csharp
public class MovementBehaviour : IEntityUpdate
{
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Get current movement state
        var velocity = entity.GetValue<Vector3>(EntityNames.VELOCITY);
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        var maxSpeed = entity.GetValue<float>(EntityNames.MAX_SPEED);
        
        // Apply movement
        position += velocity * deltaTime;
        
        // Clamp to max speed
        if (velocity.magnitude > maxSpeed)
        {
            velocity = velocity.normalized * maxSpeed;
            entity.SetValue(EntityNames.VELOCITY, velocity);
        }
        
        // Update position
        entity.SetValue(EntityNames.POSITION, position);
        
        // Update Unity Transform if present
        if (entity.TryGetValue<Transform>(EntityNames.TRANSFORM, out var transform))
        {
            transform.position = position;
        }
    }
}
```

### Health Regeneration Behaviour

```csharp
public class HealthRegenerationBehaviour : IEntityUpdate
{
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Only regenerate if alive and not at max health
        if (entity.HasTag(EntityTags.DEAD)) return;
        
        var currentHealth = entity.GetValue<float>(EntityNames.CURRENT_HEALTH);
        var maxHealth = entity.GetValue<float>(EntityNames.MAX_HEALTH);
        var regenRate = entity.GetValue<float>(EntityNames.HEALTH_REGEN_RATE);
        
        if (currentHealth < maxHealth && regenRate > 0)
        {
            // Calculate regeneration this frame
            var regenAmount = regenRate * deltaTime;
            var newHealth = Mathf.Min(currentHealth + regenAmount, maxHealth);
            
            entity.SetValue(EntityNames.CURRENT_HEALTH, newHealth);
            
            // Trigger health change event
            if (newHealth != currentHealth)
            {
                GameEvents.OnHealthChanged?.Invoke(entity, newHealth);
            }
        }
    }
}
```

### Input Processing Behaviour

```csharp
public class InputProcessorBehaviour : IEntityUpdate
{
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Only process input for controllable entities
        if (!entity.HasTag(EntityTags.CONTROLLABLE)) return;
        if (!entity.GetValue<bool>(EntityNames.INPUT_ENABLED)) return;
        
        // Get input values
        var moveInput = Input.GetAxis("Horizontal");
        var jumpInput = Input.GetButtonDown("Jump");
        var attackInput = Input.GetButtonDown("Fire1");
        
        // Process movement input
        if (Mathf.Abs(moveInput) > 0.1f)
        {
            var moveSpeed = entity.GetValue<float>(EntityNames.MOVE_SPEED);
            var velocity = new Vector3(moveInput * moveSpeed, 0, 0);
            entity.SetValue(EntityNames.VELOCITY, velocity);
            entity.AddTag(EntityTags.MOVING);
        }
        else
        {
            entity.SetValue(EntityNames.VELOCITY, Vector3.zero);
            entity.RemoveTag(EntityTags.MOVING);
        }
        
        // Process jump input
        if (jumpInput && entity.HasTag(EntityTags.GROUNDED))
        {
            var jumpForce = entity.GetValue<float>(EntityNames.JUMP_FORCE);
            entity.SetValue(EntityNames.JUMP_VELOCITY, jumpForce);
            entity.AddTag(EntityTags.JUMPING);
            entity.RemoveTag(EntityTags.GROUNDED);
        }
        
        // Process attack input
        if (attackInput && entity.CanAttack())
        {
            entity.AddTag(EntityTags.ATTACKING);
            entity.SetValue(EntityNames.ATTACK_START_TIME, Time.time);
        }
    }
}
```

### AI Decision Making Behaviour

```csharp
public class AIBehaviour : IEntityUpdate
{
    private float decisionInterval = 0.5f; // Make decisions twice per second
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Only update AI for active AI entities
        if (!entity.HasTag(EntityTags.AI_ENABLED)) return;
        if (entity.HasTag(EntityTags.STUNNED)) return;
        
        // Throttle AI decisions for performance
        var lastDecisionTime = entity.GetValue<float>(EntityNames.LAST_AI_DECISION_TIME);
        if (Time.time - lastDecisionTime < decisionInterval) return;
        
        // Update AI state
        var aiState = entity.GetValue<AIState>(EntityNames.AI_STATE);
        var targetEntity = FindNearestTarget(entity);
        
        switch (aiState)
        {
            case AIState.Idle:
                HandleIdleState(entity, targetEntity);
                break;
            case AIState.Patrol:
                HandlePatrolState(entity, deltaTime);
                break;
            case AIState.Chase:
                HandleChaseState(entity, targetEntity, deltaTime);
                break;
            case AIState.Attack:
                HandleAttackState(entity, targetEntity);
                break;
        }
        
        entity.SetValue(EntityNames.LAST_AI_DECISION_TIME, Time.time);
    }
    
    private void HandleChaseState(IEntity entity, IEntity target, float deltaTime)
    {
        if (target == null)
        {
            entity.SetValue(EntityNames.AI_STATE, AIState.Idle);
            return;
        }
        
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        var targetPosition = target.GetValue<Vector3>(EntityNames.POSITION);
        var direction = (targetPosition - position).normalized;
        var moveSpeed = entity.GetValue<float>(EntityNames.MOVE_SPEED);
        
        // Move towards target
        entity.SetValue(EntityNames.VELOCITY, direction * moveSpeed);
        
        // Check if close enough to attack
        var distance = Vector3.Distance(position, targetPosition);
        var attackRange = entity.GetValue<float>(EntityNames.ATTACK_RANGE);
        
        if (distance <= attackRange)
        {
            entity.SetValue(EntityNames.AI_STATE, AIState.Attack);
            entity.SetValue(EntityNames.AI_TARGET, target);
        }
    }
}
```

### Generic Player Update Behaviour

```csharp
public class PlayerUpdateBehaviour : IEntityUpdate<PlayerEntity>
{
    public void OnUpdate(PlayerEntity entity, float deltaTime)
    {
        // Update player-specific systems
        UpdatePlayerStats(entity, deltaTime);
        UpdatePlayerUI(entity);
        UpdatePlayerEffects(entity, deltaTime);
        UpdatePlayerInteractions(entity);
        
        // Check win/lose conditions
        CheckGameConditions(entity);
    }
    
    private void UpdatePlayerStats(PlayerEntity entity, float deltaTime)
    {
        // Update experience gain over time
        var expRate = entity.GetExperienceRate();
        if (expRate > 0)
        {
            var expGain = expRate * deltaTime;
            entity.AddExperience(expGain);
        }
        
        // Update mana regeneration
        var currentMana = entity.GetCurrentMana();
        var maxMana = entity.GetMaxMana();
        var manaRegenRate = entity.GetManaRegenRate();
        
        if (currentMana < maxMana && manaRegenRate > 0)
        {
            var manaGain = manaRegenRate * deltaTime;
            entity.SetCurrentMana(Mathf.Min(currentMana + manaGain, maxMana));
        }
    }
    
    private void UpdatePlayerUI(PlayerEntity entity)
    {
        // Update UI elements
        UIManager.UpdateHealthBar(entity.GetCurrentHealth(), entity.GetMaxHealth());
        UIManager.UpdateManaBar(entity.GetCurrentMana(), entity.GetMaxMana());
        UIManager.UpdateExperienceBar(entity.GetExperience(), entity.GetExperienceToNextLevel());
        UIManager.UpdateLevelDisplay(entity.GetPlayerLevel());
    }
}
```

### Timer and Cooldown Behaviour

```csharp
public class TimerBehaviour : IEntityUpdate
{
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Update ability cooldowns
        UpdateCooldowns(entity, deltaTime);
        
        // Update status effect durations
        UpdateStatusEffects(entity, deltaTime);
        
        // Update temporary effects
        UpdateTemporaryEffects(entity, deltaTime);
    }
    
    private void UpdateCooldowns(IEntity entity, float deltaTime)
    {
        var cooldowns = entity.GetValue<Dictionary<string, float>>(EntityNames.COOLDOWNS);
        if (cooldowns == null) return;
        
        var keysToRemove = new List<string>();
        
        foreach (var kvp in cooldowns.ToList())
        {
            var remainingTime = kvp.Value - deltaTime;
            
            if (remainingTime <= 0)
            {
                keysToRemove.Add(kvp.Key);
                GameEvents.OnCooldownCompleted?.Invoke(entity, kvp.Key);
            }
            else
            {
                cooldowns[kvp.Key] = remainingTime;
            }
        }
        
        // Remove completed cooldowns
        foreach (var key in keysToRemove)
        {
            cooldowns.Remove(key);
        }
    }
    
    private void UpdateStatusEffects(IEntity entity, float deltaTime)
    {
        if (!entity.HasTag(EntityTags.HAS_STATUS_EFFECTS)) return;
        
        var effects = entity.GetValue<List<StatusEffect>>(EntityNames.STATUS_EFFECTS);
        
        for (int i = effects.Count - 1; i >= 0; i--)
        {
            var effect = effects[i];
            effect.Duration -= deltaTime;
            
            // Apply effect per frame
            effect.ApplyEffect(entity, deltaTime);
            
            if (effect.Duration <= 0)
            {
                // Remove expired effect
                effect.OnEffectEnd(entity);
                effects.RemoveAt(i);
            }
        }
        
        // Remove tag if no effects remain
        if (effects.Count == 0)
        {
            entity.RemoveTag(EntityTags.HAS_STATUS_EFFECTS);
        }
    }
}
```

## Best Practices

### 1. Frame-Rate Independence

```csharp
public void OnUpdate(IEntity entity, float deltaTime)
{
    // GOOD: Frame-rate independent
    var speed = entity.GetValue<float>(EntityNames.SPEED);
    var distance = speed * deltaTime; // Always consistent
    
    // AVOID: Frame-rate dependent
    var badDistance = speed * 0.016f; // Assumes 60 FPS
}
```

### 2. Performance Optimization

```csharp
public void OnUpdate(IEntity entity, float deltaTime)
{
    // Cache frequently accessed values
    var position = entity.GetValue<Vector3>(EntityNames.POSITION);
    var velocity = entity.GetValue<Vector3>(EntityNames.VELOCITY);
    var transform = entity.GetValue<Transform>(EntityNames.TRANSFORM);
    
    // Perform calculations
    position += velocity * deltaTime;
    
    // Update once at the end
    entity.SetValue(EntityNames.POSITION, position);
    transform.position = position;
}
```

### 3. Conditional Execution

```csharp
public void OnUpdate(IEntity entity, float deltaTime)
{
    // Early exit for inactive entities
    if (!entity.GetValue<bool>(EntityNames.UPDATE_ENABLED)) return;
    if (entity.HasTag(EntityTags.PAUSED)) return;
    
    // Throttle expensive operations
    var lastUpdateTime = entity.GetValue<float>(EntityNames.LAST_EXPENSIVE_UPDATE);
    if (Time.time - lastUpdateTime < 0.1f) return;
    
    // Perform expensive operation
    PerformExpensiveOperation(entity);
    entity.SetValue(EntityNames.LAST_EXPENSIVE_UPDATE, Time.time);
}
```

### 4. State Machine Integration

```csharp
public void OnUpdate(IEntity entity, float deltaTime)
{
    var currentState = entity.GetValue<GameState>(EntityNames.CURRENT_STATE);
    
    switch (currentState)
    {
        case GameState.Playing:
            UpdateGameplayLogic(entity, deltaTime);
            break;
        case GameState.Paused:
            UpdatePausedLogic(entity, deltaTime);
            break;
        case GameState.Menu:
            UpdateMenuLogic(entity, deltaTime);
            break;
    }
}
```

### 5. Memory Management

```csharp
public void OnUpdate(IEntity entity, float deltaTime)
{
    // Avoid allocations in update
    var cachedList = entity.GetValue<List<IEntity>>(EntityNames.CACHED_LIST);
    cachedList.Clear(); // Reuse existing list
    
    // Use object pools for temporary objects
    var tempVector = VectorPool.Get();
    try
    {
        // Use tempVector
        PerformCalculation(tempVector);
    }
    finally
    {
        VectorPool.Return(tempVector);
    }
}
```

## Common Use Cases

### Real-Time Movement
- Position and velocity updates
- Rotation and animation blending
- Physics simulation integration

### Input Handling
- Player input processing
- Gesture recognition
- Controller state management

### AI and Automation
- Decision making systems
- Pathfinding updates
- Behavior tree execution

### Animation and Effects
- Animation parameter updates
- Particle system management
- Visual effect coordination

### Game Logic
- Score and timer updates
- Win/lose condition checking
- Rule enforcement

### User Interface
- HUD element updates
- Menu state management
- Interactive element handling

## Notes

- **Performance Critical**: OnUpdate runs every frame - optimize heavily
- **Delta Time**: Always use deltaTime for frame-rate independent behavior
- **Early Returns**: Use early returns to avoid unnecessary processing
- **State Checks**: Validate entity state before performing operations
- **Memory Awareness**: Minimize allocations to prevent garbage collection spikes
- **Throttling**: Consider throttling expensive operations to maintain frame rate