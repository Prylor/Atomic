# 🧩 IEntityFixedUpdate

The `IEntityFixedUpdate` interface defines behaviours that execute during the fixed update cycle of an entity. This interface is called automatically at a consistent time interval, making it essential for physics simulation, deterministic calculations, and frame-rate independent operations that require precise timing.

## Key Features

- **Fixed Timestep** – Called at consistent intervals regardless of frame rate
- **Physics Integration** – Perfect for physics calculations and deterministic operations
- **Consistent Timing** – Provides reliable timing for time-critical calculations
- **Generic Support** – Includes strongly-typed generic version for specific entity types
- **Deterministic Results** – Ensures reproducible results across different hardware
- **Network Friendly** – Ideal for networked games requiring synchronization

---

## Interface Definition

```csharp
public interface IEntityFixedUpdate : IEntityBehaviour
{
    void OnFixedUpdate(IEntity entity, float deltaTime);
}

public interface IEntityFixedUpdate<in T> : IEntityFixedUpdate where T : IEntity
{
    void OnFixedUpdate(T entity, float deltaTime);
}
```

## When It's Called

The `OnFixedUpdate` method is automatically invoked:
- At fixed time intervals (typically 50 times per second)
- When `IEntity.OnFixedUpdate(float deltaTime)` is called
- Independent of the frame rate and rendering performance
- Before physics simulation steps in Unity
- Only for active and spawned entities

The fixed delta time is consistent (usually 0.02 seconds) providing deterministic behavior.

## Example Implementations

### Physics Movement Behaviour

```csharp
public class PhysicsMovementBehaviour : IEntityFixedUpdate
{
    public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
    {
        // Get physics components
        if (!entity.TryGetValue<Rigidbody>(EntityNames.RIGIDBODY, out var rigidbody))
            return;
        
        // Apply movement forces
        var inputForce = entity.GetValue<Vector3>(EntityNames.INPUT_FORCE);
        var maxForce = entity.GetValue<float>(EntityNames.MAX_FORCE);
        var moveSpeed = entity.GetValue<float>(EntityNames.MOVE_SPEED);
        
        // Clamp force magnitude
        if (inputForce.magnitude > maxForce)
        {
            inputForce = inputForce.normalized * maxForce;
        }
        
        // Apply force for movement
        rigidbody.AddForce(inputForce * moveSpeed, ForceMode.Force);
        
        // Apply drag
        var drag = entity.GetValue<float>(EntityNames.DRAG_COEFFICIENT);
        rigidbody.velocity *= (1.0f - drag * fixedDeltaTime);
        
        // Update entity position from physics
        entity.SetValue(EntityNames.POSITION, rigidbody.position);
        entity.SetValue(EntityNames.VELOCITY, rigidbody.velocity);
    }
}
```

### Deterministic Timer Behaviour

```csharp
public class DeterministicTimerBehaviour : IEntityFixedUpdate
{
    public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
    {
        // Update fixed-timestep timers for deterministic behavior
        UpdateFixedTimers(entity, fixedDeltaTime);
        UpdatePeriodicActions(entity, fixedDeltaTime);
    }
    
    private void UpdateFixedTimers(IEntity entity, float fixedDeltaTime)
    {
        var timers = entity.GetValue<Dictionary<string, float>>(EntityNames.FIXED_TIMERS);
        if (timers == null) return;
        
        var expiredTimers = new List<string>();
        
        foreach (var kvp in timers.ToList())
        {
            var remainingTime = kvp.Value - fixedDeltaTime;
            
            if (remainingTime <= 0)
            {
                expiredTimers.Add(kvp.Key);
                OnTimerExpired(entity, kvp.Key);
            }
            else
            {
                timers[kvp.Key] = remainingTime;
            }
        }
        
        // Remove expired timers
        foreach (var timer in expiredTimers)
        {
            timers.Remove(timer);
        }
    }
    
    private void UpdatePeriodicActions(IEntity entity, float fixedDeltaTime)
    {
        // Execute actions at fixed intervals
        var lastActionTime = entity.GetValue<float>(EntityNames.LAST_PERIODIC_ACTION);
        var actionInterval = entity.GetValue<float>(EntityNames.PERIODIC_INTERVAL);
        
        if (Time.fixedTime - lastActionTime >= actionInterval)
        {
            ExecutePeriodicAction(entity);
            entity.SetValue(EntityNames.LAST_PERIODIC_ACTION, Time.fixedTime);
        }
    }
}
```

### Force Application Behaviour

```csharp
public class ForceApplicationBehaviour : IEntityFixedUpdate
{
    public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
    {
        if (!entity.TryGetValue<Rigidbody>(EntityNames.RIGIDBODY, out var rigidbody))
            return;
        
        // Apply gravity modifications
        var gravityMultiplier = entity.GetValue<float>(EntityNames.GRAVITY_MULTIPLIER);
        var customGravity = Physics.gravity * (gravityMultiplier - 1.0f);
        rigidbody.AddForce(customGravity, ForceMode.Acceleration);
        
        // Apply wind force
        if (entity.HasTag(EntityTags.AFFECTED_BY_WIND))
        {
            var windForce = entity.GetValue<Vector3>(EntityNames.WIND_FORCE);
            var windResistance = entity.GetValue<float>(EntityNames.WIND_RESISTANCE);
            rigidbody.AddForce(windForce * windResistance, ForceMode.Force);
        }
        
        // Apply magnetic forces
        if (entity.HasTag(EntityTags.MAGNETIC))
        {
            ApplyMagneticForces(entity, rigidbody);
        }
        
        // Apply jump force
        if (entity.HasTag(EntityTags.JUMPING))
        {
            var jumpForce = entity.GetValue<float>(EntityNames.JUMP_FORCE);
            rigidbody.AddForce(Vector3.up * jumpForce, ForceMode.Impulse);
            entity.RemoveTag(EntityTags.JUMPING);
        }
    }
    
    private void ApplyMagneticForces(IEntity entity, Rigidbody rigidbody)
    {
        var magneticEntities = EntityWorld.GetEntitiesWithTag(EntityTags.MAGNETIC);
        var position = rigidbody.position;
        var magneticStrength = entity.GetValue<float>(EntityNames.MAGNETIC_STRENGTH);
        
        foreach (var magneticEntity in magneticEntities)
        {
            if (magneticEntity == entity) continue;
            
            var otherPosition = magneticEntity.GetValue<Vector3>(EntityNames.POSITION);
            var direction = (otherPosition - position);
            var distance = direction.magnitude;
            
            if (distance > 0.1f) // Avoid division by zero
            {
                var force = (magneticStrength * magneticEntity.GetValue<float>(EntityNames.MAGNETIC_STRENGTH)) 
                           / (distance * distance);
                rigidbody.AddForce(direction.normalized * force);
            }
        }
    }
}
```

### Network Synchronization Behaviour

```csharp
public class NetworkSyncBehaviour : IEntityFixedUpdate
{
    public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
    {
        // Only sync networked entities
        if (!entity.HasTag(EntityTags.NETWORKED)) return;
        
        // Use fixed timestep for consistent network updates
        var lastSyncTime = entity.GetValue<float>(EntityNames.LAST_NETWORK_SYNC);
        var syncInterval = entity.GetValue<float>(EntityNames.NETWORK_SYNC_INTERVAL);
        
        if (Time.fixedTime - lastSyncTime >= syncInterval)
        {
            SynchronizeNetworkState(entity);
            entity.SetValue(EntityNames.LAST_NETWORK_SYNC, Time.fixedTime);
        }
        
        // Interpolate networked position
        if (entity.HasTag(EntityTags.NETWORK_INTERPOLATION))
        {
            InterpolateNetworkPosition(entity, fixedDeltaTime);
        }
    }
    
    private void SynchronizeNetworkState(IEntity entity)
    {
        var networkData = new NetworkEntityState
        {
            Position = entity.GetValue<Vector3>(EntityNames.POSITION),
            Rotation = entity.GetValue<Quaternion>(EntityNames.ROTATION),
            Velocity = entity.GetValue<Vector3>(EntityNames.VELOCITY),
            Timestamp = Time.fixedTime
        };
        
        NetworkManager.SendEntityState(entity.GetValue<int>(EntityNames.NETWORK_ID), networkData);
    }
    
    private void InterpolateNetworkPosition(IEntity entity, float fixedDeltaTime)
    {
        var targetPosition = entity.GetValue<Vector3>(EntityNames.NETWORK_TARGET_POSITION);
        var currentPosition = entity.GetValue<Vector3>(EntityNames.POSITION);
        var interpolationSpeed = entity.GetValue<float>(EntityNames.INTERPOLATION_SPEED);
        
        var newPosition = Vector3.MoveTowards(currentPosition, targetPosition, 
            interpolationSpeed * fixedDeltaTime);
        
        entity.SetValue(EntityNames.POSITION, newPosition);
        
        // Update Transform for rendering
        if (entity.TryGetValue<Transform>(EntityNames.TRANSFORM, out var transform))
        {
            transform.position = newPosition;
        }
    }
}
```

### Projectile Physics Behaviour

```csharp
public class ProjectilePhysicsBehaviour : IEntityFixedUpdate<ProjectileEntity>
{
    public void OnFixedUpdate(ProjectileEntity entity, float fixedDeltaTime)
    {
        // Calculate projectile physics
        var position = entity.GetPosition();
        var velocity = entity.GetVelocity();
        var gravity = entity.GetGravityEffect();
        var drag = entity.GetDragCoefficient();
        
        // Apply gravity
        velocity.y -= gravity * fixedDeltaTime;
        
        // Apply drag
        velocity *= (1.0f - drag * fixedDeltaTime);
        
        // Update position
        position += velocity * fixedDeltaTime;
        
        // Check for collision
        if (CheckCollision(entity, position))
        {
            HandleProjectileHit(entity, position);
            return;
        }
        
        // Update entity state
        entity.SetPosition(position);
        entity.SetVelocity(velocity);
        
        // Check lifetime
        var lifeTime = entity.GetLifeTime() - fixedDeltaTime;
        entity.SetLifeTime(lifeTime);
        
        if (lifeTime <= 0)
        {
            entity.Despawn();
        }
    }
    
    private bool CheckCollision(ProjectileEntity entity, Vector3 nextPosition)
    {
        var currentPosition = entity.GetPosition();
        var direction = nextPosition - currentPosition;
        var distance = direction.magnitude;
        
        return Physics.Raycast(currentPosition, direction.normalized, distance, 
            entity.GetCollisionLayers());
    }
}
```

### Vehicle Physics Behaviour

```csharp
public class VehiclePhysicsBehaviour : IEntityFixedUpdate
{
    public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
    {
        if (!entity.TryGetValue<Rigidbody>(EntityNames.RIGIDBODY, out var rigidbody))
            return;
        
        // Get input values
        var throttleInput = entity.GetValue<float>(EntityNames.THROTTLE_INPUT);
        var steerInput = entity.GetValue<float>(EntityNames.STEERING_INPUT);
        var brakeInput = entity.GetValue<float>(EntityNames.BRAKE_INPUT);
        
        // Vehicle parameters
        var motorPower = entity.GetValue<float>(EntityNames.MOTOR_POWER);
        var steerPower = entity.GetValue<float>(EntityNames.STEERING_POWER);
        var brakePower = entity.GetValue<float>(EntityNames.BRAKE_POWER);
        
        // Apply motor force
        var motorForce = throttleInput * motorPower;
        rigidbody.AddForce(entity.transform.forward * motorForce);
        
        // Apply steering
        if (rigidbody.velocity.magnitude > 1.0f)
        {
            var steerTorque = steerInput * steerPower * rigidbody.velocity.magnitude;
            rigidbody.AddTorque(entity.transform.up * steerTorque);
        }
        
        // Apply brakes
        if (brakeInput > 0.1f)
        {
            var brakeForce = rigidbody.velocity.normalized * brakePower * brakeInput;
            rigidbody.AddForce(-brakeForce);
        }
        
        // Update entity values
        entity.SetValue(EntityNames.SPEED, rigidbody.velocity.magnitude);
        entity.SetValue(EntityNames.POSITION, rigidbody.position);
    }
}
```

## Best Practices

### 1. Use Fixed Timestep for Physics

```csharp
public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
{
    // GOOD: Use fixedDeltaTime for physics calculations
    var acceleration = entity.GetValue<Vector3>(EntityNames.ACCELERATION);
    var velocity = entity.GetValue<Vector3>(EntityNames.VELOCITY);
    
    velocity += acceleration * fixedDeltaTime;
    entity.SetValue(EntityNames.VELOCITY, velocity);
    
    // AVOID: Using variable deltaTime in physics
    // velocity += acceleration * Time.deltaTime; // Inconsistent results
}
```

### 2. Keep Physics Separate from Rendering

```csharp
public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
{
    // Physics calculations
    UpdatePhysicsSimulation(entity, fixedDeltaTime);
    
    // Store physics results in entity state
    entity.SetValue(EntityNames.PHYSICS_POSITION, rigidbody.position);
    entity.SetValue(EntityNames.PHYSICS_VELOCITY, rigidbody.velocity);
    
    // Let rendering interpolate between physics steps
    // Don't directly update Transform here
}
```

### 3. Handle Deterministic Operations

```csharp
public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
{
    // Use fixed timestep for deterministic calculations
    var deterministicTimer = entity.GetValue<float>(EntityNames.DETERMINISTIC_TIMER);
    deterministicTimer += fixedDeltaTime;
    
    // Trigger events at exact intervals
    if (deterministicTimer >= 1.0f) // Exactly every second
    {
        TriggerDeterministicEvent(entity);
        deterministicTimer = 0.0f; // Reset exactly
    }
    
    entity.SetValue(EntityNames.DETERMINISTIC_TIMER, deterministicTimer);
}
```

### 4. Optimize Heavy Physics Calculations

```csharp
public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
{
    // Throttle expensive physics operations
    var lastComplexPhysicsTime = entity.GetValue<float>(EntityNames.LAST_COMPLEX_PHYSICS);
    var complexPhysicsInterval = 0.1f; // Every 5 fixed updates
    
    if (Time.fixedTime - lastComplexPhysicsTime >= complexPhysicsInterval)
    {
        PerformComplexPhysicsCalculation(entity);
        entity.SetValue(EntityNames.LAST_COMPLEX_PHYSICS, Time.fixedTime);
    }
    
    // Always do basic physics
    PerformBasicPhysics(entity, fixedDeltaTime);
}
```

### 5. Network Synchronization

```csharp
public void OnFixedUpdate(IEntity entity, float fixedDeltaTime)
{
    if (!entity.HasTag(EntityTags.NETWORKED)) return;
    
    // Use fixed timestep for consistent network behavior
    var networkTick = entity.GetValue<int>(EntityNames.NETWORK_TICK);
    networkTick++;
    entity.SetValue(EntityNames.NETWORK_TICK, networkTick);
    
    // Send updates at regular intervals
    if (networkTick % 3 == 0) // Every 3rd fixed update
    {
        SendNetworkUpdate(entity, networkTick);
    }
}
```

## Common Use Cases

### Physics Simulation
- Rigidbody force application
- Collision detection and response
- Vehicle and character controllers

### Deterministic Calculations
- Network game synchronization
- Replay system recording
- Consistent timer operations

### Projectile Systems
- Ballistic trajectory calculation
- Particle physics simulation
- Collision prediction

### Force-Based Systems
- Magnetic field effects
- Gravitational simulation
- Fluid dynamics

### Network Gaming
- Server tick rate synchronization
- Client prediction and rollback
- Deterministic state updates

### Animation Blending
- Physics-based animation
- Ragdoll integration
- Procedural animation

## Notes

- **Fixed Timestep**: Always runs at consistent intervals (usually 0.02s or 50Hz)
- **Physics Integration**: Perfect for Unity's physics system integration
- **Deterministic**: Provides reproducible results across different hardware
- **Performance**: More predictable performance impact than variable timestep
- **Network Friendly**: Essential for networked games requiring synchronization
- **Separate from Rendering**: Runs independently of frame rate and rendering performance