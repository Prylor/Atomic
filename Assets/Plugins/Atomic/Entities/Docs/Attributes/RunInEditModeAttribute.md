# 🛠️ RunInEditModeAttribute

`RunInEditModeAttribute` is a class-level attribute that marks entity behavior classes to indicate their entity lifecycle callbacks should be invoked during Unity Editor mode. This enables testing and development of entity systems without entering Play Mode.

## Key Features

- **Edit Mode Support** – Enables lifecycle callbacks in Unity Editor
- **Development Tool** – Perfect for testing and debugging entity behaviors
- **Class-Level Attribute** – Applied to entire behavior classes
- **Runtime Simulation** – Simulates runtime logic in Editor mode
- **IEntityBehaviour Integration** – Designed specifically for entity behaviors

---

## Attribute Definition

```csharp
[AttributeUsage(AttributeTargets.Class)]
public sealed class RunInEditModeAttribute : Attribute
{
}
```

### Target
- **AttributeTargets.Class** – Can only be applied to classes
- **Sealed class** – Cannot be inherited or extended
- **No parameters** – Simple marker attribute

### Usage Scope
- **IEntityBehaviour implementations** – Intended for entity behavior classes
- **Lifecycle callbacks** – Affects Init, Enable, Disable, Dispose methods
- **Editor mode only** – Only impacts behavior in Unity Editor

---

## Usage Patterns

### Basic Edit Mode Behavior

```csharp
[RunInEditMode]
public class DebugEntityBehaviour : IEntityBehaviour
{
    private Entity owner;
    
    public void Initialize(Entity entity)
    {
        owner = entity;
        Debug.Log($"Debug behavior initialized in {(Application.isPlaying ? "Play" : "Edit")} mode");
        
        // This will run in both Play Mode and Edit Mode
        SetupDebugVisualization();
    }
    
    public void Deinitialize()
    {
        Debug.Log($"Debug behavior deinitialized in {(Application.isPlaying ? "Play" : "Edit")} mode");
        CleanupDebugVisualization();
    }
    
    private void SetupDebugVisualization()
    {
        // Setup debug gizmos, editor GUI, etc.
        if (!Application.isPlaying)
        {
            // Editor-specific setup
            RegisterEditorCallbacks();
        }
    }
    
    private void CleanupDebugVisualization()
    {
        if (!Application.isPlaying)
        {
            // Editor-specific cleanup
            UnregisterEditorCallbacks();
        }
    }
}
```

### Development Testing Behavior

```csharp
[RunInEditMode]
public class DevelopmentTestBehaviour : IEntityBehaviour, IEntityUpdate
{
    private Entity entity;
    private float testTimer;
    
    public void Initialize(Entity owner)
    {
        entity = owner;
        Debug.Log("Development test behavior active in Edit Mode");
        
        // Initialize test data
        InitializeTestData();
    }
    
    public void OnUpdate(float deltaTime)
    {
        // This update will run in Edit Mode if the attribute is present
        testTimer += deltaTime;
        
        if (testTimer >= 1f) // Test every second
        {
            RunDevelopmentTests();
            testTimer = 0f;
        }
    }
    
    private void InitializeTestData()
    {
        // Set up test values for entity
        entity.SetValue("TestValue", 42);
        entity.AddTag("TestTag");
    }
    
    private void RunDevelopmentTests()
    {
        // Run validation tests
        ValidateEntityState();
        TestBehaviorFunctionality();
        
        if (!Application.isPlaying)
        {
            Debug.Log("Edit Mode development test completed");
        }
    }
}
```

### Editor Visualization Behavior

```csharp
[RunInEditMode]
public class EditorVisualizationBehaviour : IEntityBehaviour, IEntityGizmos
{
    private Entity entity;
    private Color visualizationColor = Color.yellow;
    
    public void Initialize(Entity owner)
    {
        entity = owner;
        
        // Set up editor visualization
        if (!Application.isPlaying)
        {
            SetupEditorVisualization();
        }
    }
    
    public void OnDrawGizmos()
    {
        if (entity?.TryGetValue<Vector3>("Position", out var position) == true)
        {
            // Draw entity representation in Scene view
            Gizmos.color = visualizationColor;
            Gizmos.DrawWireSphere(position, 1f);
            
            // Draw entity info
            DrawEntityInfo(position);
        }
    }
    
    public void OnDrawGizmosSelected()
    {
        if (entity?.TryGetValue<Vector3>("Position", out var position) == true)
        {
            // Enhanced visualization when selected
            Gizmos.color = Color.white;
            Gizmos.DrawSphere(position, 0.1f);
            
            // Draw detailed entity information
            DrawDetailedEntityInfo(position);
        }
    }
    
    private void SetupEditorVisualization()
    {
        // Configure editor-specific visualization settings
        visualizationColor = entity.HasTag("Player") ? Color.green : Color.red;
    }
    
    private void DrawEntityInfo(Vector3 position)
    {
        #if UNITY_EDITOR
        var style = new GUIStyle();
        style.normal.textColor = Color.white;
        UnityEditor.Handles.Label(position + Vector3.up * 1.5f, 
            $"Entity: {entity.Name}\nValues: {entity.ValueCount}\nTags: {entity.TagCount}", style);
        #endif
    }
}
```

### Configuration Validation Behavior

```csharp
[RunInEditMode]
public class ConfigurationValidationBehaviour : IEntityBehaviour
{
    private Entity entity;
    private bool isValidConfiguration = true;
    
    public void Initialize(Entity owner)
    {
        entity = owner;
        
        // Validate entity configuration in Edit Mode
        ValidateConfiguration();
        
        if (!isValidConfiguration && !Application.isPlaying)
        {
            Debug.LogError($"Invalid configuration detected for entity {entity.Name}");
        }
    }
    
    private void ValidateConfiguration()
    {
        var issues = new List<string>();
        
        // Check required values
        if (!entity.HasValue("Health"))
        {
            issues.Add("Missing required 'Health' value");
        }
        
        if (!entity.HasValue("Position"))
        {
            issues.Add("Missing required 'Position' value");
        }
        
        // Check required tags
        if (!entity.HasTag("EntityType"))
        {
            issues.Add("Missing required 'EntityType' tag");
        }
        
        // Validate value ranges
        if (entity.TryGetValue<int>("Health", out var health) && health <= 0)
        {
            issues.Add("Health must be greater than zero");
        }
        
        isValidConfiguration = issues.Count == 0;
        
        if (issues.Count > 0)
        {
            Debug.LogWarning($"Configuration issues for {entity.Name}:\n{string.Join("\n", issues)}");
        }
    }
    
    public void Deinitialize()
    {
        // Cleanup validation resources
    }
}
```

### Editor Tool Integration

```csharp
[RunInEditMode]
public class EditorToolIntegrationBehaviour : IEntityBehaviour
{
    private Entity entity;
    
    #if UNITY_EDITOR
    private bool isEditorTool;
    #endif
    
    public void Initialize(Entity owner)
    {
        entity = owner;
        
        #if UNITY_EDITOR
        if (!Application.isPlaying)
        {
            SetupEditorTool();
            isEditorTool = true;
        }
        #endif
    }
    
    #if UNITY_EDITOR
    private void SetupEditorTool()
    {
        // Register with Unity Editor tools
        UnityEditor.SceneView.duringSceneGui += OnSceneGUI;
        UnityEditor.EditorApplication.update += OnEditorUpdate;
        
        Debug.Log($"Editor tool registered for entity {entity.Name}");
    }
    
    private void OnSceneGUI(UnityEditor.SceneView sceneView)
    {
        if (entity?.TryGetValue<Vector3>("Position", out var position) == true)
        {
            // Draw custom handles in Scene view
            UnityEditor.Handles.color = Color.cyan;
            var newPosition = UnityEditor.Handles.PositionHandle(position, Quaternion.identity);
            
            if (newPosition != position)
            {
                entity.SetValue("Position", newPosition);
                UnityEditor.SceneView.RepaintAll();
            }
        }
    }
    
    private void OnEditorUpdate()
    {
        // Update entity in Editor
        if (entity != null && !Application.isPlaying)
        {
            UpdateEntityInEditor();
        }
    }
    
    private void UpdateEntityInEditor()
    {
        // Perform editor-specific updates
        ValidateEntityState();
        UpdateEditorVisualization();
    }
    #endif
    
    public void Deinitialize()
    {
        #if UNITY_EDITOR
        if (isEditorTool)
        {
            UnityEditor.SceneView.duringSceneGui -= OnSceneGUI;
            UnityEditor.EditorApplication.update -= OnEditorUpdate;
        }
        #endif
    }
}
```

### Prototyping Behavior

```csharp
[RunInEditMode]
public class PrototypingBehaviour : IEntityBehaviour, IEntityUpdate
{
    private Entity entity;
    private float prototypeTimer;
    private int iterationCount;
    
    public void Initialize(Entity owner)
    {
        entity = owner;
        
        if (!Application.isPlaying)
        {
            Debug.Log("Prototype behavior active - testing entity logic in Edit Mode");
            SetupPrototypeData();
        }
    }
    
    public void OnUpdate(float deltaTime)
    {
        if (!Application.isPlaying)
        {
            // Run prototype logic in Edit Mode
            prototypeTimer += deltaTime;
            
            if (prototypeTimer >= 2f) // Prototype iteration every 2 seconds
            {
                RunPrototypeIteration();
                prototypeTimer = 0f;
            }
        }
    }
    
    private void SetupPrototypeData()
    {
        // Initialize prototype testing data
        entity.SetValue("PrototypeValue", Random.Range(1, 100));
        entity.AddTag("Prototype");
    }
    
    private void RunPrototypeIteration()
    {
        iterationCount++;
        
        // Test entity modifications
        if (entity.TryGetValue<int>("PrototypeValue", out var value))
        {
            var newValue = value + Random.Range(-10, 10);
            entity.SetValue("PrototypeValue", Mathf.Max(0, newValue));
        }
        
        Debug.Log($"Prototype iteration {iterationCount}: Value = {entity.GetValue<int>("PrototypeValue")}");
        
        // Test behavior with modified values
        TestPrototypeBehavior();
    }
    
    private void TestPrototypeBehavior()
    {
        // Run prototype behavior tests
        var value = entity.GetValue<int>("PrototypeValue");
        
        if (value > 50)
        {
            entity.AddTag("HighValue");
        }
        else
        {
            entity.DelTag("HighValue");
        }
        
        Debug.Log($"Entity state: High Value = {entity.HasTag("HighValue")}");
    }
}
```

## Integration with Entity System

### System Detection

```csharp
public class EntityBehaviourManager
{
    public void InitializeBehaviour(IEntityBehaviour behaviour, Entity entity)
    {
        // Check if behavior should run in current mode
        bool shouldRun = Application.isPlaying || EntityUtils.IsRunInEditModeDefined(behaviour);
        
        if (shouldRun)
        {
            behaviour.Initialize(entity);
            Debug.Log($"Initialized {behaviour.GetType().Name} in {(Application.isPlaying ? "Play" : "Edit")} mode");
        }
        else
        {
            Debug.Log($"Skipped {behaviour.GetType().Name} - not marked for Edit Mode execution");
        }
    }
}
```

### Conditional Behavior Loading

```csharp
public static class BehaviourUtils
{
    public static bool ShouldExecuteInCurrentMode(Type behaviourType)
    {
        if (Application.isPlaying)
        {
            return true; // Always execute in Play Mode
        }
        
        // Check for RunInEditMode attribute in Edit Mode
        return behaviourType.IsDefined(typeof(RunInEditModeAttribute), false);
    }
    
    public static void ExecuteIfAllowed(IEntityBehaviour behaviour, Action action)
    {
        if (ShouldExecuteInCurrentMode(behaviour.GetType()))
        {
            action();
        }
    }
}
```

## Best Practices

### When to Use RunInEditMode
1. **Development tools** – Editor utilities and debugging aids
2. **Configuration validation** – Checking entity setup in Editor
3. **Visualization** – Custom gizmos and Scene view enhancements
4. **Prototyping** – Testing behavior logic without Play Mode
5. **Asset validation** – Ensuring entity assets are properly configured

### Performance Considerations
1. **Editor performance** – Be mindful of Editor performance impact
2. **Conditional execution** – Use `#if UNITY_EDITOR` for Editor-only code
3. **Resource cleanup** – Always clean up Editor resources properly
4. **Update frequency** – Limit expensive operations in Edit Mode

### Safety Guidelines
1. **State management** – Ensure proper state handling across mode changes
2. **Resource leaks** – Clean up Editor callbacks and subscriptions
3. **Exception handling** – Handle Editor-specific exceptions gracefully
4. **Compatibility** – Test behavior in both Play and Edit modes

## Common Use Cases

- **Debug visualization** – Custom gizmos and Scene view helpers
- **Configuration validation** – Entity setup verification
- **Development testing** – Logic testing without entering Play Mode
- **Editor tools** – Custom entity manipulation tools
- **Asset preprocessing** – Entity asset validation and setup
- **Prototyping** – Rapid iteration of entity behaviors

## Limitations

### Editor Constraints
- **Limited Unity API** – Some Unity APIs don't work in Edit Mode
- **No physics** – Physics simulation not available in Edit Mode
- **Performance impact** – Can slow down Editor operations
- **State persistence** – Editor state may not persist across sessions

### Development Considerations
- **Testing both modes** – Always test in both Edit and Play modes
- **Conditional code** – Use Editor preprocessor directives appropriately
- **Memory management** – Be extra careful with Editor memory usage
- **Threading** – Editor may have different threading behavior

## Integration Examples

### Custom Inspector Integration

```csharp
#if UNITY_EDITOR
[UnityEditor.CustomEditor(typeof(EntityComponent))]
public class EntityComponentEditor : UnityEditor.Editor
{
    public override void OnInspectorGUI()
    {
        DrawDefaultInspector();
        
        var component = target as EntityComponent;
        var entity = component.Entity;
        
        if (entity != null)
        {
            // Show which behaviors have RunInEditMode
            UnityEditor.EditorGUILayout.Space();
            UnityEditor.EditorGUILayout.LabelField("Edit Mode Behaviors:", UnityEditor.EditorStyles.boldLabel);
            
            foreach (var behaviour in entity.GetBehaviours())
            {
                bool hasAttribute = EntityUtils.IsRunInEditModeDefined(behaviour);
                string status = hasAttribute ? "✓ Edit Mode" : "○ Play Only";
                UnityEditor.EditorGUILayout.LabelField($"{behaviour.GetType().Name}: {status}");
            }
        }
    }
}
#endif
```

## Thread Safety

- **Editor thread only** – All Edit Mode operations run on Unity's main thread
- **No multithreading** – Editor behaviors should avoid threading
- **State synchronization** – No special synchronization needed
- **Unity integration** – Follows Unity's single-threaded Editor model