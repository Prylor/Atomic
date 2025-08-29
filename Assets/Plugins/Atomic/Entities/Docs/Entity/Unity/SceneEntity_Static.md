# 🔧 SceneEntity_Static

The `SceneEntity_Static` partial class provides comprehensive static factory methods and utilities for creating, managing, and working with `SceneEntity` instances. It offers flexible creation patterns, type-safe casting operations, and batch processing capabilities for entity management at scale.

## Key Features

- **Factory Methods** – Multiple patterns for entity creation
- **Prefab Instantiation** – Seamless prefab-to-entity conversion
- **Type-Safe Casting** – Safe conversion between entity types
- **Batch Operations** – Install multiple entities efficiently
- **Flexible Configuration** – Comprehensive creation parameters
- **Memory Efficient** – Optimized for performance-critical scenarios

---

## Core Data Structures

### CreateArgs Structure
```csharp
[Serializable]
public struct CreateArgs
{
    // Basic configuration
    public string name;
    public bool installOnAwake;
    public bool disposeValues;
    public bool useUnityLifecycle;
    
    // Entity components
    public IEnumerable<int> tags;
    public IReadOnlyDictionary<int, object> values;
    public IEnumerable<IEntityBehaviour> behaviours;
    public List<SceneEntityInstaller> installers;
    public List<SceneEntity> children;
    
    // Performance optimization
    public int initialTagCapacity;
    public int initialValueCapacity;
    public int initialBehaviourCapacity;
}
```

## Factory Methods

### Basic Creation
```csharp
// Create entity with CreateArgs structure
public static SceneEntity Create(in CreateArgs args)
public static E Create<E>(in CreateArgs args) where E : SceneEntity

// Create entity with individual parameters
public static SceneEntity Create(
    string name = null,
    IEnumerable<int> tags = null,
    IReadOnlyDictionary<int, object> values = null,
    IEnumerable<IEntityBehaviour> behaviours = null,
    bool installOnAwake = true,
    bool disposeValues = true,
    bool useUnityLifecycle = true,
    int initialTagCount = 1,
    int initialValueCount = 1,
    int initialBehaviourCount = 1
)

public static E Create<E>(...) where E : SceneEntity
```

### Prefab Instantiation
```csharp
// Basic prefab creation
public static SceneEntity Create(SceneEntity prefab, Transform parent = null)
public static E Create<E>(E prefab, Transform parent = null) where E : SceneEntity

// Positioned prefab creation
public static SceneEntity Create(
    SceneEntity prefab,
    Vector3 position,
    Quaternion rotation,
    Transform parent = null
)

public static E Create<E>(
    E prefab,
    Vector3 position,
    Quaternion rotation,
    Transform parent = null
) where E : SceneEntity

// Transform-based creation
public static E Create<E>(E prefab, Transform point, Transform parent) where E : SceneEntity
```

### Destruction Methods
```csharp
// Destroy entity by interface
public static void Destroy(IEntity entity, float t = 0)

// Destroy SceneEntity directly
public static void Destroy(SceneEntity entity, float t = 0)
```

## Type Casting Operations

### Safe Casting
```csharp
// Cast IEntity to SceneEntity
public static SceneEntity Cast(IEntity entity)
public static E Cast<E>(IEntity entity) where E : SceneEntity

// Try cast with result output
public static bool TryCast(IEntity entity, out SceneEntity result)
public static bool TryCast<E>(IEntity entity, out E result) where E : SceneEntity
```

## Batch Operations

### Scene-Wide Installation
```csharp
// Install all SceneEntity instances in scene
public static void InstallAll(Scene scene)

// Install all entities of specific type in scene
public static void InstallAll<E>(Scene scene) where E : SceneEntity
```

## Example Usage

### Basic Entity Creation

```csharp
public class EntityFactory : MonoBehaviour
{
    void Start()
    {
        // Example 1: Simple entity creation
        var simpleEntity = SceneEntity.Create(
            name: "Simple Entity",
            installOnAwake: false,
            useUnityLifecycle: true
        );
        
        // Example 2: Entity with initial data
        var tags = new[] { EntityTags.PLAYER, EntityTags.CONTROLLABLE };
        var values = new Dictionary<int, object>
        {
            [EntityNames.HEALTH] = 100,
            [EntityNames.MAX_HEALTH] = 100,
            [EntityNames.SPEED] = 5.0f
        };
        var behaviours = new IEntityBehaviour[]
        {
            new PlayerInputBehaviour(),
            new HealthBehaviour(),
            new MovementBehaviour()
        };
        
        var configuredEntity = SceneEntity.Create(
            name: "Player Entity",
            tags: tags,
            values: values,
            behaviours: behaviours
        );
        
        // Manual installation and spawning
        configuredEntity.Install();
        configuredEntity.Spawn();
        configuredEntity.Activate();
    }
}
```

### Advanced Entity Factory

```csharp
public class AdvancedEntityFactory : MonoBehaviour
{
    [Header("Entity Templates")]
    [SerializeField] private EntityTemplate playerTemplate;
    [SerializeField] private EntityTemplate enemyTemplate;
    [SerializeField] private EntityTemplate npcTemplate;
    
    [Header("Factory Settings")]
    [SerializeField] private Transform entityParent;
    [SerializeField] private bool autoInstall = true;
    [SerializeField] private bool autoSpawn = true;
    
    public SceneEntity CreateFromTemplate(EntityTemplate template, Vector3 position)
    {
        var args = new SceneEntity.CreateArgs
        {
            name = template.entityName,
            tags = template.GetTagIds(),
            values = template.GetValueDictionary(),
            behaviours = template.GetBehaviours(),
            installers = template.installers,
            children = null,
            
            // Performance optimization
            initialTagCapacity = template.estimatedTagCount,
            initialValueCapacity = template.estimatedValueCount,
            initialBehaviourCapacity = template.estimatedBehaviourCount,
            
            // Configuration
            installOnAwake = autoInstall,
            disposeValues = true,
            useUnityLifecycle = true
        };
        
        var entity = SceneEntity.Create(args);
        entity.transform.position = position;
        entity.transform.SetParent(entityParent);
        
        if (autoInstall)
        {
            entity.Install();
            
            if (autoSpawn)
            {
                entity.Spawn();
                entity.Activate();
            }
        }
        
        return entity;
    }
    
    public T CreateSpecializedEntity<T>(Vector3 position, EntityConfiguration config) where T : SceneEntity
    {
        var args = new SceneEntity.CreateArgs
        {
            name = $"{typeof(T).Name}_{Time.frameCount}",
            tags = config.tags,
            values = config.values,
            behaviours = config.behaviours,
            
            initialTagCapacity = config.tagCapacity,
            initialValueCapacity = config.valueCapacity,
            initialBehaviourCapacity = config.behaviourCapacity,
            
            installOnAwake = false, // Manual control
            disposeValues = true,
            useUnityLifecycle = true
        };
        
        var entity = SceneEntity.Create<T>(args);
        entity.transform.position = position;
        entity.transform.SetParent(entityParent);
        
        // Custom post-creation setup
        PostCreateSetup(entity, config);
        
        return entity;
    }
    
    private void PostCreateSetup(SceneEntity entity, EntityConfiguration config)
    {
        // Add Unity components if specified
        foreach (var componentType in config.requiredComponents)
        {
            if (entity.GetComponent(componentType) == null)
            {
                entity.gameObject.AddComponent(componentType);
            }
        }
        
        // Apply visual settings
        if (config.material != null)
        {
            var renderer = entity.GetComponent<Renderer>();
            if (renderer != null)
                renderer.material = config.material;
        }
        
        // Install and initialize
        entity.Install();
        entity.Spawn();
        entity.Activate();
    }
}

[System.Serializable]
public class EntityConfiguration
{
    public List<int> tags = new List<int>();
    public Dictionary<int, object> values = new Dictionary<int, object>();
    public List<IEntityBehaviour> behaviours = new List<IEntityBehaviour>();
    public List<System.Type> requiredComponents = new List<System.Type>();
    public Material material;
    
    [Header("Performance Settings")]
    public int tagCapacity = 4;
    public int valueCapacity = 8;
    public int behaviourCapacity = 4;
}
```

### Prefab-Based Entity System

```csharp
public class PrefabEntityManager : MonoBehaviour
{
    [Header("Entity Prefabs")]
    [SerializeField] private SceneEntity playerPrefab;
    [SerializeField] private EnemyEntity[] enemyPrefabs;
    [SerializeField] private SceneEntity npcPrefab;
    
    [Header("Spawn Configuration")]
    [SerializeField] private Transform[] spawnPoints;
    [SerializeField] private int maxEntitiesPerType = 10;
    
    private List<SceneEntity> activeEntities = new List<SceneEntity>();
    
    void Start()
    {
        CreateInitialEntities();
    }
    
    private void CreateInitialEntities()
    {
        // Create player at first spawn point
        if (playerPrefab != null && spawnPoints.Length > 0)
        {
            var player = SceneEntity.Create(playerPrefab, spawnPoints[0]);
            activeEntities.Add(player);
        }
        
        // Create enemies at random spawn points
        for (int i = 0; i < Mathf.Min(maxEntitiesPerType, enemyPrefabs.Length); i++)
        {
            var enemyPrefab = enemyPrefabs[Random.Range(0, enemyPrefabs.Length)];
            var spawnPoint = spawnPoints[Random.Range(0, spawnPoints.Length)];
            
            var enemy = SceneEntity.Create(enemyPrefab, spawnPoint);
            activeEntities.Add(enemy);
        }
    }
    
    public T CreateEntityAtPosition<T>(T prefab, Vector3 position) where T : SceneEntity
    {
        var entity = SceneEntity.Create(prefab, position, Quaternion.identity);
        activeEntities.Add(entity);
        
        Debug.Log($"Created {typeof(T).Name} at {position}");
        return entity;
    }
    
    public T CreateEntityAtSpawnPoint<T>(T prefab, int spawnIndex = -1) where T : SceneEntity
    {
        if (spawnPoints.Length == 0)
        {
            Debug.LogWarning("No spawn points defined!");
            return null;
        }
        
        if (spawnIndex < 0)
            spawnIndex = Random.Range(0, spawnPoints.Length);
        
        spawnIndex = Mathf.Clamp(spawnIndex, 0, spawnPoints.Length - 1);
        var spawnPoint = spawnPoints[spawnIndex];
        
        return CreateEntityAtPosition(prefab, spawnPoint.position);
    }
    
    public void DestroyEntity(SceneEntity entity, float delay = 0f)
    {
        if (entity != null && activeEntities.Contains(entity))
        {
            activeEntities.Remove(entity);
            SceneEntity.Destroy(entity, delay);
        }
    }
    
    public void DestroyAllEntities()
    {
        foreach (var entity in activeEntities.ToArray())
        {
            SceneEntity.Destroy(entity);
        }
        activeEntities.Clear();
    }
}
```

### Type-Safe Entity Casting

```csharp
public class EntityCastingExample : MonoBehaviour
{
    [SerializeField] private List<GameObject> entityGameObjects;
    
    void Start()
    {
        ProcessEntityGameObjects();
    }
    
    private void ProcessEntityGameObjects()
    {
        foreach (var gameObject in entityGameObjects)
        {
            // Get entity interface
            var entity = gameObject.GetComponent<IEntity>();
            if (entity == null) continue;
            
            // Try casting to SceneEntity
            if (SceneEntity.TryCast(entity, out SceneEntity sceneEntity))
            {
                Debug.Log($"Found SceneEntity: {sceneEntity.Name}");
                ProcessSceneEntity(sceneEntity);
            }
            
            // Try casting to specific entity types
            if (SceneEntity.TryCast<PlayerEntity>(entity, out PlayerEntity player))
            {
                Debug.Log($"Found PlayerEntity: {player.Name}");
                ProcessPlayerEntity(player);
            }
            
            if (SceneEntity.TryCast<EnemyEntity>(entity, out EnemyEntity enemy))
            {
                Debug.Log($"Found EnemyEntity: {enemy.Name}");
                ProcessEnemyEntity(enemy);
            }
        }
    }
    
    private void ProcessSceneEntity(SceneEntity entity)
    {
        // Generic SceneEntity processing
        if (!entity.Installed)
            entity.Install();
            
        if (!entity.Spawned)
            entity.Spawn();
            
        if (!entity.Enabled)
            entity.Activate();
    }
    
    private void ProcessPlayerEntity(PlayerEntity player)
    {
        // Player-specific processing
        player.SetValue(EntityNames.PLAYER_ID, 1);
        player.AddTag(EntityTags.MAIN_PLAYER);
    }
    
    private void ProcessEnemyEntity(EnemyEntity enemy)
    {
        // Enemy-specific processing
        enemy.SetValue(EntityNames.DIFFICULTY_LEVEL, Random.Range(1, 5));
        enemy.AddBehaviour(new EnemyAIBehaviour());
    }
}
```

### Batch Entity Management

```csharp
public class SceneEntityManager : MonoBehaviour
{
    [Header("Scene Management")]
    [SerializeField] private bool installAllOnStart = true;
    [SerializeField] private bool logInstallationProgress = true;
    
    void Start()
    {
        if (installAllOnStart)
        {
            InstallAllEntitiesInScene();
        }
    }
    
    private void InstallAllEntitiesInScene()
    {
        var currentScene = SceneManager.GetActiveScene();
        
        if (logInstallationProgress)
        {
            Debug.Log($"Installing all entities in scene: {currentScene.name}");
        }
        
        // Install all base SceneEntities
        var beforeCount = GetEntityCount<SceneEntity>(currentScene);
        SceneEntity.InstallAll(currentScene);
        var afterCount = GetEntityCount<SceneEntity>(currentScene);
        
        if (logInstallationProgress)
        {
            Debug.Log($"Installed {afterCount - beforeCount} SceneEntities");
        }
        
        // Install specific entity types if needed
        InstallSpecificEntityTypes(currentScene);
    }
    
    private void InstallSpecificEntityTypes(Scene scene)
    {
        // Install PlayerEntities
        var playersBefore = GetEntityCount<PlayerEntity>(scene);
        SceneEntity.InstallAll<PlayerEntity>(scene);
        var playersAfter = GetEntityCount<PlayerEntity>(scene);
        
        // Install EnemyEntities
        var enemiesBefore = GetEntityCount<EnemyEntity>(scene);
        SceneEntity.InstallAll<EnemyEntity>(scene);
        var enemiesAfter = GetEntityCount<EnemyEntity>(scene);
        
        if (logInstallationProgress)
        {
            Debug.Log($"Installed {playersAfter - playersBefore} PlayerEntities");
            Debug.Log($"Installed {enemiesAfter - enemiesBefore} EnemyEntities");
        }
    }
    
    private int GetEntityCount<T>(Scene scene) where T : SceneEntity
    {
        int count = 0;
        var rootGameObjects = scene.GetRootGameObjects();
        
        foreach (var gameObject in rootGameObjects)
        {
            var entities = gameObject.GetComponentsInChildren<T>();
            foreach (var entity in entities)
            {
                if (entity.Installed)
                    count++;
            }
        }
        
        return count;
    }
    
    [ContextMenu("Install All Entities")]
    public void InstallAllEntitiesManual()
    {
        InstallAllEntitiesInScene();
    }
    
    [ContextMenu("Count All Entities")]
    public void CountAllEntities()
    {
        var scene = SceneManager.GetActiveScene();
        var sceneEntityCount = GetEntityCount<SceneEntity>(scene);
        var playerEntityCount = GetEntityCount<PlayerEntity>(scene);
        var enemyEntityCount = GetEntityCount<EnemyEntity>(scene);
        
        Debug.Log($"Entity counts - Total: {sceneEntityCount}, Players: {playerEntityCount}, Enemies: {enemyEntityCount}");
    }
}
```

### Performance-Optimized Creation

```csharp
public class HighPerformanceEntityFactory : MonoBehaviour
{
    [Header("Performance Settings")]
    [SerializeField] private int batchSize = 100;
    [SerializeField] private int entitiesPerFrame = 10;
    [SerializeField] private bool useObjectPooling = true;
    
    private Queue<SceneEntity> entityPool = new Queue<SceneEntity>();
    private List<SceneEntity> activeEntities = new List<SceneEntity>();
    
    void Start()
    {
        if (useObjectPooling)
        {
            PrewarmEntityPool();
        }
    }
    
    private void PrewarmEntityPool()
    {
        StartCoroutine(PrewarmCoroutine());
    }
    
    private System.Collections.IEnumerator PrewarmCoroutine()
    {
        for (int i = 0; i < batchSize; i += entitiesPerFrame)
        {
            int createCount = Mathf.Min(entitiesPerFrame, batchSize - i);
            
            for (int j = 0; j < createCount; j++)
            {
                var entity = CreatePooledEntity();
                entity.gameObject.SetActive(false);
                entityPool.Enqueue(entity);
            }
            
            yield return null; // Spread creation across frames
        }
        
        Debug.Log($"Prewarmed entity pool with {entityPool.Count} entities");
    }
    
    private SceneEntity CreatePooledEntity()
    {
        var args = new SceneEntity.CreateArgs
        {
            name = "Pooled Entity",
            initialTagCapacity = 4,
            initialValueCapacity = 8,
            initialBehaviourCapacity = 4,
            installOnAwake = false,
            disposeValues = false, // Don't dispose pooled entities
            useUnityLifecycle = true
        };
        
        return SceneEntity.Create(args);
    }
    
    public SceneEntity GetFromPool(Vector3 position)
    {
        SceneEntity entity;
        
        if (entityPool.Count > 0)
        {
            entity = entityPool.Dequeue();
            entity.transform.position = position;
            entity.gameObject.SetActive(true);
        }
        else
        {
            // Pool exhausted, create new entity
            entity = CreatePooledEntity();
            entity.transform.position = position;
        }
        
        // Reset entity state
        entity.ClearTags();
        entity.ClearValues();
        entity.ClearBehaviours();
        
        activeEntities.Add(entity);
        return entity;
    }
    
    public void ReturnToPool(SceneEntity entity)
    {
        if (entity == null || !activeEntities.Contains(entity))
            return;
        
        activeEntities.Remove(entity);
        
        // Reset entity for pooling
        entity.Deactivate();
        entity.Despawn();
        entity.ClearTags();
        entity.ClearValues();
        entity.ClearBehaviours();
        
        entity.gameObject.SetActive(false);
        entity.transform.position = Vector3.zero;
        entity.transform.rotation = Quaternion.identity;
        entity.transform.SetParent(this.transform);
        
        entityPool.Enqueue(entity);
    }
    
    public void ReturnAllToPool()
    {
        foreach (var entity in activeEntities.ToArray())
        {
            ReturnToPool(entity);
        }
    }
}
```

## Factory Patterns

### Builder Pattern Integration
```csharp
public class EntityBuilder
{
    private SceneEntity.CreateArgs args = new SceneEntity.CreateArgs
    {
        installOnAwake = true,
        disposeValues = true,
        useUnityLifecycle = true,
        initialTagCapacity = 1,
        initialValueCapacity = 1,
        initialBehaviourCapacity = 1
    };
    
    public EntityBuilder WithName(string name)
    {
        args.name = name;
        return this;
    }
    
    public EntityBuilder WithTags(params int[] tags)
    {
        args.tags = tags;
        args.initialTagCapacity = Mathf.Max(1, tags.Length);
        return this;
    }
    
    public EntityBuilder WithValues(Dictionary<int, object> values)
    {
        args.values = values;
        args.initialValueCapacity = Mathf.Max(1, values.Count);
        return this;
    }
    
    public EntityBuilder WithBehaviours(params IEntityBehaviour[] behaviours)
    {
        args.behaviours = behaviours;
        args.initialBehaviourCapacity = Mathf.Max(1, behaviours.Length);
        return this;
    }
    
    public T Build<T>() where T : SceneEntity
    {
        return SceneEntity.Create<T>(args);
    }
    
    public SceneEntity Build()
    {
        return SceneEntity.Create(args);
    }
}
```

## Best Practices

1. **Performance Optimization** – Use appropriate initial capacities to minimize allocations
2. **Type Safety** – Use TryCast for optional casting operations
3. **Resource Management** – Always dispose entities properly with Destroy methods
4. **Batch Processing** – Use InstallAll for scene-wide operations
5. **Pooling** – Consider object pooling for frequently created/destroyed entities
6. **Error Handling** – Use safe casting methods to prevent invalid cast exceptions

## Performance Considerations

### Memory Allocation
- Set appropriate initial capacities to minimize array reallocations
- Consider pooling for frequently created entities
- Use batch operations for multiple entities

### Creation Patterns
- Struct-based CreateArgs minimizes parameter passing overhead
- Aggressive inlining improves call site performance
- Factory methods reduce boilerplate code

### Type Casting
- Pattern matching provides efficient type discrimination
- TryCast methods avoid exception handling overhead
- Support for proxy entities enables flexible architectures

## Notes

- All factory methods are aggressively inlined for performance
- Supports both immediate and deferred installation patterns
- Type-safe casting handles both direct entities and proxy wrappers
- Batch operations process entire scenes efficiently
- Prefab instantiation automatically triggers installation
- Comprehensive parameter validation prevents common errors