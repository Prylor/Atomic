# 🎨 SceneEntity_Gizmos

The `SceneEntity_Gizmos` partial class provides visual debugging capabilities for `SceneEntity` instances through Unity's gizmo system. It enables entity behaviours to draw debug visualization in the Scene view, with configurable display modes for both edit-time and runtime debugging.

## Key Features

- **Visual Entity Debugging** – Scene view visualization for entities
- **Behaviour-Based Gizmos** – Behaviours can implement `IEntityGizmos` for custom drawing
- **Configurable Display** – Control when and how gizmos are shown
- **Error Resilience** – Safe handling of gizmo drawing exceptions
- **Performance Optimized** – Efficient iteration through entity behaviours
- **Unity Integration** – Full compatibility with Unity's gizmo system

---

## Requirements

This functionality requires:
- `UNITY_EDITOR` compilation symbol
- Available only in Unity Editor environment
- Entity behaviours implementing `IEntityGizmos` interface

## Configuration Properties

### _onlySelectedGizmos
- **Type**: `bool`
- **Default**: `false`
- **Description**: Only draw gizmos when entity is selected
- **Usage**: Reduces visual clutter by showing gizmos only for selected entities

### _onlyEditModeGizmos  
- **Type**: `bool`
- **Default**: `false`
- **Description**: Only draw gizmos in edit mode, not during play
- **Usage**: Prevents gizmo interference during gameplay testing

## Core Methods

### OnDrawGizmos()
```csharp
private void OnDrawGizmos()
```
- Unity callback for drawing gizmos always
- Delegates to `OnDrawGizmosSelected()` unless `_onlySelectedGizmos` is true
- Called automatically by Unity's rendering system

### OnDrawGizmosSelected()
```csharp
private void OnDrawGizmosSelected()
```  
- Unity callback for drawing gizmos when entity is selected
- Respects `_onlyEditModeGizmos` configuration
- Handles exceptions gracefully to prevent editor issues

### ProcessGizmosDraw()
```csharp
protected virtual void ProcessGizmosDraw()
```
- Iterates through all entity behaviours
- Calls `OnGizmosDraw()` on behaviours implementing `IEntityGizmos`
- Virtual method allows for custom gizmo processing in derived classes

## Example Usage

### Basic Gizmo Implementation

```csharp
public class VisualizedPlayerEntity : SceneEntity
{
    [Header("Gizmo Configuration")]
    [SerializeField] private bool showHealthGizmos = true;
    [SerializeField] private bool showMovementGizmos = true;
    [SerializeField] private bool showInteractionRange = true;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Configure gizmo display
        this._onlySelectedGizmos = false; // Always show gizmos
        this._onlyEditModeGizmos = false; // Show in both edit and play mode
        
        // Add values for gizmo visualization
        this.AddValue(EntityNames.MAX_HEALTH, 100f);
        this.AddValue(EntityNames.HEALTH, 80f);
        this.AddValue(EntityNames.MOVE_SPEED, 5f);
        this.AddValue(EntityNames.INTERACTION_RANGE, 2f);
        
        // Add behaviours with gizmo support
        if (showHealthGizmos)
            this.AddBehaviour(new HealthGizmoBehaviour());
            
        if (showMovementGizmos)
            this.AddBehaviour(new MovementGizmoBehaviour());
            
        if (showInteractionRange)
            this.AddBehaviour(new InteractionRangeGizmoBehaviour());
    }
}

// Example behaviour with gizmo drawing
public class HealthGizmoBehaviour : IEntityBehaviour, IEntityGizmos
{
    public void OnInstall(IEntity entity) { }
    
    public void OnGizmosDraw(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        // Get health values
        float health = entity.GetValue<float>(EntityNames.HEALTH);
        float maxHealth = entity.GetValue<float>(EntityNames.MAX_HEALTH);
        float healthPercent = health / maxHealth;
        
        // Draw health bar above entity
        Vector3 position = sceneEntity.transform.position + Vector3.up * 2f;
        DrawHealthBar(position, healthPercent);
    }
    
    private void DrawHealthBar(Vector3 position, float healthPercent)
    {
        float barWidth = 1f;
        float barHeight = 0.1f;
        
        // Background bar
        Gizmos.color = Color.red;
        Gizmos.DrawCube(position, new Vector3(barWidth, barHeight, 0.1f));
        
        // Health bar
        Gizmos.color = Color.green;
        float healthWidth = barWidth * healthPercent;
        Vector3 healthPosition = position - Vector3.right * (barWidth - healthWidth) * 0.5f;
        Gizmos.DrawCube(healthPosition, new Vector3(healthWidth, barHeight, 0.1f));
        
        // Health text
        #if UNITY_EDITOR
        UnityEditor.Handles.Label(position + Vector3.up * 0.2f, 
            $"{(healthPercent * 100):F0}%");
        #endif
    }
}

public class MovementGizmoBehaviour : IEntityBehaviour, IEntityGizmos, IEntityUpdate
{
    private Vector3 lastPosition;
    private Vector3 velocity;
    
    public void OnInstall(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity != null)
            lastPosition = sceneEntity.transform.position;
    }
    
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity != null)
        {
            Vector3 currentPosition = sceneEntity.transform.position;
            velocity = (currentPosition - lastPosition) / deltaTime;
            lastPosition = currentPosition;
        }
    }
    
    public void OnGizmosDraw(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        Vector3 position = sceneEntity.transform.position;
        float speed = entity.GetValue<float>(EntityNames.MOVE_SPEED);
        
        // Draw movement direction
        if (velocity.magnitude > 0.1f)
        {
            Gizmos.color = Color.blue;
            Vector3 direction = velocity.normalized * speed;
            Gizmos.DrawRay(position, direction);
            
            // Draw arrow head
            Vector3 arrowHead = position + direction;
            Gizmos.DrawLine(arrowHead, arrowHead - direction.normalized * 0.3f + Vector3.right * 0.1f);
            Gizmos.DrawLine(arrowHead, arrowHead - direction.normalized * 0.3f - Vector3.right * 0.1f);
        }
        
        // Draw speed circle
        Gizmos.color = new Color(0, 0, 1, 0.3f);
        Gizmos.DrawWireSphere(position, speed);
    }
}

public class InteractionRangeGizmoBehaviour : IEntityBehaviour, IEntityGizmos
{
    public void OnInstall(IEntity entity) { }
    
    public void OnGizmosDraw(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        Vector3 position = sceneEntity.transform.position;
        float range = entity.GetValue<float>(EntityNames.INTERACTION_RANGE);
        
        // Draw interaction range
        Gizmos.color = new Color(1, 1, 0, 0.3f);
        Gizmos.DrawWireSphere(position, range);
        
        // Draw range segments
        Gizmos.color = Color.yellow;
        int segments = 16;
        for (int i = 0; i < segments; i++)
        {
            float angle = i * 2 * Mathf.PI / segments;
            Vector3 point = position + new Vector3(Mathf.Cos(angle), 0, Mathf.Sin(angle)) * range;
            
            if (i > 0)
            {
                float prevAngle = (i - 1) * 2 * Mathf.PI / segments;
                Vector3 prevPoint = position + new Vector3(Mathf.Cos(prevAngle), 0, Mathf.Sin(prevAngle)) * range;
                Gizmos.DrawLine(prevPoint, point);
            }
        }
    }
}
```

### Advanced Gizmo System

```csharp
public class AdvancedGizmoEntity : SceneEntity
{
    [Header("Advanced Gizmo Settings")]
    [SerializeField] private GizmoDisplayMode displayMode = GizmoDisplayMode.Always;
    [SerializeField] private Color primaryGizmoColor = Color.white;
    [SerializeField] private Color secondaryGizmoColor = Color.gray;
    [SerializeField] private bool showDebugInfo = true;
    [SerializeField] private bool showEntityBounds = true;
    
    public enum GizmoDisplayMode
    {
        Always,
        SelectedOnly,
        EditModeOnly,
        PlayModeOnly,
        Never
    }
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Configure gizmo display based on mode
        ConfigureGizmoDisplay();
        
        // Add advanced gizmo behaviours
        this.AddBehaviour(new EntityBoundsGizmoBehaviour());
        this.AddBehaviour(new DebugInfoGizmoBehaviour());
        this.AddBehaviour(new StateVisualizationBehaviour());
        this.AddBehaviour(new ConnectionGizmoBehaviour());
        
        // Store gizmo colors as values
        this.AddValue(EntityNames.PRIMARY_GIZMO_COLOR, primaryGizmoColor);
        this.AddValue(EntityNames.SECONDARY_GIZMO_COLOR, secondaryGizmoColor);
    }
    
    private void ConfigureGizmoDisplay()
    {
        switch (displayMode)
        {
            case GizmoDisplayMode.Always:
                this._onlySelectedGizmos = false;
                this._onlyEditModeGizmos = false;
                break;
                
            case GizmoDisplayMode.SelectedOnly:
                this._onlySelectedGizmos = true;
                this._onlyEditModeGizmos = false;
                break;
                
            case GizmoDisplayMode.EditModeOnly:
                this._onlySelectedGizmos = false;
                this._onlyEditModeGizmos = true;
                break;
                
            case GizmoDisplayMode.PlayModeOnly:
                this._onlySelectedGizmos = false;
                this._onlyEditModeGizmos = false;
                break;
                
            case GizmoDisplayMode.Never:
                // Remove all gizmo behaviours
                this.DelBehaviours<IEntityGizmos>();
                break;
        }
    }
    
    protected override void ProcessGizmosDraw()
    {
        // Skip gizmos based on display mode
        if (displayMode == GizmoDisplayMode.Never) return;
        
        if (displayMode == GizmoDisplayMode.PlayModeOnly && !Application.isPlaying) return;
        
        // Set gizmo matrix to entity transform
        Matrix4x4 oldMatrix = Gizmos.matrix;
        Gizmos.matrix = this.transform.localToWorldMatrix;
        
        try
        {
            base.ProcessGizmosDraw();
        }
        finally
        {
            Gizmos.matrix = oldMatrix;
        }
    }
}

public class EntityBoundsGizmoBehaviour : IEntityBehaviour, IEntityGizmos
{
    public void OnInstall(IEntity entity) { }
    
    public void OnGizmosDraw(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        // Get bounds from collider or renderer
        Bounds bounds = GetEntityBounds(sceneEntity);
        Color primaryColor = entity.GetValue<Color>(EntityNames.PRIMARY_GIZMO_COLOR);
        
        // Draw bounds wireframe
        Gizmos.color = primaryColor;
        Gizmos.DrawWireCube(bounds.center, bounds.size);
        
        // Draw bounds corners
        Vector3[] corners = GetBoundsCorners(bounds);
        Gizmos.color = new Color(primaryColor.r, primaryColor.g, primaryColor.b, 0.5f);
        
        foreach (var corner in corners)
        {
            Gizmos.DrawSphere(corner, 0.05f);
        }
    }
    
    private Bounds GetEntityBounds(SceneEntity entity)
    {
        var collider = entity.GetComponent<Collider>();
        if (collider != null)
            return collider.bounds;
            
        var renderer = entity.GetComponent<Renderer>();
        if (renderer != null)
            return renderer.bounds;
            
        // Default bounds
        return new Bounds(entity.transform.position, Vector3.one);
    }
    
    private Vector3[] GetBoundsCorners(Bounds bounds)
    {
        Vector3 center = bounds.center;
        Vector3 extents = bounds.extents;
        
        return new Vector3[]
        {
            center + new Vector3(-extents.x, -extents.y, -extents.z),
            center + new Vector3(+extents.x, -extents.y, -extents.z),
            center + new Vector3(-extents.x, +extents.y, -extents.z),
            center + new Vector3(+extents.x, +extents.y, -extents.z),
            center + new Vector3(-extents.x, -extents.y, +extents.z),
            center + new Vector3(+extents.x, -extents.y, +extents.z),
            center + new Vector3(-extents.x, +extents.y, +extents.z),
            center + new Vector3(+extents.x, +extents.y, +extents.z)
        };
    }
}

public class DebugInfoGizmoBehaviour : IEntityBehaviour, IEntityGizmos
{
    public void OnInstall(IEntity entity) { }
    
    public void OnGizmosDraw(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        #if UNITY_EDITOR
        Vector3 position = sceneEntity.transform.position;
        
        // Draw debug info text
        string debugInfo = $"Entity: {entity.Name}\n";
        debugInfo += $"ID: {entity.InstanceID}\n";
        debugInfo += $"Tags: {entity.TagCount}\n";
        debugInfo += $"Values: {entity.ValueCount}\n";
        debugInfo += $"Behaviours: {entity.BehaviourCount}\n";
        debugInfo += $"Spawned: {entity.Spawned}\n";
        debugInfo += $"Enabled: {entity.Enabled}";
        
        UnityEditor.Handles.Label(position + Vector3.up * 3f, debugInfo);
        
        // Draw state indicators
        DrawStateIndicators(entity, position);
        #endif
    }
    
    #if UNITY_EDITOR
    private void DrawStateIndicators(IEntity entity, Vector3 position)
    {
        float indicatorSize = 0.2f;
        Vector3 baseOffset = Vector3.up * 2.5f;
        
        // Spawned indicator
        Gizmos.color = entity.Spawned ? Color.green : Color.red;
        Gizmos.DrawSphere(position + baseOffset + Vector3.left * 0.5f, indicatorSize);
        UnityEditor.Handles.Label(position + baseOffset + Vector3.left * 0.5f + Vector3.up * 0.3f, "S");
        
        // Enabled indicator
        Gizmos.color = entity.Enabled ? Color.green : Color.red;
        Gizmos.DrawSphere(position + baseOffset, indicatorSize);
        UnityEditor.Handles.Label(position + baseOffset + Vector3.up * 0.3f, "E");
        
        // Installed indicator
        Gizmos.color = entity.Installed ? Color.green : Color.red;
        Gizmos.DrawSphere(position + baseOffset + Vector3.right * 0.5f, indicatorSize);
        UnityEditor.Handles.Label(position + baseOffset + Vector3.right * 0.5f + Vector3.up * 0.3f, "I");
    }
    #endif
}
```

### Gizmo Manager System

```csharp
[System.Serializable]
public class GizmoSettings
{
    [Header("Display Settings")]
    public bool enableGizmos = true;
    public bool showInPlayMode = true;
    public bool showWhenSelected = false;
    
    [Header("Visual Settings")]
    public Color primaryColor = Color.white;
    public Color secondaryColor = Color.gray;
    public float gizmoScale = 1f;
    public float alpha = 1f;
    
    [Header("Performance Settings")]
    public int maxGizmosPerFrame = 100;
    public float updateFrequency = 30f; // Hz
}

public class EntityGizmoManager : MonoBehaviour
{
    [Header("Global Gizmo Settings")]
    [SerializeField] private GizmoSettings globalSettings = new GizmoSettings();
    
    [Header("Entity Categories")]
    [SerializeField] private GizmoSettings playerSettings = new GizmoSettings();
    [SerializeField] private GizmoSettings enemySettings = new GizmoSettings();
    [SerializeField] private GizmoSettings npcSettings = new GizmoSettings();
    
    private static EntityGizmoManager instance;
    private Dictionary<SceneEntity, GizmoSettings> entitySettings = new Dictionary<SceneEntity, GizmoSettings>();
    
    public static EntityGizmoManager Instance => instance;
    
    void Awake()
    {
        instance = this;
    }
    
    void Start()
    {
        // Apply settings to existing entities
        var allEntities = FindObjectsOfType<SceneEntity>();
        foreach (var entity in allEntities)
        {
            ApplyGizmoSettings(entity);
        }
    }
    
    public void ApplyGizmoSettings(SceneEntity entity)
    {
        GizmoSettings settings = GetSettingsForEntity(entity);
        
        // Configure entity gizmo behavior
        entity._onlySelectedGizmos = settings.showWhenSelected;
        entity._onlyEditModeGizmos = !settings.showInPlayMode;
        
        // Store settings for gizmo behaviors to use
        entity.SetValue(EntityNames.GIZMO_SETTINGS, settings);
        
        entitySettings[entity] = settings;
    }
    
    private GizmoSettings GetSettingsForEntity(SceneEntity entity)
    {
        if (entity.HasTag(EntityTags.PLAYER))
            return playerSettings;
        else if (entity.HasTag(EntityTags.ENEMY))
            return enemySettings;
        else if (entity.HasTag(EntityTags.NPC))
            return npcSettings;
        else
            return globalSettings;
    }
    
    public GizmoSettings GetEntitySettings(SceneEntity entity)
    {
        return entitySettings.TryGetValue(entity, out var settings) ? settings : globalSettings;
    }
    
    [ContextMenu("Refresh All Entity Gizmos")]
    public void RefreshAllEntityGizmos()
    {
        var allEntities = FindObjectsOfType<SceneEntity>();
        foreach (var entity in allEntities)
        {
            ApplyGizmoSettings(entity);
        }
        
        Debug.Log($"Refreshed gizmo settings for {allEntities.Length} entities");
    }
    
    [ContextMenu("Disable All Gizmos")]
    public void DisableAllGizmos()
    {
        var allEntities = FindObjectsOfType<SceneEntity>();
        foreach (var entity in allEntities)
        {
            entity._onlySelectedGizmos = true;
            entity._onlyEditModeGizmos = true;
        }
        
        Debug.Log("Disabled gizmos for all entities");
    }
}
```

### Custom Gizmo Behaviours

```csharp
public class PathGizmoBehaviour : IEntityBehaviour, IEntityGizmos
{
    private List<Vector3> pathPoints = new List<Vector3>();
    private int currentPointIndex = 0;
    
    public void OnInstall(IEntity entity)
    {
        // Initialize path from waypoints or patrol points
        InitializePath(entity);
    }
    
    public void OnGizmosDraw(IEntity entity)
    {
        if (pathPoints.Count < 2) return;
        
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        var settings = EntityGizmoManager.Instance?.GetEntitySettings(sceneEntity) ?? new GizmoSettings();
        
        // Draw path lines
        Gizmos.color = new Color(settings.primaryColor.r, settings.primaryColor.g, settings.primaryColor.b, 0.7f);
        
        for (int i = 0; i < pathPoints.Count - 1; i++)
        {
            Gizmos.DrawLine(pathPoints[i], pathPoints[i + 1]);
        }
        
        // Draw waypoints
        Gizmos.color = settings.primaryColor;
        for (int i = 0; i < pathPoints.Count; i++)
        {
            float size = (i == currentPointIndex) ? 0.3f : 0.2f;
            Gizmos.DrawSphere(pathPoints[i], size * settings.gizmoScale);
            
            #if UNITY_EDITOR
            UnityEditor.Handles.Label(pathPoints[i] + Vector3.up * 0.5f, $"P{i}");
            #endif
        }
        
        // Draw progress line
        if (currentPointIndex < pathPoints.Count)
        {
            Gizmos.color = Color.yellow;
            Gizmos.DrawLine(sceneEntity.transform.position, pathPoints[currentPointIndex]);
        }
    }
    
    private void InitializePath(IEntity entity)
    {
        // Get path from entity values or generate default path
        if (entity.HasValue(EntityNames.PATH_POINTS))
        {
            pathPoints = entity.GetValue<List<Vector3>>(EntityNames.PATH_POINTS);
        }
        else
        {
            // Generate default patrol path
            var sceneEntity = entity as SceneEntity;
            if (sceneEntity != null)
            {
                Vector3 center = sceneEntity.transform.position;
                pathPoints.Add(center + Vector3.forward * 2);
                pathPoints.Add(center + Vector3.right * 2);
                pathPoints.Add(center + Vector3.back * 2);
                pathPoints.Add(center + Vector3.left * 2);
            }
        }
    }
    
    public void SetCurrentPoint(int index)
    {
        currentPointIndex = Mathf.Clamp(index, 0, pathPoints.Count - 1);
    }
}

public class FieldOfViewGizmoBehaviour : IEntityBehaviour, IEntityGizmos
{
    private float viewDistance = 5f;
    private float viewAngle = 60f;
    
    public void OnInstall(IEntity entity)
    {
        viewDistance = entity.GetValue<float>(EntityNames.VIEW_DISTANCE, 5f);
        viewAngle = entity.GetValue<float>(EntityNames.VIEW_ANGLE, 60f);
    }
    
    public void OnGizmosDraw(IEntity entity)
    {
        var sceneEntity = entity as SceneEntity;
        if (sceneEntity == null) return;
        
        Vector3 position = sceneEntity.transform.position;
        Vector3 forward = sceneEntity.transform.forward;
        
        var settings = EntityGizmoManager.Instance?.GetEntitySettings(sceneEntity) ?? new GizmoSettings();
        
        // Draw field of view cone
        Gizmos.color = new Color(settings.secondaryColor.r, settings.secondaryColor.g, settings.secondaryColor.b, 0.3f);
        
        // Calculate FOV boundaries
        Vector3 rightBoundary = Quaternion.Euler(0, viewAngle * 0.5f, 0) * forward * viewDistance;
        Vector3 leftBoundary = Quaternion.Euler(0, -viewAngle * 0.5f, 0) * forward * viewDistance;
        
        // Draw FOV lines
        Gizmos.DrawRay(position, rightBoundary);
        Gizmos.DrawRay(position, leftBoundary);
        Gizmos.DrawRay(position, forward * viewDistance);
        
        // Draw FOV arc
        int arcSegments = 20;
        Vector3 prevPoint = position + rightBoundary;
        
        for (int i = 1; i <= arcSegments; i++)
        {
            float angle = Mathf.Lerp(viewAngle * 0.5f, -viewAngle * 0.5f, (float)i / arcSegments);
            Vector3 direction = Quaternion.Euler(0, angle, 0) * forward * viewDistance;
            Vector3 point = position + direction;
            
            Gizmos.DrawLine(prevPoint, point);
            prevPoint = point;
        }
        
        #if UNITY_EDITOR
        // Draw FOV info
        UnityEditor.Handles.Label(position + forward * viewDistance * 0.5f + Vector3.up, 
            $"FOV: {viewAngle:F0}°\nRange: {viewDistance:F1}m");
        #endif
    }
}
```

## Best Practices

1. **Performance Awareness** – Gizmos can impact Scene view performance with many entities
2. **Selective Display** – Use configuration options to control when gizmos are shown
3. **Error Handling** – Always wrap gizmo code in try-catch to prevent editor issues
4. **Visual Clarity** – Use appropriate colors and sizes for gizmo elements
5. **Conditional Drawing** – Check relevant conditions before drawing expensive gizmos
6. **Unity Guidelines** – Follow Unity's gizmo drawing conventions and best practices

## Configuration Strategies

### Performance-First Configuration
```csharp
// Only show gizmos when selected and in edit mode
entity._onlySelectedGizmos = true;
entity._onlyEditModeGizmos = true;
```

### Development Configuration
```csharp
// Always show gizmos for development debugging
entity._onlySelectedGizmos = false;
entity._onlyEditModeGizmos = false;
```

### Runtime Testing Configuration
```csharp
// Show gizmos during play mode testing
entity._onlySelectedGizmos = false;
entity._onlyEditModeGizmos = false;
```

## Integration with Unity Systems

### Scene View Integration
- Gizmos appear automatically in Unity's Scene view
- Respect Unity's gizmo visibility settings
- Support for both always-visible and selection-based gizmos

### Performance Considerations
- Gizmo drawing can be expensive for complex visualizations
- Use LOD-style approaches for distant entities
- Consider frame rate impact with many entities

### Editor Integration
- Works seamlessly with Unity's inspector
- Supports custom editor scripts for gizmo controls
- Integration with Unity's scene rendering pipeline

## Common Gizmo Patterns

### Debug Visualization
- Health bars, status indicators
- Movement paths and velocities
- Interaction ranges and boundaries

### Development Tools
- Entity state visualization
- Behaviour debugging displays
- Performance monitoring gizmos

### Design Assistance
- Level design helpers
- Placement guides and measurements
- Visual feedback for game mechanics

## Notes

- Gizmo functionality is editor-only and has no runtime cost in builds
- Exception handling prevents gizmo errors from breaking the editor
- Virtual `ProcessGizmosDraw()` allows custom gizmo processing in derived classes
- Configuration properties are serialized for persistent gizmo behavior
- Integration with `IEntityGizmos` provides standardized gizmo interface
- Supports both Unity's `OnDrawGizmos` and `OnDrawGizmosSelected` callbacks