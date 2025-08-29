# 🧩 SceneEntityBaker

A Unity MonoBehaviour component that converts GameObject configurations into runtime entities through a "baking" process. Enables seamless transition from Unity's GameObject-based scene design to Atomic's entity-based runtime systems.

## Overview

`SceneEntityBaker` provides a bridge between Unity's visual scene editing and Atomic's entity system. It allows developers to configure entities visually in Unity scenes and then "bake" them into runtime entities, optionally destroying the original GameObjects to maintain clean separation between design-time and runtime representations.

## Class Definition

```csharp
#if UNITY_5_3_OR_NEWER
namespace Atomic.Entities
{
    /// <summary>
    /// A non-generic alias for SceneEntityBaker{IEntity}.
    /// </summary>
    public abstract class SceneEntityBaker : SceneEntityBaker<IEntity>
    {
    }

    public abstract class SceneEntityBaker<E> : MonoBehaviour where E : IEntity
    {
        [SerializeField] protected internal bool _destroyAfterBake = true;
        [SerializeField] protected internal ScriptableEntityFactory<E> _factory;

        public E Bake();
        protected abstract void Install(E entity);

        // Static batch baking methods
        public static E[] BakeAll(bool includeInactive = true);
        public static void BakeAll(ICollection<E> destination, bool includeInactive = true);
        public static List<E> Bake(Scene scene, bool includeInactive = true);
        public static E[] Bake(GameObject gameObject, bool includeInactive = true);
    }
}
#endif
```

## Key Features

### GameObject to Entity Conversion
- Seamless conversion from visual GameObject setups to runtime entities
- Preserves Unity's visual editing workflow while enabling entity-based systems
- Optional GameObject destruction after baking for clean runtime state

### Factory Integration
- Uses ScriptableEntityFactory for consistent entity creation
- Leverages existing factory systems for entity instantiation
- Configurable factory assignment per baker

### Batch Processing
- Static methods for baking multiple entities at once
- Scene-level baking for level initialization
- GameObject hierarchy baking for prefab-like workflows

### Unity Editor Integration
- Inspector-based configuration with tooltips and validation
- Odin Inspector support for enhanced editor experience
- Design-time vs runtime separation

## Usage Examples

### Basic Entity Baker

```csharp
using UnityEngine;
using Atomic.Entities;

public class PlayerEntityBaker : SceneEntityBaker<IEntity>
{
    [Header("Player Configuration")]
    [SerializeField] private float _health = 100f;
    [SerializeField] private float _movementSpeed = 5f;
    [SerializeField] private string[] _playerTags = { "Player", "Controllable", "Active" };
    
    [Header("Equipment")]
    [SerializeField] private GameObject _weaponPrefab;
    [SerializeField] private GameObject _armorPrefab;

    protected override void Install(IEntity entity)
    {
        // Install basic player stats
        entity.Set("Health", _health);
        entity.Set("MaxHealth", _health);
        entity.Set("MovementSpeed", _movementSpeed);
        entity.Set("Experience", 0);
        entity.Set("Level", 1);

        // Install position from GameObject transform
        entity.Set("Position", transform.position);
        entity.Set("Rotation", transform.rotation);
        entity.Set("Scale", transform.localScale);

        // Install player tags
        foreach (string tag in _playerTags)
        {
            entity.AddTag(tag);
        }

        // Install equipment references
        if (_weaponPrefab != null)
        {
            entity.Set("WeaponPrefab", _weaponPrefab);
        }
        
        if (_armorPrefab != null)
        {
            entity.Set("ArmorPrefab", _armorPrefab);
        }

        // Install player-specific data
        entity.Set("InputEnabled", true);
        entity.Set("CameraTarget", true);
        
        Debug.Log($"Baked player entity: {entity.Name} at {transform.position}");
    }
}
```

### Complex Enemy Baker

```csharp
using UnityEngine;
using Atomic.Entities;

public class EnemyEntityBaker : SceneEntityBaker<IEntity>
{
    [System.Serializable]
    public struct EnemyStats
    {
        public float health;
        public float damage;
        public float movementSpeed;
        public float attackRange;
        public float detectionRange;
        public float patrolRadius;
    }

    [Header("Enemy Configuration")]
    [SerializeField] private EnemyType _enemyType = EnemyType.Grunt;
    [SerializeField] private EnemyStats _stats;
    
    [Header("AI Behavior")]
    [SerializeField] private Transform[] _patrolPoints;
    [SerializeField] private bool _startPatrolling = true;
    [SerializeField] private AIState _initialState = AIState.Patrol;
    
    [Header("Loot Configuration")]
    [SerializeField] private LootTableScriptableObject _lootTable;
    [SerializeField] private int _experienceReward = 50;

    private enum EnemyType { Grunt, Elite, Boss }
    private enum AIState { Idle, Patrol, Chase, Attack, Return }

    protected override void Install(IEntity entity)
    {
        // Install enemy stats
        entity.Set("Health", _stats.health);
        entity.Set("MaxHealth", _stats.health);
        entity.Set("Damage", _stats.damage);
        entity.Set("MovementSpeed", _stats.movementSpeed);
        entity.Set("AttackRange", _stats.attackRange);
        entity.Set("DetectionRange", _stats.detectionRange);
        entity.Set("PatrolRadius", _stats.patrolRadius);

        // Install AI configuration
        entity.Set("AIState", _initialState.ToString());
        entity.Set("StartPatrolling", _startPatrolling);
        entity.Set("HomePosition", transform.position);

        // Install patrol points
        if (_patrolPoints != null && _patrolPoints.Length > 0)
        {
            var patrolPositions = new Vector3[_patrolPoints.Length];
            for (int i = 0; i < _patrolPoints.Length; i++)
            {
                patrolPositions[i] = _patrolPoints[i].position;
            }
            entity.Set("PatrolPoints", patrolPositions);
            entity.Set("CurrentPatrolIndex", 0);
        }

        // Install loot and rewards
        if (_lootTable != null)
        {
            entity.Set("LootTable", _lootTable);
        }
        entity.Set("ExperienceReward", _experienceReward);

        // Install enemy type-specific configuration
        InstallEnemyTypeConfiguration(entity);

        // Install transform data
        entity.Set("Position", transform.position);
        entity.Set("Rotation", transform.rotation);
        entity.Set("Forward", transform.forward);

        Debug.Log($"Baked {_enemyType} enemy: {entity.Name} with {_stats.health} health");
    }

    private void InstallEnemyTypeConfiguration(IEntity entity)
    {
        entity.AddTag("Enemy");
        entity.AddTag("Hostile");
        entity.AddTag($"Enemy_{_enemyType}");

        switch (_enemyType)
        {
            case EnemyType.Grunt:
                entity.AddTag("Grunt");
                entity.Set("Threat", 1);
                break;
                
            case EnemyType.Elite:
                entity.AddTag("Elite");
                entity.Set("Threat", 3);
                entity.Set("EliteAbility", "PowerAttack");
                break;
                
            case EnemyType.Boss:
                entity.AddTag("Boss");
                entity.AddTag("MiniBoss");
                entity.Set("Threat", 5);
                entity.Set("BossPhases", 3);
                entity.Set("CurrentPhase", 1);
                break;
        }
    }
}
```

### Level Initialization System

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using Atomic.Entities;

public class LevelInitializer : MonoBehaviour
{
    [Header("Baking Configuration")]
    [SerializeField] private bool _bakeOnStart = true;
    [SerializeField] private bool _includeInactive = false;
    [SerializeField] private bool _logBakeResults = true;
    
    private EntityCollection<IEntity> _bakedEntities = new();

    void Start()
    {
        if (_bakeOnStart)
        {
            BakeAllEntitiesInLevel();
        }
    }

    [ContextMenu("Bake All Entities")]
    public void BakeAllEntitiesInLevel()
    {
        Debug.Log("Starting entity baking process...");
        
        // Clear previous baked entities
        _bakedEntities.Clear();

        // Bake all entity bakers in the scene
        SceneEntityBaker.BakeAll(_bakedEntities, _includeInactive);

        if (_logBakeResults)
        {
            LogBakeResults();
        }

        // Initialize game systems with baked entities
        InitializeGameSystems();
        
        Debug.Log($"Entity baking completed. Total entities: {_bakedEntities.Count}");
    }

    public void BakeSpecificScene(Scene scene)
    {
        var sceneEntities = SceneEntityBaker.Bake(scene, _includeInactive);
        
        foreach (var entity in sceneEntities)
        {
            _bakedEntities.Add(entity);
        }
        
        Debug.Log($"Baked {sceneEntities.Count} entities from scene: {scene.name}");
    }

    public void BakeGameObjectHierarchy(GameObject rootObject)
    {
        var hierarchyEntities = SceneEntityBaker.Bake(rootObject, _includeInactive);
        
        foreach (var entity in hierarchyEntities)
        {
            _bakedEntities.Add(entity);
        }
        
        Debug.Log($"Baked {hierarchyEntities.Length} entities from hierarchy: {rootObject.name}");
    }

    private void LogBakeResults()
    {
        var entityTypeCount = new Dictionary<string, int>();
        var tagStats = new Dictionary<string, int>();

        foreach (var entity in _bakedEntities)
        {
            // Count entity types (simplified - would need actual type checking)
            string entityType = entity.HasTag("Player") ? "Player" :
                               entity.HasTag("Enemy") ? "Enemy" :
                               entity.HasTag("Collectible") ? "Collectible" : "Other";
            
            entityTypeCount[entityType] = entityTypeCount.GetValueOrDefault(entityType, 0) + 1;

            // Count common tags
            if (entity.HasTag("Active")) tagStats["Active"] = tagStats.GetValueOrDefault("Active", 0) + 1;
            if (entity.HasTag("Hostile")) tagStats["Hostile"] = tagStats.GetValueOrDefault("Hostile", 0) + 1;
            if (entity.HasTag("Collectible")) tagStats["Collectible"] = tagStats.GetValueOrDefault("Collectible", 0) + 1;
        }

        Debug.Log("=== Baking Results ===");
        foreach (var kvp in entityTypeCount)
        {
            Debug.Log($"{kvp.Key}: {kvp.Value} entities");
        }
        
        Debug.Log("=== Tag Statistics ===");
        foreach (var kvp in tagStats)
        {
            Debug.Log($"{kvp.Key}: {kvp.Value} entities");
        }
    }

    private void InitializeGameSystems()
    {
        // Initialize various game systems with baked entities
        var enemyManager = FindObjectOfType<EnemyManager>();
        var playerManager = FindObjectOfType<PlayerManager>();
        var collectibleManager = FindObjectOfType<CollectibleManager>();

        foreach (var entity in _bakedEntities)
        {
            if (entity.HasTag("Player") && playerManager != null)
            {
                playerManager.RegisterPlayer(entity);
            }
            else if (entity.HasTag("Enemy") && enemyManager != null)
            {
                enemyManager.RegisterEnemy(entity);
            }
            else if (entity.HasTag("Collectible") && collectibleManager != null)
            {
                collectibleManager.RegisterCollectible(entity);
            }
        }
    }

    public int GetBakedEntityCount() => _bakedEntities.Count;
    
    public IReadOnlyCollection<IEntity> GetBakedEntities()
    {
        var entities = new List<IEntity>();
        _bakedEntities.CopyTo(entities);
        return entities;
    }

    void OnDestroy()
    {
        _bakedEntities.Dispose();
    }
}
```

### Runtime Baker for Dynamic Content

```csharp
using UnityEngine;
using Atomic.Entities;
using System.Collections;

public class RuntimeEntityBaker : MonoBehaviour
{
    [Header("Runtime Baking")]
    [SerializeField] private bool _bakeOnTriggerEnter = true;
    [SerializeField] private LayerMask _triggerLayers = -1;
    [SerializeField] private float _bakeDelay = 0f;

    private EntityCollection<IEntity> _runtimeBakedEntities = new();

    void Start()
    {
        _runtimeBakedEntities.OnAdded += OnEntityRuntimeBaked;
        _runtimeBakedEntities.OnRemoved += OnEntityRuntimeRemoved;
    }

    void OnTriggerEnter(Collider other)
    {
        if (!_bakeOnTriggerEnter)
            return;

        if (((1 << other.gameObject.layer) & _triggerLayers) == 0)
            return;

        var bakers = other.GetComponentsInChildren<SceneEntityBaker>();
        if (bakers.Length > 0)
        {
            StartCoroutine(BakeWithDelay(bakers));
        }
    }

    private IEnumerator BakeWithDelay(SceneEntityBaker[] bakers)
    {
        if (_bakeDelay > 0)
        {
            yield return new WaitForSeconds(_bakeDelay);
        }

        foreach (var baker in bakers)
        {
            if (baker != null)
            {
                var entity = baker.Bake();
                _runtimeBakedEntities.Add(entity);
            }
        }
    }

    [ContextMenu("Bake All In Trigger")]
    public void BakeAllInTrigger()
    {
        var collidersInTrigger = Physics.OverlapBox(
            transform.position,
            transform.localScale / 2,
            transform.rotation
        );

        foreach (var collider in collidersInTrigger)
        {
            var bakers = collider.GetComponentsInChildren<SceneEntityBaker>();
            foreach (var baker in bakers)
            {
                if (baker != null)
                {
                    var entity = baker.Bake();
                    _runtimeBakedEntities.Add(entity);
                }
            }
        }
    }

    public void BakeSpecificPrefab(GameObject prefab)
    {
        var instance = Instantiate(prefab);
        var bakers = instance.GetComponentsInChildren<SceneEntityBaker>();
        
        foreach (var baker in bakers)
        {
            var entity = baker.Bake();
            _runtimeBakedEntities.Add(entity);
        }
        
        // Clean up the temporary instance
        Destroy(instance);
    }

    private void OnEntityRuntimeBaked(IEntity entity)
    {
        Debug.Log($"Runtime baked entity: {entity.Name}");
        entity.AddTag("RuntimeBaked");
        entity.Set("BakeTime", Time.time);
        
        GameEvents.OnEntityRuntimeBaked?.Invoke(entity);
    }

    private void OnEntityRuntimeRemoved(IEntity entity)
    {
        Debug.Log($"Runtime baked entity removed: {entity.Name}");
        GameEvents.OnRuntimeBakedEntityRemoved?.Invoke(entity);
    }

    public int GetRuntimeBakedCount() => _runtimeBakedEntities.Count;

    void OnDestroy()
    {
        _runtimeBakedEntities.Dispose();
    }
}
```

## Integration with Atomic Framework

### Factory-Based Baking System

```csharp
using UnityEngine;
using Atomic.Entities;

public class FactoryBasedBaker : SceneEntityBaker<IEntity>
{
    [Header("Factory Configuration")]
    [SerializeField] private ScriptableEntityFactory<IEntity> _primaryFactory;
    [SerializeField] private ScriptableEntityFactory<IEntity> _secondaryFactory;
    [SerializeField] private bool _useSecondaryFactory = false;
    
    [Header("Conditional Baking")]
    [SerializeField] private bool _bakingEnabled = true;
    [SerializeField] private string[] _requiredSceneTags;

    protected override void Install(IEntity entity)
    {
        if (!_bakingEnabled || !ShouldBakeInCurrentScene())
        {
            Debug.Log($"Skipping baking for {gameObject.name}");
            return;
        }

        // Use appropriate factory
        var factory = _useSecondaryFactory && _secondaryFactory != null 
            ? _secondaryFactory 
            : _factory;

        if (factory == null)
        {
            Debug.LogWarning($"No factory assigned for baker on {gameObject.name}");
            return;
        }

        // Install factory-specific configuration
        InstallFactoryConfiguration(entity, factory);
        
        // Install GameObject-specific overrides
        InstallGameObjectConfiguration(entity);
        
        Debug.Log($"Baked entity using factory: {factory.name}");
    }

    private bool ShouldBakeInCurrentScene()
    {
        if (_requiredSceneTags == null || _requiredSceneTags.Length == 0)
            return true;

        // Check if current scene has required tags (implementation would depend on scene tagging system)
        string currentScene = UnityEngine.SceneManagement.SceneManager.GetActiveScene().name;
        
        // Simplified scene tag checking
        foreach (string requiredTag in _requiredSceneTags)
        {
            if (currentScene.Contains(requiredTag))
                return true;
        }
        
        return false;
    }

    private void InstallFactoryConfiguration(IEntity entity, ScriptableEntityFactory<IEntity> factory)
    {
        // Let factory configure the entity first
        // This would require access to factory's configuration logic
        // Simplified example:
        entity.AddTag("FactoryCreated");
        entity.Set("CreatedByFactory", factory.name);
    }

    private void InstallGameObjectConfiguration(IEntity entity)
    {
        // Override or supplement factory configuration with GameObject-specific settings
        entity.Set("OriginalGameObjectName", gameObject.name);
        entity.Set("ScenePosition", transform.position);
        entity.Set("SceneRotation", transform.rotation);
        
        // Apply any component-based configuration
        var renderers = GetComponentsInChildren<Renderer>();
        if (renderers.Length > 0)
        {
            entity.Set("VisualComponents", renderers.Length);
            entity.AddTag("HasVisuals");
        }
        
        var colliders = GetComponentsInChildren<Collider>();
        if (colliders.Length > 0)
        {
            entity.Set("PhysicsComponents", colliders.Length);
            entity.AddTag("HasPhysics");
        }
    }
}
```

## Implementation Notes

### Unity Version Compatibility
- Uses conditional compilation for Unity 5.3 or newer
- Supports both legacy and modern FindObjects APIs
- Handles Unity 2023.1+ changes automatically

### Performance Characteristics
- Batch baking operations for efficiency
- Optional GameObject destruction to reduce memory usage
- Aggressive inlining for critical path operations

### Editor Integration
- Inspector-based configuration with tooltips
- Odin Inspector enhancements when available
- Design-time validation and feedback

## Best Practices

### Baking Strategy
- Use batch baking methods for level initialization
- Consider performance impact of large-scale baking
- Implement conditional baking for different contexts

### Factory Integration
- Always assign appropriate factories to bakers
- Use factory-specific configuration where possible
- Override factory settings only when necessary

### GameObject Management
- Set _destroyAfterBake appropriately based on use case
- Consider GameObject lifecycle in relation to entity lifecycle
- Clean up Unity-specific references after baking

## Common Patterns

### Conditional Baking Pattern

```csharp
public class ConditionalBaker : SceneEntityBaker<IEntity>
{
    [SerializeField] private bool _bakeInPlayMode = true;
    [SerializeField] private bool _bakeInEditor = false;
    
    protected override void Install(IEntity entity)
    {
        if (Application.isPlaying && !_bakeInPlayMode)
            return;
            
        if (!Application.isPlaying && !_bakeInEditor)
            return;
            
        // Perform actual baking
        DoBaking(entity);
    }
}
```

### Multi-Stage Baking Pattern

```csharp
public class MultiStageBaker : SceneEntityBaker<IEntity>
{
    [SerializeField] private BakingStage[] _stages;
    
    protected override void Install(IEntity entity)
    {
        foreach (var stage in _stages)
        {
            if (stage.enabled)
            {
                stage.ApplyTo(entity);
            }
        }
    }
}
```

The `SceneEntityBaker` provides essential functionality for converting Unity's GameObject-based scene design into Atomic's entity-based runtime systems, enabling seamless integration between visual design tools and performant entity architectures within the Atomic framework.