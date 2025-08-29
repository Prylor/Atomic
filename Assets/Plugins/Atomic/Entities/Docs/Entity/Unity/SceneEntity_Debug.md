# 🐛 SceneEntity_Debug

The `SceneEntity_Debug` partial class provides comprehensive debugging and inspection capabilities for `SceneEntity` instances in the Unity Editor when Odin Inspector is available. It offers read-only visualization of entity state, interactive management of tags, values, and behaviours directly from the inspector.

## Key Features

- **Editor-Only Debug UI** – Available only in Unity Editor with Odin Inspector
- **Live Entity State Inspection** – Real-time view of tags, values, and behaviours
- **Interactive Management** – Remove tags, values, and behaviours from inspector
- **Searchable Lists** – Find specific elements quickly
- **Sorted Display** – Alphabetically sorted elements for easy browsing
- **Type-Safe Removal** – Safe deletion of entity components from inspector

---

## Requirements

This functionality requires:
- `UNITY_EDITOR` compilation symbol
- `ODIN_INSPECTOR` compilation symbol  
- Odin Inspector asset in project

## Debug Elements

### TagElement Structure
```csharp
[InlineProperty]
private struct TagElement : IComparable<TagElement>
{
    [ShowInInspector, ReadOnly]
    internal string name;     // Display name from EntityNames
    internal readonly int id; // Internal tag ID
    
    public int CompareTo(TagElement other); // Alphabetical sorting
}
```

### ValueElement Structure
```csharp
[InlineProperty]
private struct ValueElement : IComparable<ValueElement>
{
    [HorizontalGroup(200), ShowInInspector, ReadOnly, HideLabel]
    public string name;       // Display name from EntityNames
    
    [HorizontalGroup, ShowInInspector, HideLabel]
    public object value;      // Actual stored value
    
    internal readonly int id; // Internal value key
    
    public int CompareTo(ValueElement other); // Alphabetical sorting
}
```

### BehaviourElement Structure
```csharp
[InlineProperty]
private struct BehaviourElement : IComparable<BehaviourElement>
{
    [ShowInInspector, ReadOnly]
    public string name;                      // Behaviour type name
    internal readonly IEntityBehaviour value; // Behaviour instance
    
    public int CompareTo(BehaviourElement other); // Alphabetical sorting
}
```

## Inspector Properties

### Tags Display
```csharp
[Searchable]
[FoldoutGroup("Debug")]
[LabelText("Tags")]
[ShowInInspector, PropertyOrder(100)]
private List<TagElement> TagElements { get; }
```
- Shows all entity tags with their display names
- Searchable list for quick filtering
- Custom remove functions for tag deletion
- Read-only display names resolved from EntityNames

### Values Display  
```csharp
[Searchable]
[FoldoutGroup("Debug")]
[LabelText("Values")]
[ShowInInspector, PropertyOrder(100)]
private List<ValueElement> ValueElements { get; }
```
- Displays all entity values with names and current values
- Horizontal layout showing name and value side-by-side
- Values are live-updated as entity state changes
- Supports removal of individual values

### Behaviours Display
```csharp
[Searchable] 
[FoldoutGroup("Debug")]
[LabelText("Behaviours")]
[ShowInInspector, PropertyOrder(100)]
private List<BehaviourElement> BehaviourElements { get; }
```
- Lists all attached behaviours by type name
- Shows actual behaviour instances
- Allows removal of specific behaviours
- Sorted alphabetically by type name

## Example Usage

### Basic Debug Inspection

```csharp
public class DebuggablePlayerEntity : SceneEntity
{
    [Header("Player Configuration")]
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private int maxHealth = 100;
    [SerializeField] private string playerName = "Player";
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Add tags for debugging
        this.AddTag(EntityTags.PLAYER);
        this.AddTag(EntityTags.CONTROLLABLE);
        this.AddTag(EntityTags.DAMAGEABLE);
        
        // Add values for debugging
        this.AddValue(EntityNames.PLAYER_NAME, playerName);
        this.AddValue(EntityNames.MAX_HEALTH, maxHealth);
        this.AddValue(EntityNames.HEALTH, maxHealth);
        this.AddValue(EntityNames.MOVE_SPEED, moveSpeed);
        this.AddValue(EntityNames.LEVEL, 1);
        this.AddValue(EntityNames.EXPERIENCE, 0);
        
        // Add behaviours for debugging
        this.AddBehaviour(new PlayerInputBehaviour());
        this.AddBehaviour(new MovementBehaviour());
        this.AddBehaviour(new HealthBehaviour());
        this.AddBehaviour(new AnimationBehaviour());
        this.AddBehaviour(new InventoryBehaviour());
        
        Debug.Log("Player entity setup complete - check Debug foldout in inspector");
    }
    
    [Button("Simulate Damage")]
    private void SimulateDamage()
    {
        int currentHealth = this.GetValue<int>(EntityNames.HEALTH);
        int newHealth = Mathf.Max(0, currentHealth - 25);
        this.SetValue(EntityNames.HEALTH, newHealth);
        
        if (newHealth <= 0)
        {
            this.AddTag(EntityTags.DEAD);
            this.DelTag(EntityTags.CONTROLLABLE);
        }
    }
    
    [Button("Add Random Tag")]
    private void AddRandomTag()
    {
        var randomTags = new[]
        {
            EntityTags.BURNING,
            EntityTags.FROZEN,
            EntityTags.POISONED,
            EntityTags.BUFFED,
            EntityTags.INVISIBLE
        };
        
        var randomTag = randomTags[UnityEngine.Random.Range(0, randomTags.Length)];
        this.AddTag(randomTag);
    }
    
    [Button("Level Up")]
    private void LevelUp()
    {
        int currentLevel = this.GetValue<int>(EntityNames.LEVEL);
        int currentExp = this.GetValue<int>(EntityNames.EXPERIENCE);
        
        this.SetValue(EntityNames.LEVEL, currentLevel + 1);
        this.SetValue(EntityNames.EXPERIENCE, currentExp + 100);
        this.SetValue(EntityNames.MAX_HEALTH, maxHealth + (currentLevel * 10));
    }
}
```

### Advanced Debug Monitoring

```csharp
#if UNITY_EDITOR
using UnityEditor;

[CustomEditor(typeof(DebuggablePlayerEntity))]
public class DebuggablePlayerEntityEditor : Editor
{
    private DebuggablePlayerEntity entity;
    private bool showLiveStats = true;
    private double lastUpdateTime;
    
    void OnEnable()
    {
        entity = (DebuggablePlayerEntity)target;
        EditorApplication.update += OnEditorUpdate;
    }
    
    void OnDisable()
    {
        EditorApplication.update -= OnEditorUpdate;
    }
    
    void OnEditorUpdate()
    {
        // Update inspector every 0.5 seconds during play mode
        if (Application.isPlaying && EditorApplication.timeSinceStartup - lastUpdateTime > 0.5)
        {
            Repaint();
            lastUpdateTime = EditorApplication.timeSinceStartup;
        }
    }
    
    public override void OnInspectorGUI()
    {
        base.OnInspectorGUI();
        
        if (!Application.isPlaying) return;
        
        EditorGUILayout.Space();
        showLiveStats = EditorGUILayout.Foldout(showLiveStats, "Live Debug Stats");
        
        if (showLiveStats)
        {
            EditorGUI.indentLevel++;
            
            // Show real-time statistics
            EditorGUILayout.LabelField("Entity Statistics", EditorStyles.boldLabel);
            EditorGUILayout.LabelField($"Tag Count: {entity.TagCount}");
            EditorGUILayout.LabelField($"Value Count: {entity.ValueCount}");
            EditorGUILayout.LabelField($"Behaviour Count: {entity.BehaviourCount}");
            
            EditorGUILayout.Space();
            
            // Show current state
            EditorGUILayout.LabelField("Current State", EditorStyles.boldLabel);
            EditorGUILayout.LabelField($"Installed: {entity.Installed}");
            EditorGUILayout.LabelField($"Spawned: {entity.Spawned}");
            EditorGUILayout.LabelField($"Enabled: {entity.Enabled}");
            
            EditorGUILayout.Space();
            
            // Show key values
            if (entity.HasValue(EntityNames.HEALTH))
            {
                int health = entity.GetValue<int>(EntityNames.HEALTH);
                int maxHealth = entity.GetValue<int>(EntityNames.MAX_HEALTH);
                float healthPercent = (float)health / maxHealth;
                
                EditorGUILayout.LabelField("Health", EditorStyles.boldLabel);
                EditorGUI.ProgressBar(GUILayoutUtility.GetRect(200, 20), 
                    healthPercent, $"{health}/{maxHealth}");
            }
            
            EditorGUI.indentLevel--;
        }
        
        // Debug buttons
        EditorGUILayout.Space();
        EditorGUILayout.LabelField("Debug Actions", EditorStyles.boldLabel);
        
        using (new EditorGUILayout.HorizontalScope())
        {
            if (GUILayout.Button("Refresh Inspector"))
            {
                Repaint();
            }
            
            if (GUILayout.Button("Log Entity State"))
            {
                LogEntityState();
            }
        }
        
        using (new EditorGUILayout.HorizontalScope())
        {
            if (GUILayout.Button("Clear All Tags"))
            {
                if (EditorUtility.DisplayDialog("Clear Tags", 
                    "Remove all tags from this entity?", "Yes", "Cancel"))
                {
                    entity.ClearTags();
                }
            }
            
            if (GUILayout.Button("Clear All Values"))
            {
                if (EditorUtility.DisplayDialog("Clear Values", 
                    "Remove all values from this entity?", "Yes", "Cancel"))
                {
                    entity.ClearValues();
                }
            }
        }
    }
    
    private void LogEntityState()
    {
        Debug.Log($"=== Entity State: {entity.Name} ===");
        
        // Log tags
        Debug.Log($"Tags ({entity.TagCount}):");
        var tagEnumerator = entity.GetTagEnumerator();
        while (tagEnumerator.MoveNext())
        {
            var tagName = EntityNames.IdToName(tagEnumerator.Current);
            Debug.Log($"  - {tagName} ({tagEnumerator.Current})");
        }
        
        // Log values
        Debug.Log($"Values ({entity.ValueCount}):");
        var valueEnumerator = entity.GetValueEnumerator();
        while (valueEnumerator.MoveNext())
        {
            var (id, value) = valueEnumerator.Current;
            var valueName = EntityNames.IdToName(id);
            Debug.Log($"  - {valueName}: {value} ({value?.GetType().Name})");
        }
        
        // Log behaviours
        Debug.Log($"Behaviours ({entity.BehaviourCount}):");
        for (int i = 0; i < entity.BehaviourCount; i++)
        {
            var behaviour = entity.GetBehaviourAt(i);
            Debug.Log($"  - {behaviour.GetType().Name}");
        }
    }
}
#endif
```

### Debug-Enabled Entity Factory

```csharp
public class DebugEntityFactory : MonoBehaviour
{
    [Header("Debug Configuration")]
    [SerializeField] private bool enableDebugMode = true;
    [SerializeField] private bool logEntityCreation = true;
    
    public SceneEntity CreateDebugEntity(string entityName)
    {
        var entityGO = new GameObject($"Debug_{entityName}");
        var entity = entityGO.AddComponent<SceneEntity>();
        entity.Name = entityName;
        
        if (enableDebugMode)
        {
            SetupDebugEntity(entity);
        }
        
        entity.Install();
        
        if (logEntityCreation)
        {
            Debug.Log($"Created debug entity: {entityName}");
            LogEntityConfiguration(entity);
        }
        
        return entity;
    }
    
    private void SetupDebugEntity(SceneEntity entity)
    {
        // Add debug tags
        entity.AddTag(EntityTags.DEBUG);
        entity.AddTag(EntityTags.RUNTIME_CREATED);
        
        // Add debug values
        entity.AddValue(EntityNames.CREATION_TIME, Time.time);
        entity.AddValue(EntityNames.CREATION_FRAME, Time.frameCount);
        entity.AddValue(EntityNames.DEBUG_ID, System.Guid.NewGuid().ToString());
        
        // Add debug behaviours
        entity.AddBehaviour(new DebugLogBehaviour());
        entity.AddBehaviour(new PerformanceMonitorBehaviour());
        
        // Subscribe to debug events
        entity.OnStateChanged += () => Debug.Log($"Debug entity state changed: {entity.Name}");
        
        entity.OnBehaviourAdded += (e, b) => 
            Debug.Log($"Behaviour added to {e.Name}: {b.GetType().Name}");
            
        entity.OnBehaviourDeleted += (e, b) => 
            Debug.Log($"Behaviour removed from {e.Name}: {b.GetType().Name}");
    }
    
    private void LogEntityConfiguration(SceneEntity entity)
    {
        Debug.Log($"=== Debug Entity Configuration: {entity.Name} ===");
        Debug.Log($"Instance ID: {entity.InstanceID}");
        Debug.Log($"Tags: {entity.TagCount}");
        Debug.Log($"Values: {entity.ValueCount}");
        Debug.Log($"Behaviours: {entity.BehaviourCount}");
        
        // In editor with Odin Inspector, the debug foldout will show detailed information
        #if UNITY_EDITOR && ODIN_INSPECTOR
        Debug.Log("Check the Debug foldout in the inspector for detailed entity state");
        #endif
    }
    
    [Button("Create Test Entity")]
    private void CreateTestEntity()
    {
        var entity = CreateDebugEntity($"TestEntity_{Time.frameCount}");
        
        // Add some test data
        entity.AddValue(EntityNames.HEALTH, UnityEngine.Random.Range(50, 100));
        entity.AddValue(EntityNames.LEVEL, UnityEngine.Random.Range(1, 10));
        entity.AddTag(UnityEngine.Random.value > 0.5f ? EntityTags.FRIENDLY : EntityTags.ENEMY);
    }
}
```

### Performance Debug Monitor

```csharp
public class EntityPerformanceDebugger : MonoBehaviour
{
    [Header("Performance Monitoring")]
    [SerializeField] private bool monitorMemoryUsage = true;
    [SerializeField] private bool monitorUpdatePerformance = true;
    [SerializeField] private float updateInterval = 1.0f;
    
    private List<SceneEntity> monitoredEntities = new List<SceneEntity>();
    private Dictionary<SceneEntity, EntityMetrics> entityMetrics = new Dictionary<SceneEntity, EntityMetrics>();
    
    private struct EntityMetrics
    {
        public int tagCount;
        public int valueCount;
        public int behaviourCount;
        public long memorySnapshot;
        public float lastUpdateTime;
    }
    
    void Start()
    {
        // Find all debug entities
        var allEntities = FindObjectsOfType<SceneEntity>();
        foreach (var entity in allEntities)
        {
            if (entity.HasTag(EntityTags.DEBUG))
            {
                monitoredEntities.Add(entity);
                entityMetrics[entity] = new EntityMetrics();
            }
        }
        
        if (monitoredEntities.Count > 0)
        {
            InvokeRepeating(nameof(UpdateMetrics), updateInterval, updateInterval);
        }
    }
    
    private void UpdateMetrics()
    {
        foreach (var entity in monitoredEntities.ToArray())
        {
            if (entity == null)
            {
                monitoredEntities.Remove(entity);
                entityMetrics.Remove(entity);
                continue;
            }
            
            var metrics = entityMetrics[entity];
            var newMetrics = new EntityMetrics
            {
                tagCount = entity.TagCount,
                valueCount = entity.ValueCount,
                behaviourCount = entity.BehaviourCount,
                lastUpdateTime = Time.time
            };
            
            if (monitorMemoryUsage)
            {
                newMetrics.memorySnapshot = System.GC.GetTotalMemory(false);
            }
            
            // Check for significant changes
            if (HasSignificantChanges(metrics, newMetrics))
            {
                LogEntityChanges(entity, metrics, newMetrics);
            }
            
            entityMetrics[entity] = newMetrics;
        }
    }
    
    private bool HasSignificantChanges(EntityMetrics old, EntityMetrics current)
    {
        return old.tagCount != current.tagCount ||
               old.valueCount != current.valueCount ||
               old.behaviourCount != current.behaviourCount;
    }
    
    private void LogEntityChanges(SceneEntity entity, EntityMetrics old, EntityMetrics current)
    {
        Debug.Log($"[Performance] Entity changes detected: {entity.Name}");
        
        if (old.tagCount != current.tagCount)
            Debug.Log($"  Tags: {old.tagCount} -> {current.tagCount}");
            
        if (old.valueCount != current.valueCount)
            Debug.Log($"  Values: {old.valueCount} -> {current.valueCount}");
            
        if (old.behaviourCount != current.behaviourCount)
            Debug.Log($"  Behaviours: {old.behaviourCount} -> {current.behaviourCount}");
    }
    
    [Button("Generate Performance Report")]
    private void GeneratePerformanceReport()
    {
        Debug.Log("=== Entity Performance Report ===");
        Debug.Log($"Monitored entities: {monitoredEntities.Count}");
        
        int totalTags = 0, totalValues = 0, totalBehaviours = 0;
        
        foreach (var entity in monitoredEntities)
        {
            if (entity != null)
            {
                totalTags += entity.TagCount;
                totalValues += entity.ValueCount;
                totalBehaviours += entity.BehaviourCount;
                
                Debug.Log($"Entity: {entity.Name} | T:{entity.TagCount} V:{entity.ValueCount} B:{entity.BehaviourCount}");
            }
        }
        
        Debug.Log($"Totals - Tags: {totalTags}, Values: {totalValues}, Behaviours: {totalBehaviours}");
        
        if (monitorMemoryUsage)
        {
            long totalMemory = System.GC.GetTotalMemory(false);
            Debug.Log($"Current memory usage: {totalMemory / 1024 / 1024} MB");
        }
    }
}
```

## Debug Features

### Live Inspector Updates
- Real-time reflection of entity state changes
- Automatic refresh during gameplay
- Searchable and sortable element lists

### Interactive Management
- Remove tags, values, and behaviours directly from inspector
- Custom remove functions for each element type
- Safe deletion with proper cleanup

### Performance Monitoring
- Track entity component counts over time
- Monitor memory usage patterns
- Identify performance bottlenecks

### Integration with Development Tools
- Works seamlessly with Odin Inspector attributes
- Custom editor support for extended functionality
- Integration with Unity's profiler and debugging tools

## Best Practices

1. **Editor-Only Usage** – Never rely on debug functionality in builds
2. **Performance Awareness** – Debug UI can impact editor performance with many entities
3. **Odin Inspector Integration** – Leverage Odin's advanced inspector features
4. **Custom Editors** – Extend with custom editor scripts for specific needs
5. **Memory Monitoring** – Use debug tools to track memory usage patterns
6. **Development Workflow** – Integrate into development and testing workflows

## Limitations

### Compilation Requirements
- Only available with both `UNITY_EDITOR` and `ODIN_INSPECTOR` defined
- Requires Odin Inspector asset in project
- Not available in builds or without Odin Inspector

### Performance Considerations
- Inspector updates can be expensive with many entities
- Sorting and searching operations have computational cost
- Live value updates may impact editor responsiveness

### Read-Only Limitations
- Values are displayed but not directly editable
- Tag and value names are resolved from EntityNames
- Behaviour instances are not directly modifiable

## Integration with Testing

### Unit Testing Support
```csharp
#if UNITY_EDITOR
[Test]
public void TestEntityDebugInformation()
{
    var entity = new GameObject().AddComponent<SceneEntity>();
    entity.Install();
    
    entity.AddTag(EntityTags.TEST);
    entity.AddValue(EntityNames.TEST_VALUE, 42);
    entity.AddBehaviour(new TestBehaviour());
    
    // Debug information should be available in editor
    Assert.AreEqual(1, entity.TagCount);
    Assert.AreEqual(1, entity.ValueCount);
    Assert.AreEqual(1, entity.BehaviourCount);
    
    // Debug UI should reflect these values
    // (Actual UI testing would require Odin Inspector test framework)
}
#endif
```

### Debug Validation
- Validate entity state consistency
- Check for memory leaks in debug builds
- Verify proper cleanup of debug resources

## Notes

- Debug functionality is completely editor-only and has no runtime impact
- Requires Odin Inspector for full functionality
- All debug elements are sorted alphabetically for consistency  
- Custom remove functions ensure proper entity state management
- Searchable lists help with entities containing many components
- Static caches improve performance for repeated inspector updates
- Integration with Unity's serialization system for proper inspector display