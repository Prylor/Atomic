# 🧩 IEntityDeactivate

The `IEntityDeactivate` interface defines behaviours that execute when an entity is deactivated or disabled. This interface handles the transition from active to inactive state, making it ideal for pausing operations, disabling systems, and conserving resources while preserving entity state.

## Key Features

- **Automatic Invocation** – Called automatically when `IActivatable.Deactivate()` is executed
- **State Preservation** – Perfect for pausing operations while maintaining state
- **Generic Support** – Includes strongly-typed generic version for specific entity types
- **Resource Conservation** – Helps optimize performance by disabling unnecessary systems
- **Reversible Process** – Designed to be paired with activation for seamless state transitions

---

## Interface Definition

```csharp
public interface IEntityDeactivate : IEntityBehaviour
{
    void OnDeactivate(IEntity entity);
}

public interface IEntityDeactivate<in T> : IEntityDeactivate where T : IEntity
{
    void OnDeactivate(T entity);
}
```

## When It's Called

The `OnDeactivate` method is automatically invoked when:
- An entity transitions from active to inactive state
- `IActivatable.Deactivate()` is called on an active entity
- An entity is paused or disabled temporarily
- Scene entities are disabled in the Unity hierarchy
- Game is paused and entities need to suspend operations

The deactivate behaviour is called **before** despawn behaviours, ensuring proper state preservation.

## Example Implementations

### System Disablement Behaviour

```csharp
public class SystemDeactivatorBehaviour : IEntityDeactivate
{
    public void OnDeactivate(IEntity entity)
    {
        // Disable core systems while preserving state
        entity.SetValue(EntityNames.PHYSICS_ENABLED, false);
        entity.SetValue(EntityNames.COLLISION_ENABLED, false);
        entity.SetValue(EntityNames.UPDATE_ENABLED, false);
        
        // Preserve current state for reactivation
        entity.SetValue(EntityNames.DEACTIVATION_TIME, Time.time);
        entity.AddTag(EntityTags.WAS_ACTIVE);
        entity.RemoveTag(EntityTags.ACTIVE);
        
        // Disable Unity components
        if (entity.TryGetValue<Rigidbody>(EntityNames.RIGIDBODY, out var rigidbody))
        {
            entity.SetValue(EntityNames.VELOCITY_BEFORE_PAUSE, rigidbody.velocity);
            rigidbody.isKinematic = true;
        }
        
        Debug.Log($"Entity {entity.Name} systems deactivated");
    }
}
```

### Animation Pauser Behaviour

```csharp
public class AnimationPauserBehaviour : IEntityDeactivate
{
    public void OnDeactivate(IEntity entity)
    {
        // Pause animations while preserving state
        if (entity.TryGetValue<Animator>(EntityNames.ANIMATOR, out var animator))
        {
            // Save current animation state
            var currentState = animator.GetCurrentAnimatorStateInfo(0);
            entity.SetValue(EntityNames.ANIMATION_STATE_HASH, currentState.fullPathHash);
            entity.SetValue(EntityNames.ANIMATION_TIME, currentState.normalizedTime);
            entity.SetValue(EntityNames.ANIMATION_PAUSE_TIME, Time.time);
            
            // Pause the animator
            animator.SetBool("IsActive", false);
            animator.speed = 0f;
            
            entity.AddTag(EntityTags.ANIMATION_PAUSED);
        }
        
        // Pause particle effects
        if (entity.TryGetValue<ParticleSystem>(EntityNames.PARTICLE_SYSTEM, out var particles))
        {
            particles.Pause();
            entity.AddTag(EntityTags.PARTICLES_PAUSED);
        }
    }
}
```

### Input Disabler Behaviour

```csharp
public class InputDeactivatorBehaviour : IEntityDeactivate
{
    public void OnDeactivate(IEntity entity)
    {
        // Disable input processing while preserving bindings
        if (entity.HasTag(EntityTags.CONTROLLABLE))
        {
            var inputSystem = entity.GetValue<InputSystem>(EntityNames.INPUT_SYSTEM);
            inputSystem?.Disable();
            
            // Save input state
            entity.SetValue(EntityNames.INPUT_ENABLED, false);
            entity.SetValue(EntityNames.LAST_INPUT_TIME, Time.time);
            
            // Preserve input bindings for reactivation
            var bindings = InputManager.GetBindings(entity);
            entity.SetValue(EntityNames.SAVED_INPUT_BINDINGS, bindings);
            
            entity.RemoveTag(EntityTags.INPUT_ACTIVE);
            entity.AddTag(EntityTags.INPUT_PAUSED);
            
            Debug.Log($"Input deactivated for entity {entity.Name}");
        }
    }
}
```

### Generic Player Deactivation

```csharp
public class PlayerDeactivationBehaviour : IEntityDeactivate<PlayerEntity>
{
    public void OnDeactivate(PlayerEntity entity)
    {
        // Save current player state
        var playerState = new PlayerState
        {
            Position = entity.GetValue<Vector3>(EntityNames.POSITION),
            Health = entity.GetCurrentHealth(),
            Experience = entity.GetExperience(),
            ActiveQuests = entity.GetActiveQuests()
        };
        
        entity.SavePlayerState(playerState);
        
        // Disable player-specific systems
        entity.DisablePlayerInput();
        entity.DisablePlayerUI();
        entity.PausePlayerCamera();
        
        // Update UI
        UIManager.ShowPauseOverlay();
        UIManager.HidePlayerHUD();
        
        // Notify game systems
        GameEvents.OnPlayerDeactivated?.Invoke(entity);
        
        entity.RemoveTag(EntityTags.PLAYER_ACTIVE);
        entity.AddTag(EntityTags.PLAYER_PAUSED);
    }
}
```

### Audio Pauser Behaviour

```csharp
public class AudioPauserBehaviour : IEntityDeactivate
{
    public void OnDeactivate(IEntity entity)
    {
        // Pause audio while preserving playback state
        if (entity.TryGetValue<AudioSource>(EntityNames.AUDIO_SOURCE, out var audioSource))
        {
            if (audioSource.isPlaying)
            {
                entity.SetValue(EntityNames.AUDIO_WAS_PLAYING, true);
                entity.SetValue(EntityNames.AUDIO_PAUSE_TIME, audioSource.time);
                audioSource.Pause();
            }
            
            entity.AddTag(EntityTags.AUDIO_PAUSED);
        }
        
        // Stop ambient sounds but save their state
        AudioSystem.PauseAmbient(entity);
        
        entity.SetValue(EntityNames.AUDIO_ENABLED, false);
        entity.SetValue(EntityNames.AUDIO_DEACTIVATION_TIME, Time.time);
    }
}
```

### Network Pause Behaviour

```csharp
public class NetworkPauserBehaviour : IEntityDeactivate
{
    public void OnDeactivate(IEntity entity)
    {
        // Pause network synchronization
        if (entity.HasTag(EntityTags.NETWORKED))
        {
            var networkId = entity.GetValue<int>(EntityNames.NETWORK_ID);
            
            // Save current network state
            var networkState = NetworkManager.GetEntityState(networkId);
            entity.SetValue(EntityNames.NETWORK_STATE_BACKUP, networkState);
            
            // Disable synchronization but maintain connection
            NetworkManager.PauseSynchronization(networkId);
            
            entity.SetValue(EntityNames.NETWORK_SYNC_ENABLED, false);
            entity.RemoveTag(EntityTags.NETWORK_ACTIVE);
            entity.AddTag(EntityTags.NETWORK_PAUSED);
        }
    }
}
```

## Best Practices

### 1. Preserve State for Reactivation

```csharp
public void OnDeactivate(IEntity entity)
{
    // Save time-sensitive information
    entity.SetValue(EntityNames.PAUSE_START_TIME, Time.time);
    
    // Preserve running processes
    if (entity.TryGetValue<Coroutine>(EntityNames.ACTIVE_COROUTINE, out var coroutine))
    {
        entity.SetValue(EntityNames.PAUSED_COROUTINE, coroutine);
        StopCoroutine(coroutine);
        entity.RemoveValue(EntityNames.ACTIVE_COROUTINE);
    }
    
    // Mark as paused for reactivation
    entity.AddTag(EntityTags.WAS_PAUSED);
}
```

### 2. Graceful Resource Management

```csharp
public void OnDeactivate(IEntity entity)
{
    // Reduce resource usage without losing state
    if (entity.TryGetValue<MeshRenderer>(EntityNames.RENDERER, out var renderer))
    {
        entity.SetValue(EntityNames.WAS_VISIBLE, renderer.enabled);
        renderer.enabled = false;
    }
    
    // Pause expensive operations
    if (entity.HasTag(EntityTags.AI_ENABLED))
    {
        AISystem.PauseAI(entity);
        entity.AddTag(EntityTags.AI_PAUSED);
    }
    
    // Reduce update frequency
    entity.SetValue(EntityNames.UPDATE_FREQUENCY, 0f);
}
```

### 3. Handle Partial Deactivation

```csharp
public void OnDeactivate(IEntity entity)
{
    // Allow selective system deactivation
    var systemsToDeactivate = entity.GetValue<HashSet<int>>(EntityNames.SYSTEMS_TO_PAUSE);
    
    if (systemsToDeactivate.Contains(SystemIds.PHYSICS))
    {
        DeactivatePhysics(entity);
    }
    
    if (systemsToDeactivate.Contains(SystemIds.RENDERING))
    {
        DeactivateRendering(entity);
    }
    
    if (systemsToDeactivate.Contains(SystemIds.AUDIO))
    {
        DeactivateAudio(entity);
    }
}
```

### 4. Validate Current State

```csharp
public void OnDeactivate(IEntity entity)
{
    // Ensure entity is in correct state for deactivation
    if (!entity.HasTag(EntityTags.ACTIVE))
    {
        Debug.LogWarning($"Attempting to deactivate inactive entity: {entity.Name}");
        return;
    }
    
    if (entity.HasTag(EntityTags.CRITICAL_OPERATION))
    {
        Debug.LogWarning($"Cannot deactivate entity during critical operation: {entity.Name}");
        return;
    }
    
    // Safe to proceed
    PerformDeactivation(entity);
}
```

### 5. Exception-Safe Deactivation

```csharp
public void OnDeactivate(IEntity entity)
{
    var systemsDeactivated = new List<string>();
    
    try
    {
        DeactivatePhysics(entity);
        systemsDeactivated.Add("Physics");
        
        DeactivateRendering(entity);
        systemsDeactivated.Add("Rendering");
        
        DeactivateAudio(entity);
        systemsDeactivated.Add("Audio");
    }
    catch (Exception ex)
    {
        Debug.LogError($"Deactivation failed after {string.Join(", ", systemsDeactivated)}: {ex.Message}");
        
        // Store partial deactivation state
        entity.SetValue(EntityNames.PARTIALLY_DEACTIVATED, systemsDeactivated);
        entity.AddTag(EntityTags.PARTIAL_DEACTIVATION);
    }
}
```

## Common Use Cases

### Game Pause
- Pausing all entity operations during menu screens
- Preserving game state during interruptions
- Maintaining entity relationships while paused

### Performance Optimization
- Disabling off-screen entities
- Reducing update frequency for distant objects
- Pausing expensive AI calculations

### UI Interaction
- Pausing game during dialog boxes
- Disabling player input during cutscenes
- Temporarily stopping game flow

### Level Transitions
- Pausing current level entities
- Preserving state during scene changes
- Managing entity lifecycle during transitions

### Network Optimization
- Reducing network traffic for inactive players
- Pausing synchronization for background entities
- Managing bandwidth during high-traffic periods

### Save System Integration
- Pausing entities during save operations
- Ensuring consistent state for serialization
- Managing entity state during checkpoints

## Notes

- **Call Order**: OnDeactivate is called before OnDespawn in the entity lifecycle
- **State Preservation**: Focus on preserving state rather than cleaning up resources
- **Reversibility**: Always design deactivation to be reversible by activation
- **Performance**: Use deactivation to optimize performance without losing entity data
- **Pairing**: Always pair with `IEntityActivate` for complete state management
- **Error Recovery**: Handle deactivation failures gracefully to prevent entity corruption