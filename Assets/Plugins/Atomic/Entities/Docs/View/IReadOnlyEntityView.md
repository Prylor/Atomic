# 👁️ IReadOnlyEntityView

`IReadOnlyEntityView` is an interface that defines a read-only contract for entity view implementations. It provides access to view properties and entity association without allowing modification of the view's internal state, making it suitable for systems that need to inspect but not modify views.

## Key Features

- **Read-Only Access** – Immutable view interface for safe inspection
- **Entity Association** – Access to bound entity without modification rights
- **Type Safety** – Compile-time interface compliance
- **System Integration** – Compatible with view pools and management systems
- **Minimal Interface** – Focused on essential read-only operations

---

## Interface Definition

```csharp
public interface IReadOnlyEntityView
{
    /// <summary>
    /// Gets the entity associated with this view, if any.
    /// </summary>
    IEntity Entity { get; }
    
    /// <summary>
    /// Gets a value indicating whether this view is currently bound to an entity.
    /// </summary>
    bool IsBound { get; }
}
```

## Implementation Examples

### Basic Read-Only View Implementation

```csharp
public class ReadOnlyEntityView : MonoBehaviour, IReadOnlyEntityView
{
    private IEntity boundEntity;
    
    public IEntity Entity => boundEntity;
    public bool IsBound => boundEntity != null;
    
    // Internal method for binding (not part of read-only interface)
    internal void BindToEntity(IEntity entity)
    {
        boundEntity = entity;
        OnEntityBound(entity);
    }
    
    // Internal method for unbinding (not part of read-only interface)  
    internal void UnbindFromEntity()
    {
        var previousEntity = boundEntity;
        boundEntity = null;
        OnEntityUnbound(previousEntity);
    }
    
    protected virtual void OnEntityBound(IEntity entity)
    {
        // Subclasses can override for custom binding behavior
    }
    
    protected virtual void OnEntityUnbound(IEntity entity)
    {
        // Subclasses can override for custom unbinding behavior
    }
}
```

### View Inspector System

```csharp
public class EntityViewInspector : MonoBehaviour
{
    [Header("Inspector Settings")]
    [SerializeField] private bool inspectViewsAutomatically = true;
    [SerializeField] private float inspectionInterval = 1.0f;
    [SerializeField] private bool logInspectionResults = false;
    
    private List<IReadOnlyEntityView> inspectedViews = new List<IReadOnlyEntityView>();
    private Dictionary<IReadOnlyEntityView, ViewInspectionData> inspectionData = 
        new Dictionary<IReadOnlyEntityView, ViewInspectionData>();
    
    [System.Serializable]
    public class ViewInspectionData
    {
        public string viewName;
        public string entityName;
        public bool isBound;
        public int entityInstanceId;
        public Vector3 worldPosition;
        public bool isActive;
        public float lastInspectionTime;
    }
    
    void Start()
    {
        if (inspectViewsAutomatically)
        {
            InvokeRepeating(nameof(InspectAllViews), 0f, inspectionInterval);
        }
    }
    
    private void InspectAllViews()
    {
        // Find all views in scene
        var allViews = FindObjectsOfType<MonoBehaviour>()
            .OfType<IReadOnlyEntityView>()
            .ToList();
        
        inspectedViews.Clear();
        
        foreach (var view in allViews)
        {
            var data = InspectView(view);
            inspectedViews.Add(view);
            inspectionData[view] = data;
            
            if (logInspectionResults)
            {
                LogInspectionData(data);
            }
        }
        
        if (logInspectionResults)
        {
            Debug.Log($"Inspected {inspectedViews.Count} views");
        }
    }
    
    private ViewInspectionData InspectView(IReadOnlyEntityView view)
    {
        var data = new ViewInspectionData
        {
            lastInspectionTime = Time.time,
            isBound = view.IsBound
        };
        
        if (view is MonoBehaviour mb)
        {
            data.viewName = mb.name;
            data.worldPosition = mb.transform.position;
            data.isActive = mb.gameObject.activeInHierarchy;
        }
        
        if (view.IsBound && view.Entity != null)
        {
            data.entityName = view.Entity.Name;
            data.entityInstanceId = view.Entity.InstanceID;
        }
        else
        {
            data.entityName = "None";
            data.entityInstanceId = -1;
        }
        
        return data;
    }
    
    private void LogInspectionData(ViewInspectionData data)
    {
        string status = data.isBound ? $"bound to {data.entityName}" : "unbound";
        Debug.Log($"View '{data.viewName}' is {status} at {data.worldPosition}");
    }
    
    public List<IReadOnlyEntityView> GetUnboundViews()
    {
        return inspectedViews.Where(view => !view.IsBound).ToList();
    }
    
    public List<IReadOnlyEntityView> GetBoundViews()
    {
        return inspectedViews.Where(view => view.IsBound).ToList();
    }
    
    public ViewInspectionData GetInspectionData(IReadOnlyEntityView view)
    {
        return inspectionData.TryGetValue(view, out var data) ? data : null;
    }
    
    // Manual inspection methods
    [ContextMenu("Inspect All Views Now")]
    public void InspectAllViewsManual()
    {
        InspectAllViews();
    }
    
    [ContextMenu("Print View Summary")]
    public void PrintViewSummary()
    {
        var boundCount = inspectedViews.Count(view => view.IsBound);
        var unboundCount = inspectedViews.Count - boundCount;
        
        Debug.Log($"View Summary: {inspectedViews.Count} total, {boundCount} bound, {unboundCount} unbound");
        
        foreach (var view in inspectedViews.Take(10)) // Show first 10
        {
            var data = inspectionData[view];
            Debug.Log($"  - {data.viewName}: {(data.isBound ? data.entityName : "unbound")}");
        }
    }
}
```

### View Query System

```csharp
public class EntityViewQuerySystem : MonoBehaviour
{
    private readonly List<IReadOnlyEntityView> cachedViews = new List<IReadOnlyEntityView>();
    private float lastCacheUpdate = 0f;
    private const float CACHE_REFRESH_INTERVAL = 1.0f;
    
    public IEnumerable<IReadOnlyEntityView> GetAllViews()
    {
        RefreshCacheIfNeeded();
        return cachedViews.AsReadOnly();
    }
    
    public IEnumerable<IReadOnlyEntityView> GetBoundViews()
    {
        RefreshCacheIfNeeded();
        return cachedViews.Where(view => view.IsBound);
    }
    
    public IEnumerable<IReadOnlyEntityView> GetUnboundViews()
    {
        RefreshCacheIfNeeded();
        return cachedViews.Where(view => !view.IsBound);
    }
    
    public IReadOnlyEntityView FindViewForEntity(IEntity entity)
    {
        RefreshCacheIfNeeded();
        return cachedViews.FirstOrDefault(view => view.IsBound && view.Entity == entity);
    }
    
    public IEnumerable<IReadOnlyEntityView> FindViewsForEntityTag(int tagId)
    {
        RefreshCacheIfNeeded();
        return cachedViews.Where(view => 
            view.IsBound && 
            view.Entity != null && 
            view.Entity.HasTag(tagId));
    }
    
    public IEnumerable<IReadOnlyEntityView> FindViewsAtPosition(Vector3 position, float radius)
    {
        RefreshCacheIfNeeded();
        return cachedViews.Where(view =>
        {
            if (view is MonoBehaviour mb)
            {
                return Vector3.Distance(mb.transform.position, position) <= radius;
            }
            return false;
        });
    }
    
    public IEnumerable<T> GetViewsOfType<T>() where T : class, IReadOnlyEntityView
    {
        RefreshCacheIfNeeded();
        return cachedViews.OfType<T>();
    }
    
    public int GetViewCount()
    {
        RefreshCacheIfNeeded();
        return cachedViews.Count;
    }
    
    public int GetBoundViewCount()
    {
        RefreshCacheIfNeeded();
        return cachedViews.Count(view => view.IsBound);
    }
    
    private void RefreshCacheIfNeeded()
    {
        if (Time.time - lastCacheUpdate > CACHE_REFRESH_INTERVAL)
        {
            RefreshCache();
        }
    }
    
    private void RefreshCache()
    {
        cachedViews.Clear();
        
        // Find all IReadOnlyEntityView implementations in scene
        var monoBehaviours = FindObjectsOfType<MonoBehaviour>();
        
        foreach (var mb in monoBehaviours)
        {
            if (mb is IReadOnlyEntityView view)
            {
                cachedViews.Add(view);
            }
        }
        
        lastCacheUpdate = Time.time;
    }
    
    // Events for view state changes (if implemented by view system)
    public static event System.Action<IReadOnlyEntityView> OnViewBound;
    public static event System.Action<IReadOnlyEntityView> OnViewUnbound;
    
    // Static methods for triggering events (called by view implementations)
    public static void NotifyViewBound(IReadOnlyEntityView view)
    {
        OnViewBound?.Invoke(view);
    }
    
    public static void NotifyViewUnbound(IReadOnlyEntityView view)
    {
        OnViewUnbound?.Invoke(view);
    }
}
```

### View Statistics System

```csharp
public class EntityViewStatistics : MonoBehaviour
{
    [System.Serializable]
    public class ViewStats
    {
        public int totalViews;
        public int boundViews;
        public int unboundViews;
        public int activeViews;
        public int inactiveViews;
        public Dictionary<string, int> viewsByType;
        public Dictionary<string, int> viewsByEntityType;
        public float lastUpdateTime;
    }
    
    [Header("Statistics Settings")]
    [SerializeField] private bool enableStatistics = true;
    [SerializeField] private float updateInterval = 2.0f;
    [SerializeField] private bool displayUI = false;
    
    private ViewStats currentStats = new ViewStats();
    private EntityViewQuerySystem querySystem;
    
    void Start()
    {
        querySystem = FindObjectOfType<EntityViewQuerySystem>();
        if (querySystem == null)
        {
            querySystem = gameObject.AddComponent<EntityViewQuerySystem>();
        }
        
        if (enableStatistics)
        {
            InvokeRepeating(nameof(UpdateStatistics), 0f, updateInterval);
        }
    }
    
    private void UpdateStatistics()
    {
        var allViews = querySystem.GetAllViews().ToList();
        
        currentStats.totalViews = allViews.Count;
        currentStats.boundViews = allViews.Count(view => view.IsBound);
        currentStats.unboundViews = currentStats.totalViews - currentStats.boundViews;
        currentStats.activeViews = 0;
        currentStats.inactiveViews = 0;
        currentStats.lastUpdateTime = Time.time;
        
        currentStats.viewsByType = new Dictionary<string, int>();
        currentStats.viewsByEntityType = new Dictionary<string, int>();
        
        foreach (var view in allViews)
        {
            // Count by view type
            string viewType = view.GetType().Name;
            if (currentStats.viewsByType.ContainsKey(viewType))
                currentStats.viewsByType[viewType]++;
            else
                currentStats.viewsByType[viewType] = 1;
            
            // Count active/inactive
            if (view is MonoBehaviour mb)
            {
                if (mb.gameObject.activeInHierarchy)
                    currentStats.activeViews++;
                else
                    currentStats.inactiveViews++;
            }
            
            // Count by entity type
            if (view.IsBound && view.Entity != null)
            {
                string entityType = view.Entity.GetType().Name;
                if (currentStats.viewsByEntityType.ContainsKey(entityType))
                    currentStats.viewsByEntityType[entityType]++;
                else
                    currentStats.viewsByEntityType[entityType] = 1;
            }
        }
    }
    
    public ViewStats GetCurrentStatistics()
    {
        return currentStats;
    }
    
    void OnGUI()
    {
        if (!displayUI || !enableStatistics) return;
        
        GUILayout.BeginArea(new Rect(10, 10, 300, 400));
        
        GUILayout.Label("Entity View Statistics", GUI.skin.box);
        GUILayout.Label($"Total Views: {currentStats.totalViews}");
        GUILayout.Label($"Bound: {currentStats.boundViews}");
        GUILayout.Label($"Unbound: {currentStats.unboundViews}");
        GUILayout.Label($"Active: {currentStats.activeViews}");
        GUILayout.Label($"Inactive: {currentStats.inactiveViews}");
        
        GUILayout.Space(10);
        GUILayout.Label("Views by Type:", GUI.skin.box);
        
        if (currentStats.viewsByType != null)
        {
            foreach (var kvp in currentStats.viewsByType.Take(5))
            {
                GUILayout.Label($"{kvp.Key}: {kvp.Value}");
            }
        }
        
        GUILayout.Space(10);
        GUILayout.Label("Views by Entity Type:", GUI.skin.box);
        
        if (currentStats.viewsByEntityType != null)
        {
            foreach (var kvp in currentStats.viewsByEntityType.Take(5))
            {
                GUILayout.Label($"{kvp.Key}: {kvp.Value}");
            }
        }
        
        GUILayout.EndArea();
    }
    
    [ContextMenu("Print Detailed Statistics")]
    public void PrintDetailedStatistics()
    {
        UpdateStatistics();
        
        Debug.Log("=== Entity View Statistics ===");
        Debug.Log($"Total Views: {currentStats.totalViews}");
        Debug.Log($"Bound: {currentStats.boundViews}, Unbound: {currentStats.unboundViews}");
        Debug.Log($"Active: {currentStats.activeViews}, Inactive: {currentStats.inactiveViews}");
        
        Debug.Log("Views by Type:");
        if (currentStats.viewsByType != null)
        {
            foreach (var kvp in currentStats.viewsByType)
            {
                Debug.Log($"  {kvp.Key}: {kvp.Value}");
            }
        }
        
        Debug.Log("Views by Entity Type:");
        if (currentStats.viewsByEntityType != null)
        {
            foreach (var kvp in currentStats.viewsByEntityType)
            {
                Debug.Log($"  {kvp.Key}: {kvp.Value}");
            }
        }
    }
}
```

### Debug Utilities

```csharp
public static class ReadOnlyEntityViewDebug
{
    public static void LogViewInfo(IReadOnlyEntityView view, string prefix = "")
    {
        if (view == null)
        {
            Debug.Log($"{prefix}View is null");
            return;
        }
        
        string viewName = view is MonoBehaviour mb ? mb.name : view.GetType().Name;
        
        if (view.IsBound)
        {
            string entityName = view.Entity?.Name ?? "Unknown";
            int entityId = view.Entity?.InstanceID ?? -1;
            Debug.Log($"{prefix}View '{viewName}' bound to entity '{entityName}' (ID: {entityId})");
        }
        else
        {
            Debug.Log($"{prefix}View '{viewName}' is unbound");
        }
    }
    
    public static void LogViewHierarchy(IReadOnlyEntityView view, string prefix = "")
    {
        LogViewInfo(view, prefix);
        
        if (view is MonoBehaviour mb)
        {
            Debug.Log($"{prefix}  Position: {mb.transform.position}");
            Debug.Log($"{prefix}  Active: {mb.gameObject.activeInHierarchy}");
            Debug.Log($"{prefix}  Parent: {(mb.transform.parent?.name ?? "None")}");
            Debug.Log($"{prefix}  Children: {mb.transform.childCount}");
        }
    }
    
    public static void ValidateViewState(IReadOnlyEntityView view)
    {
        if (view == null)
        {
            Debug.LogError("View validation failed: view is null");
            return;
        }
        
        bool hasEntity = view.Entity != null;
        bool isBound = view.IsBound;
        
        if (isBound && !hasEntity)
        {
            Debug.LogError($"View state inconsistency: IsBound is true but Entity is null");
        }
        
        if (!isBound && hasEntity)
        {
            Debug.LogWarning($"View state inconsistency: IsBound is false but Entity is not null");
        }
        
        if (view is MonoBehaviour mb && !mb.gameObject.activeInHierarchy && isBound)
        {
            Debug.LogWarning($"View '{mb.name}' is bound but GameObject is inactive");
        }
    }
}
```

## Usage Patterns

### Safe View Inspection
```csharp
public void InspectView(IReadOnlyEntityView view)
{
    if (view == null) return;
    
    // Safe to call - read-only interface
    bool isBound = view.IsBound;
    IEntity entity = view.Entity;
    
    if (isBound && entity != null)
    {
        // Can safely read entity properties
        string entityName = entity.Name;
        int instanceId = entity.InstanceID;
        
        // Cannot modify entity through read-only view
        // entity.AddTag(...) // Would require IEntity reference
    }
}
```

### View System Integration
```csharp
public class ViewManager
{
    private readonly List<IReadOnlyEntityView> managedViews = new List<IReadOnlyEntityView>();
    
    public void RegisterView(IReadOnlyEntityView view)
    {
        if (view != null && !managedViews.Contains(view))
        {
            managedViews.Add(view);
        }
    }
    
    public IEnumerable<IReadOnlyEntityView> GetViewsForEntity(IEntity entity)
    {
        return managedViews.Where(view => 
            view.IsBound && view.Entity == entity);
    }
}
```

## Best Practices

1. **Immutability** – Respect the read-only contract in implementing classes
2. **Null Safety** – Always check for null entities and views
3. **Performance** – Cache view collections when doing frequent queries
4. **State Validation** – Ensure IsBound and Entity states are consistent
5. **Interface Segregation** – Use read-only interface when modification is not needed
6. **Documentation** – Clearly document when views are safe to inspect

## Integration Benefits

### System Safety
- Prevents accidental modification of view state
- Enables safe sharing of view references
- Reduces coupling between view consumers and view implementations

### Performance
- Allows for optimized query systems
- Enables efficient caching strategies
- Reduces overhead of capability checking

### Architecture
- Clear separation of concerns between read and write operations
- Better API design through interface segregation
- Improved testability and maintainability

## Notes

- IReadOnlyEntityView provides a safe, inspection-only interface to entity views
- Implementation should ensure IsBound and Entity properties are always consistent
- Designed for systems that need to inspect but not modify view state
- Compatible with pooling systems and view management frameworks
- Essential for building robust, maintainable view query and inspection systems