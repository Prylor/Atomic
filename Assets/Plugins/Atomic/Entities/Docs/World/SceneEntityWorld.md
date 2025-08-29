# 🌍 SceneEntityWorld

`SceneEntityWorld` is a Unity MonoBehaviour implementation of `IEntityWorld` that manages collections of `SceneEntity` instances within a Unity scene. It bridges the gap between Atomic's entity system and Unity's component-based architecture, providing automatic entity discovery, lifecycle management, and seamless integration with Unity's update loops.

## Key Features

- **Unity MonoBehaviour Integration** – Works as a standard Unity component
- **Automatic Entity Discovery** – Scans scene for entities on startup
- **Lifecycle Synchronization** – Syncs with Unity's Start/OnEnable/OnDisable
- **Generic Type Support** – Can manage specialized SceneEntity types
- **Update Loop Integration** – Automatic registration with Atomic's update system
- **Scene Persistence** – Can optionally persist across scene loads
- **Inspector Configuration** – Full configuration via Unity Inspector

---

## Class Hierarchy

```csharp
// Non-generic convenience class
[AddComponentMenu("Atomic/Entities/Entity World")]
[DisallowMultipleComponent]
[DefaultExecutionOrder(-1000)]
public class SceneEntityWorld : SceneEntityWorld<SceneEntity>

// Generic base class
public class SceneEntityWorld<E> : MonoBehaviour, IEntityWorld<E> where E : SceneEntity
```

## Configuration Properties

### scanEntitiesOnAwake
- **Default**: `true`
- **Description**: Automatically scans scene for entities during Awake()
- **Usage**: Enables automatic entity discovery and registration

### includeInactiveOnScan
- **Default**: `true`
- **Description**: Includes inactive GameObjects when scanning for entities
- **Usage**: Allows managing entities that start disabled

### useUnityLifecycle
- **Default**: `true`
- **Description**: Automatically syncs world lifecycle with Unity MonoBehaviour events
- **Mapping**:
  - `Start()` → `Spawn()` + `Activate()`
  - `OnEnable()` → `Activate()` (if already started)
  - `OnDisable()` → `Deactivate()`
  - `OnDestroy()` → `Despawn()` + `Dispose()`

### _dontDestroyOnLoad
- **Default**: `false`
- **Description**: Prevents destruction when loading new scenes
- **Usage**: Creates persistent entity worlds across scenes

## Core Functionality

### Entity Management
```csharp
// Add/Remove entities
public bool Add(E entity)
public bool Remove(E entity)
public void Clear()
public bool Contains(E entity)

// Collection properties
public int Count { get; }
public bool IsReadOnly { get; }

// Events
public event Action<E> OnAdded
public event Action<E> OnRemoved
```

### Lifecycle Management
```csharp
// Lifecycle methods
public void Spawn()
public void Activate()
public void Deactivate()
public void Despawn()

// Lifecycle state
public bool IsSpawned { get; }
public bool IsActive { get; }

// Lifecycle events
public event Action OnSpawned
public event Action OnDespawned
public event Action OnActivated
public event Action OnDeactivated
```

### Update Integration
```csharp
// Update methods
public void OnUpdate(float deltaTime)
public void OnFixedUpdate(float deltaTime)
public void OnLateUpdate(float deltaTime)

// Update events
public event Action<float> OnUpdated
public event Action<float> OnFixedUpdated
public event Action<float> OnLateUpdated
```

## Example Usage

### Basic Scene World Setup

```csharp
public class GameManager : MonoBehaviour
{
    [SerializeField] private SceneEntityWorld entityWorld;
    
    void Start()
    {
        // World automatically scans and manages entities
        // Access managed entities
        foreach (var entity in entityWorld)
        {
            Debug.Log($"Managing entity: {entity.Name}");
        }
        
        // Listen for entity changes
        entityWorld.OnAdded += OnEntityAdded;
        entityWorld.OnRemoved += OnEntityRemoved;
    }
    
    private void OnEntityAdded(SceneEntity entity)
    {
        Debug.Log($"Entity added to world: {entity.Name}");
        
        // Configure newly added entity
        if (entity.HasTag(EntityTags.ENEMY))
        {
            entity.AddBehaviour(new EnemyAIBehaviour());
        }
    }
    
    private void OnEntityRemoved(SceneEntity entity)
    {
        Debug.Log($"Entity removed from world: {entity.Name}");
    }
}
```

### Specialized Entity World

```csharp
// Custom entity type
public class PlayerEntity : SceneEntity
{
    [SerializeField] private int playerId;
    public int PlayerId => playerId;
}

// Specialized world for player entities
public class PlayerWorld : SceneEntityWorld<PlayerEntity>
{
    [SerializeField] private int maxPlayers = 4;
    
    protected override void Awake()
    {
        base.Awake();
        
        // Custom initialization
        this.OnAdded += OnPlayerAdded;
    }
    
    private void OnPlayerAdded(PlayerEntity player)
    {
        if (this.Count > maxPlayers)
        {
            Debug.LogWarning("Maximum players exceeded!");
        }
        
        // Setup player-specific systems
        player.AddValue(EntityNames.PLAYER_ID, player.PlayerId);
        player.AddBehaviour(new PlayerInputBehaviour());
    }
    
    public PlayerEntity GetPlayer(int playerId)
    {
        foreach (var player in this)
        {
            if (player.PlayerId == playerId)
                return player;
        }
        return null;
    }
}

// Usage
public class GameController : MonoBehaviour
{
    [SerializeField] private PlayerWorld playerWorld;
    
    void Start()
    {
        var mainPlayer = playerWorld.GetPlayer(0);
        if (mainPlayer != null)
        {
            // Focus camera on main player
            Camera.main.GetComponent<CameraFollow>().target = mainPlayer.transform;
        }
    }
}
```

### Runtime World Creation

```csharp
public class WorldManager : MonoBehaviour
{
    private SceneEntityWorld dynamicWorld;
    
    void Start()
    {
        // Create world programmatically
        dynamicWorld = SceneEntityWorld.Create(
            name: "Dynamic Entity World",
            scanEntities: false, // Don't auto-scan
            useUnityLifecycle: true
        );
        
        // Create and add entities manually
        CreatePlayerEntity();
        CreateEnemyEntity();
    }
    
    private void CreatePlayerEntity()
    {
        var playerGO = new GameObject("Player");
        var playerEntity = playerGO.AddComponent<SceneEntity>();
        
        // Configure entity
        playerEntity.AddValue(EntityNames.HEALTH, 100);
        playerEntity.AddTag(EntityTags.PLAYER);
        
        // Add to world
        dynamicWorld.Add(playerEntity);
    }
    
    private void CreateEnemyEntity()
    {
        var enemyGO = new GameObject("Enemy");
        var enemyEntity = enemyGO.AddComponent<SceneEntity>();
        
        // Configure entity
        enemyEntity.AddValue(EntityNames.HEALTH, 50);
        enemyEntity.AddTag(EntityTags.ENEMY);
        
        // Add to world
        dynamicWorld.Add(enemyEntity);
    }
    
    void OnDestroy()
    {
        // Clean up dynamic world
        if (dynamicWorld != null)
        {
            SceneEntityWorld.Destroy(dynamicWorld);
        }
    }
}
```

### Multi-Scene World Persistence

```csharp
public class PersistentEntityWorld : MonoBehaviour
{
    private static PersistentEntityWorld instance;
    private SceneEntityWorld entityWorld;
    
    void Awake()
    {
        // Singleton pattern
        if (instance != null)
        {
            Destroy(gameObject);
            return;
        }
        
        instance = this;
        DontDestroyOnLoad(gameObject);
        
        // Create persistent world
        entityWorld = SceneEntityWorld.Create(
            name: "Persistent World",
            scanEntities: false,
            useUnityLifecycle: true
        );
        
        // Make world persistent too
        entityWorld._dontDestroyOnLoad = true;
        DontDestroyOnLoad(entityWorld.gameObject);
    }
    
    void OnEnable()
    {
        SceneManager.sceneLoaded += OnSceneLoaded;
    }
    
    void OnDisable()
    {
        SceneManager.sceneLoaded -= OnSceneLoaded;
    }
    
    private void OnSceneLoaded(Scene scene, LoadSceneMode mode)
    {
        // Add persistent entities that should exist in all scenes
        CreatePersistentUI();
        CreatePersistentManagers();
    }
    
    private void CreatePersistentUI()
    {
        var uiGO = new GameObject("Persistent UI");
        var uiEntity = uiGO.AddComponent<SceneEntity>();
        
        uiEntity.AddValue(EntityNames.UI_CANVAS, uiGO.GetComponent<Canvas>());
        uiEntity.AddTag(EntityTags.UI);
        uiEntity.AddTag(EntityTags.PERSISTENT);
        
        entityWorld.Add(uiEntity);
    }
}
```

### Custom Update Loop Integration

```csharp
public class TimedEntityWorld : SceneEntityWorld
{
    [SerializeField] private float updateInterval = 0.1f;
    private float lastUpdateTime;
    
    protected override void Start()
    {
        // Don't use automatic Unity lifecycle
        useUnityLifecycle = false;
        
        // Manual lifecycle control
        this.Spawn();
        this.Activate();
        
        // Custom update registration
        StartCoroutine(CustomUpdateLoop());
    }
    
    private IEnumerator CustomUpdateLoop()
    {
        while (IsActive)
        {
            yield return new WaitForSeconds(updateInterval);
            
            float deltaTime = Time.time - lastUpdateTime;
            this.OnUpdate(deltaTime);
            lastUpdateTime = Time.time;
        }
    }
    
    protected override void OnDestroy()
    {
        this.Deactivate();
        this.Despawn();
        base.OnDestroy();
    }
}
```

### Entity Query and Filtering

```csharp
public class EntityQueryManager : MonoBehaviour
{
    [SerializeField] private SceneEntityWorld entityWorld;
    
    void Start()
    {
        // Wait for world initialization
        StartCoroutine(QueryEntities());
    }
    
    private IEnumerator QueryEntities()
    {
        yield return new WaitForEndOfFrame();
        
        // Query by tags
        var enemies = GetEntitiesWithTag(EntityTags.ENEMY);
        var players = GetEntitiesWithTag(EntityTags.PLAYER);
        var powerups = GetEntitiesWithTag(EntityTags.POWERUP);
        
        Debug.Log($"Found {enemies.Count} enemies, {players.Count} players, {powerups.Count} powerups");
        
        // Query by values
        var damagedEntities = GetEntitiesWithHealthBelow(50);
        Debug.Log($"Found {damagedEntities.Count} damaged entities");
    }
    
    private List<SceneEntity> GetEntitiesWithTag(object tag)
    {
        var results = new List<SceneEntity>();
        foreach (var entity in entityWorld)
        {
            if (entity.HasTag(tag))
                results.Add(entity);
        }
        return results;
    }
    
    private List<SceneEntity> GetEntitiesWithHealthBelow(int threshold)
    {
        var results = new List<SceneEntity>();
        foreach (var entity in entityWorld)
        {
            if (entity.HasValue(EntityNames.HEALTH))
            {
                var health = entity.GetValue<int>(EntityNames.HEALTH);
                if (health < threshold)
                    results.Add(entity);
            }
        }
        return results;
    }
}
```

### Debug and Development Tools

```csharp
#if UNITY_EDITOR
[CustomEditor(typeof(SceneEntityWorld), true)]
public class SceneEntityWorldEditor : Editor
{
    public override void OnInspectorGUI()
    {
        base.OnInspectorGUI();
        
        var world = (SceneEntityWorld)target;
        
        EditorGUILayout.Space();
        EditorGUILayout.LabelField("World State", EditorStyles.boldLabel);
        
        GUI.enabled = false;
        EditorGUILayout.IntField("Entity Count", world.Count);
        EditorGUILayout.Toggle("Is Spawned", world.IsSpawned);
        EditorGUILayout.Toggle("Is Active", world.IsActive);
        GUI.enabled = true;
        
        if (Application.isPlaying)
        {
            EditorGUILayout.Space();
            EditorGUILayout.LabelField("Runtime Controls", EditorStyles.boldLabel);
            
            using (new EditorGUILayout.HorizontalScope())
            {
                if (GUILayout.Button("Spawn"))
                    world.Spawn();
                if (GUILayout.Button("Despawn"))
                    world.Despawn();
            }
            
            using (new EditorGUILayout.HorizontalScope())
            {
                if (GUILayout.Button("Activate"))
                    world.Activate();
                if (GUILayout.Button("Deactivate"))
                    world.Deactivate();
            }
            
            if (GUILayout.Button("Clear All Entities"))
            {
                if (EditorUtility.DisplayDialog("Clear Entities", 
                    "Remove all entities from this world?", "Yes", "Cancel"))
                {
                    world.Clear();
                }
            }
            
            EditorGUILayout.Space();
            EditorGUILayout.LabelField("Entities", EditorStyles.boldLabel);
            
            foreach (var entity in world)
            {
                using (new EditorGUILayout.HorizontalScope())
                {
                    EditorGUILayout.ObjectField(entity, typeof(SceneEntity), true);
                    if (GUILayout.Button("Remove", GUILayout.Width(60)))
                    {
                        world.Remove(entity);
                        break;
                    }
                }
            }
        }
    }
}
#endif
```

## Static Factory Methods

### Create Method
```csharp
public static T Create<T>(
    string name = null,
    bool scanEntities = true,
    bool useUnityLifecycle = true
) where T : SceneEntityWorld<E>
```
Creates a new GameObject with attached world component.

### Destroy Method
```csharp
public static void Destroy(SceneEntityWorld<E> world, float t = 0)
```
Destroys the world's GameObject safely.

## Best Practices

1. **Single World Per Scene** – Use one primary world per scene for organization
2. **Specialized Worlds** – Create typed worlds for specific entity categories
3. **Lifecycle Management** – Let Unity lifecycle handle spawning/activation automatically
4. **Entity Discovery** – Use automatic scanning for scene-placed entities
5. **Manual Addition** – Add runtime-created entities manually
6. **World Persistence** – Use DontDestroyOnLoad sparingly for truly persistent worlds
7. **Custom Updates** – Override update behavior only when necessary

## Unity-Specific Considerations

### Execution Order
- Default execution order: -1000 (runs early)
- Ensures entities are managed before other systems

### Component Restrictions
- DisallowMultipleComponent prevents duplicates
- Use specialized worlds for different entity types

### Scene Management
- Worlds are destroyed with scene changes by default
- Use persistence flags for cross-scene worlds

### Performance
- Automatic scanning has overhead proportional to scene size
- Consider disabling scan for dynamic-only worlds
- Update loop integration is efficient but has Unity overhead

## Integration Patterns

### With Entity Pools
```csharp
public class PooledEntityWorld : SceneEntityWorld
{
    [SerializeField] private PrefabEntityPool entityPool;
    
    public SceneEntity SpawnFromPool()
    {
        var entity = entityPool.Get();
        this.Add(entity);
        return entity;
    }
    
    public void ReturnToPool(SceneEntity entity)
    {
        this.Remove(entity);
        entityPool.Return(entity);
    }
}
```

### With Entity Filters
```csharp
public class FilteredEntityWorld : SceneEntityWorld
{
    [SerializeField] private EntityFilter enemyFilter;
    [SerializeField] private EntityFilter playerFilter;
    
    protected override void Awake()
    {
        base.Awake();
        
        // Setup filters after entities are added
        this.OnAdded += entity =>
        {
            enemyFilter.TryAdd(entity);
            playerFilter.TryAdd(entity);
        };
    }
    
    public IReadOnlyEntityCollection GetEnemies() => enemyFilter;
    public IReadOnlyEntityCollection GetPlayers() => playerFilter;
}
```

## Common Issues and Solutions

### Entity Not Found
- Ensure scanEntitiesOnAwake is enabled
- Check includeInactiveOnScan setting
- Verify entities have SceneEntity components

### Lifecycle Sync Issues
- Check useUnityLifecycle setting
- Ensure proper MonoBehaviour lifecycle
- Avoid manual lifecycle calls when using Unity sync

### Performance Problems
- Disable scanning for runtime-only worlds
- Use specialized entity types to reduce casting
- Consider update frequency and deltaTime usage

## Notes

- SceneEntityWorld requires UNITY_5_3_OR_NEWER
- Supports Odin Inspector attributes for enhanced editor experience
- Generic type parameter must inherit from SceneEntity
- Implements full IEntityWorld interface for compatibility
- Integration with Atomic's UpdateLoop system for efficient updates