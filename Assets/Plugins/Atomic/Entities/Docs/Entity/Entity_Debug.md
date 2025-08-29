# 🐛 Entity_Debug

`Entity_Debug` is a partial class implementation that provides comprehensive debug visualization and inspection capabilities for entities when using Unity Editor with Odin Inspector. It offers readable debug views for tags, values, and behaviors with interactive editing capabilities.

## Key Features

- **Odin Inspector Integration** – Rich debug visualization in Unity Inspector
- **Interactive Debugging** – Live editing of entity state in Inspector
- **Sorted Display** – Alphabetically sorted debug information
- **Searchable Collections** – Find specific tags, values, or behaviors quickly
- **Custom Removal Functions** – Remove items directly from Inspector
- **Conditional Compilation** – Only active in Unity Editor with Odin Inspector

---

## Debug Visualization Components

### Debug Properties

```csharp
[ShowInInspector, LabelText("Name")]
private string DebugName { get; set; }           // Entity name editor

[ShowInInspector, LabelText("Spawned")] 
private bool DebugSpawned => IsSpawned;          // Spawn state display

[ShowInInspector, LabelText("Active")]
private bool DebugActive => IsActive;            // Active state display
```

### Debug Structures

```csharp
// Tag representation
private readonly struct DebugTag : IComparable<DebugTag>
{
    internal readonly string name;               // Human-readable name
    internal readonly int id;                    // Numeric ID
}

// Value representation  
private struct DebugValue : IComparable<DebugValue>
{
    public readonly string name;                 // Human-readable name
    public object value;                         // Editable value
    internal readonly int id;                    // Numeric ID
}

// Behavior representation
private struct DebugBehaviour : IComparable<DebugBehaviour>
{
    public string name;                          // Type name
    internal readonly IEntityBehaviour value;   // Behavior reference
}
```

---

## Debug Collections

### Tags Debug View

```csharp
[Searchable, LabelText("Tags"), ShowInInspector, PropertyOrder(100)]
[ListDrawerSettings(
    CustomRemoveElementFunction = nameof(DebugDelTag),
    CustomRemoveIndexFunction = nameof(DebugDelTagAt),
    HideAddButton = true
)]
private List<DebugTag> DebugTags { get; set; }
```

**Features:**
- Displays all entity tags sorted alphabetically
- Shows both human-readable names and numeric IDs  
- Searchable for finding specific tags
- Custom removal functions for interactive editing
- Read-only display (add button hidden)

### Values Debug View

```csharp
[Searchable, LabelText("Values"), ShowInInspector, PropertyOrder(100)]
[ListDrawerSettings(
    CustomRemoveElementFunction = nameof(DebugDelValue),
    CustomRemoveIndexFunction = nameof(DebugDelValueAt), 
    HideAddButton = true
)]
private List<DebugValue> DebugValues { get; set; }
```

**Features:**
- Displays all entity values sorted alphabetically
- Shows name-value pairs in horizontal layout
- Values are editable in Inspector (updates entity)
- Searchable for finding specific values
- Custom removal functions for interactive editing

### Behaviors Debug View

```csharp
[Searchable, LabelText("Behaviours"), ShowInInspector, PropertyOrder(100)]  
[ListDrawerSettings(
    CustomRemoveElementFunction = nameof(DebugDelBehaviour),
    CustomRemoveIndexFunction = nameof(DebugDelBehaviourAt),
    HideAddButton = true
)]
private List<DebugBehaviour> DebugBehaviours { get; set; }
```

**Features:**
- Displays all entity behaviors sorted alphabetically
- Shows behavior type names
- Searchable for finding specific behaviors
- Custom removal functions for interactive editing
- Cached list for performance optimization

---

## Usage Patterns

### Debug-Enabled Entity Development

```csharp
#if UNITY_EDITOR && ODIN_INSPECTOR
public class DebugEntityExample : MonoBehaviour
{
    [ShowInInspector]
    private Entity debugEntity;
    
    [Button("Create Debug Entity")]
    private void CreateDebugEntity()
    {
        debugEntity = new Entity("Debug Entity");
        
        // Add sample data for debugging
        debugEntity.AddTag(EntityNames.NameToId("Player"));
        debugEntity.AddTag(EntityNames.NameToId("Alive"));
        debugEntity.AddTag(EntityNames.NameToId("Controllable"));
        
        debugEntity.SetValue(EntityNames.NameToId("Health"), 100);
        debugEntity.SetValue(EntityNames.NameToId("Position"), Vector3.zero);
        debugEntity.SetValue(EntityNames.NameToId("Speed"), 5.5f);
        debugEntity.SetValue(EntityNames.NameToId("Name"), "Test Player");
        
        debugEntity.AddBehaviour(new MovementBehaviour());
        debugEntity.AddBehaviour(new HealthBehaviour());
        debugEntity.AddBehaviour(new InputBehaviour());
        
        debugEntity.Spawn();
        debugEntity.Activate();
    }
    
    [Button("Modify Entity State")]
    private void ModifyEntityState()
    {
        if (debugEntity != null)
        {
            // Add debug tag
            debugEntity.AddTag(EntityNames.NameToId("Debug"));
            
            // Modify value
            var currentHealth = debugEntity.GetValue<int>(EntityNames.NameToId("Health"));
            debugEntity.SetValue(EntityNames.NameToId("Health"), currentHealth - 10);
            
            // Add debug behavior
            debugEntity.AddBehaviour(new DebugBehaviour());
        }
    }
}
#endif
```

### Custom Debug Inspector

```csharp
#if UNITY_EDITOR && ODIN_INSPECTOR
[CustomEditor(typeof(EntityComponent))]
public class EntityComponentEditor : OdinEditor
{
    public override void OnInspectorGUI()
    {
        var component = target as EntityComponent;
        if (component?.Entity != null)
        {
            DrawEntityDebugInfo(component.Entity);
        }
        
        base.OnInspectorGUI();
    }
    
    private void DrawEntityDebugInfo(Entity entity)
    {
        EditorGUILayout.Space();
        EditorGUILayout.LabelField("Entity Debug Information", EditorStyles.boldLabel);
        
        // Entity state
        EditorGUILayout.LabelField($"Name: {entity.Name}");
        EditorGUILayout.LabelField($"Spawned: {entity.IsSpawned}");
        EditorGUILayout.LabelField($"Active: {entity.IsActive}");
        
        // Statistics
        EditorGUILayout.LabelField($"Tags: {entity.TagCount}");
        EditorGUILayout.LabelField($"Values: {entity.ValueCount}");
        EditorGUILayout.LabelField($"Behaviours: {entity.BehaviourCount}");
        
        // Quick actions
        EditorGUILayout.Space();
        if (GUILayout.Button("Log Entity State"))
        {
            LogEntityState(entity);
        }
        
        if (GUILayout.Button("Validate Entity"))
        {
            ValidateEntity(entity);
        }
    }
    
    private void LogEntityState(Entity entity)
    {
        Debug.Log($"=== Entity State: {entity.Name} ===");
        Debug.Log($"Spawned: {entity.IsSpawned}, Active: {entity.IsActive}");
        
        // Log tags
        var tags = entity.GetTags();
        Debug.Log($"Tags ({tags.Length}): {string.Join(", ", tags.Select(t => EntityNames.IdToName(t)))}");
        
        // Log values
        var values = entity.GetValues();
        Debug.Log($"Values ({values.Length}):");
        foreach (var kvp in values)
        {
            string name = EntityNames.IdToName(kvp.Key);
            Debug.Log($"  {name}: {kvp.Value} ({kvp.Value?.GetType().Name})");
        }
        
        // Log behaviors
        var behaviors = entity.GetBehaviours();
        Debug.Log($"Behaviours ({behaviors.Length}): {string.Join(", ", behaviors.Select(b => b.GetType().Name))}");
    }
    
    private void ValidateEntity(Entity entity)
    {
        bool isValid = true;
        var issues = new List<string>();
        
        // Validate state consistency
        if (entity.IsActive && !entity.IsSpawned)
        {
            issues.Add("Entity is active but not spawned");
            isValid = false;
        }
        
        // Validate required values
        if (!entity.HasValue(EntityNames.NameToId("Position")))
        {
            issues.Add("Missing Position value");
            isValid = false;
        }
        
        // Validate behavior compatibility
        if (entity.HasBehaviour<MovementBehaviour>() && !entity.HasValue(EntityNames.NameToId("Speed")))
        {
            issues.Add("MovementBehaviour requires Speed value");
            isValid = false;
        }
        
        if (isValid)
        {
            Debug.Log($"Entity {entity.Name} validation passed");
        }
        else
        {
            Debug.LogWarning($"Entity {entity.Name} validation failed:\n{string.Join("\n", issues)}");
        }
    }
}
#endif
```

### Runtime Debug Utilities

```csharp
public static class EntityDebugUtils
{
    public static void PrintEntityInfo(Entity entity)
    {
        if (entity == null)
        {
            Debug.Log("Entity is null");
            return;
        }
        
        Debug.Log($"=== Entity: {entity.Name} ===");
        Debug.Log($"Instance ID: {entity.InstanceID}");
        Debug.Log($"Spawned: {entity.IsSpawned}");
        Debug.Log($"Active: {entity.IsActive}");
        
        PrintEntityTags(entity);
        PrintEntityValues(entity);  
        PrintEntityBehaviours(entity);
    }
    
    private static void PrintEntityTags(Entity entity)
    {
        var tags = entity.GetTags();
        Debug.Log($"Tags ({tags.Length}):");
        
        foreach (int tagId in tags)
        {
            string tagName = EntityNames.IdToName(tagId);
            Debug.Log($"  - {tagName} (ID: {tagId})");
        }
    }
    
    private static void PrintEntityValues(Entity entity)
    {
        var values = entity.GetValues();
        Debug.Log($"Values ({values.Length}):");
        
        foreach (var kvp in values.OrderBy(v => EntityNames.IdToName(v.Key)))
        {
            string valueName = EntityNames.IdToName(kvp.Key);
            string typeName = kvp.Value?.GetType().Name ?? "null";
            Debug.Log($"  - {valueName}: {kvp.Value} ({typeName})");
        }
    }
    
    private static void PrintEntityBehaviours(Entity entity)
    {
        var behaviours = entity.GetBehaviours();
        Debug.Log($"Behaviours ({behaviours.Length}):");
        
        foreach (var behaviour in behaviours.OrderBy(b => b.GetType().Name))
        {
            string typeName = behaviour.GetType().Name;
            var interfaces = behaviour.GetType().GetInterfaces()
                .Where(i => i.Namespace == "Atomic.Entities")
                .Select(i => i.Name);
            
            Debug.Log($"  - {typeName} ({string.Join(", ", interfaces)})");
        }
    }
    
    public static void CompareEntities(Entity entity1, Entity entity2)
    {
        Debug.Log($"=== Comparing {entity1?.Name} vs {entity2?.Name} ===");
        
        if (entity1 == null || entity2 == null)
        {
            Debug.Log("One or both entities are null");
            return;
        }
        
        CompareTags(entity1, entity2);
        CompareValues(entity1, entity2);
        CompareBehaviours(entity1, entity2);
    }
    
    private static void CompareTags(Entity entity1, Entity entity2)
    {
        var tags1 = new HashSet<int>(entity1.GetTags());
        var tags2 = new HashSet<int>(entity2.GetTags());
        
        var onlyIn1 = tags1.Except(tags2);
        var onlyIn2 = tags2.Except(tags1);
        var common = tags1.Intersect(tags2);
        
        Debug.Log($"Tag comparison:");
        Debug.Log($"  Common ({common.Count()}): {string.Join(", ", common.Select(EntityNames.IdToName))}");
        Debug.Log($"  Only in {entity1.Name} ({onlyIn1.Count()}): {string.Join(", ", onlyIn1.Select(EntityNames.IdToName))}");
        Debug.Log($"  Only in {entity2.Name} ({onlyIn2.Count()}): {string.Join(", ", onlyIn2.Select(EntityNames.IdToName))}");
    }
    
    private static void CompareValues(Entity entity1, Entity entity2)
    {
        var values1 = entity1.GetValues().ToDictionary(kvp => kvp.Key, kvp => kvp.Value);
        var values2 = entity2.GetValues().ToDictionary(kvp => kvp.Key, kvp => kvp.Value);
        
        var allKeys = values1.Keys.Union(values2.Keys);
        
        Debug.Log($"Value comparison:");
        foreach (int key in allKeys.OrderBy(k => EntityNames.IdToName(k)))
        {
            string name = EntityNames.IdToName(key);
            bool has1 = values1.TryGetValue(key, out var value1);
            bool has2 = values2.TryGetValue(key, out var value2);
            
            if (has1 && has2)
            {
                bool equal = Equals(value1, value2);
                Debug.Log($"  {name}: {value1} vs {value2} {(equal ? "✓" : "✗")}");
            }
            else if (has1)
            {
                Debug.Log($"  {name}: {value1} vs <missing>");
            }
            else
            {
                Debug.Log($"  {name}: <missing> vs {value2}");
            }
        }
    }
    
    private static void CompareBehaviours(Entity entity1, Entity entity2)
    {
        var behaviours1 = entity1.GetBehaviours().Select(b => b.GetType()).ToHashSet();
        var behaviours2 = entity2.GetBehaviours().Select(b => b.GetType()).ToHashSet();
        
        var onlyIn1 = behaviours1.Except(behaviours2);
        var onlyIn2 = behaviours2.Except(behaviours1);
        var common = behaviours1.Intersect(behaviours2);
        
        Debug.Log($"Behaviour comparison:");
        Debug.Log($"  Common ({common.Count()}): {string.Join(", ", common.Select(t => t.Name))}");
        Debug.Log($"  Only in {entity1.Name} ({onlyIn1.Count()}): {string.Join(", ", onlyIn1.Select(t => t.Name))}");
        Debug.Log($"  Only in {entity2.Name} ({onlyIn2.Count()}): {string.Join(", ", onlyIn2.Select(t => t.Name))}");
    }
}
```

## Debug View Features

### Interactive Editing
- **Tag removal** – Click to remove tags directly from Inspector
- **Value editing** – Modify values inline and see changes immediately  
- **Behavior removal** – Remove behaviors directly from Inspector
- **Name editing** – Change entity name through debug property

### Visual Organization
- **Alphabetical sorting** – All collections sorted for easy browsing
- **Searchable lists** – Find items quickly with search functionality
- **Grouped display** – Related information grouped together
- **Clean layout** – Optimized Inspector layout for readability

### Performance Optimization
- **Conditional compilation** – Only active when Odin Inspector available
- **Cached collections** – Debug behavior list cached for efficiency
- **Lazy evaluation** – Debug properties computed on-demand
- **Memory efficient** – Minimal memory overhead for debug features

## Compilation Conditions

The debug functionality is only available when:
```csharp
#if UNITY_EDITOR && ODIN_INSPECTOR
```

This ensures:
- **No runtime overhead** in builds
- **Odin Inspector dependency** managed properly
- **Editor-only features** don't leak to runtime
- **Conditional compilation** for clean builds

## Best Practices

### Debug Usage
1. **Use for development** – Debug features are development tools only
2. **Inspector interaction** – Take advantage of interactive editing
3. **Search functionality** – Use search to find specific items quickly
4. **State monitoring** – Monitor entity state changes in real-time

### Performance Considerations
1. **Editor only** – No impact on runtime performance
2. **Lazy computation** – Debug properties computed only when viewed
3. **Efficient caching** – Collections cached where appropriate
4. **Minimal allocation** – Debug structures optimized for memory usage

## Limitations

### Editor Dependency
- **Odin Inspector required** – Features unavailable without Odin
- **Unity Editor only** – No debug features in builds
- **Inspector access** – Must view in Inspector to see debug info

### Functional Limitations
- **Read-only where appropriate** – Some properties are display-only
- **No add functionality** – Cannot add new items through debug interface
- **Type constraints** – Value editing limited by Inspector capabilities