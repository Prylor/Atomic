# 🧩 SceneEntity

`SceneEntity` is a Unity MonoBehaviour implementation of `IEntity` that bridges the Atomic framework with Unity's component system. It allows entities to be configured directly in the Unity Inspector and participate in Unity's lifecycle.

## Key Features

- **Unity Integration** – Full MonoBehaviour compatibility
- **Inspector Configuration** – Configure entities visually in Unity Editor
- **Automatic Installation** – Setup entities via installers and child entities
- **Unity Lifecycle Sync** – Automatic synchronization with Start/OnEnable/OnDisable
- **Scene Persistence** – Entities can be saved with Unity scenes
- **Editor Support** – Edit-mode installation for preview
- **Child Entity Support** – Hierarchical entity composition

---

## Class Definition

```csharp
[AddComponentMenu("Atomic/Entities/Entity")]
[DisallowMultipleComponent, DefaultExecutionOrder(-1000)]
public partial class SceneEntity : MonoBehaviour, IEntity, ISerializationCallbackReceiver
{
    public event Action OnStateChanged;
    public int InstanceID { get; }
    public string Name { get; set; }
    public bool Installed { get; }
    
    // Configuration
    public bool installOnAwake = true;
    public bool installInEditMode = false;
    public bool disposeValues = true;
    public bool useUnityLifecycle = true;
    
    // Composition
    public List<SceneEntityInstaller> installers;
    public List<SceneEntity> children;
}
```

## Configuration Properties

### installOnAwake
- **Default**: `true`
- **Description**: Automatically calls Install() during Awake()
- **Usage**: Enable for automatic setup

### installInEditMode
- **Default**: `false`
- **Description**: Calls Install() during OnValidate in Editor
- **Warning**: Disable if Install() creates heavy objects

### disposeValues
- **Default**: `true`
- **Description**: Disposes values when GameObject is destroyed
- **Usage**: Prevents memory leaks

### useUnityLifecycle
- **Default**: `true`
- **Description**: Syncs entity lifecycle with Unity MonoBehaviour
- **Mapping**:
  - `Start()` → `Spawn()`
  - `OnEnable()` → `Activate()`
  - `OnDisable()` → `Deactivate()`
  - `OnDestroy()` → `Despawn()` and `Dispose()`

## Installation System

### Install Method
```csharp
public void Install()
```
- Executes installers in order
- Installs child entities
- Sets up initial state
- Only runs once (idempotent)

### Installers
```csharp
[SerializeField]
internal List<SceneEntityInstaller> installers;
```
- Configure entity with reusable setups
- Applied in list order
- Can add values, tags, behaviours

### Child Entities
```csharp
[SerializeField]
internal List<SceneEntity> children;
```
- Automatically installed with parent
- Creates entity hierarchies
- Useful for complex game objects

## Example Usage

### Basic Scene Entity

```csharp
public class PlayerEntity : SceneEntity
{
    [SerializeField] private int maxHealth = 100;
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private GameObject model;
    
    protected override void OnInstall()
    {
        // Add initial values
        this.AddValue(EntityNames.MAX_HEALTH, maxHealth);
        this.AddValue(EntityNames.HEALTH, maxHealth);
        this.AddValue(EntityNames.SPEED, moveSpeed);
        this.AddValue(EntityNames.MODEL, model);
        
        // Add tags
        this.AddTag(EntityTags.PLAYER);
        this.AddTag(EntityTags.CONTROLLABLE);
        
        // Add behaviours
        this.AddBehaviour(new PlayerInputBehaviour());
        this.AddBehaviour(new HealthBehaviour());
    }
}
```

### Using Installers

```csharp
[CreateAssetMenu(menuName = "Entities/Enemy Installer")]
public class EnemyInstaller : ScriptableEntityInstaller
{
    [SerializeField] private int health = 50;
    [SerializeField] private int damage = 10;
    [SerializeField] private float patrolSpeed = 2f;
    
    public override void Install(IEntity entity)
    {
        entity.AddValue(EntityNames.HEALTH, health);
        entity.AddValue(EntityNames.DAMAGE, damage);
        entity.AddValue(EntityNames.PATROL_SPEED, patrolSpeed);
        
        entity.AddTag(EntityTags.ENEMY);
        entity.AddBehaviour(new EnemyAIBehaviour());
    }
}
```

### Hierarchical Entities

```csharp
public class VehicleEntity : SceneEntity
{
    // Child entities configured in Inspector:
    // - EngineEntity
    // - WeaponEntity
    // - ShieldEntity
    
    protected override void OnInstall()
    {
        // Parent setup
        this.AddValue(EntityNames.FUEL, 100f);
        
        // Children are automatically installed
        // Access them after installation
        var engine = children[0];
        var weapon = children[1];
        var shield = children[2];
        
        // Link children
        this.AddValue(EntityNames.ENGINE, engine);
        this.AddValue(EntityNames.WEAPON, weapon);
        this.AddValue(EntityNames.SHIELD, shield);
    }
}
```

### Unity Lifecycle Integration

```csharp
public class AnimatedEntity : SceneEntity
{
    private Animator animator;
    
    protected override void OnSpawn()
    {
        base.OnSpawn();
        animator = GetComponent<Animator>();
        Debug.Log("Entity spawned with Unity Start()");
    }
    
    protected override void OnActivate()
    {
        base.OnActivate();
        animator.enabled = true;
        Debug.Log("Entity activated with Unity OnEnable()");
    }
    
    protected override void OnDeactivate()
    {
        base.OnDeactivate();
        animator.enabled = false;
        Debug.Log("Entity deactivated with Unity OnDisable()");
    }
    
    protected override void OnDespawn()
    {
        base.OnDespawn();
        Debug.Log("Entity despawned with Unity OnDestroy()");
    }
}
```

### Edit Mode Preview

```csharp
public class PreviewableEntity : SceneEntity
{
    protected override void OnInstall()
    {
        // This runs in edit mode if installInEditMode is true
        #if UNITY_EDITOR
        if (!Application.isPlaying)
        {
            // Preview setup
            this.AddValue(EntityNames.PREVIEW_COLOR, Color.blue);
            UpdatePreview();
        }
        else
        #endif
        {
            // Runtime setup
            this.AddValue(EntityNames.COLOR, Color.white);
        }
    }
    
    #if UNITY_EDITOR
    private void UpdatePreview()
    {
        var renderer = GetComponent<Renderer>();
        if (renderer != null)
        {
            renderer.sharedMaterial.color = 
                this.GetValue<Color>(EntityNames.PREVIEW_COLOR);
        }
    }
    #endif
}
```

### Procedural Operations

Following Atomic's procedural pattern:

```csharp
public static class SceneEntityUtils
{
    public static T GetOrAddEntity<T>(GameObject gameObject) where T : SceneEntity
    {
        var entity = gameObject.GetComponent<T>();
        if (entity == null)
        {
            entity = gameObject.AddComponent<T>();
            entity.Install();
        }
        return entity;
    }
    
    public static void SetupPlayerEntity(GameObject player)
    {
        var entity = GetOrAddEntity<SceneEntity>(player);
        
        // Configure entity
        entity.AddValue(EntityNames.HEALTH, 100);
        entity.AddTag(EntityTags.PLAYER);
        entity.AddBehaviour(new PlayerBehaviour());
        
        // Setup Unity components
        var rigidbody = player.GetComponent<Rigidbody>();
        if (rigidbody != null)
        {
            entity.AddValue(EntityNames.RIGIDBODY, rigidbody);
        }
    }
    
    public static void ConvertToEntity(GameObject gameObject)
    {
        // Add entity component
        var entity = gameObject.AddComponent<SceneEntity>();
        
        // Auto-detect and add values from components
        var transform = gameObject.transform;
        entity.AddValue(EntityNames.POSITION, transform.position);
        entity.AddValue(EntityNames.ROTATION, transform.rotation);
        
        var renderer = gameObject.GetComponent<Renderer>();
        if (renderer != null)
        {
            entity.AddValue(EntityNames.RENDERER, renderer);
        }
        
        // Install and spawn
        entity.Install();
        if (Application.isPlaying)
        {
            entity.Spawn();
        }
    }
}
```

### Custom Inspector

```csharp
#if UNITY_EDITOR
using UnityEditor;

[CustomEditor(typeof(SceneEntity), true)]
public class SceneEntityEditor : Editor
{
    public override void OnInspectorGUI()
    {
        base.OnInspectorGUI();
        
        var entity = (SceneEntity)target;
        
        EditorGUILayout.Space();
        EditorGUILayout.LabelField("Runtime Info", EditorStyles.boldLabel);
        
        GUI.enabled = false;
        EditorGUILayout.IntField("Instance ID", entity.InstanceID);
        EditorGUILayout.Toggle("Installed", entity.Installed);
        EditorGUILayout.Toggle("Spawned", entity.Spawned);
        EditorGUILayout.Toggle("Enabled", entity.Enabled);
        GUI.enabled = true;
        
        if (Application.isPlaying)
        {
            EditorGUILayout.Space();
            if (GUILayout.Button("Install"))
                entity.Install();
            if (GUILayout.Button("Spawn"))
                entity.Spawn();
            if (GUILayout.Button("Despawn"))
                entity.Despawn();
        }
    }
}
#endif
```

## Best Practices

1. **Use Installers** – Keep entity configuration in reusable installers
2. **Avoid Heavy Operations** – Don't create GameObjects in Install() if using edit mode
3. **Lifecycle Awareness** – Understand Unity lifecycle mapping
4. **Child Management** – Use child entities for complex hierarchies
5. **Editor Preview** – Use installInEditMode carefully
6. **Disposal** – Always enable disposeValues to prevent leaks

## Unity-Specific Considerations

### Prefab Workflow
- SceneEntity works with prefabs
- Installers can be prefab overrides
- Child entities maintain prefab links

### Scene Management
- Entities persist with scene saves
- Installation state is not serialized
- Runtime values are not saved

### Performance
- Installation happens once
- Unity lifecycle has overhead
- Consider pooling for frequently spawned entities

## Common Patterns

### Entity Prefab Pool
```csharp
public class SceneEntityPool : MonoBehaviour
{
    [SerializeField] private SceneEntity prefab;
    [SerializeField] private int initialSize = 10;
    
    private Stack<SceneEntity> pool = new Stack<SceneEntity>();
    
    void Start()
    {
        for (int i = 0; i < initialSize; i++)
        {
            var instance = Instantiate(prefab);
            instance.gameObject.SetActive(false);
            pool.Push(instance);
        }
    }
    
    public SceneEntity Get()
    {
        SceneEntity entity = pool.Count > 0 
            ? pool.Pop() 
            : Instantiate(prefab);
            
        entity.gameObject.SetActive(true);
        entity.Install();
        entity.Spawn();
        return entity;
    }
    
    public void Return(SceneEntity entity)
    {
        entity.Despawn();
        entity.ClearValues();
        entity.ClearTags();
        entity.ClearBehaviours();
        entity.gameObject.SetActive(false);
        pool.Push(entity);
    }
}
```

## Notes

- SceneEntity is Unity-specific (requires UNITY_5_3_OR_NEWER)
- Implements ISerializationCallbackReceiver for Unity serialization
- Default execution order is -1000 (runs early)
- DisallowMultipleComponent prevents multiple entities per GameObject
- Supports Odin Inspector attributes for enhanced editor experience