# 🧩 IEntityLateUpdate

The `IEntityLateUpdate` interface defines behaviours that execute during the late update phase of the entity lifecycle. This interface is called after all regular update operations have completed, making it perfect for post-processing, camera following, UI updates, and order-dependent calculations that need to happen after the main game logic.

## Key Features

- **Post-Processing Timing** – Called after all OnUpdate methods complete
- **Order-Dependent Operations** – Perfect for operations that depend on other updates
- **Camera and UI Updates** – Ideal for camera following and UI synchronization
- **Generic Support** – Includes strongly-typed generic version for specific entity types
- **Final Calculations** – Last chance to modify entity state before rendering
- **Transform Synchronization** – Perfect for visual updates and transform corrections

---

## Interface Definition

```csharp
public interface IEntityLateUpdate : IEntityBehaviour
{
    void OnLateUpdate(IEntity entity, float deltaTime);
}

public interface IEntityLateUpdate<in T> : IEntityLateUpdate where T : IEntity
{
    void OnLateUpdate(T entity, float deltaTime);
}
```

## When It's Called

The `OnLateUpdate` method is automatically invoked:
- After all OnUpdate methods have completed
- When `IEntity.OnLateUpdate(float deltaTime)` is called
- Before the final rendering pass
- Every frame for active and spawned entities
- After physics calculations but before rendering

This makes it ideal for operations that need to respond to or finalize the results of regular updates.

## Example Implementations

### Camera Follow Behaviour

```csharp
public class CameraFollowBehaviour : IEntityLateUpdate
{
    public void OnLateUpdate(IEntity entity, float deltaTime)
    {
        // Only update if this entity is the camera target
        if (!entity.HasTag(EntityTags.CAMERA_TARGET)) return;
        
        var cameraEntity = EntityWorld.GetEntityWithTag(EntityTags.MAIN_CAMERA);
        if (cameraEntity == null) return;
        
        // Get target position (after all movement updates)
        var targetPosition = entity.GetValue<Vector3>(EntityNames.POSITION);
        var cameraOffset = entity.GetValue<Vector3>(EntityNames.CAMERA_OFFSET);
        var followSpeed = entity.GetValue<float>(EntityNames.CAMERA_FOLLOW_SPEED);
        
        // Calculate desired camera position
        var desiredPosition = targetPosition + cameraOffset;
        
        // Get current camera position
        var currentCameraPosition = cameraEntity.GetValue<Vector3>(EntityNames.POSITION);
        
        // Smoothly move camera
        var newCameraPosition = Vector3.Lerp(currentCameraPosition, desiredPosition, 
            followSpeed * deltaTime);
        
        cameraEntity.SetValue(EntityNames.POSITION, newCameraPosition);
        
        // Update Unity Camera Transform
        if (cameraEntity.TryGetValue<Transform>(EntityNames.TRANSFORM, out var cameraTransform))
        {
            cameraTransform.position = newCameraPosition;
        }
        
        // Update camera look direction
        if (entity.HasTag(EntityTags.CAMERA_LOOK_AT_TARGET))
        {
            var lookDirection = (targetPosition - newCameraPosition).normalized;
            var lookRotation = Quaternion.LookRotation(lookDirection);
            cameraTransform.rotation = lookRotation;
        }
    }
}
```

### UI Synchronization Behaviour

```csharp
public class UIUpdateBehaviour : IEntityLateUpdate
{
    public void OnLateUpdate(IEntity entity, float deltaTime)
    {
        // Update UI elements based on final entity state
        UpdateHealthBar(entity);
        UpdateNameplate(entity);
        UpdateStatusIndicators(entity);
        UpdateWorldSpaceUI(entity);
    }
    
    private void UpdateHealthBar(IEntity entity)
    {
        if (!entity.HasTag(EntityTags.HAS_HEALTH_BAR)) return;
        
        var healthBarUI = entity.GetValue<HealthBarUI>(EntityNames.HEALTH_BAR_UI);
        var currentHealth = entity.GetValue<float>(EntityNames.CURRENT_HEALTH);
        var maxHealth = entity.GetValue<float>(EntityNames.MAX_HEALTH);
        
        healthBarUI.UpdateHealth(currentHealth, maxHealth);
        
        // Position health bar above entity
        var worldPosition = entity.GetValue<Vector3>(EntityNames.POSITION);
        var screenPosition = Camera.main.WorldToScreenPoint(worldPosition + Vector3.up * 2f);
        healthBarUI.SetScreenPosition(screenPosition);
    }
    
    private void UpdateNameplate(IEntity entity)
    {
        if (!entity.HasTag(EntityTags.HAS_NAMEPLATE)) return;
        
        var nameplate = entity.GetValue<NameplateUI>(EntityNames.NAMEPLATE);
        var worldPosition = entity.GetValue<Vector3>(EntityNames.POSITION);
        var nameplateOffset = entity.GetValue<Vector3>(EntityNames.NAMEPLATE_OFFSET);
        
        var screenPosition = Camera.main.WorldToScreenPoint(worldPosition + nameplateOffset);
        nameplate.SetScreenPosition(screenPosition);
        
        // Update nameplate visibility based on distance
        var cameraPosition = Camera.main.transform.position;
        var distance = Vector3.Distance(worldPosition, cameraPosition);
        var maxDistance = entity.GetValue<float>(EntityNames.NAMEPLATE_MAX_DISTANCE);
        
        nameplate.SetVisible(distance <= maxDistance);
    }
}
```

### Animation Post-Processing Behaviour

```csharp
public class AnimationPostProcessBehaviour : IEntityLateUpdate
{
    public void OnLateUpdate(IEntity entity, float deltaTime)
    {
        // Apply animation corrections after all updates
        ApplyIKCorrections(entity);
        ApplyLookAtConstraints(entity);
        ApplyAnimationBlending(entity);
        SynchronizeAnimatorParameters(entity);
    }
    
    private void ApplyIKCorrections(IEntity entity)
    {
        if (!entity.TryGetValue<Animator>(EntityNames.ANIMATOR, out var animator)) return;
        if (!entity.HasTag(EntityTags.IK_ENABLED)) return;
        
        // Apply foot IK
        if (entity.HasTag(EntityTags.FOOT_IK))
        {
            var leftFootTarget = entity.GetValue<Transform>(EntityNames.LEFT_FOOT_IK_TARGET);
            var rightFootTarget = entity.GetValue<Transform>(EntityNames.RIGHT_FOOT_IK_TARGET);
            
            if (leftFootTarget != null)
            {
                animator.SetIKPositionWeight(AvatarIKGoal.LeftFoot, 1.0f);
                animator.SetIKPosition(AvatarIKGoal.LeftFoot, leftFootTarget.position);
            }
            
            if (rightFootTarget != null)
            {
                animator.SetIKPositionWeight(AvatarIKGoal.RightFoot, 1.0f);
                animator.SetIKPosition(AvatarIKGoal.RightFoot, rightFootTarget.position);
            }
        }
    }
    
    private void ApplyLookAtConstraints(IEntity entity)
    {
        if (!entity.HasTag(EntityTags.LOOK_AT_ENABLED)) return;
        
        var lookAtTarget = entity.GetValue<Transform>(EntityNames.LOOK_AT_TARGET);
        if (lookAtTarget == null) return;
        
        var headTransform = entity.GetValue<Transform>(EntityNames.HEAD_TRANSFORM);
        var lookAtSpeed = entity.GetValue<float>(EntityNames.LOOK_AT_SPEED);
        
        if (headTransform != null)
        {
            var direction = (lookAtTarget.position - headTransform.position).normalized;
            var targetRotation = Quaternion.LookRotation(direction);
            
            headTransform.rotation = Quaternion.Slerp(headTransform.rotation, 
                targetRotation, lookAtSpeed * deltaTime);
        }
    }
}
```

### Physics Interpolation Behaviour

```csharp
public class PhysicsInterpolationBehaviour : IEntityLateUpdate
{
    public void OnLateUpdate(IEntity entity, float deltaTime)
    {
        // Interpolate visual position between physics steps
        InterpolatePhysicsTransforms(entity, deltaTime);
        SmoothRigidbodyMovement(entity);
        UpdateTrails(entity);
    }
    
    private void InterpolatePhysicsTransforms(IEntity entity, float deltaTime)
    {
        if (!entity.TryGetValue<Rigidbody>(EntityNames.RIGIDBODY, out var rigidbody)) return;
        if (!entity.TryGetValue<Transform>(EntityNames.VISUAL_TRANSFORM, out var visualTransform)) return;
        
        // Get physics position and visual position
        var physicsPosition = rigidbody.position;
        var currentVisualPosition = visualTransform.position;
        
        // Interpolate for smooth movement
        var interpolationSpeed = entity.GetValue<float>(EntityNames.INTERPOLATION_SPEED);
        var targetPosition = Vector3.Lerp(currentVisualPosition, physicsPosition, 
            interpolationSpeed * deltaTime);
        
        visualTransform.position = targetPosition;
        
        // Also interpolate rotation
        var targetRotation = Quaternion.Slerp(visualTransform.rotation, rigidbody.rotation,
            interpolationSpeed * deltaTime);
        
        visualTransform.rotation = targetRotation;
        
        // Update entity visual state
        entity.SetValue(EntityNames.VISUAL_POSITION, targetPosition);
        entity.SetValue(EntityNames.VISUAL_ROTATION, targetRotation);
    }
    
    private void UpdateTrails(IEntity entity)
    {
        if (!entity.TryGetValue<TrailRenderer>(EntityNames.TRAIL_RENDERER, out var trail)) return;
        
        var position = entity.GetValue<Vector3>(EntityNames.VISUAL_POSITION);
        var velocity = entity.GetValue<Vector3>(EntityNames.VELOCITY);
        
        // Update trail width based on velocity
        var speed = velocity.magnitude;
        var maxSpeed = entity.GetValue<float>(EntityNames.MAX_SPEED);
        var trailWidth = Mathf.Lerp(0.1f, 1.0f, speed / maxSpeed);
        
        trail.startWidth = trailWidth;
        trail.endWidth = trailWidth * 0.1f;
    }
}
```

### Generic Player Post-Update Behaviour

```csharp
public class PlayerPostUpdateBehaviour : IEntityLateUpdate<PlayerEntity>
{
    public void OnLateUpdate(PlayerEntity entity, float deltaTime)
    {
        // Final player state calculations
        UpdatePlayerUI(entity);
        UpdatePlayerEffects(entity);
        UpdatePlayerAudio(entity);
        ValidatePlayerState(entity);
    }
    
    private void UpdatePlayerUI(PlayerEntity entity)
    {
        // Update all UI elements with final values
        UIManager.UpdatePlayerStats(
            entity.GetCurrentHealth(),
            entity.GetMaxHealth(),
            entity.GetCurrentMana(),
            entity.GetMaxMana(),
            entity.GetExperience(),
            entity.GetExperienceToNextLevel()
        );
        
        // Update minimap position
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        UIManager.UpdateMinimapPlayerPosition(position);
        
        // Update crosshair based on final aim direction
        if (entity.HasTag(EntityTags.AIMING))
        {
            var aimDirection = entity.GetValue<Vector3>(EntityNames.AIM_DIRECTION);
            UIManager.UpdateCrosshair(aimDirection);
        }
    }
    
    private void UpdatePlayerEffects(PlayerEntity entity)
    {
        // Update particle effects based on final state
        if (entity.IsRunning())
        {
            var dustEffect = entity.GetValue<ParticleSystem>(EntityNames.DUST_EFFECT);
            dustEffect?.Play();
        }
        
        // Update screen effects
        if (entity.GetCurrentHealth() < entity.GetMaxHealth() * 0.2f)
        {
            ScreenEffects.ShowLowHealthEffect();
        }
        else
        {
            ScreenEffects.HideLowHealthEffect();
        }
    }
}
```

### Transform Synchronization Behaviour

```csharp
public class TransformSyncBehaviour : IEntityLateUpdate
{
    public void OnLateUpdate(IEntity entity, float deltaTime)
    {
        // Ensure Unity Transforms match entity state after all updates
        SyncTransformToEntity(entity);
        SyncChildTransforms(entity);
        UpdateBounds(entity);
    }
    
    private void SyncTransformToEntity(IEntity entity)
    {
        if (!entity.TryGetValue<Transform>(EntityNames.TRANSFORM, out var transform)) return;
        
        // Get final entity values
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        var rotation = entity.GetValue<Quaternion>(EntityNames.ROTATION);
        var scale = entity.GetValue<Vector3>(EntityNames.SCALE);
        
        // Only update if values changed (avoid unnecessary Unity calls)
        if (Vector3.Distance(transform.position, position) > 0.001f)
        {
            transform.position = position;
        }
        
        if (Quaternion.Angle(transform.rotation, rotation) > 0.1f)
        {
            transform.rotation = rotation;
        }
        
        if (Vector3.Distance(transform.localScale, scale) > 0.001f)
        {
            transform.localScale = scale;
        }
    }
    
    private void SyncChildTransforms(IEntity entity)
    {
        var childEntities = entity.GetValue<List<IEntity>>(EntityNames.CHILD_ENTITIES);
        if (childEntities == null) return;
        
        foreach (var child in childEntities)
        {
            // Update child transforms relative to parent
            UpdateChildTransform(entity, child);
        }
    }
}
```

## Best Practices

### 1. Use for Order-Dependent Operations

```csharp
public void OnLateUpdate(IEntity entity, float deltaTime)
{
    // GOOD: Operations that depend on other updates being complete
    var finalPosition = entity.GetValue<Vector3>(EntityNames.POSITION);
    var camera = Camera.main;
    var screenPos = camera.WorldToScreenPoint(finalPosition);
    UpdateUIElement(screenPos);
    
    // AVOID: Independent calculations that could go in OnUpdate
    // var health = entity.GetValue<float>(EntityNames.HEALTH);
    // health += regenRate * deltaTime; // This should be in OnUpdate
}
```

### 2. Minimize Heavy Calculations

```csharp
public void OnLateUpdate(IEntity entity, float deltaTime)
{
    // Cache expensive operations
    if (!_cachedScreenPosition.HasValue || 
        Vector3.Distance(_lastWorldPosition, entity.GetValue<Vector3>(EntityNames.POSITION)) > 0.1f)
    {
        var worldPos = entity.GetValue<Vector3>(EntityNames.POSITION);
        _cachedScreenPosition = Camera.main.WorldToScreenPoint(worldPos);
        _lastWorldPosition = worldPos;
    }
    
    UpdateUI(_cachedScreenPosition.Value);
}
```

### 3. Handle Transform Updates Efficiently

```csharp
public void OnLateUpdate(IEntity entity, float deltaTime)
{
    // Only update transforms when necessary
    var currentPosition = entity.GetValue<Vector3>(EntityNames.POSITION);
    var lastSyncedPosition = entity.GetValue<Vector3>(EntityNames.LAST_SYNCED_POSITION);
    
    if (Vector3.Distance(currentPosition, lastSyncedPosition) > 0.001f)
    {
        SyncTransform(entity, currentPosition);
        entity.SetValue(EntityNames.LAST_SYNCED_POSITION, currentPosition);
    }
}
```

### 4. Batch UI Updates

```csharp
public void OnLateUpdate(IEntity entity, float deltaTime)
{
    // Collect all UI updates into a batch
    var uiUpdates = new UIUpdateBatch();
    
    if (entity.HasTag(EntityTags.HAS_HEALTH_BAR))
    {
        uiUpdates.AddHealthUpdate(entity.GetValue<float>(EntityNames.CURRENT_HEALTH));
    }
    
    if (entity.HasTag(EntityTags.HAS_NAMEPLATE))
    {
        var worldPos = entity.GetValue<Vector3>(EntityNames.POSITION);
        uiUpdates.AddPositionUpdate(Camera.main.WorldToScreenPoint(worldPos));
    }
    
    // Execute batch update
    UIManager.ProcessUpdateBatch(uiUpdates);
}
```

### 5. Error Recovery

```csharp
public void OnLateUpdate(IEntity entity, float deltaTime)
{
    try
    {
        PerformLateUpdateOperations(entity, deltaTime);
    }
    catch (Exception ex)
    {
        Debug.LogError($"LateUpdate error for entity {entity.Name}: {ex.Message}");
        
        // Ensure entity remains in valid state
        ValidateAndRepairEntityState(entity);
    }
}
```

## Common Use Cases

### Camera Systems
- Following player movement
- Updating camera bounds
- Applying camera shake effects

### UI Synchronization
- World-to-screen position updates
- Health bar positioning
- Nameplate updates

### Animation Post-Processing
- IK corrections
- Look-at constraints
- Animation blending

### Physics Interpolation
- Smoothing rigidbody movement
- Visual transform updates
- Trail and effect updates

### Transform Management
- Syncing Unity Transforms
- Parent-child relationships
- Bounds calculations

### Final Validation
- State consistency checks
- Error correction
- Cleanup operations

## Notes

- **Execution Order**: Runs after OnUpdate but before rendering
- **Post-Processing**: Perfect for operations that need final state
- **Performance**: Keep operations lightweight - this runs every frame
- **Transform Sync**: Ideal for synchronizing Unity Transforms with entity state
- **UI Updates**: Best place for world-to-screen UI positioning
- **Order Dependency**: Use when operations depend on other updates completing first