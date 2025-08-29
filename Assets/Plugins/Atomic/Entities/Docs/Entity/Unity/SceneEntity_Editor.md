# ⚡ SceneEntity_Editor

The `SceneEntity_Editor` partial class provides comprehensive editor-time support for `SceneEntity` instances in the Unity Editor. It enables edit-mode installation, automatic refresh capabilities, simulated lifecycle events, and development workflow optimizations for entity authoring and testing.

## Key Features

- **Edit-Mode Installation** – Install and preview entities in Edit Mode
- **Automatic Refresh** – OnValidate-driven updates when properties change
- **Simulated Lifecycle** – Full entity lifecycle simulation in editor
- **Auto-Discovery** – Automatic detection of installers and child entities
- **Compilation System** – Complete entity compilation and precompilation
- **Development Tools** – Editor buttons and context menu integration

---

## Requirements

This functionality requires:
- `UNITY_EDITOR` compilation symbol
- Available only in Unity Editor environment
- Optional Odin Inspector integration for enhanced UI

## Core Editor Methods

### Reset Method
```csharp
private void Reset()
```
- Automatically gathers installers and child entities
- Called when component is added or reset
- Populates installer and children lists automatically
- Removes self from children list to prevent recursion

### OnValidate Method  
```csharp
private void OnValidate()
```
- Automatically refreshes entity when properties change
- Only active when `installInEditMode` is enabled
- Skipped during play mode or compilation
- Sets up refresh callbacks on installers

### Compile Method
```csharp
private void Compile()
```
- Complete entity compilation for editor preview
- Simulates full entity lifecycle
- Handles prefab vs scene instance differences
- Provides detailed success/error logging

## Editor Lifecycle Simulation

### Initialization Simulation
```csharp
private void InitInEditMode()
```
- Simulates `OnSpawn()` for edit-mode compatible behaviours
- Respects `RunInEditModeAttribute` on behaviours
- Only runs if entity not already spawned

### Activation Simulation
```csharp
private void EnableInEditMode()
```
- Simulates `OnActivate()` for edit-mode compatible behaviours
- Respects `RunInEditModeAttribute` on behaviours
- Only runs if entity not already active

### Deactivation Simulation
```csharp
private void DisableInEditMode()
```
- Simulates `OnDeactivate()` for edit-mode compatible behaviours
- Proper cleanup before recompilation
- Only runs if entity is currently active

### Disposal Simulation
```csharp
private void DisposeInEditMode()
```
- Simulates `OnDespawn()` for edit-mode compatible behaviours
- Ensures proper cleanup of editor state
- Only runs if entity is currently spawned

## Example Usage

### Basic Editor Integration

```csharp
[AddComponentMenu("Game/Editable Player Entity")]
public class EditablePlayerEntity : SceneEntity
{
    [Header("Edit-Mode Configuration")]
    [SerializeField] private bool enableEditPreview = true;
    
    [Header("Player Settings")]
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private int maxHealth = 100;
    [SerializeField] private Color playerColor = Color.blue;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Configure installInEditMode based on preview setting
        this.installInEditMode = enableEditPreview;
        
        // Add values that work in both edit and play modes
        this.AddValue(EntityNames.MOVE_SPEED, moveSpeed);
        this.AddValue(EntityNames.MAX_HEALTH, maxHealth);
        this.AddValue(EntityNames.PLAYER_COLOR, playerColor);
        
        // Add edit-mode compatible behaviours
        this.AddBehaviour(new EditModeVisualizationBehaviour());
        this.AddBehaviour(new PropertySyncBehaviour());
        
        // Add runtime-only behaviours with proper attributes
        this.AddBehaviour(new PlayerInputBehaviour()); // Has RunInEditMode attribute
        this.AddBehaviour(new RuntimeOnlyBehaviour()); // Doesn't run in edit mode
    }
    
    // This will be called when properties change in inspector
    protected override void OnValidate()
    {
        base.OnValidate();
        
        // Update values when serialized fields change
        if (this.Installed)
        {
            this.SetValue(EntityNames.MOVE_SPEED, moveSpeed);
            this.SetValue(EntityNames.MAX_HEALTH, maxHealth);
            this.SetValue(EntityNames.PLAYER_COLOR, playerColor);
            
            // Update visual representation in edit mode
            UpdateEditModeVisualization();
        }
    }
    
    private void UpdateEditModeVisualization()
    {
        if (!Application.isPlaying)
        {
            var renderer = GetComponent<Renderer>();
            if (renderer != null)
            {
                renderer.sharedMaterial.color = playerColor;
            }
        }
    }
}

// Example of edit-mode compatible behaviour
[RunInEditMode]
public class EditModeVisualizationBehaviour : IEntityBehaviour, IEntityActivate
{
    public void OnInstall(IEntity entity) { }
    
    public void OnActivate(IEntity entity)
    {
        if (!Application.isPlaying)
        {
            // Safe edit-mode visualization
            var sceneEntity = entity as SceneEntity;
            if (sceneEntity != null)
            {
                var color = sceneEntity.GetValue<Color>(EntityNames.PLAYER_COLOR);
                UpdateVisualization(sceneEntity, color);
            }
        }
    }
    
    private void UpdateVisualization(SceneEntity entity, Color color)
    {
        var renderer = entity.GetComponent<Renderer>();
        if (renderer != null && renderer.sharedMaterial != null)
        {
            // Create temporary material for edit mode
            var material = new Material(renderer.sharedMaterial);
            material.color = color;
            renderer.sharedMaterial = material;
        }
    }
}
```

### Advanced Editor Workflow

```csharp
public class AdvancedEditableEntity : SceneEntity
{
    [Header("Editor Configuration")]
    [SerializeField] private bool autoRefreshOnChange = true;
    [SerializeField] private bool previewInEditMode = true;
    [SerializeField] private bool logCompilationSteps = false;
    
    [Header("Entity Configuration")]
    [SerializeField] private EntityType entityType = EntityType.Player;
    [SerializeField] private List<string> customTags = new List<string>();
    [SerializeField] private EntitySettings settings;
    
    public enum EntityType { Player, Enemy, NPC, Prop }
    
    [System.Serializable]
    public class EntitySettings
    {
        public float health = 100f;
        public float speed = 5f;
        public bool canAttack = true;
        public int level = 1;
    }
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Configure editor behavior
        this.installInEditMode = previewInEditMode;
        
        // Setup based on entity type
        ConfigureByType();
        
        // Add custom tags
        foreach (var tagName in customTags)
        {
            if (!string.IsNullOrEmpty(tagName))
            {
                this.AddTag(EntityNames.NameToId(tagName));
            }
        }
        
        // Add editor-friendly behaviours
        if (previewInEditMode)
        {
            this.AddBehaviour(new EditModePreviewBehaviour(entityType));
            this.AddBehaviour(new PropertyValidationBehaviour());
        }
        
        if (logCompilationSteps)
        {
            this.AddBehaviour(new CompilationLoggerBehaviour());
        }
    }
    
    private void ConfigureByType()
    {
        switch (entityType)
        {
            case EntityType.Player:
                this.AddTag(EntityTags.PLAYER);
                this.AddTag(EntityTags.CONTROLLABLE);
                this.AddValue(EntityNames.PLAYER_SETTINGS, settings);
                break;
                
            case EntityType.Enemy:
                this.AddTag(EntityTags.ENEMY);
                this.AddTag(EntityTags.HOSTILE);
                this.AddValue(EntityNames.AI_SETTINGS, settings);
                break;
                
            case EntityType.NPC:
                this.AddTag(EntityTags.NPC);
                this.AddTag(EntityTags.INTERACTABLE);
                this.AddValue(EntityNames.NPC_SETTINGS, settings);
                break;
                
            case EntityType.Prop:
                this.AddTag(EntityTags.PROP);
                this.AddValue(EntityNames.PROP_SETTINGS, settings);
                break;
        }
        
        // Common settings
        this.AddValue(EntityNames.HEALTH, settings.health);
        this.AddValue(EntityNames.MAX_HEALTH, settings.health);
        this.AddValue(EntityNames.SPEED, settings.speed);
        this.AddValue(EntityNames.LEVEL, settings.level);
    }
    
    protected override void OnValidate()
    {
        base.OnValidate();
        
        // Auto-refresh when properties change
        if (autoRefreshOnChange && this.Installed && !Application.isPlaying)
        {
            // Delay compilation to avoid OnValidate loops
            EditorApplication.delayCall += () =>
            {
                if (this != null) // Check if object still exists
                {
                    this.Compile();
                }
            };
        }
    }
    
    // Custom editor buttons
    #if ODIN_INSPECTOR
    [Button("Validate Configuration")]
    [PropertyOrder(90)]
    private void ValidateConfiguration()
    {
        var issues = new List<string>();
        
        // Validate settings
        if (settings.health <= 0)
            issues.Add("Health must be greater than 0");
            
        if (settings.speed < 0)
            issues.Add("Speed cannot be negative");
            
        if (settings.level < 1)
            issues.Add("Level must be at least 1");
        
        // Validate installers
        foreach (var installer in installers)
        {
            if (installer == null)
                issues.Add("Found null installer reference");
        }
        
        // Validate children
        foreach (var child in children)
        {
            if (child == null)
                issues.Add("Found null child entity reference");
        }
        
        // Report results
        if (issues.Count == 0)
        {
            Debug.Log($"<color=#00FF00>Configuration validated successfully for {this.name}</color>", this);
        }
        else
        {
            Debug.LogWarning($"<color=#FFAA00>Configuration issues found in {this.name}:</color>\n" +
                           string.Join("\n", issues), this);
        }
    }
    
    [Button("Test Edit Mode Lifecycle")]
    [PropertyOrder(91)]
    private void TestEditModeLifecycle()
    {
        Debug.Log($"Testing edit mode lifecycle for {this.name}");
        
        try
        {
            // Manually trigger edit mode lifecycle
            this.DisableInEditMode();
            this.DisposeInEditMode();
            this.InitInEditMode();
            this.EnableInEditMode();
            
            Debug.Log($"<color=#00FF00>Edit mode lifecycle test completed successfully</color>", this);
        }
        catch (System.Exception e)
        {
            Debug.LogError($"<color=#FF0000>Edit mode lifecycle test failed: {e.Message}</color>", this);
        }
    }
    
    [Button("Generate Entity Report")]
    [PropertyOrder(92)]
    private void GenerateEntityReport()
    {
        var report = $"=== Entity Report: {this.name} ===\n";
        report += $"Entity Type: {entityType}\n";
        report += $"Install in Edit Mode: {installInEditMode}\n";
        report += $"Installed: {this.Installed}\n";
        report += $"Tags: {this.TagCount}\n";
        report += $"Values: {this.ValueCount}\n";
        report += $"Behaviours: {this.BehaviourCount}\n";
        report += $"Installers: {installers.Count}\n";
        report += $"Children: {children.Count}\n";
        
        Debug.Log(report, this);
    }
    #endif
}

// Edit-mode compatible behaviour for logging compilation steps
[RunInEditMode]
public class CompilationLoggerBehaviour : IEntityBehaviour, IEntitySpawn, IEntityActivate, 
    IEntityDeactivate, IEntityDespawn
{
    public void OnInstall(IEntity entity)
    {
        if (!Application.isPlaying)
            Debug.Log($"[Edit Mode] {entity.Name}: Behaviour installed");
    }
    
    public void OnSpawn(IEntity entity)
    {
        if (!Application.isPlaying)
            Debug.Log($"[Edit Mode] {entity.Name}: Entity spawned");
    }
    
    public void OnActivate(IEntity entity)
    {
        if (!Application.isPlaying)
            Debug.Log($"[Edit Mode] {entity.Name}: Entity activated");
    }
    
    public void OnDeactivate(IEntity entity)
    {
        if (!Application.isPlaying)
            Debug.Log($"[Edit Mode] {entity.Name}: Entity deactivated");
    }
    
    public void OnDespawn(IEntity entity)
    {
        if (!Application.isPlaying)
            Debug.Log($"[Edit Mode] {entity.Name}: Entity despawned");
    }
}
```

### Custom Editor Integration

```csharp
#if UNITY_EDITOR
[CustomEditor(typeof(AdvancedEditableEntity))]
public class AdvancedEditableEntityEditor : Editor
{
    private AdvancedEditableEntity entity;
    private bool showAdvancedOptions = false;
    
    void OnEnable()
    {
        entity = (AdvancedEditableEntity)target;
    }
    
    public override void OnInspectorGUI()
    {
        // Show default inspector
        base.OnInspectorGUI();
        
        if (!Application.isPlaying)
        {
            EditorGUILayout.Space();
            EditorGUILayout.LabelField("Edit Mode Tools", EditorStyles.boldLabel);
            
            using (new EditorGUILayout.HorizontalScope())
            {
                if (GUILayout.Button("Compile Entity"))
                {
                    entity.Compile();
                }
                
                if (GUILayout.Button("Reset Entity"))
                {
                    entity.Reset();
                }
            }
            
            showAdvancedOptions = EditorGUILayout.Foldout(showAdvancedOptions, "Advanced Options");
            if (showAdvancedOptions)
            {
                EditorGUI.indentLevel++;
                
                if (GUILayout.Button("Force Refresh Installers"))
                {
                    entity.SetRefreshCallbackToInstallers();
                    EditorUtility.SetDirty(entity);
                }
                
                if (GUILayout.Button("Precompile Capacities"))
                {
                    entity.Precompile();
                    EditorUtility.SetDirty(entity);
                }
                
                if (GUILayout.Button("Test Edit Mode Lifecycle"))
                {
                    TestEditModeLifecycle();
                }
                
                EditorGUI.indentLevel--;
            }
        }
        else
        {
            EditorGUILayout.Space();
            EditorGUILayout.HelpBox("Entity is running in Play Mode. Edit mode tools are disabled.", 
                MessageType.Info);
        }
    }
    
    private void TestEditModeLifecycle()
    {
        Debug.Log("Testing edit mode lifecycle manually...");
        
        entity.DisableInEditMode();
        entity.DisposeInEditMode();
        entity.InitInEditMode();
        entity.EnableInEditMode();
        
        Debug.Log("Edit mode lifecycle test completed");
    }
}
#endif
```

### Editor Asset Integration

```csharp
[CreateAssetMenu(menuName = "Entities/Editor Entity Template")]
public class EntityTemplate : ScriptableObject
{
    [Header("Template Configuration")]
    public string entityName = "New Entity";
    public EntityType entityType = EntityType.Player;
    
    [Header("Default Values")]
    public float defaultHealth = 100f;
    public float defaultSpeed = 5f;
    public int defaultLevel = 1;
    
    [Header("Default Tags")]
    public List<string> defaultTags = new List<string>();
    
    [Header("Required Installers")]
    public List<SceneEntityInstaller> requiredInstallers = new List<SceneEntityInstaller>();
    
    public enum EntityType { Player, Enemy, NPC, Prop }
    
    [ContextMenu("Apply Template to Selected Entity")]
    private void ApplyTemplateToSelected()
    {
        #if UNITY_EDITOR
        var selectedObject = Selection.activeGameObject;
        if (selectedObject == null)
        {
            Debug.LogWarning("No GameObject selected");
            return;
        }
        
        var sceneEntity = selectedObject.GetComponent<SceneEntity>();
        if (sceneEntity == null)
        {
            sceneEntity = selectedObject.AddComponent<SceneEntity>();
        }
        
        ApplyTemplate(sceneEntity);
        #endif
    }
    
    public void ApplyTemplate(SceneEntity entity)
    {
        #if UNITY_EDITOR
        // Set basic properties
        entity.name = entityName;
        entity.installInEditMode = true;
        
        // Apply installers
        var installersList = new List<SceneEntityInstaller>(entity.installers);
        foreach (var installer in requiredInstallers)
        {
            if (installer != null && !installersList.Contains(installer))
            {
                installersList.Add(installer);
            }
        }
        entity.installers = installersList;
        
        // Compile to apply template
        entity.Compile();
        
        // Set default values after compilation
        entity.SetValue(EntityNames.HEALTH, defaultHealth);
        entity.SetValue(EntityNames.MAX_HEALTH, defaultHealth);
        entity.SetValue(EntityNames.SPEED, defaultSpeed);
        entity.SetValue(EntityNames.LEVEL, defaultLevel);
        
        // Add default tags
        foreach (var tagName in defaultTags)
        {
            if (!string.IsNullOrEmpty(tagName))
            {
                entity.AddTag(EntityNames.NameToId(tagName));
            }
        }
        
        EditorUtility.SetDirty(entity);
        Debug.Log($"Applied template '{this.name}' to entity '{entity.name}'", entity);
        #endif
    }
}
```

## Editor Integration Features

### Odin Inspector Integration
- Custom button styling with `GUIColor` attributes
- Conditional display with `HideInPlayMode` 
- Property ordering with `PropertyOrder`
- Button integration for common operations

### Unity Editor Integration
- Context menu commands for easy access
- Proper prefab handling in compilation
- EditorApplication integration for timing
- Automatic serialization and undo support

### Development Workflow
- Automatic installer discovery
- Live property updates via OnValidate
- Edit-mode lifecycle simulation
- Comprehensive error handling and logging

## Performance Considerations

### Editor Performance
- OnValidate can be expensive with many entities
- Consider disabling auto-refresh for complex scenes
- Use delayed compilation to avoid inspector loops
- Cache frequently accessed components

### Memory Management
- Edit-mode behaviours may hold editor-only references
- Proper cleanup in DisposeInEditMode
- Avoid memory leaks from edit-mode allocations

## Best Practices

1. **Edit Mode Safety** – Only use edit-compatible operations in edit mode
2. **Behaviour Attributes** – Use `RunInEditModeAttribute` appropriately
3. **Property Validation** – Implement proper OnValidate logic
4. **Error Handling** – Wrap edit operations in try-catch blocks
5. **Performance** – Consider disabling edit mode for complex entities
6. **Prefab Awareness** – Handle prefabs differently from scene instances
7. **Development Workflow** – Use editor tools to streamline entity creation

## Common Issues and Solutions

### OnValidate Loops
- Use delayed calls to prevent infinite loops
- Check if object exists before delayed operations
- Avoid triggering OnValidate from within OnValidate

### Prefab Issues
- Check prefab status before edit operations
- Handle prefab instances vs prefab assets correctly
- Use proper prefab utilities for detection

### Edit Mode Compatibility
- Not all behaviours work safely in edit mode
- Use `RunInEditModeAttribute` to control edit compatibility
- Test edit mode operations thoroughly

## Integration with Development Tools

### Version Control
- Serialized precompiled capacities help with merging
- Edit mode changes are properly serialized
- Template assets support team workflows

### Asset Pipeline
- Integration with Unity's asset import pipeline
- Proper handling of asset dependencies
- Support for scriptable object templates

### Testing Framework
- Edit mode compilation supports automated testing
- Lifecycle simulation enables thorough testing
- Template system supports test entity creation

## Notes

- Editor functionality is completely stripped in builds
- Requires careful handling of editor-only references
- Supports both manual and automatic workflow patterns
- Integration with Unity's undo system for proper editor experience
- Comprehensive error handling prevents editor crashes
- Template system enables efficient entity authoring workflows