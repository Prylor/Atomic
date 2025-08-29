# 🧩 SceneEntityProxy

`SceneEntityProxy` is a Unity MonoBehaviour that acts as a proxy or reference to an existing `SceneEntity`. It allows multiple GameObjects to share and reference the same entity instance, enabling flexible entity architectures.

## Key Features

- **Entity Reference** – Points to an existing SceneEntity
- **Proxy Pattern** – Multiple proxies can reference one entity
- **Inspector Configuration** – Set entity reference in Unity Editor
- **Null Safety** – Handles missing entity references gracefully
- **Delegation** – Forwards IEntity interface calls to target

---

## Class Definition

```csharp
[AddComponentMenu("Atomic/Entities/Entity Proxy")]
public class SceneEntityProxy : MonoBehaviour, IEntity
{
    [SerializeField]
    private SceneEntity targetEntity;
    
    public SceneEntity Target
    {
        get => targetEntity;
        set => targetEntity = value;
    }
    
    // IEntity implementation delegates to target
    public int InstanceID => targetEntity?.InstanceID ?? -1;
    public string Name 
    { 
        get => targetEntity?.Name ?? gameObject.name;
        set { if (targetEntity != null) targetEntity.Name = value; }
    }
    
    // All IEntity methods delegate to targetEntity
}
```

## Use Cases

### Shared Entity Reference

```csharp
public class WeaponSystem : MonoBehaviour
{
    private SceneEntityProxy entityProxy;
    
    void Awake()
    {
        entityProxy = GetComponent<SceneEntityProxy>();
    }
    
    public void Fire()
    {
        // Access the actual entity through proxy
        var entity = entityProxy.Target;
        if (entity != null)
        {
            var damage = entity.GetValue<int>(EntityNames.DAMAGE);
            var ammo = entity.GetValue<int>(EntityNames.AMMO);
            
            if (ammo > 0)
            {
                PerformAttack(damage);
                entity.SetValue(EntityNames.AMMO, ammo - 1);
            }
        }
    }
}
```

### Multi-Component Architecture

```csharp
// Single entity, multiple component GameObjects
public class VehicleSetup : MonoBehaviour
{
    [SerializeField] private SceneEntity vehicleEntity;
    [SerializeField] private GameObject[] componentObjects;
    
    void Start()
    {
        // Give each component a proxy to the main entity
        foreach (var obj in componentObjects)
        {
            var proxy = obj.AddComponent<SceneEntityProxy>();
            proxy.Target = vehicleEntity;
        }
    }
}
```

### Entity Switching

```csharp
public class CharacterController : MonoBehaviour
{
    private SceneEntityProxy proxy;
    private List<SceneEntity> availableCharacters;
    private int currentIndex;
    
    void Start()
    {
        proxy = GetComponent<SceneEntityProxy>();
    }
    
    public void SwitchCharacter()
    {
        currentIndex = (currentIndex + 1) % availableCharacters.Count;
        proxy.Target = availableCharacters[currentIndex];
        
        // UI and systems automatically use new entity
        OnCharacterSwitched?.Invoke(proxy.Target);
    }
}
```

### Procedural Proxy Management

```csharp
public static class ProxyUtils
{
    public static SceneEntityProxy CreateProxy(GameObject obj, SceneEntity target)
    {
        var proxy = obj.AddComponent<SceneEntityProxy>();
        proxy.Target = target;
        return proxy;
    }
    
    public static void UpdateProxyTargets(SceneEntity newTarget, params GameObject[] objects)
    {
        foreach (var obj in objects)
        {
            var proxy = obj.GetComponent<SceneEntityProxy>();
            if (proxy != null)
            {
                proxy.Target = newTarget;
            }
        }
    }
    
    public static List<SceneEntityProxy> FindProxiesFor(SceneEntity entity)
    {
        var allProxies = Object.FindObjectsOfType<SceneEntityProxy>();
        return allProxies.Where(p => p.Target == entity).ToList();
    }
}
```

## Null Safety Pattern

```csharp
public class SafeProxyAccess : MonoBehaviour
{
    private SceneEntityProxy proxy;
    
    void Update()
    {
        // Safe access pattern
        if (proxy?.Target != null)
        {
            // Entity exists and is valid
            ProcessEntity(proxy.Target);
        }
    }
    
    private bool TryGetEntityValue<T>(int key, out T value)
    {
        value = default;
        
        if (proxy?.Target == null)
            return false;
            
        return proxy.Target.TryGetValue(key, out value);
    }
}
```

## Best Practices

1. **Null Check Target** – Always verify Target is not null
2. **Cache References** – Store proxy reference in Awake/Start
3. **Event Cleanup** – Unsubscribe when target changes
4. **Validate in Editor** – Add custom inspector to validate target
5. **Use for Sharing** – Best for multiple objects needing same entity

## Common Patterns

### Proxy Chain
```csharp
// Proxy pointing to another proxy
var proxy1 = obj1.GetComponent<SceneEntityProxy>();
var proxy2 = obj2.GetComponent<SceneEntityProxy>();
proxy2.Target = proxy1.Target; // Share same entity
```

### Dynamic Binding
```csharp
// Bind proxy at runtime
void BindToNearestEnemy()
{
    var nearestEnemy = FindNearestEnemy();
    proxy.Target = nearestEnemy;
}
```

## Notes

- Proxy adds minimal overhead (one reference)
- Useful for decoupling systems from specific entities
- Can create proxy chains but avoid deep nesting
- Consider using events when target changes