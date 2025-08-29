# 🧩 IEntityView

`IEntityView` is the core interface for single entity presentation in the Atomic framework. It separates presentation logic from entity data, following Atomic's procedural pattern where views observe entity state and update the UI accordingly without holding their own state.

## Key Features

- **Presentation Layer** – Handles visual representation of entity data
- **State Observation** – Reacts to entity state changes rather than maintaining its own state
- **Show/Hide Control** – Provides visibility management for entity presentation
- **Entity Association** – Links views to specific entity instances
- **Unity Integration** – Designed to work seamlessly with Unity's GameObject system
- **Stateless Design** – Views derive their state from associated entities

---

## Interface Definition

```csharp
public interface IEntityView : IReadOnlyEntityView
{
    /// <summary>
    /// Displays the view for the specified entity and associates it with this view instance.
    /// </summary>
    /// <param name="entity">The entity to associate with and display through this view.</param>
    void Show(IEntity entity);

    /// <summary>
    /// Hides or deactivates the current view, removing its association with the entity.
    /// </summary>
    void Hide();
}
```

## Inheritance

- **IReadOnlyEntityView** – Provides read-only access to view state and entity reference

## Core Properties (Inherited)

### Name
```csharp
string Name { get; }
```
- **Description**: Display name or identifier of the view
- **Usage**: Debugging, editor tools, runtime identification

### Entity
```csharp
IEntity Entity { get; }
```
- **Description**: The entity instance currently associated with this view
- **Usage**: Access entity data for presentation updates

### IsVisible
```csharp
bool IsVisible { get; }
```
- **Description**: Whether the view is currently visible (active in scene or UI)
- **Usage**: State checking, conditional rendering logic

## Core Methods

### Show
```csharp
void Show(IEntity entity);
```
- **Description**: Displays the view and associates it with the specified entity
- **Parameters**: 
  - `entity` – The entity to visualize through this view
- **Behavior**:
  - Links view to entity instance
  - Activates visual components
  - Sets `IsVisible` to true
  - Triggers view initialization logic

### Hide
```csharp
void Hide();
```
- **Description**: Hides the view and removes entity association
- **Behavior**:
  - Deactivates visual components
  - Sets `IsVisible` to false
  - Clears entity reference
  - Triggers cleanup logic

## Example Usage

### Basic View Implementation

```csharp
using UnityEngine;
using UnityEngine.UI;

public class HealthBarView : MonoBehaviour, IEntityView
{
    [SerializeField] private Slider healthSlider;
    [SerializeField] private Text healthText;
    
    private IEntity _entity;
    private bool _isVisible;
    
    public string Name => gameObject.name;
    public IEntity Entity => _entity;
    public bool IsVisible => _isVisible;
    
    public void Show(IEntity entity)
    {
        _entity = entity ?? throw new ArgumentNullException(nameof(entity));
        _isVisible = true;
        
        // Subscribe to entity state changes
        entity.OnStateChanged += OnEntityStateChanged;
        
        // Initial UI update
        UpdateHealthDisplay();
        
        // Show the view
        gameObject.SetActive(true);
    }
    
    public void Hide()
    {
        if (!_isVisible) return;
        
        // Unsubscribe from entity events
        if (_entity != null)
        {
            _entity.OnStateChanged -= OnEntityStateChanged;
        }
        
        _isVisible = false;
        _entity = null;
        
        // Hide the view
        gameObject.SetActive(false);
    }
    
    private void OnEntityStateChanged()
    {
        if (_isVisible && _entity != null)
        {
            UpdateHealthDisplay();
        }
    }
    
    private void UpdateHealthDisplay()
    {
        if (_entity == null) return;
        
        int currentHealth = _entity.GetValue<int>("Health");
        int maxHealth = _entity.GetValue<int>("MaxHealth");
        
        healthSlider.value = (float)currentHealth / maxHealth;
        healthText.text = $"{currentHealth}/{maxHealth}";
    }
}
```

### Procedural View Management

Following Atomic's procedural approach:

```csharp
// Static utility methods for view management
public static class ViewUtils
{
    public static void ShowEntityView<T>(T view, IEntity entity) where T : IEntityView
    {
        if (view == null || entity == null) return;
        
        view.Show(entity);
        Debug.Log($"Showing view '{view.Name}' for entity '{entity.Name}'");
    }
    
    public static void HideEntityView<T>(T view) where T : IEntityView
    {
        if (view == null || !view.IsVisible) return;
        
        string viewName = view.Name;
        string entityName = view.Entity?.Name ?? "Unknown";
        
        view.Hide();
        Debug.Log($"Hidden view '{viewName}' for entity '{entityName}'");
    }
    
    public static bool IsViewBoundTo(IEntityView view, IEntity entity)
    {
        return view != null && view.IsVisible && view.Entity == entity;
    }
}

// Usage example
public class GameManager : MonoBehaviour
{
    [SerializeField] private HealthBarView healthBarPrefab;
    
    private readonly Dictionary<IEntity, IEntityView> activeViews = new();
    
    public void ShowPlayerHealth(IEntity player)
    {
        if (!activeViews.ContainsKey(player))
        {
            var healthBar = Instantiate(healthBarPrefab);
            ViewUtils.ShowEntityView(healthBar, player);
            activeViews[player] = healthBar;
        }
    }
    
    public void HidePlayerHealth(IEntity player)
    {
        if (activeViews.TryGetValue(player, out var view))
        {
            ViewUtils.HideEntityView(view);
            activeViews.Remove(player);
            
            if (view is MonoBehaviour mb)
                Destroy(mb.gameObject);
        }
    }
}
```

### View State Synchronization

```csharp
public class PlayerInfoView : MonoBehaviour, IEntityView
{
    [SerializeField] private Text playerName;
    [SerializeField] private Text level;
    [SerializeField] private Image avatar;
    [SerializeField] private GameObject[] statusEffects;
    
    private IEntity _entity;
    private bool _isVisible;
    
    public string Name => "Player Info Panel";
    public IEntity Entity => _entity;
    public bool IsVisible => _isVisible;
    
    public void Show(IEntity entity)
    {
        _entity = entity;
        _isVisible = true;
        
        // Subscribe to all relevant entity changes
        entity.OnStateChanged += SynchronizeWithEntity;
        
        // Initial synchronization
        SynchronizeWithEntity();
        gameObject.SetActive(true);
    }
    
    public void Hide()
    {
        if (!_isVisible) return;
        
        _entity.OnStateChanged -= SynchronizeWithEntity;
        _isVisible = false;
        _entity = null;
        
        gameObject.SetActive(false);
    }
    
    private void SynchronizeWithEntity()
    {
        if (_entity == null) return;
        
        // Update UI based on entity state
        playerName.text = _entity.Name;
        level.text = $"Level {_entity.GetValue<int>("Level")}";
        
        // Update status effects based on entity tags
        UpdateStatusEffects();
        
        // Load avatar based on entity data
        LoadAvatar(_entity.GetValue<string>("AvatarPath"));
    }
    
    private void UpdateStatusEffects()
    {
        // Show/hide status effect icons based on entity tags
        statusEffects[0].SetActive(_entity.HasTag("Poisoned"));
        statusEffects[1].SetActive(_entity.HasTag("Buffed"));
        statusEffects[2].SetActive(_entity.HasTag("Stunned"));
    }
    
    private void LoadAvatar(string avatarPath)
    {
        if (string.IsNullOrEmpty(avatarPath)) return;
        
        // Load and set avatar sprite
        var sprite = Resources.Load<Sprite>(avatarPath);
        if (sprite != null)
            avatar.sprite = sprite;
    }
}
```

## Best Practices

### View Design
- **Stateless Views** – Derive all state from the associated entity, don't store duplicate data
- **Reactive Updates** – Subscribe to `OnStateChanged` for automatic UI synchronization
- **Null Safety** – Always check entity and view state before operations
- **Resource Management** – Properly unsubscribe from events in `Hide()`

### Entity Association
```csharp
// ✅ Good: Check entity state, don't store duplicate data
private void UpdateDisplay()
{
    if (_entity == null) return;
    
    int health = _entity.GetValue<int>("Health");
    UpdateHealthBar(health);
}

// ❌ Bad: Storing duplicate state in view
private int _cachedHealth;
private void UpdateDisplay()
{
    UpdateHealthBar(_cachedHealth); // Out of sync risk
}
```

### Event Management
```csharp
// ✅ Good: Proper event lifecycle
public void Show(IEntity entity)
{
    _entity = entity;
    entity.OnStateChanged += OnEntityStateChanged;
    gameObject.SetActive(true);
}

public void Hide()
{
    if (_entity != null)
    {
        _entity.OnStateChanged -= OnEntityStateChanged; // Clean up!
    }
    gameObject.SetActive(false);
}

// ❌ Bad: Memory leak through missing cleanup
public void Hide()
{
    gameObject.SetActive(false); // Event subscription remains!
}
```

## Performance Considerations

### Efficient Updates
- **Selective Updates** – Only update UI elements that actually changed
- **Batched Updates** – Group multiple UI updates when possible
- **Update Frequency** – Consider update rates for expensive operations

```csharp
private void OnEntityStateChanged()
{
    // Only update if view is visible and entity exists
    if (!_isVisible || _entity == null) return;
    
    // Batch UI updates
    StartCoroutine(BatchedUIUpdate());
}

private IEnumerator BatchedUIUpdate()
{
    yield return null; // Wait one frame to batch changes
    
    // Update all UI elements at once
    SynchronizeWithEntity();
}
```

### Memory Management
- **Event Cleanup** – Always unsubscribe from entity events
- **Reference Management** – Clear entity references when hiding views
- **Pool Views** – Reuse view instances when possible

## Integration with Unity UI

### Canvas Integration
```csharp
public class CanvasEntityView : MonoBehaviour, IEntityView
{
    [SerializeField] private Canvas canvas;
    
    public void Show(IEntity entity)
    {
        _entity = entity;
        _isVisible = true;
        
        // Enable canvas for UI visibility
        canvas.enabled = true;
        
        // Update UI elements
        SynchronizeWithEntity();
    }
    
    public void Hide()
    {
        canvas.enabled = false;
        _isVisible = false;
    }
}
```

### World Space UI
```csharp
public class WorldSpaceInfoView : MonoBehaviour, IEntityView
{
    [SerializeField] private Canvas worldCanvas;
    
    public void Show(IEntity entity)
    {
        _entity = entity;
        _isVisible = true;
        
        // Position view in world space based on entity
        if (entity.TryGetValue("Position", out Vector3 position))
        {
            transform.position = position + Vector3.up * 2f; // Above entity
        }
        
        worldCanvas.enabled = true;
    }
}
```

## Notes

- **Separation of Concerns** – Views handle presentation, entities hold data
- **Reactive Architecture** – Views react to entity changes, maintaining consistency
- **Unity Integration** – Designed to work with GameObjects, Canvas, and Unity UI
- **Testability** – Interface-based design enables easy mocking and unit testing
- **Performance** – Efficient event-driven updates minimize unnecessary UI refreshes