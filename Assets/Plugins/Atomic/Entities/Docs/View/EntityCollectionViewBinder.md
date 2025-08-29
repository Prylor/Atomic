# 🔗 EntityCollectionViewBinder

`EntityCollectionViewBinder` is a component that automatically binds collections of entities to corresponding collections of views, maintaining synchronization between entity collections and their visual representations. It provides a seamless bridge between the entity system and view system for managing multiple related entities.

## Key Features

- **Automatic Synchronization** – Keeps view collections in sync with entity collections
- **Event-Driven Updates** – Responds to entity collection changes in real-time
- **Flexible View Creation** – Supports various view creation strategies
- **Performance Optimized** – Efficient handling of collection changes
- **Configurable Binding** – Customizable binding rules and behaviors

---

## Core Functionality

```csharp
public class EntityCollectionViewBinder : MonoBehaviour
{
    // Collection binding
    public void BindToCollection(IReadOnlyEntityCollection entityCollection);
    public void UnbindFromCollection();
    
    // View management
    protected virtual IReadOnlyEntityView CreateViewForEntity(IEntity entity);
    protected virtual void DestroyViewForEntity(IEntity entity, IReadOnlyEntityView view);
    
    // Synchronization
    protected virtual void OnEntityAdded(IEntity entity);
    protected virtual void OnEntityRemoved(IEntity entity);
}
```

## Implementation Examples

### Basic Collection View Binder

```csharp
public class BasicEntityCollectionViewBinder : MonoBehaviour
{
    [Header("Binding Configuration")]
    [SerializeField] private Transform viewParent;
    [SerializeField] private GameObject defaultViewPrefab;
    [SerializeField] private bool autoBindOnStart = true;
    [SerializeField] private bool useViewPooling = false;
    
    [Header("Collection Settings")]
    [SerializeField] private bool maintainOrder = true;
    [SerializeField] private float viewSpacing = 1.0f;
    
    private IReadOnlyEntityCollection boundCollection;
    private Dictionary<IEntity, IReadOnlyEntityView> entityToView = new Dictionary<IEntity, IReadOnlyEntityView>();
    private List<IReadOnlyEntityView> orderedViews = new List<IReadOnlyEntityView>();
    
    private IEntityViewPool viewPool;
    
    void Start()
    {
        if (autoBindOnStart)
        {
            // Find a collection to bind to
            var collection = FindObjectOfType<EntityCollection>();
            if (collection != null)
            {
                BindToCollection(collection);
            }
        }
        
        if (useViewPooling)
        {
            viewPool = FindObjectOfType<EntityViewPool>();
        }
    }
    
    public void BindToCollection(IReadOnlyEntityCollection collection)
    {
        if (boundCollection != null)
        {
            UnbindFromCollection();
        }
        
        boundCollection = collection;
        
        if (collection != null)
        {
            // Subscribe to collection events
            collection.OnAdded += OnEntityAdded;
            collection.OnRemoved += OnEntityRemoved;
            
            // Create views for existing entities
            foreach (var entity in collection)
            {
                OnEntityAdded(entity);
            }
            
            Debug.Log($"Bound to collection with {collection.Count} entities");
        }
    }
    
    public void UnbindFromCollection()
    {
        if (boundCollection != null)
        {
            // Unsubscribe from events
            boundCollection.OnAdded -= OnEntityAdded;
            boundCollection.OnRemoved -= OnEntityRemoved;
            
            // Remove all views
            var entitiesToRemove = new List<IEntity>(entityToView.Keys);
            foreach (var entity in entitiesToRemove)
            {
                OnEntityRemoved(entity);
            }
            
            boundCollection = null;
            Debug.Log("Unbound from collection");
        }
    }
    
    protected virtual void OnEntityAdded(IEntity entity)
    {
        if (entityToView.ContainsKey(entity))
        {
            Debug.LogWarning($"Entity {entity.Name} already has a view");
            return;
        }
        
        var view = CreateViewForEntity(entity);
        if (view != null)
        {
            entityToView[entity] = view;
            orderedViews.Add(view);
            
            if (maintainOrder)
            {
                UpdateViewPositions();
            }
            
            Debug.Log($"Created view for entity: {entity.Name}");
        }
    }
    
    protected virtual void OnEntityRemoved(IEntity entity)
    {
        if (entityToView.TryGetValue(entity, out var view))
        {
            entityToView.Remove(entity);
            orderedViews.Remove(view);
            
            DestroyViewForEntity(entity, view);
            
            if (maintainOrder)
            {
                UpdateViewPositions();
            }
            
            Debug.Log($"Removed view for entity: {entity.Name}");
        }
    }
    
    protected virtual IReadOnlyEntityView CreateViewForEntity(IEntity entity)
    {
        IReadOnlyEntityView view = null;
        
        if (useViewPooling && viewPool != null)
        {
            // Get view from pool
            string viewName = DetermineViewName(entity);
            view = viewPool.Rent(viewName);
        }
        else
        {
            // Instantiate from prefab
            var prefab = DetermineViewPrefab(entity);
            if (prefab != null)
            {
                var instance = Instantiate(prefab, viewParent);
                view = instance.GetComponent<IReadOnlyEntityView>();
            }
        }
        
        // Bind view to entity
        if (view is IEntityView bindableView)
        {
            bindableView.BindToEntity(entity);
        }
        
        return view;
    }
    
    protected virtual void DestroyViewForEntity(IEntity entity, IReadOnlyEntityView view)
    {
        // Unbind view from entity
        if (view is IEntityView bindableView)
        {
            bindableView.UnbindFromEntity();
        }
        
        if (useViewPooling && viewPool != null)
        {
            // Return to pool
            string viewName = DetermineViewName(entity);
            viewPool.Return(viewName, view);
        }
        else
        {
            // Destroy GameObject
            if (view is MonoBehaviour mb)
            {
                Destroy(mb.gameObject);
            }
        }
    }
    
    protected virtual GameObject DetermineViewPrefab(IEntity entity)
    {
        // Override in subclasses for custom view selection logic
        return defaultViewPrefab;
    }
    
    protected virtual string DetermineViewName(IEntity entity)
    {
        // Override in subclasses for custom view naming logic
        if (entity.HasTag(EntityTags.PLAYER))
            return "PlayerView";
        else if (entity.HasTag(EntityTags.ENEMY))
            return "EnemyView";
        else
            return "DefaultView";
    }
    
    private void UpdateViewPositions()
    {
        for (int i = 0; i < orderedViews.Count; i++)
        {
            var view = orderedViews[i];
            if (view is MonoBehaviour mb)
            {
                Vector3 position = viewParent.position + Vector3.right * (i * viewSpacing);
                mb.transform.position = position;
            }
        }
    }
    
    public IReadOnlyList<IReadOnlyEntityView> GetOrderedViews()
    {
        return orderedViews.AsReadOnly();
    }
    
    public IReadOnlyEntityView GetViewForEntity(IEntity entity)
    {
        return entityToView.TryGetValue(entity, out var view) ? view : null;
    }
    
    void OnDestroy()
    {
        UnbindFromCollection();
    }
}
```

### Advanced UI Collection Binder

```csharp
public class UIEntityCollectionBinder : MonoBehaviour
{
    [Header("UI Configuration")]
    [SerializeField] private RectTransform contentParent;
    [SerializeField] private GameObject uiItemPrefab;
    [SerializeField] private ScrollRect scrollRect;
    [SerializeField] private bool useVirtualization = false;
    
    [Header("Layout Settings")]
    [SerializeField] private float itemHeight = 50f;
    [SerializeField] private float itemSpacing = 5f;
    [SerializeField] private bool reverseOrder = false;
    
    [Header("Filtering")]
    [SerializeField] private string[] visibleTags;
    [SerializeField] private string[] hiddenTags;
    
    private IReadOnlyEntityCollection boundCollection;
    private List<IEntity> filteredEntities = new List<IEntity>();
    private Dictionary<IEntity, GameObject> entityToUIItem = new Dictionary<IEntity, GameObject>();
    
    // Virtualization support
    private ObjectPool<GameObject> uiItemPool;
    private List<GameObject> activeItems = new List<GameObject>();
    private int firstVisibleIndex = 0;
    private int lastVisibleIndex = -1;
    
    void Start()
    {
        InitializeUISystem();
    }
    
    private void InitializeUISystem()
    {
        if (useVirtualization)
        {
            SetupVirtualization();
        }
        
        if (scrollRect != null)
        {
            scrollRect.onValueChanged.AddListener(OnScrollValueChanged);
        }
    }
    
    private void SetupVirtualization()
    {
        uiItemPool = new ObjectPool<GameObject>(
            createFunc: () => Instantiate(uiItemPrefab, contentParent),
            actionOnGet: item => item.SetActive(true),
            actionOnRelease: item => item.SetActive(false),
            actionOnDestroy: item => Destroy(item),
            defaultCapacity: 20,
            maxSize: 100
        );
    }
    
    public void BindToCollection(IReadOnlyEntityCollection collection)
    {
        UnbindFromCollection();
        
        boundCollection = collection;
        
        if (collection != null)
        {
            collection.OnAdded += OnEntityAdded;
            collection.OnRemoved += OnEntityRemoved;
            
            RefreshFilteredEntities();
            RebuildUI();
        }
    }
    
    public void UnbindFromCollection()
    {
        if (boundCollection != null)
        {
            boundCollection.OnAdded -= OnEntityAdded;
            boundCollection.OnRemoved -= OnEntityRemoved;
            
            ClearUI();
            boundCollection = null;
        }
    }
    
    private void OnEntityAdded(IEntity entity)
    {
        if (ShouldShowEntity(entity))
        {
            if (!filteredEntities.Contains(entity))
            {
                InsertEntitySorted(entity);
                RebuildUI();
            }
        }
    }
    
    private void OnEntityRemoved(IEntity entity)
    {
        if (filteredEntities.Remove(entity))
        {
            RebuildUI();
        }
    }
    
    private bool ShouldShowEntity(IEntity entity)
    {
        // Check visible tags
        if (visibleTags.Length > 0)
        {
            bool hasVisibleTag = false;
            foreach (var tagName in visibleTags)
            {
                int tagId = EntityNames.NameToId(tagName);
                if (entity.HasTag(tagId))
                {
                    hasVisibleTag = true;
                    break;
                }
            }
            if (!hasVisibleTag) return false;
        }
        
        // Check hidden tags
        foreach (var tagName in hiddenTags)
        {
            int tagId = EntityNames.NameToId(tagName);
            if (entity.HasTag(tagId))
            {
                return false;
            }
        }
        
        return true;
    }
    
    private void RefreshFilteredEntities()
    {
        filteredEntities.Clear();
        
        if (boundCollection != null)
        {
            foreach (var entity in boundCollection)
            {
                if (ShouldShowEntity(entity))
                {
                    filteredEntities.Add(entity);
                }
            }
            
            SortFilteredEntities();
        }
    }
    
    private void InsertEntitySorted(IEntity entity)
    {
        int insertIndex = filteredEntities.Count;
        
        // Simple alphabetical sort by name
        for (int i = 0; i < filteredEntities.Count; i++)
        {
            if (string.Compare(entity.Name, filteredEntities[i].Name) < 0)
            {
                insertIndex = i;
                break;
            }
        }
        
        filteredEntities.Insert(insertIndex, entity);
    }
    
    private void SortFilteredEntities()
    {
        filteredEntities.Sort((a, b) => string.Compare(a.Name, b.Name));
        
        if (reverseOrder)
        {
            filteredEntities.Reverse();
        }
    }
    
    private void RebuildUI()
    {
        if (useVirtualization)
        {
            RebuildVirtualizedUI();
        }
        else
        {
            RebuildFullUI();
        }
        
        UpdateContentSize();
    }
    
    private void RebuildFullUI()
    {
        ClearUI();
        
        for (int i = 0; i < filteredEntities.Count; i++)
        {
            var entity = filteredEntities[i];
            var uiItem = CreateUIItemForEntity(entity, i);
            entityToUIItem[entity] = uiItem;
        }
    }
    
    private void RebuildVirtualizedUI()
    {
        // Return all active items to pool
        foreach (var item in activeItems)
        {
            uiItemPool.Release(item);
        }
        activeItems.Clear();
        entityToUIItem.Clear();
        
        // Calculate visible range
        CalculateVisibleRange();
        
        // Create items for visible range
        for (int i = firstVisibleIndex; i <= lastVisibleIndex && i < filteredEntities.Count; i++)
        {
            var entity = filteredEntities[i];
            var uiItem = uiItemPool.Get();
            
            SetupUIItem(uiItem, entity, i);
            activeItems.Add(uiItem);
            entityToUIItem[entity] = uiItem;
        }
    }
    
    private GameObject CreateUIItemForEntity(IEntity entity, int index)
    {
        var uiItem = Instantiate(uiItemPrefab, contentParent);
        SetupUIItem(uiItem, entity, index);
        return uiItem;
    }
    
    private void SetupUIItem(GameObject uiItem, IEntity entity, int index)
    {
        // Position the item
        var rectTransform = uiItem.GetComponent<RectTransform>();
        float yPosition = -(index * (itemHeight + itemSpacing));
        rectTransform.anchoredPosition = new Vector2(0, yPosition);
        
        // Configure the UI item with entity data
        var uiComponent = uiItem.GetComponent<EntityUIItem>();
        if (uiComponent != null)
        {
            uiComponent.SetupForEntity(entity);
        }
        
        // Bind to entity if the UI item supports it
        var view = uiItem.GetComponent<IEntityView>();
        if (view != null)
        {
            view.BindToEntity(entity);
        }
    }
    
    private void CalculateVisibleRange()
    {
        if (scrollRect == null || filteredEntities.Count == 0)
        {
            firstVisibleIndex = 0;
            lastVisibleIndex = filteredEntities.Count - 1;
            return;
        }
        
        var viewportHeight = scrollRect.viewport.rect.height;
        var contentPosition = scrollRect.content.anchoredPosition.y;
        
        firstVisibleIndex = Mathf.Max(0, Mathf.FloorToInt(contentPosition / (itemHeight + itemSpacing)));
        
        int visibleItemCount = Mathf.CeilToInt(viewportHeight / (itemHeight + itemSpacing)) + 2; // +2 for buffer
        lastVisibleIndex = Mathf.Min(filteredEntities.Count - 1, firstVisibleIndex + visibleItemCount);
    }
    
    private void OnScrollValueChanged(Vector2 scrollPosition)
    {
        if (useVirtualization)
        {
            RebuildVirtualizedUI();
        }
    }
    
    private void UpdateContentSize()
    {
        if (contentParent != null)
        {
            float totalHeight = filteredEntities.Count * (itemHeight + itemSpacing);
            contentParent.sizeDelta = new Vector2(contentParent.sizeDelta.x, totalHeight);
        }
    }
    
    private void ClearUI()
    {
        if (useVirtualization)
        {
            foreach (var item in activeItems)
            {
                uiItemPool.Release(item);
            }
            activeItems.Clear();
        }
        else
        {
            foreach (var uiItem in entityToUIItem.Values)
            {
                if (uiItem != null)
                {
                    Destroy(uiItem);
                }
            }
        }
        
        entityToUIItem.Clear();
    }
    
    // Public API
    public int GetFilteredEntityCount() => filteredEntities.Count;
    
    public IEntity GetEntityAtIndex(int index)
    {
        return index >= 0 && index < filteredEntities.Count ? filteredEntities[index] : null;
    }
    
    public GameObject GetUIItemForEntity(IEntity entity)
    {
        return entityToUIItem.TryGetValue(entity, out var item) ? item : null;
    }
    
    [ContextMenu("Refresh Filter")]
    public void RefreshFilter()
    {
        RefreshFilteredEntities();
        RebuildUI();
    }
    
    void OnDestroy()
    {
        UnbindFromCollection();
    }
}

// Supporting UI component
public class EntityUIItem : MonoBehaviour
{
    [Header("UI Elements")]
    [SerializeField] private Text nameText;
    [SerializeField] private Text healthText;
    [SerializeField] private Image iconImage;
    [SerializeField] private Button selectButton;
    
    private IEntity boundEntity;
    
    public void SetupForEntity(IEntity entity)
    {
        boundEntity = entity;
        
        if (nameText != null)
            nameText.text = entity.Name;
        
        if (healthText != null && entity.TryGetValue<float>(EntityNames.HEALTH_PERCENTAGE, out var health))
            healthText.text = $"{health * 100:F0}%";
        
        if (selectButton != null)
        {
            selectButton.onClick.RemoveAllListeners();
            selectButton.onClick.AddListener(() => OnSelectEntity(entity));
        }
        
        UpdateIcon(entity);
    }
    
    private void UpdateIcon(IEntity entity)
    {
        if (iconImage == null) return;
        
        // Set icon based on entity type
        string iconPath = "Icons/Default";
        
        if (entity.HasTag(EntityTags.PLAYER))
            iconPath = "Icons/Player";
        else if (entity.HasTag(EntityTags.ENEMY))
            iconPath = "Icons/Enemy";
        else if (entity.HasTag(EntityTags.NPC))
            iconPath = "Icons/NPC";
        
        var sprite = Resources.Load<Sprite>(iconPath);
        if (sprite != null)
            iconImage.sprite = sprite;
    }
    
    private void OnSelectEntity(IEntity entity)
    {
        // Notify selection system
        var selectionManager = FindObjectOfType<EntitySelectionManager>();
        if (selectionManager != null)
        {
            selectionManager.SelectEntity(entity);
        }
        
        Debug.Log($"Selected entity: {entity.Name}");
    }
}
```

## Best Practices

1. **Event Management** – Properly subscribe/unsubscribe from collection events
2. **View Lifecycle** – Ensure views are properly created and destroyed
3. **Performance** – Use virtualization for large collections
4. **Filtering** – Implement efficient filtering for relevant entities
5. **Memory Management** – Clean up views when unbinding from collections
6. **Error Handling** – Handle cases where view creation fails
7. **Customization** – Make view selection logic easily customizable

## Integration Patterns

### Collection Manager Integration
```csharp
public class CollectionViewManager : MonoBehaviour
{
    [SerializeField] private EntityCollectionViewBinder[] binders;
    private EntityCollection managedCollection;
    
    void Start()
    {
        managedCollection = GetComponent<EntityCollection>();
        
        foreach (var binder in binders)
        {
            binder.BindToCollection(managedCollection);
        }
    }
}
```

## Performance Considerations

### Memory Management
- Use object pooling for frequently created/destroyed views
- Implement virtualization for large collections
- Clean up event subscriptions properly

### Update Efficiency
- Batch UI updates when possible
- Use dirty flags to minimize unnecessary updates
- Implement view frustum culling for 3D views

## Notes

- EntityCollectionViewBinder provides automatic synchronization between entity collections and view collections
- Designed for both 2D UI and 3D world view scenarios
- Supports various view creation strategies including pooling
- Virtualization support enables handling of large collections efficiently
- Filtering capabilities allow selective display of collection subsets