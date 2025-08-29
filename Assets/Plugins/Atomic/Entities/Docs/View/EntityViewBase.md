# 🎭 EntityViewBase

`EntityViewBase` is an abstract base class that provides common functionality for entity view implementations. It serves as the foundation for creating concrete view classes that represent entities in Unity's visual and interactive systems.

## Key Features

- **Abstract Base Implementation** – Common view functionality foundation
- **Entity Binding** – Standardized entity association patterns
- **Unity Integration** – Seamless MonoBehaviour compatibility
- **Lifecycle Management** – View state and lifecycle coordination
- **Extensible Design** – Virtual methods for customization

---

## Base Implementation Structure

```csharp
public abstract class EntityViewBase : MonoBehaviour, IReadOnlyEntityView
{
    // Core properties
    public IEntity Entity { get; protected set; }
    public bool IsBound => Entity != null;
    
    // Abstract methods for concrete implementation
    protected abstract void OnEntityBound(IEntity entity);
    protected abstract void OnEntityUnbound(IEntity entity);
    protected abstract void OnEntityStateChanged();
    
    // Virtual methods for optional override
    protected virtual void OnViewActivated() { }
    protected virtual void OnViewDeactivated() { }
    protected virtual void OnViewDestroyed() { }
    
    // Public interface
    public void BindToEntity(IEntity entity) { /* ... */ }
    public void UnbindFromEntity() { /* ... */ }
}
```

## Implementation Examples

### Basic Entity View

```csharp
public class BasicEntityView : EntityViewBase
{
    [Header("View Components")]
    [SerializeField] private Renderer viewRenderer;
    [SerializeField] private Collider viewCollider;
    [SerializeField] private TextMesh nameLabel;
    
    [Header("Visual Settings")]
    [SerializeField] private Color defaultColor = Color.white;
    [SerializeField] private Material defaultMaterial;
    
    protected override void OnEntityBound(IEntity entity)
    {
        Debug.Log($"View bound to entity: {entity.Name}");
        
        // Update visual representation
        UpdateVisuals();
        
        // Subscribe to entity events
        entity.OnStateChanged += OnEntityStateChanged;
        
        // Set initial state
        if (nameLabel != null)
        {
            nameLabel.text = entity.Name;
        }
    }
    
    protected override void OnEntityUnbound(IEntity entity)
    {
        Debug.Log($"View unbound from entity: {entity.Name}");
        
        // Unsubscribe from events
        entity.OnStateChanged -= OnEntityStateChanged;
        
        // Reset visual state
        ResetVisuals();
    }
    
    protected override void OnEntityStateChanged()
    {
        UpdateVisuals();
    }
    
    private void UpdateVisuals()
    {
        if (!IsBound) return;
        
        // Update color based on entity state
        Color viewColor = defaultColor;
        
        if (Entity.HasTag(EntityTags.SELECTED))
        {
            viewColor = Color.yellow;
        }
        else if (Entity.HasTag(EntityTags.DAMAGED))
        {
            viewColor = Color.red;
        }
        else if (Entity.HasTag(EntityTags.HEALED))
        {
            viewColor = Color.green;
        }
        
        if (viewRenderer != null)
        {
            viewRenderer.material.color = viewColor;
        }
        
        // Update collider based on entity state
        if (viewCollider != null)
        {
            viewCollider.enabled = !Entity.HasTag(EntityTags.INTANGIBLE);
        }
    }
    
    private void ResetVisuals()
    {
        if (viewRenderer != null)
        {
            viewRenderer.material.color = defaultColor;
        }
        
        if (nameLabel != null)
        {
            nameLabel.text = "";
        }
    }
    
    protected override void OnViewActivated()
    {
        base.OnViewActivated();
        gameObject.SetActive(true);
    }
    
    protected override void OnViewDeactivated()
    {
        base.OnViewDeactivated();
        gameObject.SetActive(false);
    }
}
```

### Advanced Entity View with Animation

```csharp
public class AnimatedEntityView : EntityViewBase
{
    [Header("Animation")]
    [SerializeField] private Animator animator;
    [SerializeField] private AnimationClip idleClip;
    [SerializeField] private AnimationClip moveClip;
    [SerializeField] private AnimationClip attackClip;
    
    [Header("Movement")]
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private bool smoothMovement = true;
    
    private Vector3 targetPosition;
    private Vector3 lastPosition;
    private bool isMoving;
    
    protected override void OnEntityBound(IEntity entity)
    {
        base.OnEntityBound(entity);
        
        // Initialize animation
        if (animator == null)
        {
            animator = GetComponent<Animator>();
        }
        
        // Set initial position
        if (entity.TryGetValue<Vector3>(EntityNames.POSITION, out var position))
        {
            transform.position = position;
            targetPosition = position;
            lastPosition = position;
        }
        
        // Subscribe to value changes
        entity.OnValueChanged += OnEntityValueChanged;
        
        PlayAnimation("Idle");
    }
    
    protected override void OnEntityUnbound(IEntity entity)
    {
        entity.OnValueChanged -= OnEntityValueChanged;
        base.OnEntityUnbound(entity);
    }
    
    private void OnEntityValueChanged(IEntity entity, int valueKey)
    {
        // Handle position changes
        if (valueKey == EntityNames.POSITION)
        {
            if (entity.TryGetValue<Vector3>(EntityNames.POSITION, out var newPosition))
            {
                targetPosition = newPosition;
                
                if (!smoothMovement)
                {
                    transform.position = targetPosition;
                }
            }
        }
        
        // Handle animation state changes
        if (valueKey == EntityNames.ANIMATION_STATE)
        {
            if (entity.TryGetValue<string>(EntityNames.ANIMATION_STATE, out var animState))
            {
                PlayAnimation(animState);
            }
        }
    }
    
    void Update()
    {
        if (!IsBound) return;
        
        UpdateMovement();
        UpdateAnimationState();
    }
    
    private void UpdateMovement()
    {
        if (smoothMovement && Vector3.Distance(transform.position, targetPosition) > 0.01f)
        {
            var previousPosition = transform.position;
            transform.position = Vector3.MoveTowards(
                transform.position, 
                targetPosition, 
                moveSpeed * Time.deltaTime
            );
            
            // Check if we're moving
            bool wasMoving = isMoving;
            isMoving = Vector3.Distance(previousPosition, transform.position) > 0.001f;
            
            if (isMoving != wasMoving)
            {
                PlayAnimation(isMoving ? "Move" : "Idle");
            }
            
            // Update entity position if we're controlling movement
            if (Entity.HasTag(EntityTags.VIEW_CONTROLLED_POSITION))
            {
                Entity.SetValue(EntityNames.POSITION, transform.position);
            }
        }
    }
    
    private void UpdateAnimationState()
    {
        // Update animator parameters based on entity state
        if (animator != null)
        {
            animator.SetBool("IsMoving", isMoving);
            animator.SetBool("IsAttacking", Entity.HasTag(EntityTags.ATTACKING));
            animator.SetBool("IsDead", Entity.HasTag(EntityTags.DEAD));
            
            if (Entity.TryGetValue<float>(EntityNames.HEALTH_PERCENTAGE, out var healthPercent))
            {
                animator.SetFloat("HealthPercent", healthPercent);
            }
        }
    }
    
    private void PlayAnimation(string animationName)
    {
        if (animator != null && animator.runtimeAnimatorController != null)
        {
            animator.SetTrigger(animationName);
        }
    }
    
    // Animation event handlers
    public void OnAttackAnimationComplete()
    {
        if (IsBound && Entity.HasTag(EntityTags.ATTACKING))
        {
            Entity.DelTag(EntityTags.ATTACKING);
            PlayAnimation("Idle");
        }
    }
    
    public void OnDeathAnimationComplete()
    {
        if (IsBound)
        {
            Entity.AddTag(EntityTags.DEATH_ANIMATION_COMPLETE);
        }
    }
}
```

### UI Entity View

```csharp
public class UIEntityView : EntityViewBase
{
    [Header("UI Components")]
    [SerializeField] private Text nameText;
    [SerializeField] private Slider healthBar;
    [SerializeField] private Image iconImage;
    [SerializeField] private Button actionButton;
    
    [Header("UI Settings")]
    [SerializeField] private bool followWorldPosition = true;
    [SerializeField] private Vector3 worldOffset = Vector3.up * 2f;
    [SerializeField] private Camera targetCamera;
    
    private Canvas parentCanvas;
    private RectTransform rectTransform;
    
    protected override void OnEntityBound(IEntity entity)
    {
        base.OnEntityBound(entity);
        
        // Initialize UI components
        rectTransform = GetComponent<RectTransform>();
        parentCanvas = GetComponentInParent<Canvas>();
        
        if (targetCamera == null)
        {
            targetCamera = Camera.main;
        }
        
        // Setup UI event handlers
        if (actionButton != null)
        {
            actionButton.onClick.AddListener(OnActionButtonClicked);
        }
        
        // Subscribe to entity events
        entity.OnValueChanged += OnEntityValueChanged;
        
        UpdateUIElements();
    }
    
    protected override void OnEntityUnbound(IEntity entity)
    {
        entity.OnValueChanged -= OnEntityValueChanged;
        
        if (actionButton != null)
        {
            actionButton.onClick.RemoveListener(OnActionButtonClicked);
        }
        
        base.OnEntityUnbound(entity);
    }
    
    void Update()
    {
        if (!IsBound) return;
        
        if (followWorldPosition)
        {
            UpdateWorldToScreenPosition();
        }
    }
    
    private void OnEntityValueChanged(IEntity entity, int valueKey)
    {
        UpdateUIElements();
    }
    
    private void UpdateUIElements()
    {
        if (!IsBound) return;
        
        // Update name
        if (nameText != null)
        {
            nameText.text = Entity.Name;
        }
        
        // Update health bar
        if (healthBar != null && Entity.TryGetValue<float>(EntityNames.HEALTH_PERCENTAGE, out var healthPercent))
        {
            healthBar.value = healthPercent;
            
            // Color coding
            var healthBarImage = healthBar.fillRect.GetComponent<Image>();
            if (healthBarImage != null)
            {
                healthBarImage.color = Color.Lerp(Color.red, Color.green, healthPercent);
            }
        }
        
        // Update icon based on entity type
        if (iconImage != null)
        {
            if (Entity.HasTag(EntityTags.PLAYER))
            {
                iconImage.sprite = Resources.Load<Sprite>("Icons/Player");
            }
            else if (Entity.HasTag(EntityTags.ENEMY))
            {
                iconImage.sprite = Resources.Load<Sprite>("Icons/Enemy");
            }
        }
        
        // Update button state
        if (actionButton != null)
        {
            actionButton.interactable = Entity.HasTag(EntityTags.INTERACTABLE) && 
                                       !Entity.HasTag(EntityTags.BUSY);
        }
    }
    
    private void UpdateWorldToScreenPosition()
    {
        if (Entity.TryGetValue<Vector3>(EntityNames.POSITION, out var worldPosition))
        {
            Vector3 screenPoint = targetCamera.WorldToScreenPoint(worldPosition + worldOffset);
            
            if (screenPoint.z > 0) // In front of camera
            {
                Vector2 screenPosition;
                RectTransformUtility.ScreenPointToLocalPointInRectangle(
                    parentCanvas.transform as RectTransform,
                    screenPoint,
                    parentCanvas.worldCamera,
                    out screenPosition
                );
                
                rectTransform.localPosition = screenPosition;
                gameObject.SetActive(true);
            }
            else
            {
                gameObject.SetActive(false);
            }
        }
    }
    
    private void OnActionButtonClicked()
    {
        if (IsBound)
        {
            // Trigger entity interaction
            Entity.AddTag(EntityTags.INTERACTION_REQUESTED);
            
            // Or call specific interaction behavior
            if (Entity.TryGetBehaviour<IInteractableBehaviour>(out var interaction))
            {
                interaction.Interact();
            }
        }
    }
    
    protected override void OnViewActivated()
    {
        base.OnViewActivated();
        gameObject.SetActive(true);
    }
    
    protected override void OnViewDeactivated()
    {
        base.OnViewDeactivated();
        gameObject.SetActive(false);
    }
}
```

### Poolable Entity View

```csharp
public class PoolableEntityView : EntityViewBase, IPoolableView, IResettableView
{
    [Header("Pooling")]
    [SerializeField] private bool resetTransformOnReturn = true;
    [SerializeField] private bool resetMaterialsOnReturn = true;
    
    private Vector3 originalPosition;
    private Quaternion originalRotation;
    private Vector3 originalScale;
    private Material[] originalMaterials;
    
    void Awake()
    {
        // Store original values for pooling reset
        originalPosition = transform.position;
        originalRotation = transform.rotation;
        originalScale = transform.localScale;
        
        var renderers = GetComponentsInChildren<Renderer>();
        originalMaterials = new Material[renderers.Length];
        for (int i = 0; i < renderers.Length; i++)
        {
            originalMaterials[i] = renderers[i].material;
        }
    }
    
    // IPoolableView implementation
    public void OnRentFromPool()
    {
        gameObject.SetActive(true);
        Debug.Log($"View rented from pool: {name}");
    }
    
    public void OnReturnToPool()
    {
        // Unbind from entity
        if (IsBound)
        {
            UnbindFromEntity();
        }
        
        // Reset state
        Reset();
        
        gameObject.SetActive(false);
        Debug.Log($"View returned to pool: {name}");
    }
    
    public void PrepareForPooling()
    {
        // Additional cleanup before pooling
        StopAllCoroutines();
        
        // Clear any temporary components
        var tempComponents = GetComponents<ITempComponent>();
        foreach (var temp in tempComponents)
        {
            Destroy(temp as Component);
        }
    }
    
    // IResettableView implementation
    public void Reset()
    {
        if (resetTransformOnReturn)
        {
            transform.position = originalPosition;
            transform.rotation = originalRotation;
            transform.localScale = originalScale;
        }
        
        if (resetMaterialsOnReturn)
        {
            var renderers = GetComponentsInChildren<Renderer>();
            for (int i = 0; i < renderers.Length && i < originalMaterials.Length; i++)
            {
                renderers[i].material = originalMaterials[i];
            }
        }
        
        // Reset animation state
        var animator = GetComponent<Animator>();
        if (animator != null)
        {
            animator.Rebind();
            animator.Update(0f);
        }
        
        // Reset particle systems
        var particleSystems = GetComponentsInChildren<ParticleSystem>();
        foreach (var ps in particleSystems)
        {
            ps.Stop();
            ps.Clear();
        }
    }
    
    protected override void OnEntityBound(IEntity entity)
    {
        base.OnEntityBound(entity);
        
        // View is now active and bound
        Debug.Log($"Pooled view bound to entity: {entity.Name}");
    }
    
    protected override void OnEntityUnbound(IEntity entity)
    {
        Debug.Log($"Pooled view unbound from entity: {entity.Name}");
        base.OnEntityUnbound(entity);
    }
}
```

## Base Class Utilities

### Common Helper Methods
```csharp
protected void ValidateEntity(IEntity entity)
{
    if (entity == null)
        throw new ArgumentNullException(nameof(entity));
}

protected void SafeUnsubscribeFromEntity(IEntity entity)
{
    if (entity != null)
    {
        entity.OnStateChanged -= OnEntityStateChanged;
        entity.OnValueChanged -= OnEntityValueChanged;
    }
}

protected void UpdateTransformFromEntity()
{
    if (!IsBound) return;
    
    if (Entity.TryGetValue<Vector3>(EntityNames.POSITION, out var position))
    {
        transform.position = position;
    }
    
    if (Entity.TryGetValue<Quaternion>(EntityNames.ROTATION, out var rotation))
    {
        transform.rotation = rotation;
    }
}
```

## Best Practices

1. **Entity Binding** – Always validate entity binding state
2. **Event Management** – Properly subscribe/unsubscribe from entity events
3. **Resource Cleanup** – Clean up resources in OnEntityUnbound
4. **State Synchronization** – Keep view state synchronized with entity
5. **Performance** – Cache component references and avoid frequent lookups
6. **Pooling Support** – Implement pooling interfaces for reusable views
7. **Error Handling** – Handle null entities and missing components gracefully

## Integration Patterns

### View Factory Integration
```csharp
public class ViewFactory
{
    public EntityViewBase CreateViewForEntity(IEntity entity, string viewType)
    {
        var prefab = Resources.Load<GameObject>($"Views/{viewType}");
        var instance = Object.Instantiate(prefab);
        var view = instance.GetComponent<EntityViewBase>();
        
        if (view != null)
        {
            view.BindToEntity(entity);
        }
        
        return view;
    }
}
```

## Notes

- EntityViewBase provides a solid foundation for all entity view implementations
- Abstract methods enforce proper entity binding and state management
- Virtual methods allow customization while maintaining core functionality
- Designed for Unity's component system and MonoBehaviour lifecycle
- Supports both immediate and pooled view usage patterns
- Integration with entity events enables reactive view updates