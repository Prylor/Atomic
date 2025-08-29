# 🔧 EntityViewInstaller

`EntityViewInstaller` is an abstract Unity MonoBehaviour base class that provides a standardized interface for installing and configuring `EntityView` instances. It enables modular, reusable installation logic that can be attached to GameObjects for flexible view setup patterns.

## Key Features

- **Abstract Installation Interface** – Standardized installation contract
- **MonoBehaviour Integration** – Full Unity component system compatibility
- **Modular Design** – Reusable installation logic across different views
- **Inheritance-Based** – Extend for custom installation behaviors
- **Unity-Specific** – Designed for Unity's component-based architecture

---

## Abstract Interface

```csharp
public abstract class EntityViewInstaller : MonoBehaviour
{
    /// <summary>
    /// Performs the installation logic for the specified EntityView.
    /// Must be implemented in derived classes.
    /// </summary>
    /// <param name="view">The EntityView instance to install.</param>
    public abstract void Install(EntityView view);
}
```

## Implementation Examples

### Basic View Configuration Installer

```csharp
[AddComponentMenu("Atomic/Views/Basic View Installer")]
public class BasicViewInstaller : EntityViewInstaller
{
    [Header("View Configuration")]
    [SerializeField] private string viewName = "Basic View";
    [SerializeField] private Color viewColor = Color.white;
    [SerializeField] private float updateInterval = 0.1f;
    
    public override void Install(EntityView view)
    {
        // Basic configuration
        view.name = viewName;
        
        // Apply visual settings
        var renderer = view.GetComponent<Renderer>();
        if (renderer != null)
        {
            renderer.material.color = viewColor;
        }
        
        // Set update interval if view supports it
        if (view is IUpdatableView updatable)
        {
            updatable.SetUpdateInterval(updateInterval);
        }
        
        Debug.Log($"Installed basic view configuration: {viewName}");
    }
}
```

### Component-Based View Installer

```csharp
[AddComponentMenu("Atomic/Views/Component View Installer")]
public class ComponentViewInstaller : EntityViewInstaller
{
    [Header("Required Components")]
    [SerializeField] private bool requireRenderer = true;
    [SerializeField] private bool requireCollider = true;
    [SerializeField] private bool requireRigidbody = false;
    
    [Header("Component Settings")]
    [SerializeField] private Material defaultMaterial;
    [SerializeField] private PhysicMaterial physicMaterial;
    [SerializeField] private float mass = 1.0f;
    
    public override void Install(EntityView view)
    {
        GameObject viewObject = view.gameObject;
        
        // Add required components
        if (requireRenderer)
        {
            var renderer = EnsureComponent<Renderer>(viewObject);
            if (defaultMaterial != null)
            {
                renderer.material = defaultMaterial;
            }
        }
        
        if (requireCollider)
        {
            var collider = EnsureComponent<Collider>(viewObject);
            if (physicMaterial != null)
            {
                collider.material = physicMaterial;
            }
        }
        
        if (requireRigidbody)
        {
            var rigidbody = EnsureComponent<Rigidbody>(viewObject);
            rigidbody.mass = mass;
        }
        
        Debug.Log($"Installed components for view: {view.name}");
    }
    
    private T EnsureComponent<T>(GameObject obj) where T : Component
    {
        var component = obj.GetComponent<T>();
        if (component == null)
        {
            component = obj.AddComponent<T>();
        }
        return component;
    }
}
```

### Animated View Installer

```csharp
[AddComponentMenu("Atomic/Views/Animated View Installer")]
public class AnimatedViewInstaller : EntityViewInstaller
{
    [Header("Animation Configuration")]
    [SerializeField] private AnimationClip idleAnimation;
    [SerializeField] private AnimationClip spawnAnimation;
    [SerializeField] private AnimationClip despawnAnimation;
    [SerializeField] private bool autoPlay = true;
    
    public override void Install(EntityView view)
    {
        // Setup Animation component
        var animation = view.GetComponent<Animation>();
        if (animation == null)
        {
            animation = view.gameObject.AddComponent<Animation>();
        }
        
        // Add animation clips
        if (idleAnimation != null)
        {
            animation.AddClip(idleAnimation, "Idle");
        }
        
        if (spawnAnimation != null)
        {
            animation.AddClip(spawnAnimation, "Spawn");
        }
        
        if (despawnAnimation != null)
        {
            animation.AddClip(despawnAnimation, "Despawn");
        }
        
        // Setup view animation integration
        if (view is IAnimatedView animatedView)
        {
            animatedView.SetAnimationClips(idleAnimation, spawnAnimation, despawnAnimation);
            
            if (autoPlay && idleAnimation != null)
            {
                animation.Play("Idle");
            }
        }
        
        Debug.Log($"Installed animation system for view: {view.name}");
    }
}
```

### Data-Driven View Installer

```csharp
[CreateAssetMenu(menuName = "Atomic/Views/View Installation Data")]
public class ViewInstallationData : ScriptableObject
{
    [Header("View Properties")]
    public string viewName;
    public Color primaryColor = Color.white;
    public Color secondaryColor = Color.gray;
    public Vector3 scale = Vector3.one;
    
    [Header("Materials")]
    public Material[] materials;
    
    [Header("Audio")]
    public AudioClip spawnSound;
    public AudioClip despawnSound;
    
    [Header("Effects")]
    public ParticleSystem spawnEffect;
    public ParticleSystem idleEffect;
}

[AddComponentMenu("Atomic/Views/Data Driven View Installer")]
public class DataDrivenViewInstaller : EntityViewInstaller
{
    [Header("Installation Data")]
    [SerializeField] private ViewInstallationData installationData;
    
    public override void Install(EntityView view)
    {
        if (installationData == null)
        {
            Debug.LogWarning("No installation data provided!");
            return;
        }
        
        ApplyBasicProperties(view);
        ApplyMaterials(view);
        SetupAudio(view);
        SetupEffects(view);
    }
    
    private void ApplyBasicProperties(EntityView view)
    {
        view.name = installationData.viewName;
        view.transform.localScale = installationData.scale;
        
        // Apply colors to renderer
        var renderer = view.GetComponent<Renderer>();
        if (renderer != null)
        {
            renderer.material.color = installationData.primaryColor;
        }
    }
    
    private void ApplyMaterials(EntityView view)
    {
        var renderers = view.GetComponentsInChildren<Renderer>();
        
        for (int i = 0; i < renderers.Length && i < installationData.materials.Length; i++)
        {
            if (installationData.materials[i] != null)
            {
                renderers[i].material = installationData.materials[i];
            }
        }
    }
    
    private void SetupAudio(EntityView view)
    {
        if (installationData.spawnSound != null || installationData.despawnSound != null)
        {
            var audioSource = view.GetComponent<AudioSource>();
            if (audioSource == null)
            {
                audioSource = view.gameObject.AddComponent<AudioSource>();
            }
            
            if (view is IAudioView audioView)
            {
                audioView.SetAudioClips(installationData.spawnSound, installationData.despawnSound);
            }
        }
    }
    
    private void SetupEffects(EntityView view)
    {
        if (installationData.spawnEffect != null)
        {
            var spawnEffect = Instantiate(installationData.spawnEffect, view.transform);
            spawnEffect.name = "Spawn Effect";
        }
        
        if (installationData.idleEffect != null)
        {
            var idleEffect = Instantiate(installationData.idleEffect, view.transform);
            idleEffect.name = "Idle Effect";
        }
    }
}
```

### Conditional View Installer

```csharp
[AddComponentMenu("Atomic/Views/Conditional View Installer")]
public class ConditionalViewInstaller : EntityViewInstaller
{
    [Header("Installation Conditions")]
    [SerializeField] private InstallationCondition[] conditions;
    [SerializeField] private EntityViewInstaller[] conditionalInstallers;
    
    [System.Serializable]
    public class InstallationCondition
    {
        public string conditionName;
        public ConditionType type;
        public string targetTag;
        public string targetValue;
        public object expectedValue;
        
        public enum ConditionType
        {
            HasEntityTag,
            EntityValueEquals,
            ComponentExists,
            SceneContains
        }
    }
    
    public override void Install(EntityView view)
    {
        var entity = view.Entity;
        if (entity == null)
        {
            Debug.LogWarning("View has no associated entity for conditional installation");
            return;
        }
        
        // Evaluate conditions and install appropriate components
        for (int i = 0; i < conditions.Length && i < conditionalInstallers.Length; i++)
        {
            if (EvaluateCondition(conditions[i], view, entity))
            {
                var installer = conditionalInstallers[i];
                if (installer != null)
                {
                    installer.Install(view);
                    Debug.Log($"Applied conditional installer: {installer.name}");
                }
            }
        }
    }
    
    private bool EvaluateCondition(InstallationCondition condition, EntityView view, IEntity entity)
    {
        switch (condition.type)
        {
            case InstallationCondition.ConditionType.HasEntityTag:
                int tagId = EntityNames.NameToId(condition.targetTag);
                return entity.HasTag(tagId);
                
            case InstallationCondition.ConditionType.EntityValueEquals:
                int valueId = EntityNames.NameToId(condition.targetValue);
                if (entity.TryGetValue(valueId, out object value))
                {
                    return value?.Equals(condition.expectedValue) == true;
                }
                return false;
                
            case InstallationCondition.ConditionType.ComponentExists:
                System.Type componentType = System.Type.GetType(condition.targetTag);
                return componentType != null && view.GetComponent(componentType) != null;
                
            case InstallationCondition.ConditionType.SceneContains:
                return FindObjectsOfType<GameObject>()
                    .Any(go => go.name.Contains(condition.targetTag));
                
            default:
                return false;
        }
    }
}
```

## Best Practices

1. **Single Responsibility** – Each installer should handle a specific aspect of view configuration
2. **Null Checking** – Always validate inputs and component existence
3. **Error Handling** – Gracefully handle missing dependencies
4. **Component Management** – Use helper methods for component addition/configuration
5. **Data-Driven Design** – Consider using ScriptableObjects for complex configurations
6. **Performance** – Cache expensive operations and component lookups
7. **Modularity** – Create small, focused installers that can be combined

## Integration Patterns

### Installer Chain Pattern
```csharp
public class InstallerChain : EntityViewInstaller
{
    [SerializeField] private EntityViewInstaller[] installers;
    
    public override void Install(EntityView view)
    {
        foreach (var installer in installers)
        {
            if (installer != null)
            {
                installer.Install(view);
            }
        }
    }
}
```

### Factory Integration
```csharp
public class ViewFactory : MonoBehaviour
{
    public EntityView CreateView(GameObject prefab, IEntity entity)
    {
        var viewObject = Instantiate(prefab);
        var view = viewObject.GetComponent<EntityView>();
        
        // Apply all installers found on the prefab
        var installers = viewObject.GetComponents<EntityViewInstaller>();
        foreach (var installer in installers)
        {
            installer.Install(view);
        }
        
        // Bind to entity
        view.BindToEntity(entity);
        
        return view;
    }
}
```

## Notes

- EntityViewInstaller is Unity-specific and requires UNITY_5_3_OR_NEWER
- Abstract class enforces implementation of Install method in derived classes
- Designed to work seamlessly with Unity's component system
- Supports composition through multiple installers on the same GameObject
- Can be used with both runtime and editor workflows
- Integration with ScriptableObjects enables data-driven view configuration