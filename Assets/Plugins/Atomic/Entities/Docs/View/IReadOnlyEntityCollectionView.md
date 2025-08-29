# 👁️ IReadOnlyEntityCollectionView

`IReadOnlyEntityCollectionView` is an interface that defines a read-only contract for entity collection view implementations. It provides access to collection view properties and bound entity collections without allowing modification of the view's internal state, ensuring safe inspection of collection views across different systems.

## Key Features

- **Read-Only Access** – Immutable interface for safe collection view inspection
- **Collection Association** – Access to bound entity collections without modification rights
- **Type Safety** – Compile-time interface compliance for collection views
- **System Integration** – Compatible with view management and monitoring systems
- **Minimal Interface** – Focused on essential read-only collection operations

---

## Interface Definition

```csharp
public interface IReadOnlyEntityCollectionView
{
    /// <summary>
    /// Gets the entity collection associated with this view, if any.
    /// </summary>
    IReadOnlyEntityCollection Collection { get; }
    
    /// <summary>
    /// Gets a value indicating whether this view is currently bound to a collection.
    /// </summary>
    bool IsBoundToCollection { get; }
    
    /// <summary>
    /// Gets the number of items currently displayed in this collection view.
    /// </summary>
    int DisplayedItemCount { get; }
}
```

## Implementation Examples

### Basic Read-Only Collection View

```csharp
public class ReadOnlyEntityCollectionView : MonoBehaviour, IReadOnlyEntityCollectionView
{
    private IReadOnlyEntityCollection boundCollection;
    private int displayedItems;
    
    public IReadOnlyEntityCollection Collection => boundCollection;
    public bool IsBoundToCollection => boundCollection != null;
    public int DisplayedItemCount => displayedItems;
    
    // Internal methods for binding (not part of read-only interface)
    internal void BindToCollection(IReadOnlyEntityCollection collection)
    {
        boundCollection = collection;
        OnCollectionBound(collection);
    }
    
    internal void UnbindFromCollection()
    {
        var previousCollection = boundCollection;
        boundCollection = null;
        displayedItems = 0;
        OnCollectionUnbound(previousCollection);
    }
    
    protected virtual void OnCollectionBound(IReadOnlyEntityCollection collection)
    {
        if (collection != null)
        {
            displayedItems = collection.Count;
            Debug.Log($"Collection view bound to collection with {displayedItems} items");
        }
    }
    
    protected virtual void OnCollectionUnbound(IReadOnlyEntityCollection collection)
    {
        Debug.Log("Collection view unbound from collection");
    }
    
    protected void UpdateDisplayedItemCount(int count)
    {
        displayedItems = count;
    }
}
```

### Collection View Inspector System

```csharp
public class EntityCollectionViewInspector : MonoBehaviour
{
    [Header("Inspector Settings")]
    [SerializeField] private bool inspectViewsAutomatically = true;
    [SerializeField] private float inspectionInterval = 2.0f;
    [SerializeField] private bool logInspectionResults = false;
    
    private List<IReadOnlyEntityCollectionView> inspectedViews = new List<IReadOnlyEntityCollectionView>();
    private Dictionary<IReadOnlyEntityCollectionView, CollectionViewInspectionData> inspectionData = 
        new Dictionary<IReadOnlyEntityCollectionView, CollectionViewInspectionData>();
    
    [System.Serializable]
    public class CollectionViewInspectionData
    {
        public string viewName;
        public bool isBoundToCollection;
        public int collectionCount;
        public int displayedItemCount;
        public float syncPercentage;
        public bool isFullySynced;
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
        // Find all collection views in scene
        var allViews = FindObjectsOfType<MonoBehaviour>()
            .OfType<IReadOnlyEntityCollectionView>()
            .ToList();
        
        inspectedViews.Clear();
        
        foreach (var view in allViews)
        {
            var data = InspectCollectionView(view);
            inspectedViews.Add(view);
            inspectionData[view] = data;
            
            if (logInspectionResults)
            {
                LogInspectionData(data);
            }
        }
        
        if (logInspectionResults)
        {
            Debug.Log($"Inspected {inspectedViews.Count} collection views");
        }
    }
    
    private CollectionViewInspectionData InspectCollectionView(IReadOnlyEntityCollectionView view)
    {
        var data = new CollectionViewInspectionData
        {
            lastInspectionTime = Time.time,
            isBoundToCollection = view.IsBoundToCollection,
            displayedItemCount = view.DisplayedItemCount
        };
        
        if (view is MonoBehaviour mb)
        {
            data.viewName = mb.name;
        }
        
        if (view.IsBoundToCollection && view.Collection != null)
        {
            data.collectionCount = view.Collection.Count;
            data.syncPercentage = data.collectionCount > 0 
                ? (float)data.displayedItemCount / data.collectionCount 
                : 1.0f;
            data.isFullySynced = data.displayedItemCount == data.collectionCount;
        }
        else
        {
            data.collectionCount = 0;
            data.syncPercentage = 0.0f;
            data.isFullySynced = data.displayedItemCount == 0;
        }
        
        return data;
    }
    
    private void LogInspectionData(CollectionViewInspectionData data)
    {
        string bindingStatus = data.isBoundToCollection 
            ? $"bound to collection ({data.collectionCount} items)" 
            : "unbound";
        
        string syncStatus = data.isFullySynced 
            ? "fully synced" 
            : $"{data.syncPercentage * 100:F1}% synced";
        
        Debug.Log($"Collection view '{data.viewName}' is {bindingStatus}, " +
                 $"displaying {data.displayedItemCount} items, {syncStatus}");
    }
    
    public List<IReadOnlyEntityCollectionView> GetUnboundViews()
    {
        return inspectedViews.Where(view => !view.IsBoundToCollection).ToList();
    }
    
    public List<IReadOnlyEntityCollectionView> GetBoundViews()
    {
        return inspectedViews.Where(view => view.IsBoundToCollection).ToList();
    }
    
    public List<IReadOnlyEntityCollectionView> GetUnsyncedViews()
    {
        return inspectedViews.Where(view =>
        {
            if (inspectionData.TryGetValue(view, out var data))
            {
                return !data.isFullySynced;
            }
            return false;
        }).ToList();
    }
    
    public CollectionViewInspectionData GetInspectionData(IReadOnlyEntityCollectionView view)
    {
        return inspectionData.TryGetValue(view, out var data) ? data : null;
    }
    
    // Manual inspection methods
    [ContextMenu("Inspect All Collection Views Now")]
    public void InspectAllViewsManual()
    {
        InspectAllViews();
    }
    
    [ContextMenu("Print Collection View Summary")]
    public void PrintCollectionViewSummary()
    {
        var boundCount = inspectedViews.Count(view => view.IsBoundToCollection);
        var unboundCount = inspectedViews.Count - boundCount;
        var syncedCount = inspectedViews.Count(view => 
        {
            if (inspectionData.TryGetValue(view, out var data))
                return data.isFullySynced;
            return false;
        });
        
        Debug.Log($"Collection View Summary: {inspectedViews.Count} total, " +
                 $"{boundCount} bound, {unboundCount} unbound, {syncedCount} synced");
        
        foreach (var view in inspectedViews.Take(10)) // Show first 10
        {
            var data = inspectionData[view];
            Debug.Log($"  - {data.viewName}: {data.displayedItemCount} items " +
                     $"({data.syncPercentage * 100:F1}% synced)");
        }
    }
}
```

### Collection View Query System

```csharp
public class EntityCollectionViewQuerySystem : MonoBehaviour
{
    private readonly List<IReadOnlyEntityCollectionView> cachedViews = new List<IReadOnlyEntityCollectionView>();
    private float lastCacheUpdate = 0f;
    private const float CACHE_REFRESH_INTERVAL = 1.0f;
    
    public IEnumerable<IReadOnlyEntityCollectionView> GetAllCollectionViews()
    {
        RefreshCacheIfNeeded();
        return cachedViews.AsReadOnly();
    }
    
    public IEnumerable<IReadOnlyEntityCollectionView> GetBoundCollectionViews()
    {
        RefreshCacheIfNeeded();
        return cachedViews.Where(view => view.IsBoundToCollection);
    }
    
    public IEnumerable<IReadOnlyEntityCollectionView> GetUnboundCollectionViews()
    {
        RefreshCacheIfNeeded();
        return cachedViews.Where(view => !view.IsBoundToCollection);
    }
    
    public IReadOnlyEntityCollectionView FindViewForCollection(IReadOnlyEntityCollection collection)
    {
        RefreshCacheIfNeeded();
        return cachedViews.FirstOrDefault(view => 
            view.IsBoundToCollection && view.Collection == collection);
    }
    
    public IEnumerable<IReadOnlyEntityCollectionView> FindViewsWithItemCount(int itemCount)
    {
        RefreshCacheIfNeeded();
        return cachedViews.Where(view => view.DisplayedItemCount == itemCount);
    }
    
    public IEnumerable<IReadOnlyEntityCollectionView> FindViewsInItemCountRange(int minCount, int maxCount)
    {
        RefreshCacheIfNeeded();
        return cachedViews.Where(view => 
            view.DisplayedItemCount >= minCount && view.DisplayedItemCount <= maxCount);
    }
    
    public IEnumerable<T> GetCollectionViewsOfType<T>() where T : class, IReadOnlyEntityCollectionView
    {
        RefreshCacheIfNeeded();
        return cachedViews.OfType<T>();
    }
    
    public int GetTotalDisplayedItems()
    {
        RefreshCacheIfNeeded();
        return cachedViews.Sum(view => view.DisplayedItemCount);
    }
    
    public float GetAverageItemsPerView()
    {
        RefreshCacheIfNeeded();
        if (cachedViews.Count == 0) return 0f;
        
        return (float)GetTotalDisplayedItems() / cachedViews.Count;
    }
    
    public Dictionary<int, int> GetItemCountDistribution()
    {
        RefreshCacheIfNeeded();
        var distribution = new Dictionary<int, int>();
        
        foreach (var view in cachedViews)
        {
            int itemCount = view.DisplayedItemCount;
            if (distribution.ContainsKey(itemCount))
                distribution[itemCount]++;
            else
                distribution[itemCount] = 1;
        }
        
        return distribution;
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
        
        // Find all IReadOnlyEntityCollectionView implementations in scene
        var monoBehaviours = FindObjectsOfType<MonoBehaviour>();
        
        foreach (var mb in monoBehaviours)
        {
            if (mb is IReadOnlyEntityCollectionView view)
            {
                cachedViews.Add(view);
            }
        }
        
        lastCacheUpdate = Time.time;
    }
    
    // Events for collection view state changes (if implemented by view system)
    public static event System.Action<IReadOnlyEntityCollectionView> OnCollectionViewBound;
    public static event System.Action<IReadOnlyEntityCollectionView> OnCollectionViewUnbound;
    public static event System.Action<IReadOnlyEntityCollectionView, int> OnDisplayedItemCountChanged;
    
    // Static methods for triggering events (called by view implementations)
    public static void NotifyCollectionViewBound(IReadOnlyEntityCollectionView view)
    {
        OnCollectionViewBound?.Invoke(view);
    }
    
    public static void NotifyCollectionViewUnbound(IReadOnlyEntityCollectionView view)
    {
        OnCollectionViewUnbound?.Invoke(view);
    }
    
    public static void NotifyDisplayedItemCountChanged(IReadOnlyEntityCollectionView view, int newCount)
    {
        OnDisplayedItemCountChanged?.Invoke(view, newCount);
    }
}
```

### Collection View Statistics System

```csharp
public class EntityCollectionViewStatistics : MonoBehaviour
{
    [System.Serializable]
    public class CollectionViewStats
    {
        public int totalCollectionViews;
        public int boundCollectionViews;
        public int unboundCollectionViews;
        public int totalDisplayedItems;
        public int averageItemsPerView;
        public int maxItemsInSingleView;
        public int minItemsInSingleView;
        public Dictionary<string, int> viewsByType;
        public Dictionary<int, int> itemCountDistribution;
        public float lastUpdateTime;
    }
    
    [Header("Statistics Settings")]
    [SerializeField] private bool enableStatistics = true;
    [SerializeField] private float updateInterval = 3.0f;
    [SerializeField] private bool displayUI = false;
    
    private CollectionViewStats currentStats = new CollectionViewStats();
    private EntityCollectionViewQuerySystem querySystem;
    
    void Start()
    {
        querySystem = FindObjectOfType<EntityCollectionViewQuerySystem>();
        if (querySystem == null)
        {
            querySystem = gameObject.AddComponent<EntityCollectionViewQuerySystem>();
        }
        
        if (enableStatistics)
        {
            InvokeRepeating(nameof(UpdateStatistics), 0f, updateInterval);
        }
    }
    
    private void UpdateStatistics()
    {
        var allViews = querySystem.GetAllCollectionViews().ToList();
        
        currentStats.totalCollectionViews = allViews.Count;
        currentStats.boundCollectionViews = allViews.Count(view => view.IsBoundToCollection);
        currentStats.unboundCollectionViews = currentStats.totalCollectionViews - currentStats.boundCollectionViews;
        currentStats.totalDisplayedItems = querySystem.GetTotalDisplayedItems();
        currentStats.lastUpdateTime = Time.time;
        
        if (currentStats.totalCollectionViews > 0)
        {
            currentStats.averageItemsPerView = (int)querySystem.GetAverageItemsPerView();
            currentStats.maxItemsInSingleView = allViews.Max(view => view.DisplayedItemCount);
            currentStats.minItemsInSingleView = allViews.Min(view => view.DisplayedItemCount);
        }
        
        // Calculate distribution
        currentStats.itemCountDistribution = querySystem.GetItemCountDistribution();
        
        // Calculate view types
        currentStats.viewsByType = new Dictionary<string, int>();
        foreach (var view in allViews)
        {
            string typeName = view.GetType().Name;
            if (currentStats.viewsByType.ContainsKey(typeName))
                currentStats.viewsByType[typeName]++;
            else
                currentStats.viewsByType[typeName] = 1;
        }
    }
    
    public CollectionViewStats GetCurrentStatistics()
    {
        return currentStats;
    }
    
    void OnGUI()
    {
        if (!displayUI || !enableStatistics) return;
        
        GUILayout.BeginArea(new Rect(10, 10, 350, 500));
        
        GUILayout.Label("Entity Collection View Statistics", GUI.skin.box);
        GUILayout.Label($"Total Collection Views: {currentStats.totalCollectionViews}");
        GUILayout.Label($"Bound: {currentStats.boundCollectionViews}");
        GUILayout.Label($"Unbound: {currentStats.unboundCollectionViews}");
        GUILayout.Label($"Total Displayed Items: {currentStats.totalDisplayedItems}");
        
        if (currentStats.totalCollectionViews > 0)
        {
            GUILayout.Label($"Avg Items/View: {currentStats.averageItemsPerView}");
            GUILayout.Label($"Max Items: {currentStats.maxItemsInSingleView}");
            GUILayout.Label($"Min Items: {currentStats.minItemsInSingleView}");
        }
        
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
        GUILayout.Label("Item Count Distribution:", GUI.skin.box);
        
        if (currentStats.itemCountDistribution != null)
        {
            foreach (var kvp in currentStats.itemCountDistribution.Take(5))
            {
                GUILayout.Label($"{kvp.Key} items: {kvp.Value} views");
            }
        }
        
        GUILayout.EndArea();
    }
    
    [ContextMenu("Print Detailed Statistics")]
    public void PrintDetailedStatistics()
    {
        UpdateStatistics();
        
        Debug.Log("=== Entity Collection View Statistics ===");
        Debug.Log($"Total Views: {currentStats.totalCollectionViews}");
        Debug.Log($"Bound: {currentStats.boundCollectionViews}, Unbound: {currentStats.unboundCollectionViews}");
        Debug.Log($"Total Items: {currentStats.totalDisplayedItems}");
        
        if (currentStats.totalCollectionViews > 0)
        {
            Debug.Log($"Average Items/View: {currentStats.averageItemsPerView}");
            Debug.Log($"Range: {currentStats.minItemsInSingleView} - {currentStats.maxItemsInSingleView} items");
        }
        
        Debug.Log("Views by Type:");
        if (currentStats.viewsByType != null)
        {
            foreach (var kvp in currentStats.viewsByType)
            {
                Debug.Log($"  {kvp.Key}: {kvp.Value}");
            }
        }
        
        Debug.Log("Item Count Distribution:");
        if (currentStats.itemCountDistribution != null)
        {
            foreach (var kvp in currentStats.itemCountDistribution)
            {
                Debug.Log($"  {kvp.Key} items: {kvp.Value} views");
            }
        }
    }
}
```

### Debug Utilities

```csharp
public static class ReadOnlyEntityCollectionViewDebug
{
    public static void LogCollectionViewInfo(IReadOnlyEntityCollectionView view, string prefix = "")
    {
        if (view == null)
        {
            Debug.Log($"{prefix}Collection view is null");
            return;
        }
        
        string viewName = view is MonoBehaviour mb ? mb.name : view.GetType().Name;
        
        if (view.IsBoundToCollection)
        {
            int collectionCount = view.Collection?.Count ?? 0;
            Debug.Log($"{prefix}Collection view '{viewName}' bound to collection " +
                     $"({collectionCount} entities), displaying {view.DisplayedItemCount} items");
        }
        else
        {
            Debug.Log($"{prefix}Collection view '{viewName}' is unbound, " +
                     $"displaying {view.DisplayedItemCount} items");
        }
    }
    
    public static void ValidateCollectionViewState(IReadOnlyEntityCollectionView view)
    {
        if (view == null)
        {
            Debug.LogError("Collection view validation failed: view is null");
            return;
        }
        
        bool hasCollection = view.Collection != null;
        bool isBound = view.IsBoundToCollection;
        
        if (isBound && !hasCollection)
        {
            Debug.LogError($"Collection view state inconsistency: IsBoundToCollection is true but Collection is null");
        }
        
        if (!isBound && hasCollection)
        {
            Debug.LogWarning($"Collection view state inconsistency: IsBoundToCollection is false but Collection is not null");
        }
        
        if (isBound && hasCollection)
        {
            int collectionCount = view.Collection.Count;
            int displayedCount = view.DisplayedItemCount;
            
            if (displayedCount > collectionCount)
            {
                Debug.LogWarning($"Collection view displaying more items ({displayedCount}) than collection contains ({collectionCount})");
            }
            
            if (displayedCount < collectionCount)
            {
                Debug.Log($"Collection view displaying fewer items ({displayedCount}) than collection contains ({collectionCount}) - may be filtered");
            }
        }
    }
    
    public static void CompareCollectionViews(IReadOnlyEntityCollectionView view1, IReadOnlyEntityCollectionView view2)
    {
        if (view1 == null || view2 == null)
        {
            Debug.LogError("Cannot compare null collection views");
            return;
        }
        
        string name1 = view1 is MonoBehaviour mb1 ? mb1.name : view1.GetType().Name;
        string name2 = view2 is MonoBehaviour mb2 ? mb2.name : view2.GetType().Name;
        
        Debug.Log($"Comparing collection views '{name1}' and '{name2}':");
        Debug.Log($"  Bound: {view1.IsBoundToCollection} vs {view2.IsBoundToCollection}");
        Debug.Log($"  Items: {view1.DisplayedItemCount} vs {view2.DisplayedItemCount}");
        
        if (view1.IsBoundToCollection && view2.IsBoundToCollection)
        {
            bool sameCollection = view1.Collection == view2.Collection;
            Debug.Log($"  Same Collection: {sameCollection}");
            
            if (sameCollection)
            {
                Debug.Log($"  Both views are bound to the same collection with {view1.Collection.Count} entities");
            }
        }
    }
}
```

## Usage Patterns

### Safe Collection View Inspection
```csharp
public void InspectCollectionView(IReadOnlyEntityCollectionView view)
{
    if (view == null) return;
    
    // Safe to call - read-only interface
    bool isBound = view.IsBoundToCollection;
    int displayedCount = view.DisplayedItemCount;
    IReadOnlyEntityCollection collection = view.Collection;
    
    if (isBound && collection != null)
    {
        // Can safely read collection properties
        int collectionCount = collection.Count;
        
        // Analyze synchronization
        bool fullySync = displayedCount == collectionCount;
        float syncPercentage = collectionCount > 0 ? (float)displayedCount / collectionCount : 1.0f;
        
        Debug.Log($"Collection view sync: {syncPercentage * 100:F1}% ({displayedCount}/{collectionCount})");
    }
}
```

### Collection View Manager Integration
```csharp
public class CollectionViewManager
{
    private readonly List<IReadOnlyEntityCollectionView> managedViews = new List<IReadOnlyEntityCollectionView>();
    
    public void RegisterCollectionView(IReadOnlyEntityCollectionView view)
    {
        if (view != null && !managedViews.Contains(view))
        {
            managedViews.Add(view);
        }
    }
    
    public IEnumerable<IReadOnlyEntityCollectionView> GetViewsForCollection(IReadOnlyEntityCollection collection)
    {
        return managedViews.Where(view => 
            view.IsBoundToCollection && view.Collection == collection);
    }
}
```

## Best Practices

1. **State Consistency** – Ensure IsBoundToCollection and Collection properties are always consistent
2. **Null Safety** – Always check for null collections and views
3. **Performance** – Cache view collections when doing frequent queries
4. **Synchronization Monitoring** – Track displayed item count vs collection count for debugging
5. **Interface Compliance** – Implement read-only interface without exposing modification capabilities
6. **Documentation** – Clearly document when collection views are safe to inspect

## Integration Benefits

### System Safety
- Prevents accidental modification of collection view state
- Enables safe sharing of collection view references
- Reduces coupling between view consumers and view implementations

### Monitoring and Analytics
- Allows for comprehensive collection view monitoring systems
- Enables collection view performance analysis
- Supports debugging and diagnostics of view synchronization

### Architecture
- Clear separation of concerns between read and write operations
- Better API design through interface segregation
- Improved testability and maintainability

## Notes

- IReadOnlyEntityCollectionView provides a safe, inspection-only interface to entity collection views
- Implementation should ensure all properties reflect the current view state accurately
- Designed for systems that need to inspect but not modify collection view state
- Compatible with collection view pooling and management frameworks
- Essential for building robust collection view monitoring and analytics systems