# 📚 EntityViewCatalog

`EntityViewCatalog` is a management system that provides centralized registration, lookup, and creation of entity view types. It serves as a registry for different view implementations and enables dynamic view creation based on entity characteristics or requirements.

## Key Features

- **Centralized View Registry** – Single source for all view type management
- **Dynamic View Creation** – Create views based on runtime criteria
- **Type-Safe Registration** – Compile-time type safety for view registration
- **Flexible Lookup** – Multiple strategies for view type resolution
- **Memory Efficient** – Lazy loading and caching of view definitions

---

## Core Structure

```csharp
public class EntityViewCatalog
{
    private readonly Dictionary<string, ViewDefinition> viewDefinitions;
    private readonly Dictionary<Type, string> typeToName;
    private readonly IEntityViewPool viewPool;
    
    public void RegisterView<T>(string name, GameObject prefab) where T : IReadOnlyEntityView;
    public string ResolveViewName(IEntity entity);
    public IReadOnlyEntityView CreateView(string viewName);
    public IReadOnlyEntityView CreateViewForEntity(IEntity entity);
}
```

## Implementation Examples

### Basic View Catalog

```csharp
[CreateAssetMenu(menuName = "Atomic/Views/View Catalog")]
public class EntityViewCatalog : ScriptableObject
{
    [System.Serializable]
    public class ViewDefinition
    {
        public string viewName;
        public GameObject prefab;
        public ViewCategory category;
        public string[] requiredTags;
        public string[] excludedTags;
        public int priority = 1;
    }
    
    public enum ViewCategory
    {
        World3D,
        UI,
        Effect,
        Debug
    }
    
    [Header("View Definitions")]
    [SerializeField] private ViewDefinition[] viewDefinitions;
    
    [Header("Default Views")]
    [SerializeField] private GameObject defaultWorldView;
    [SerializeField] private GameObject defaultUIView;
    
    private Dictionary<string, ViewDefinition> definitionMap;
    private Dictionary<ViewCategory, List<ViewDefinition>> categoryMap;
    
    void OnEnable()
    {
        InitializeCatalog();
    }
    
    private void InitializeCatalog()
    {
        definitionMap = new Dictionary<string, ViewDefinition>();
        categoryMap = new Dictionary<ViewCategory, List<ViewDefinition>>();
        
        // Initialize category lists
        foreach (ViewCategory category in System.Enum.GetValues(typeof(ViewCategory)))
        {
            categoryMap[category] = new List<ViewDefinition>();
        }
        
        // Build lookup tables
        foreach (var definition in viewDefinitions)
        {
            definitionMap[definition.viewName] = definition;
            categoryMap[definition.category].Add(definition);
        }
        
        Debug.Log($"Initialized view catalog with {viewDefinitions.Length} view definitions");
    }
    
    public ViewDefinition GetViewDefinition(string viewName)
    {
        return definitionMap.TryGetValue(viewName, out var definition) ? definition : null;
    }
    
    public ViewDefinition ResolveViewDefinitionForEntity(IEntity entity)
    {
        var candidates = new List<(ViewDefinition definition, int score)>();
        
        foreach (var definition in viewDefinitions)
        {
            int score = CalculateMatchScore(entity, definition);
            if (score > 0)
            {
                candidates.Add((definition, score));
            }
        }
        
        if (candidates.Count == 0)
        {
            return GetDefaultDefinition(entity);
        }
        
        // Return highest scoring definition
        candidates.Sort((a, b) => b.score.CompareTo(a.score));
        return candidates[0].definition;
    }
    
    private int CalculateMatchScore(IEntity entity, ViewDefinition definition)
    {
        int score = definition.priority;
        
        // Check required tags
        foreach (var tagName in definition.requiredTags)
        {
            int tagId = EntityNames.NameToId(tagName);
            if (!entity.HasTag(tagId))
            {
                return 0; // Required tag missing
            }
            score += 10; // Bonus for matching required tag
        }
        
        // Check excluded tags
        foreach (var tagName in definition.excludedTags)
        {
            int tagId = EntityNames.NameToId(tagName);
            if (entity.HasTag(tagId))
            {
                return 0; // Excluded tag present
            }
        }
        
        return score;
    }
    
    private ViewDefinition GetDefaultDefinition(IEntity entity)
    {
        // Determine category based on entity characteristics
        if (entity.HasTag(EntityTags.UI_ELEMENT))
        {
            return new ViewDefinition { viewName = "DefaultUI", prefab = defaultUIView, category = ViewCategory.UI };
        }
        
        return new ViewDefinition { viewName = "DefaultWorld", prefab = defaultWorldView, category = ViewCategory.World3D };
    }
    
    public List<ViewDefinition> GetViewsByCategory(ViewCategory category)
    {
        return categoryMap.TryGetValue(category, out var list) ? new List<ViewDefinition>(list) : new List<ViewDefinition>();
    }
    
    public GameObject GetViewPrefab(string viewName)
    {
        var definition = GetViewDefinition(viewName);
        return definition?.prefab;
    }
}
```

### Dynamic View Catalog with Rules

```csharp
public class DynamicEntityViewCatalog : MonoBehaviour
{
    [System.Serializable]
    public class ViewRule
    {
        public string ruleName;
        public ViewCondition[] conditions;
        public string targetViewName;
        public int priority = 1;
        public bool enabled = true;
    }
    
    [System.Serializable]
    public class ViewCondition
    {
        public ConditionType type;
        public string key;
        public object expectedValue;
        public ComparisonOperator comparison = ComparisonOperator.Equals;
        
        public enum ConditionType
        {
            HasTag,
            HasValue,
            ValueEquals,
            ValueGreaterThan,
            ValueLessThan
        }
        
        public enum ComparisonOperator
        {
            Equals,
            NotEquals,
            GreaterThan,
            GreaterOrEqual,
            LessThan,
            LessOrEqual
        }
    }
    
    [Header("View Rules")]
    [SerializeField] private ViewRule[] viewRules;
    
    [Header("Catalog Reference")]
    [SerializeField] private EntityViewCatalog baseCatalog;
    
    private List<ViewRule> sortedRules;
    
    void Start()
    {
        InitializeRules();
    }
    
    private void InitializeRules()
    {
        sortedRules = new List<ViewRule>(viewRules);
        sortedRules.RemoveAll(rule => !rule.enabled);
        sortedRules.Sort((a, b) => b.priority.CompareTo(a.priority));
        
        Debug.Log($"Initialized {sortedRules.Count} view rules");
    }
    
    public string ResolveViewNameForEntity(IEntity entity)
    {
        foreach (var rule in sortedRules)
        {
            if (EvaluateRule(entity, rule))
            {
                Debug.Log($"Entity {entity.Name} matched rule '{rule.ruleName}' -> {rule.targetViewName}");
                return rule.targetViewName;
            }
        }
        
        // Fall back to base catalog
        var baseDefinition = baseCatalog.ResolveViewDefinitionForEntity(entity);
        return baseDefinition?.viewName ?? "DefaultWorld";
    }
    
    private bool EvaluateRule(IEntity entity, ViewRule rule)
    {
        foreach (var condition in rule.conditions)
        {
            if (!EvaluateCondition(entity, condition))
            {
                return false;
            }
        }
        
        return true;
    }
    
    private bool EvaluateCondition(IEntity entity, ViewCondition condition)
    {
        switch (condition.type)
        {
            case ViewCondition.ConditionType.HasTag:
                int tagId = EntityNames.NameToId(condition.key);
                return entity.HasTag(tagId);
                
            case ViewCondition.ConditionType.HasValue:
                int valueId = EntityNames.NameToId(condition.key);
                return entity.HasValue(valueId);
                
            case ViewCondition.ConditionType.ValueEquals:
                return EvaluateValueCondition(entity, condition, ViewCondition.ComparisonOperator.Equals);
                
            case ViewCondition.ConditionType.ValueGreaterThan:
                return EvaluateValueCondition(entity, condition, ViewCondition.ComparisonOperator.GreaterThan);
                
            case ViewCondition.ConditionType.ValueLessThan:
                return EvaluateValueCondition(entity, condition, ViewCondition.ComparisonOperator.LessThan);
                
            default:
                return false;
        }
    }
    
    private bool EvaluateValueCondition(IEntity entity, ViewCondition condition, ViewCondition.ComparisonOperator op)
    {
        int valueId = EntityNames.NameToId(condition.key);
        
        if (!entity.TryGetValue(valueId, out object currentValue))
        {
            return false;
        }
        
        if (currentValue == null || condition.expectedValue == null)
        {
            return (currentValue == null) == (condition.expectedValue == null);
        }
        
        try
        {
            switch (op)
            {
                case ViewCondition.ComparisonOperator.Equals:
                    return currentValue.Equals(condition.expectedValue);
                    
                case ViewCondition.ComparisonOperator.NotEquals:
                    return !currentValue.Equals(condition.expectedValue);
                    
                case ViewCondition.ComparisonOperator.GreaterThan:
                    return ((IComparable)currentValue).CompareTo(condition.expectedValue) > 0;
                    
                case ViewCondition.ComparisonOperator.LessThan:
                    return ((IComparable)currentValue).CompareTo(condition.expectedValue) < 0;
                    
                default:
                    return false;
            }
        }
        catch
        {
            return false;
        }
    }
}
```

### Cached View Catalog

```csharp
public class CachedEntityViewCatalog : MonoBehaviour
{
    private EntityViewCatalog baseCatalog;
    private Dictionary<string, string> entityToViewCache;
    private Dictionary<string, WeakReference> viewInstanceCache;
    
    [Header("Cache Settings")]
    [SerializeField] private int maxCacheSize = 1000;
    [SerializeField] private float cacheCleanupInterval = 60f;
    
    void Start()
    {
        baseCatalog = FindObjectOfType<EntityViewCatalog>();
        entityToViewCache = new Dictionary<string, string>();
        viewInstanceCache = new Dictionary<string, WeakReference>();
        
        InvokeRepeating(nameof(CleanupCache), cacheCleanupInterval, cacheCleanupInterval);
    }
    
    public string ResolveViewNameForEntity(IEntity entity)
    {
        string cacheKey = GenerateCacheKey(entity);
        
        if (entityToViewCache.TryGetValue(cacheKey, out string cachedViewName))
        {
            return cachedViewName;
        }
        
        // Resolve using base catalog
        var definition = baseCatalog.ResolveViewDefinitionForEntity(entity);
        string resolvedViewName = definition?.viewName ?? "DefaultWorld";
        
        // Cache result
        CacheEntityViewName(cacheKey, resolvedViewName);
        
        return resolvedViewName;
    }
    
    public IReadOnlyEntityView GetCachedViewInstance(string viewName)
    {
        if (viewInstanceCache.TryGetValue(viewName, out var weakRef))
        {
            if (weakRef.Target is IReadOnlyEntityView view && view != null)
            {
                return view;
            }
            else
            {
                // Remove dead reference
                viewInstanceCache.Remove(viewName);
            }
        }
        
        return null;
    }
    
    public void CacheViewInstance(string viewName, IReadOnlyEntityView view)
    {
        viewInstanceCache[viewName] = new WeakReference(view);
    }
    
    private string GenerateCacheKey(IEntity entity)
    {
        // Generate cache key based on relevant entity characteristics
        var keyBuilder = new System.Text.StringBuilder();
        keyBuilder.Append(entity.GetType().Name);
        
        // Include relevant tags in cache key
        var relevantTags = new[]
        {
            EntityTags.PLAYER, EntityTags.ENEMY, EntityTags.NPC,
            EntityTags.UI_ELEMENT, EntityTags.EFFECT
        };
        
        foreach (var tag in relevantTags)
        {
            if (entity.HasTag(tag))
            {
                keyBuilder.Append($"_TAG_{tag}");
            }
        }
        
        // Include relevant values in cache key
        if (entity.TryGetValue<string>(EntityNames.ENTITY_TYPE, out var entityType))
        {
            keyBuilder.Append($"_TYPE_{entityType}");
        }
        
        if (entity.TryGetValue<int>(EntityNames.LEVEL, out var level))
        {
            keyBuilder.Append($"_LEVEL_{level}");
        }
        
        return keyBuilder.ToString();
    }
    
    private void CacheEntityViewName(string cacheKey, string viewName)
    {
        if (entityToViewCache.Count >= maxCacheSize)
        {
            // Remove oldest entries (simple FIFO)
            var firstKey = entityToViewCache.Keys.First();
            entityToViewCache.Remove(firstKey);
        }
        
        entityToViewCache[cacheKey] = viewName;
    }
    
    private void CleanupCache()
    {
        // Clean up dead weak references
        var deadKeys = new List<string>();
        
        foreach (var kvp in viewInstanceCache)
        {
            if (kvp.Value.Target == null)
            {
                deadKeys.Add(kvp.Key);
            }
        }
        
        foreach (var key in deadKeys)
        {
            viewInstanceCache.Remove(key);
        }
        
        if (deadKeys.Count > 0)
        {
            Debug.Log($"Cleaned up {deadKeys.Count} dead view references from cache");
        }
    }
    
    [ContextMenu("Clear Cache")]
    public void ClearCache()
    {
        entityToViewCache.Clear();
        viewInstanceCache.Clear();
        Debug.Log("Cleared view catalog cache");
    }
    
    [ContextMenu("Print Cache Statistics")]
    public void PrintCacheStatistics()
    {
        Debug.Log($"Cache Statistics:");
        Debug.Log($"  Entity->View mappings: {entityToViewCache.Count}");
        Debug.Log($"  Cached view instances: {viewInstanceCache.Count}");
        Debug.Log($"  Memory usage: ~{(entityToViewCache.Count + viewInstanceCache.Count) * 50} bytes");
    }
}
```

## Integration Patterns

### View Factory Integration
```csharp
public class CatalogViewFactory : MonoBehaviour
{
    [SerializeField] private EntityViewCatalog catalog;
    [SerializeField] private IEntityViewPool viewPool;
    
    public IReadOnlyEntityView CreateViewForEntity(IEntity entity)
    {
        string viewName = catalog.ResolveViewNameForEntity(entity);
        var view = viewPool.Rent(viewName);
        
        if (view is IEntityView bindableView)
        {
            bindableView.BindToEntity(entity);
        }
        
        return view;
    }
}
```

### Editor Integration
```csharp
#if UNITY_EDITOR
[CustomEditor(typeof(EntityViewCatalog))]
public class EntityViewCatalogEditor : Editor
{
    public override void OnInspectorGUI()
    {
        base.OnInspectorGUI();
        
        var catalog = (EntityViewCatalog)target;
        
        EditorGUILayout.Space();
        EditorGUILayout.LabelField("Catalog Statistics", EditorStyles.boldLabel);
        
        if (Application.isPlaying)
        {
            var definitions = catalog.GetAllDefinitions();
            EditorGUILayout.LabelField($"Total Definitions: {definitions.Count}");
            
            foreach (var category in System.Enum.GetValues(typeof(EntityViewCatalog.ViewCategory)))
            {
                var categoryViews = catalog.GetViewsByCategory((EntityViewCatalog.ViewCategory)category);
                EditorGUILayout.LabelField($"{category}: {categoryViews.Count}");
            }
        }
        
        if (GUILayout.Button("Validate All Prefabs"))
        {
            ValidateAllPrefabs(catalog);
        }
    }
    
    private void ValidateAllPrefabs(EntityViewCatalog catalog)
    {
        var definitions = catalog.GetAllDefinitions();
        int validCount = 0;
        int invalidCount = 0;
        
        foreach (var definition in definitions)
        {
            if (definition.prefab != null)
            {
                var view = definition.prefab.GetComponent<IReadOnlyEntityView>();
                if (view != null)
                {
                    validCount++;
                }
                else
                {
                    Debug.LogError($"Prefab '{definition.prefab.name}' does not have a view component");
                    invalidCount++;
                }
            }
            else
            {
                Debug.LogError($"View definition '{definition.viewName}' has null prefab");
                invalidCount++;
            }
        }
        
        Debug.Log($"Prefab validation complete: {validCount} valid, {invalidCount} invalid");
    }
}
#endif
```

## Best Practices

1. **Clear Naming** – Use consistent and descriptive view names
2. **Priority System** – Use priorities to handle overlapping rules
3. **Cache Management** – Implement caching for frequently resolved views
4. **Validation** – Validate view definitions and prefabs
5. **Performance** – Optimize lookup algorithms for large catalogs
6. **Modularity** – Separate catalog definitions from runtime logic
7. **Extensibility** – Design for easy addition of new view types

## Performance Considerations

### Lookup Optimization
- Use hash tables for O(1) view name lookups
- Cache entity->view name mappings
- Pre-sort rules by priority

### Memory Management
- Use weak references for cached instances
- Implement cache size limits
- Regular cleanup of dead references

## Notes

- EntityViewCatalog serves as the central registry for all view types
- Supports both static definitions and dynamic rule-based resolution
- Designed for extensibility and performance at scale
- Integration with caching systems improves runtime performance
- Editor tools help validate and manage large view catalogs