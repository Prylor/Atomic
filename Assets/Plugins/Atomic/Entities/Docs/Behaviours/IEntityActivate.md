# 🧩 IEntityActivate

The `IEntityActivate` interface defines behaviours that execute when an entity is activated or enabled. This interface handles the transition from inactive to active state, making it ideal for resuming operations, enabling systems, and restoring functionality after being paused or disabled.

## Key Features

- **Automatic Invocation** – Called automatically when `IActivatable.Activate()` is executed
- **State Resumption** – Perfect for resuming paused operations and enabling systems
- **Generic Support** – Includes strongly-typed generic version for specific entity types
- **Reactive Design** – Works with entity state changes rather than storing internal state
- **Lifecycle Integration** – Part of the entity activation/deactivation cycle

---

## Interface Definition

```csharp
public interface IEntityActivate : IEntityBehaviour
{
    void OnActivate(IEntity entity);
}

public interface IEntityActivate<in T> : IEntityActivate where T : IEntity
{
    void OnActivate(T entity);
}
```

## When It's Called

The `OnActivate` method is automatically invoked when:
- An entity transitions from inactive to active state
- `IActivatable.Activate()` is called on an inactive entity
- An entity is spawned (spawn occurs before activate)
- An entity resumes after being paused or disabled
- Scene entities are enabled in the Unity hierarchy

The activate behaviour is called **after** spawn behaviours but **before** update behaviours begin.

## Example Implementations

### System Enablement Behaviour

```csharp
public class SystemActivatorBehaviour : IEntityActivate
{
    public void OnActivate(IEntity entity)
    {
        // Enable core systems
        entity.SetValue(EntityNames.PHYSICS_ENABLED, true);
        entity.SetValue(EntityNames.RENDERING_ENABLED, true);
        entity.SetValue(EntityNames.COLLISION_ENABLED, true);
        
        // Start update loops
        entity.AddTag(EntityTags.UPDATE_ENABLED);
        entity.AddTag(EntityTags.ACTIVE);
        
        // Enable Unity components
        if (entity.TryGetValue<Rigidbody>(EntityNames.RIGIDBODY, out var rigidbody))
        {
            rigidbody.isKinematic = false;
        }
        
        Debug.Log($"Entity {entity.Name} systems activated");
    }
}
```

### Animation Controller Behaviour

```csharp
public class AnimationActivatorBehaviour : IEntityActivate
{
    public void OnActivate(IEntity entity)
    {
        // Resume animations
        if (entity.TryGetValue<Animator>(EntityNames.ANIMATOR, out var animator))
        {
            animator.enabled = true;
            animator.SetBool("IsActive", true);
            
            // Resume from paused state
            if (entity.HasTag(EntityTags.ANIMATION_PAUSED))
            {
                var pausedTime = entity.GetValue<float>(EntityNames.ANIMATION_PAUSE_TIME);
                var currentTime = Time.time;
                animator.SetFloat("ResumeOffset", currentTime - pausedTime);
                
                entity.RemoveTag(EntityTags.ANIMATION_PAUSED);
                entity.RemoveValue(EntityNames.ANIMATION_PAUSE_TIME);
            }
        }
        
        // Start particle effects
        if (entity.TryGetValue<ParticleSystem>(EntityNames.PARTICLE_SYSTEM, out var particles))
        {
            particles.Play();
        }
    }
}
```

### Input Handler Activation

```csharp
public class InputActivatorBehaviour : IEntityActivate
{
    public void OnActivate(IEntity entity)
    {
        // Enable input processing
        if (entity.HasTag(EntityTags.CONTROLLABLE))
        {
            var inputSystem = entity.GetValue<InputSystem>(EntityNames.INPUT_SYSTEM);
            inputSystem?.Enable();
            
            entity.SetValue(EntityNames.INPUT_ENABLED, true);
            entity.AddTag(EntityTags.INPUT_ACTIVE);
            
            // Restore input bindings
            RestoreInputBindings(entity);
            
            Debug.Log($"Input activated for entity {entity.Name}");
        }
    }
    
    private void RestoreInputBindings(IEntity entity)
    {
        var bindings = entity.GetValue<Dictionary<string, KeyCode>>(EntityNames.INPUT_BINDINGS);
        foreach (var binding in bindings)
        {
            InputManager.RegisterBinding(entity, binding.Key, binding.Value);
        }
    }
}
```

### Generic Player Activation

```csharp
public class PlayerActivationBehaviour : IEntityActivate<PlayerEntity>
{
    public void OnActivate(PlayerEntity entity)
    {
        // Enable player-specific systems
        entity.EnablePlayerInput();
        entity.EnablePlayerUI();
        entity.EnablePlayerCamera();
        
        // Resume player state
        if (entity.HasPlayerSave())
        {
            entity.LoadPlayerState();
        }
        
        // Update UI
        UIManager.ShowPlayerHUD();
        UIManager.UpdateHealthBar(entity.GetCurrentHealth(), entity.GetMaxHealth());
        
        // Register player events
        GameEvents.OnPlayerActivated?.Invoke(entity);
        
        entity.AddTag(EntityTags.PLAYER_ACTIVE);
    }
}
```

### Audio System Activation

```csharp
public class AudioActivatorBehaviour : IEntityActivate
{
    public void OnActivate(IEntity entity)
    {
        // Enable audio components
        if (entity.TryGetValue<AudioSource>(EntityNames.AUDIO_SOURCE, out var audioSource))
        {
            audioSource.enabled = true;
            
            // Resume paused audio
            if (entity.HasTag(EntityTags.AUDIO_PAUSED))
            {
                audioSource.UnPause();
                entity.RemoveTag(EntityTags.AUDIO_PAUSED);
            }
        }
        
        // Start ambient sounds
        var ambientClip = entity.GetValue<AudioClip>(EntityNames.AMBIENT_SOUND);
        if (ambientClip != null)
        {
            AudioSystem.PlayAmbient(entity, ambientClip);
        }
        
        entity.SetValue(EntityNames.AUDIO_ENABLED, true);
    }
}
```

### Network Synchronization Activation

```csharp
public class NetworkActivatorBehaviour : IEntityActivate
{
    public void OnActivate(IEntity entity)
    {
        // Re-enable network synchronization
        if (entity.HasTag(EntityTags.NETWORKED))
        {
            var networkId = entity.GetValue<int>(EntityNames.NETWORK_ID);
            NetworkManager.EnableSynchronization(networkId);
            
            // Send activation message to clients
            NetworkManager.SendActivationMessage(networkId);
            
            // Resume position synchronization
            entity.SetValue(EntityNames.NETWORK_SYNC_ENABLED, true);
            entity.AddTag(EntityTags.NETWORK_ACTIVE);
        }
    }
}
```

## Best Practices

### 1. Resume vs Restart Logic

```csharp
public void OnActivate(IEntity entity)
{
    // Check if resuming from pause or fresh activation
    if (entity.HasTag(EntityTags.WAS_PAUSED))
    {
        // Resume from paused state
        var pausedTime = entity.GetValue<float>(EntityNames.PAUSE_TIME);
        var deltaTime = Time.time - pausedTime;
        
        // Adjust time-dependent values
        AdjustTimeDependentValues(entity, deltaTime);
        
        entity.RemoveTag(EntityTags.WAS_PAUSED);
        entity.RemoveValue(EntityNames.PAUSE_TIME);
    }
    else
    {
        // Fresh activation
        InitializeFreshState(entity);
    }
    
    entity.AddTag(EntityTags.ACTIVE);
}
```

### 2. Validate Entity State

```csharp
public void OnActivate(IEntity entity)
{
    // Ensure entity is in correct state for activation
    if (!entity.IsSpawned)
    {
        Debug.LogWarning($"Attempting to activate non-spawned entity: {entity.Name}");
        return;
    }
    
    if (entity.HasTag(EntityTags.DESTROYED))
    {
        Debug.LogError($"Cannot activate destroyed entity: {entity.Name}");
        return;
    }
    
    // Proceed with activation
    PerformActivation(entity);
}
```

### 3. Handle Dependencies

```csharp
public void OnActivate(IEntity entity)
{
    // Check for required dependencies
    var requiredSystems = new[]
    {
        EntityNames.TRANSFORM,
        EntityNames.RENDERER,
        EntityNames.COLLIDER
    };
    
    foreach (var system in requiredSystems)
    {
        if (!entity.HasValue(system))
        {
            Debug.LogError($"Entity {entity.Name} missing required system: {system}");
            entity.AddTag(EntityTags.ACTIVATION_FAILED);
            return;
        }
    }
    
    // All dependencies satisfied
    EnableAllSystems(entity);
    entity.AddTag(EntityTags.FULLY_ACTIVATED);
}
```

### 4. Gradual Activation

```csharp
public void OnActivate(IEntity entity)
{
    // Activate systems in stages to avoid frame drops
    StartCoroutine(GradualActivation(entity));
}

private IEnumerator GradualActivation(IEntity entity)
{
    // Stage 1: Critical systems
    ActivateCriticalSystems(entity);
    yield return null;
    
    // Stage 2: Rendering
    ActivateRenderingSystems(entity);
    yield return null;
    
    // Stage 3: Audio and effects
    ActivateAudioAndEffects(entity);
    yield return null;
    
    // Final stage
    entity.AddTag(EntityTags.FULLY_ACTIVATED);
}
```

### 5. Error Recovery

```csharp
public void OnActivate(IEntity entity)
{
    try
    {
        ActivateCoreSystems(entity);
        ActivateOptionalSystems(entity);
        
        entity.AddTag(EntityTags.ACTIVE);
    }
    catch (Exception ex)
    {
        Debug.LogError($"Activation failed for {entity.Name}: {ex.Message}");
        
        // Attempt graceful degradation
        ActivateMinimalSystems(entity);
        entity.AddTag(EntityTags.PARTIAL_ACTIVATION);
    }
}
```

## Common Use Cases

### System Resumption
- Re-enabling physics simulation
- Resuming animation playback
- Activating AI decision making

### Input Handling
- Enabling player controls
- Activating interaction systems
- Resuming input processing

### Rendering and Effects
- Enabling mesh renderers
- Starting particle systems
- Resuming visual effects

### Audio Management
- Unpausing audio sources
- Starting ambient sounds
- Enabling sound effects

### Network Operations
- Re-enabling network synchronization
- Resuming data transmission
- Activating network callbacks

### UI Integration
- Showing interface elements
- Enabling user interactions
- Updating display information

## Notes

- **Call Order**: OnActivate is called after OnSpawn and before update methods
- **State Management**: Use entity values to track activation state rather than internal fields
- **Performance**: Consider spreading heavy activation work across multiple frames
- **Error Handling**: Gracefully handle activation failures without breaking the entity
- **Pairing**: Always pair with `IEntityDeactivate` for proper state management
- **Dependencies**: Validate required components and systems before activation