# 🧩 IEntityDespawn

The `IEntityDespawn` interface defines behaviours that execute cleanup logic when an entity is despawned from the world. This interface is called automatically during the entity despawn process, making it essential for resource cleanup, event unsubscription, and proper memory management.

## Key Features

- **Automatic Invocation** – Called automatically when `IEntity.Despawn()` is executed
- **Resource Cleanup** – Perfect for releasing resources and cleaning up references
- **Generic Support** – Includes strongly-typed generic version for specific entity types
- **Memory Management** – Prevents memory leaks through proper cleanup
- **Event-Driven** – Reactive to entity lifecycle events

---

## Interface Definition

```csharp
public interface IEntityDespawn : IEntityBehaviour
{
    void OnDespawn(IEntity entity);
}

public interface IEntityDespawn<in T> : IEntityDespawn where T : IEntity
{
    void OnDespawn(T entity);
}
```

## When It's Called

The `OnDespawn` method is automatically invoked when:
- An entity transitions from spawned to despawned state
- `IEntity.Despawn()` is called on a spawned entity
- An entity is returned to an object pool
- An entity is destroyed or removed from the world

The despawn behaviour is called **after** deactivate behaviours, ensuring proper cleanup order.

## Example Implementations

### Resource Cleanup Behaviour

```csharp
public class ResourceCleanupBehaviour : IEntityDespawn
{
    public void OnDespawn(IEntity entity)
    {
        // Clean up loaded assets
        if (entity.TryGetValue<Object>(EntityNames.LOADED_ASSET, out var asset))
        {
            Resources.UnloadAsset(asset);
            entity.RemoveValue(EntityNames.LOADED_ASSET);
        }
        
        // Clear temporary data
        entity.RemoveValue(EntityNames.TEMP_DATA);
        entity.RemoveTag(EntityTags.ASSETS_LOADED);
        
        Debug.Log($"Entity {entity.Name} resources cleaned up");
    }
}
```

### Event Unsubscription Behaviour

```csharp
public class EventUnsubscriberBehaviour : IEntityDespawn
{
    public void OnDespawn(IEntity entity)
    {
        // Unsubscribe from entity events
        entity.OnValueChanged -= HandleValueChanged;
        entity.OnTagAdded -= HandleTagAdded;
        entity.OnTagRemoved -= HandleTagRemoved;
        
        // Unregister from global systems
        GameEvents.OnLevelChanged -= HandleLevelChange;
        
        entity.RemoveTag(EntityTags.EVENT_SUBSCRIBED);
        
        Debug.Log($"Entity {entity.Name} unsubscribed from all events");
    }
    
    private void HandleValueChanged(IEntity entity, int key) { /* ... */ }
    private void HandleTagAdded(IEntity entity, int tag) { /* ... */ }
    private void HandleTagRemoved(IEntity entity, int tag) { /* ... */ }
    private void HandleLevelChange(int level) { /* ... */ }
}
```

### Animation and Effects Cleanup

```csharp
public class EffectsCleanupBehaviour : IEntityDespawn
{
    public void OnDespawn(IEntity entity)
    {
        // Stop all particle effects
        if (entity.TryGetValue<ParticleSystem>(EntityNames.PARTICLE_SYSTEM, out var particles))
        {
            particles.Stop(true, ParticleSystemStopBehavior.StopEmittingAndClear);
        }
        
        // Cancel running animations
        if (entity.TryGetValue<Animator>(EntityNames.ANIMATOR, out var animator))
        {
            animator.SetBool("IsAlive", false);
        }
        
        // Play despawn effects
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        EffectsSystem.PlayDespawnEffect(position);
        
        // Clean up audio
        AudioSystem.StopAllSounds(entity);
    }
}
```

### Generic Typed Implementation

```csharp
public class PlayerDespawnBehaviour : IEntityDespawn<PlayerEntity>
{
    public void OnDespawn(PlayerEntity entity)
    {
        // Save player progress before despawning
        var saveData = new PlayerSaveData
        {
            Level = entity.GetPlayerLevel(),
            Experience = entity.GetExperience(),
            Position = entity.GetValue<Vector3>(EntityNames.POSITION),
            Inventory = entity.GetInventoryData()
        };
        
        SaveSystem.SavePlayerData(saveData);
        
        // Clean up player-specific resources
        entity.ClearInventory();
        entity.UnregisterFromLeaderboard();
        
        // Remove player tags
        entity.RemoveTag(EntityTags.PLAYER);
        entity.RemoveTag(EntityTags.CONTROLLABLE);
        
        Debug.Log($"Player {entity.GetPlayerName()} despawned and saved");
    }
}
```

### Pool Return Behaviour

```csharp
public class PoolReturnBehaviour : IEntityDespawn
{
    public void OnDespawn(IEntity entity)
    {
        // Reset entity state for pooling
        entity.SetValue(EntityNames.HEALTH, entity.GetValue<int>(EntityNames.MAX_HEALTH));
        entity.SetValue(EntityNames.VELOCITY, Vector3.zero);
        entity.SetValue(EntityNames.ROTATION, Quaternion.identity);
        
        // Clear temporary tags
        entity.RemoveTag(EntityTags.DEAD);
        entity.RemoveTag(EntityTags.DAMAGED);
        entity.RemoveTag(EntityTags.POISONED);
        
        // Clear temporary values
        entity.RemoveValue(EntityNames.DAMAGE_TAKEN);
        entity.RemoveValue(EntityNames.LAST_ATTACKER);
        
        // Mark as ready for reuse
        entity.AddTag(EntityTags.POOL_READY);
    }
}
```

### Network Cleanup Behaviour

```csharp
public class NetworkCleanupBehaviour : IEntityDespawn
{
    public void OnDespawn(IEntity entity)
    {
        // Notify other clients about entity despawn
        if (entity.HasTag(EntityTags.NETWORKED))
        {
            var networkId = entity.GetValue<int>(EntityNames.NETWORK_ID);
            NetworkManager.SendDespawnMessage(networkId);
            
            // Clean up network references
            NetworkManager.UnregisterEntity(networkId);
            entity.RemoveValue(EntityNames.NETWORK_ID);
            entity.RemoveTag(EntityTags.NETWORKED);
        }
        
        // Cancel pending network operations
        NetworkManager.CancelPendingOperations(entity);
    }
}
```

## Best Practices

### 1. Always Clean Up Resources

```csharp
public void OnDespawn(IEntity entity)
{
    // Unload assets
    if (entity.TryGetValue<Texture2D>(EntityNames.TEXTURE, out var texture))
    {
        Resources.UnloadAsset(texture);
    }
    
    // Stop coroutines
    if (entity.TryGetValue<Coroutine>(EntityNames.ACTIVE_COROUTINE, out var coroutine))
    {
        StopCoroutine(coroutine);
    }
    
    // Clean up event subscriptions
    UnsubscribeFromEvents(entity);
}
```

### 2. Handle Paired Operations

```csharp
// If spawn behaviour subscribes to events
public class PairedCleanupBehaviour : IEntitySpawn, IEntityDespawn
{
    public void OnSpawn(IEntity entity)
    {
        GameEvents.OnPlayerDied += HandlePlayerDied;
        entity.AddTag(EntityTags.EVENT_SUBSCRIBED);
    }
    
    public void OnDespawn(IEntity entity)
    {
        // Always pair with spawn operations
        GameEvents.OnPlayerDied -= HandlePlayerDied;
        entity.RemoveTag(EntityTags.EVENT_SUBSCRIBED);
    }
    
    private void HandlePlayerDied(PlayerEntity player) { /* ... */ }
}
```

### 3. Safe Resource Disposal

```csharp
public void OnDespawn(IEntity entity)
{
    // Safe disposal pattern
    if (entity.TryGetValue<IDisposable>(EntityNames.DISPOSABLE_RESOURCE, out var resource))
    {
        try
        {
            resource?.Dispose();
        }
        catch (Exception ex)
        {
            Debug.LogError($"Error disposing resource: {ex.Message}");
        }
        finally
        {
            entity.RemoveValue(EntityNames.DISPOSABLE_RESOURCE);
        }
    }
}
```

### 4. State Validation

```csharp
public void OnDespawn(IEntity entity)
{
    // Validate entity state before cleanup
    if (!entity.IsSpawned)
    {
        Debug.LogWarning($"Attempting to despawn already despawned entity: {entity.Name}");
        return;
    }
    
    // Perform cleanup
    CleanupResources(entity);
    
    // Final validation
    ValidateCleanup(entity);
}
```

### 5. Avoid Exceptions

```csharp
public void OnDespawn(IEntity entity)
{
    try
    {
        // Critical cleanup operations
        UnsubscribeFromEvents(entity);
        ReleaseResources(entity);
    }
    catch (Exception ex)
    {
        // Log but don't rethrow - despawn must succeed
        Debug.LogError($"Error during despawn cleanup: {ex.Message}");
    }
    finally
    {
        // Ensure entity is properly marked as despawned
        entity.RemoveTag(EntityTags.SPAWNED);
    }
}
```

## Common Use Cases

### Memory Management
- Unloading textures and assets
- Disposing of unmanaged resources
- Clearing large data structures

### Event Cleanup
- Unsubscribing from entity events
- Removing global event listeners
- Cancelling callback registrations

### System Deregistration
- Removing from update loops
- Clearing spatial partitioning entries
- Unregistering from physics systems

### Network Synchronization
- Sending despawn messages to clients
- Cleaning up network state
- Cancelling pending network operations

### Object Pooling
- Resetting state for reuse
- Clearing temporary modifications
- Preparing for pool return

### Save Data
- Persisting important state before cleanup
- Creating checkpoints
- Backing up critical information

## Notes

- **Call Order**: OnDespawn is called after OnDeactivate in the entity lifecycle
- **Exception Safety**: Avoid throwing exceptions - despawn operations should always succeed
- **Paired Operations**: Always pair with spawn behaviours for proper resource management
- **Performance**: Keep despawn logic efficient for frequently despawned entities
- **State Reset**: Consider whether to reset or preserve state for pooled entities
- **Cleanup Validation**: Verify that all resources were properly released