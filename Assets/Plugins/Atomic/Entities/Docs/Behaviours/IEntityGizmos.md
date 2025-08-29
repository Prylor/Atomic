# 🧩 IEntityGizmos

The `IEntityGizmos` interface defines behaviours that handle debug visualization and gizmo drawing for entities in the Unity Editor. This interface is called automatically during the Unity gizmo rendering phase, making it perfect for debugging entity state, visualizing ranges, and displaying editor-only information.

## Key Features

- **Editor-Only Rendering** – Only available in Unity Editor, automatically excluded from builds
- **Debug Visualization** – Perfect for visualizing entity data, ranges, and states
- **Automatic Invocation** – Called during Unity's gizmo rendering phase
- **Generic Support** – Includes strongly-typed generic version for specific entity types
- **Scene View Integration** – Draws directly in the Unity Scene view
- **Conditional Compilation** – Automatically disabled in builds for performance

---

## Interface Definition

```csharp
#if UNITY_5_3_OR_NEWER
public interface IEntityGizmos : IEntityBehaviour
{
    void OnGizmosDraw(IEntity entity);
}

public interface IEntityGizmos<in T> : IEntityGizmos where T : IEntity
{
    void OnGizmosDraw(T entity);
}
#endif
```

## When It's Called

The `OnGizmosDraw` method is automatically invoked:
- During Unity's `OnDrawGizmos()` phase in the Editor
- When `OnDrawGizmosSelected()` is called for selected entities
- Only in the Unity Editor (excluded from builds)
- For both selected and unselected entities (depending on implementation)
- During Scene view rendering

This interface is only available when `UNITY_5_3_OR_NEWER` is defined and is automatically excluded from builds.

## Example Implementations

### Movement Range Visualization

```csharp
#if UNITY_5_3_OR_NEWER
public class MovementRangeGizmoBehaviour : IEntityGizmos
{
    public void OnGizmosDraw(IEntity entity)
    {
        // Draw movement range for AI entities
        if (!entity.HasTag(EntityTags.AI_ENABLED)) return;
        
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        var movementRange = entity.GetValue<float>(EntityNames.MOVEMENT_RANGE);
        var patrolCenter = entity.GetValue<Vector3>(EntityNames.PATROL_CENTER);
        
        // Draw patrol area
        Gizmos.color = Color.yellow;
        Gizmos.DrawWireSphere(patrolCenter, movementRange);
        
        // Draw line from entity to patrol center
        Gizmos.color = Color.blue;
        Gizmos.DrawLine(position, patrolCenter);
        
        // Draw movement path if available
        if (entity.TryGetValue<Vector3[]>(EntityNames.PATROL_POINTS, out var patrolPoints))
        {
            DrawPatrolPath(patrolPoints);
        }
    }
    
    private void DrawPatrolPath(Vector3[] points)
    {
        if (points.Length < 2) return;
        
        Gizmos.color = Color.green;
        for (int i = 0; i < points.Length; i++)
        {
            // Draw point
            Gizmos.DrawSphere(points[i], 0.2f);
            
            // Draw line to next point
            int nextIndex = (i + 1) % points.Length;
            Gizmos.DrawLine(points[i], points[nextIndex]);
        }
    }
}
#endif
```

### Combat System Visualization

```csharp
#if UNITY_5_3_OR_NEWER
public class CombatGizmoBehaviour : IEntityGizmos
{
    public void OnGizmosDraw(IEntity entity)
    {
        if (!entity.HasTag(EntityTags.COMBATANT)) return;
        
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        var forward = entity.GetValue<Vector3>(EntityNames.FORWARD_DIRECTION);
        
        // Draw attack range
        DrawAttackRange(entity, position);
        
        // Draw detection cone
        DrawDetectionCone(entity, position, forward);
        
        // Draw health status
        DrawHealthBar(entity, position);
        
        // Draw current target line
        DrawTargetLine(entity, position);
    }
    
    private void DrawAttackRange(IEntity entity, Vector3 position)
    {
        var attackRange = entity.GetValue<float>(EntityNames.ATTACK_RANGE);
        
        if (entity.HasTag(EntityTags.IN_COMBAT))
        {
            Gizmos.color = Color.red;
        }
        else
        {
            Gizmos.color = new Color(1f, 0f, 0f, 0.3f);
        }
        
        Gizmos.DrawWireSphere(position, attackRange);
    }
    
    private void DrawDetectionCone(IEntity entity, Vector3 position, Vector3 forward)
    {
        var detectionRange = entity.GetValue<float>(EntityNames.DETECTION_RANGE);
        var detectionAngle = entity.GetValue<float>(EntityNames.DETECTION_ANGLE);
        
        Gizmos.color = Color.cyan;
        
        // Draw detection cone
        var halfAngle = detectionAngle * 0.5f;
        var leftDirection = Quaternion.Euler(0, -halfAngle, 0) * forward;
        var rightDirection = Quaternion.Euler(0, halfAngle, 0) * forward;
        
        Gizmos.DrawRay(position, leftDirection * detectionRange);
        Gizmos.DrawRay(position, rightDirection * detectionRange);
        Gizmos.DrawRay(position, forward * detectionRange);
        
        // Draw arc
        DrawArc(position, forward, detectionAngle, detectionRange);
    }
    
    private void DrawHealthBar(IEntity entity, Vector3 position)
    {
        var currentHealth = entity.GetValue<float>(EntityNames.CURRENT_HEALTH);
        var maxHealth = entity.GetValue<float>(EntityNames.MAX_HEALTH);
        var healthPercent = currentHealth / maxHealth;
        
        var barPosition = position + Vector3.up * 2.5f;
        var barWidth = 2f;
        var barHeight = 0.2f;
        
        // Background
        Gizmos.color = Color.black;
        Gizmos.DrawCube(barPosition, new Vector3(barWidth, barHeight, 0.1f));
        
        // Health fill
        var healthColor = Color.Lerp(Color.red, Color.green, healthPercent);
        Gizmos.color = healthColor;
        
        var fillWidth = barWidth * healthPercent;
        var fillPosition = barPosition + Vector3.left * (barWidth - fillWidth) * 0.5f;
        Gizmos.DrawCube(fillPosition, new Vector3(fillWidth, barHeight * 0.8f, 0.1f));
    }
}
#endif
```

### Physics Debug Visualization

```csharp
#if UNITY_5_3_OR_NEWER
public class PhysicsDebugGizmoBehaviour : IEntityGizmos
{
    public void OnGizmosDraw(IEntity entity)
    {
        if (!entity.HasTag(EntityTags.PHYSICS_ENABLED)) return;
        
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        var velocity = entity.GetValue<Vector3>(EntityNames.VELOCITY);
        var acceleration = entity.GetValue<Vector3>(EntityNames.ACCELERATION);
        
        // Draw velocity vector
        if (velocity.magnitude > 0.1f)
        {
            Gizmos.color = Color.blue;
            Gizmos.DrawRay(position, velocity);
            
            // Draw velocity magnitude as sphere size
            var speedIndicatorSize = Mathf.Clamp(velocity.magnitude * 0.1f, 0.1f, 1f);
            Gizmos.DrawWireSphere(position + velocity.normalized * 2f, speedIndicatorSize);
        }
        
        // Draw acceleration vector
        if (acceleration.magnitude > 0.1f)
        {
            Gizmos.color = Color.red;
            Gizmos.DrawRay(position, acceleration * 5f); // Scale for visibility
        }
        
        // Draw collision bounds
        DrawCollisionBounds(entity, position);
        
        // Draw ground check
        DrawGroundCheck(entity, position);
    }
    
    private void DrawCollisionBounds(IEntity entity, Vector3 position)
    {
        if (!entity.TryGetValue<Collider>(EntityNames.COLLIDER, out var collider)) return;
        
        Gizmos.color = Color.green;
        
        switch (collider)
        {
            case BoxCollider box:
                var boxMatrix = Matrix4x4.TRS(box.transform.position, box.transform.rotation, box.transform.lossyScale);
                Gizmos.matrix = boxMatrix;
                Gizmos.DrawWireCube(box.center, box.size);
                Gizmos.matrix = Matrix4x4.identity;
                break;
                
            case SphereCollider sphere:
                Gizmos.DrawWireSphere(sphere.transform.position + sphere.center, 
                    sphere.radius * sphere.transform.lossyScale.x);
                break;
                
            case CapsuleCollider capsule:
                DrawWireCapsule(capsule.transform.position + capsule.center, 
                    capsule.radius * capsule.transform.lossyScale.x, 
                    capsule.height * capsule.transform.lossyScale.y, 
                    capsule.direction);
                break;
        }
    }
}
#endif
```

### AI State Visualization

```csharp
#if UNITY_5_3_OR_NEWER
public class AIStateGizmoBehaviour : IEntityGizmos
{
    public void OnGizmosDraw(IEntity entity)
    {
        if (!entity.HasTag(EntityTags.AI_ENABLED)) return;
        
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        var aiState = entity.GetValue<AIState>(EntityNames.AI_STATE);
        
        // Draw state-specific gizmos
        switch (aiState)
        {
            case AIState.Idle:
                DrawIdleState(entity, position);
                break;
            case AIState.Patrol:
                DrawPatrolState(entity, position);
                break;
            case AIState.Chase:
                DrawChaseState(entity, position);
                break;
            case AIState.Attack:
                DrawAttackState(entity, position);
                break;
            case AIState.Flee:
                DrawFleeState(entity, position);
                break;
        }
        
        // Draw state name
        DrawStateLabel(position, aiState.ToString());
    }
    
    private void DrawIdleState(IEntity entity, Vector3 position)
    {
        // Draw as calm blue sphere
        Gizmos.color = Color.blue;
        Gizmos.DrawWireSphere(position + Vector3.up * 3f, 0.5f);
    }
    
    private void DrawChaseState(IEntity entity, Vector3 position)
    {
        // Draw target line and pursuit indicator
        if (entity.TryGetValue<IEntity>(EntityNames.AI_TARGET, out var target))
        {
            var targetPosition = target.GetValue<Vector3>(EntityNames.POSITION);
            
            Gizmos.color = Color.red;
            Gizmos.DrawLine(position, targetPosition);
            
            // Draw pursuit arrow
            var direction = (targetPosition - position).normalized;
            DrawArrow(position + Vector3.up * 1f, direction * 2f, Color.red);
        }
        
        // Pulsing red sphere
        var pulseSize = 0.5f + Mathf.Sin(Time.time * 5f) * 0.2f;
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(position + Vector3.up * 3f, pulseSize);
    }
    
    private void DrawStateLabel(Vector3 position, string text)
    {
        #if UNITY_EDITOR
        UnityEditor.Handles.Label(position + Vector3.up * 4f, text);
        #endif
    }
}
#endif
```

### Generic Player Gizmo Behaviour

```csharp
#if UNITY_5_3_OR_NEWER
public class PlayerGizmoBehaviour : IEntityGizmos<PlayerEntity>
{
    public void OnGizmosDraw(PlayerEntity entity)
    {
        var position = entity.GetValue<Vector3>(EntityNames.POSITION);
        
        // Draw player-specific visualizations
        DrawPlayerStats(entity, position);
        DrawInventoryIndicator(entity, position);
        DrawQuestMarkers(entity, position);
        DrawInteractionRange(entity, position);
    }
    
    private void DrawPlayerStats(PlayerEntity entity, Vector3 position)
    {
        var level = entity.GetPlayerLevel();
        var experience = entity.GetExperience();
        var expToNext = entity.GetExperienceToNextLevel();
        
        // Draw level indicator
        Gizmos.color = Color.gold;
        for (int i = 0; i < level; i++)
        {
            var starPos = position + Vector3.up * 4f + Vector3.right * (i * 0.3f - level * 0.15f);
            DrawStar(starPos, 0.1f);
        }
        
        // Draw experience bar
        var expPercent = experience / (float)expToNext;
        DrawProgressBar(position + Vector3.up * 3.5f, expPercent, Color.cyan, 2f);
    }
    
    private void DrawInteractionRange(PlayerEntity entity, Vector3 position)
    {
        var interactionRange = entity.GetInteractionRange();
        
        Gizmos.color = new Color(0f, 1f, 0f, 0.2f);
        Gizmos.DrawSphere(position, interactionRange);
        
        Gizmos.color = Color.green;
        Gizmos.DrawWireSphere(position, interactionRange);
    }
    
    private void DrawQuestMarkers(PlayerEntity entity, Vector3 position)
    {
        var activeQuests = entity.GetActiveQuests();
        
        foreach (var quest in activeQuests)
        {
            if (quest.HasTargetLocation())
            {
                var targetLocation = quest.GetTargetLocation();
                
                // Draw quest line
                Gizmos.color = Color.yellow;
                Gizmos.DrawLine(position + Vector3.up, targetLocation + Vector3.up);
                
                // Draw quest marker
                Gizmos.color = Color.yellow;
                Gizmos.DrawSphere(targetLocation + Vector3.up * 2f, 0.3f);
            }
        }
    }
}
#endif
```

### Utility Drawing Methods

```csharp
#if UNITY_5_3_OR_NEWER
public static class GizmoUtilities
{
    public static void DrawArrow(Vector3 position, Vector3 direction, Color color)
    {
        Gizmos.color = color;
        Gizmos.DrawRay(position, direction);
        
        var arrowHeadLength = direction.magnitude * 0.2f;
        var arrowHeadAngle = 20f;
        
        var right = Quaternion.LookRotation(direction) * Quaternion.Euler(0, 180 + arrowHeadAngle, 0) * Vector3.forward;
        var left = Quaternion.LookRotation(direction) * Quaternion.Euler(0, 180 - arrowHeadAngle, 0) * Vector3.forward;
        
        var arrowTip = position + direction;
        Gizmos.DrawRay(arrowTip, right * arrowHeadLength);
        Gizmos.DrawRay(arrowTip, left * arrowHeadLength);
    }
    
    public static void DrawArc(Vector3 center, Vector3 forward, float angle, float radius)
    {
        var steps = Mathf.CeilToInt(angle / 5f);
        var stepAngle = angle / steps;
        var startAngle = -angle * 0.5f;
        
        var previousPoint = center + Quaternion.Euler(0, startAngle, 0) * forward * radius;
        
        for (int i = 1; i <= steps; i++)
        {
            var currentAngle = startAngle + stepAngle * i;
            var currentPoint = center + Quaternion.Euler(0, currentAngle, 0) * forward * radius;
            
            Gizmos.DrawLine(previousPoint, currentPoint);
            previousPoint = currentPoint;
        }
    }
    
    public static void DrawProgressBar(Vector3 position, float progress, Color fillColor, float width)
    {
        var height = 0.2f;
        
        // Background
        Gizmos.color = Color.black;
        Gizmos.DrawCube(position, new Vector3(width, height, 0.1f));
        
        // Fill
        Gizmos.color = fillColor;
        var fillWidth = width * progress;
        var fillPosition = position + Vector3.left * (width - fillWidth) * 0.5f;
        Gizmos.DrawCube(fillPosition, new Vector3(fillWidth, height * 0.8f, 0.1f));
    }
    
    public static void DrawWireCapsule(Vector3 position, float radius, float height, int direction)
    {
        var half = height * 0.5f - radius;
        
        switch (direction)
        {
            case 0: // X-axis
                Gizmos.DrawWireSphere(position + Vector3.right * half, radius);
                Gizmos.DrawWireSphere(position + Vector3.left * half, radius);
                break;
            case 1: // Y-axis
                Gizmos.DrawWireSphere(position + Vector3.up * half, radius);
                Gizmos.DrawWireSphere(position + Vector3.down * half, radius);
                break;
            case 2: // Z-axis
                Gizmos.DrawWireSphere(position + Vector3.forward * half, radius);
                Gizmos.DrawWireSphere(position + Vector3.back * half, radius);
                break;
        }
    }
}
#endif
```

## Best Practices

### 1. Use Color Coding

```csharp
public void OnGizmosDraw(IEntity entity)
{
    // Consistent color scheme
    if (entity.HasTag(EntityTags.HOSTILE))
    {
        Gizmos.color = Color.red;
    }
    else if (entity.HasTag(EntityTags.FRIENDLY))
    {
        Gizmos.color = Color.green;
    }
    else
    {
        Gizmos.color = Color.yellow;
    }
    
    DrawEntityGizmo(entity);
}
```

### 2. Conditional Rendering

```csharp
public void OnGizmosDraw(IEntity entity)
{
    // Only draw for specific entity types
    if (!entity.HasTag(EntityTags.DEBUG_VISUALIZE)) return;
    
    // Only draw when entity is selected in hierarchy
    #if UNITY_EDITOR
    if (UnityEditor.Selection.activeGameObject != entity.GetValue<GameObject>(EntityNames.GAME_OBJECT))
        return;
    #endif
    
    DrawDebugVisualization(entity);
}
```

### 3. Performance Considerations

```csharp
public void OnGizmosDraw(IEntity entity)
{
    // Avoid expensive calculations in gizmo drawing
    var cachedPosition = entity.GetValue<Vector3>(EntityNames.CACHED_POSITION);
    var lastUpdate = entity.GetValue<float>(EntityNames.LAST_GIZMO_UPDATE);
    
    if (Time.time - lastUpdate > 0.1f) // Update cache every 100ms
    {
        cachedPosition = CalculateExpensivePosition(entity);
        entity.SetValue(EntityNames.CACHED_POSITION, cachedPosition);
        entity.SetValue(EntityNames.LAST_GIZMO_UPDATE, Time.time);
    }
    
    Gizmos.DrawSphere(cachedPosition, 0.5f);
}
```

### 4. Layer-Based Visibility

```csharp
public void OnGizmosDraw(IEntity entity)
{
    // Different gizmo layers for different information
    if (ShowMovementGizmos)
    {
        DrawMovementVisualization(entity);
    }
    
    if (ShowCombatGizmos)
    {
        DrawCombatVisualization(entity);
    }
    
    if (ShowPhysicsGizmos)
    {
        DrawPhysicsVisualization(entity);
    }
}
```

## Common Use Cases

### Debug Visualization
- Entity state representation
- Value visualization
- State machine debugging

### Range Indicators
- Attack ranges
- Detection areas
- Interaction zones

### Pathfinding Debug
- Movement paths
- Waypoints
- Navigation mesh

### Physics Debugging
- Collision bounds
- Force vectors
- Velocity indicators

### AI Visualization
- Behavior states
- Target tracking
- Decision trees

### Level Design Tools
- Spawn points
- Trigger areas
- Gameplay zones

## Notes

- **Editor Only**: Automatically excluded from builds using conditional compilation
- **Performance**: Minimize expensive calculations in gizmo drawing
- **Visual Clarity**: Use consistent colors and shapes for similar concepts
- **Selective Display**: Consider adding toggles for different gizmo types
- **Unity Integration**: Works with Unity's built-in gizmo system
- **Debug Tool**: Perfect for debugging entity behavior and state visualization