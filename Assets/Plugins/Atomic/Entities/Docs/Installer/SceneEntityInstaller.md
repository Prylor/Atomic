# 🧩 SceneEntityInstaller

A Unity MonoBehaviour-based installer that provides declarative entity configuration directly in Unity scenes. Enables visual configuration of entities through the Unity Inspector with automatic refresh capabilities in the editor.

## Overview

`SceneEntityInstaller` allows developers to configure entities directly in Unity scenes using MonoBehaviour components. Perfect for level designers and developers who need visual, inspector-based entity configuration with immediate feedback in the Unity Editor.

## Class Definition

```csharp
#if UNITY_5_3_OR_NEWER
namespace Atomic.Entities
{
    /// <summary>
    /// A Unity MonoBehaviour that can be attached to a GameObject
    /// to perform installation logic on an IEntity during runtime or initialization.
    /// </summary>
    public abstract class SceneEntityInstaller : MonoBehaviour, IEntityInstaller
    {
#if UNITY_EDITOR
        /// <summary>
        /// Editor-only callback used to signal the need to refresh editor state when the component is modified.
        /// </summary>
        internal Action refreshCallback;
#endif

        /// <summary>
        /// Installs data or behavior into the specified entity.
        /// </summary>
        /// <param name="entity">The entity to install configuration or components into.</param>
        public abstract void Install(IEntity entity);

        /// <summary>
        /// Called by Unity when the component is modified in the Inspector.
        /// Triggers the refreshCallback in Editor to update related systems or previews.
        /// </summary>
        protected virtual void OnValidate()
        {
#if UNITY_EDITOR
            try
            {
                if (!EditorApplication.isPlaying && !EditorApplication.isCompiling)
                    refreshCallback?.Invoke();
            }
            catch (Exception)
            {
                // ignored
            }
#endif
        }
    }

    /// <summary>
    /// A strongly-typed version of SceneEntityInstaller for entities of type T.
    /// </summary>
    /// <typeparam name="T">The specific type of IEntity this installer operates on.</typeparam>
    public abstract class SceneEntityInstaller<T> : SceneEntityInstaller where T : class, IEntity
    {
        /// <inheritdoc/>
        public sealed override void Install(IEntity entity) => this.Install((T) entity);

        /// <summary>
        /// Installs data or behavior into a strongly-typed entity.
        /// </summary>
        /// <param name="entity">The entity to install.</param>
        protected abstract void Install(T entity);
    }
}
#endif
```

## Key Features

### Unity Integration
- Native MonoBehaviour integration for scene-based configuration
- Visual configuration through Unity Inspector
- Automatic editor refresh when component values change

### Type Safety
- Generic version eliminates casting in derived classes
- Compile-time type checking for entity-specific installers
- Seamless integration with Unity's component system

### Editor Support
- OnValidate callback for immediate visual feedback
- Editor-only refresh callbacks for custom tooling
- Non-playing mode safety checks

## Usage Examples

### Basic Scene Stats Installer

```csharp
using UnityEngine;
using Atomic.Entities;

[System.Serializable]
public class EntityStats
{
    public float health = 100f;
    public float mana = 50f;
    public float stamina = 75f;
    public float movementSpeed = 5f;
    public int level = 1;
}

public class SceneStatsInstaller : SceneEntityInstaller
{
    [Header("Entity Stats")]
    [SerializeField] private EntityStats _stats = new EntityStats();
    
    [Header("Tags")]
    [SerializeField] private string[] _tags = { "Active", "Alive" };
    
    [Header("Advanced Settings")]
    [SerializeField] private bool _randomizeHealth = false;
    [SerializeField] private Vector2 _healthRange = new Vector2(80f, 120f);

    public override void Install(IEntity entity)
    {
        // Install basic stats
        float finalHealth = _randomizeHealth 
            ? Random.Range(_healthRange.x, _healthRange.y) 
            : _stats.health;
            
        entity.Set("Health", finalHealth);
        entity.Set("MaxHealth", finalHealth);
        entity.Set("Mana", _stats.mana);
        entity.Set("MaxMana", _stats.mana);
        entity.Set("Stamina", _stats.stamina);
        entity.Set("MaxStamina", _stats.stamina);
        entity.Set("MovementSpeed", _stats.movementSpeed);
        entity.Set("Level", _stats.level);

        // Install tags
        foreach (string tag in _tags)
        {
            if (!string.IsNullOrEmpty(tag))
            {
                entity.AddTag(tag.Trim());
            }
        }

        // Install position from GameObject transform
        entity.Set("Position", transform.position);
        entity.Set("Rotation", transform.rotation);
        entity.Set("Scale", transform.localScale);

        Debug.Log($"Scene stats installed for entity: {entity.Name}");
    }
}
```

### Equipment Scene Installer

```csharp
using UnityEngine;
using Atomic.Entities;
using System.Collections.Generic;

[System.Serializable]
public class EquipmentSlot
{
    public string slotName;
    public GameObject itemPrefab;
    public int itemId;
    public string itemName;
    
    [Header("Item Stats")]
    public int attackBonus;
    public int defenseBonus;
    public int durability = 100;
}

public class SceneEquipmentInstaller : SceneEntityInstaller
{
    [Header("Equipment Configuration")]
    [SerializeField] private EquipmentSlot[] _equipment;
    
    [Header("Visual Settings")]
    [SerializeField] private bool _hideEquipmentInScene = false;
    [SerializeField] private Transform _equipmentParent;

    public override void Install(IEntity entity)
    {
        var installedEquipment = new List<string>();

        foreach (var slot in _equipment)
        {
            if (slot.itemPrefab != null || slot.itemId > 0)
            {
                InstallEquipmentSlot(entity, slot);
                installedEquipment.Add(slot.slotName);
            }
        }

        // Update equipment tags
        UpdateEquipmentTags(entity, installedEquipment);
        
        // Handle visual equipment
        if (_hideEquipmentInScene && _equipmentParent != null)
        {
            _equipmentParent.gameObject.SetActive(false);
        }

        Debug.Log($"Equipment installed for entity {entity.Name}: {string.Join(", ", installedEquipment)}");
    }

    private void InstallEquipmentSlot(IEntity entity, EquipmentSlot slot)
    {
        // Create equipment data
        var equipmentData = new Dictionary<string, object>
        {
            ["Name"] = slot.itemName,
            ["ItemId"] = slot.itemId,
            ["AttackBonus"] = slot.attackBonus,
            ["DefenseBonus"] = slot.defenseBonus,
            ["Durability"] = slot.durability,
            ["MaxDurability"] = slot.durability
        };

        // Install equipment to entity
        entity.Set(slot.slotName, equipmentData);

        // Apply stat bonuses
        ApplyEquipmentBonuses(entity, slot);
    }

    private void ApplyEquipmentBonuses(IEntity entity, EquipmentSlot slot)
    {
        if (slot.attackBonus != 0)
        {
            float currentAttack = entity.HasValue("Attack") ? entity.Get<float>("Attack") : 0f;
            entity.Set("Attack", currentAttack + slot.attackBonus);
        }

        if (slot.defenseBonus != 0)
        {
            float currentDefense = entity.HasValue("Defense") ? entity.Get<float>("Defense") : 0f;
            entity.Set("Defense", currentDefense + slot.defenseBonus);
        }
    }

    private void UpdateEquipmentTags(IEntity entity, List<string> equippedSlots)
    {
        entity.SetTagState("Armed", equippedSlots.Contains("MainHand"));
        entity.SetTagState("Armored", equippedSlots.Contains("Chest"));
        entity.SetTagState("FullyEquipped", equippedSlots.Count >= 4);
    }
}
```

### AI Configuration Scene Installer

```csharp
using UnityEngine;
using Atomic.Entities;

[System.Serializable]
public class AIBehaviorSettings
{
    [Header("Movement")]
    public float movementSpeed = 3.5f;
    public float rotationSpeed = 180f;
    public float stopDistance = 1f;
    
    [Header("Combat")]
    public float attackRange = 2f;
    public float attackCooldown = 2f;
    public float aggroRange = 8f;
    
    [Header("Vision")]
    public float viewDistance = 10f;
    public float viewAngle = 90f;
    public LayerMask obstacleLayer = -1;
    
    [Header("Patrol")]
    public bool enablePatrol = true;
    public Transform[] patrolPoints;
    public float patrolWaitTime = 2f;
}

public class SceneAIInstaller : SceneEntityInstaller
{
    [Header("AI Configuration")]
    [SerializeField] private AIType _aiType = AIType.Passive;
    [SerializeField] private AIBehaviorSettings _behaviorSettings = new AIBehaviorSettings();
    
    [Header("State Machine")]
    [SerializeField] private AIState _initialState = AIState.Idle;
    [SerializeField] private bool _debugStateChanges = false;
    
    [Header("Visual Debugging")]
    [SerializeField] private bool _drawGizmos = true;
    [SerializeField] private Color _gizmoColor = Color.red;

    public override void Install(IEntity entity)
    {
        // Install AI type and state
        entity.Set("AIType", _aiType.ToString());
        entity.Set("CurrentState", _initialState.ToString());
        entity.Set("PreviousState", "None");
        entity.Set("StateChangeTime", Time.time);

        // Install movement settings
        entity.Set("MovementSpeed", _behaviorSettings.movementSpeed);
        entity.Set("RotationSpeed", _behaviorSettings.rotationSpeed);
        entity.Set("StopDistance", _behaviorSettings.stopDistance);

        // Install combat settings
        entity.Set("AttackRange", _behaviorSettings.attackRange);
        entity.Set("AttackCooldown", _behaviorSettings.attackCooldown);
        entity.Set("AggroRange", _behaviorSettings.aggroRange);
        entity.Set("LastAttackTime", 0f);

        // Install vision settings
        entity.Set("ViewDistance", _behaviorSettings.viewDistance);
        entity.Set("ViewAngle", _behaviorSettings.viewAngle);
        entity.Set("ObstacleLayer", (int)_behaviorSettings.obstacleLayer);

        // Install patrol settings
        if (_behaviorSettings.enablePatrol && _behaviorSettings.patrolPoints != null)
        {
            var patrolPositions = new Vector3[_behaviorSettings.patrolPoints.Length];
            for (int i = 0; i < _behaviorSettings.patrolPoints.Length; i++)
            {
                patrolPositions[i] = _behaviorSettings.patrolPoints[i].position;
            }
            
            entity.Set("PatrolPoints", patrolPositions);
            entity.Set("CurrentPatrolIndex", 0);
            entity.Set("PatrolWaitTime", _behaviorSettings.patrolWaitTime);
            entity.Set("PatrolTimer", 0f);
        }

        // Install AI tags
        InstallAITags(entity);

        // Install debug settings
        entity.Set("DebugStateChanges", _debugStateChanges);
        entity.Set("DrawGizmos", _drawGizmos);
        entity.Set("GizmoColor", _gizmoColor);

        Debug.Log($"AI configuration '{_aiType}' installed for entity: {entity.Name}");
    }

    private void InstallAITags(IEntity entity)
    {
        entity.AddTag("HasAI");
        entity.AddTag($"AI_{_aiType}");

        switch (_aiType)
        {
            case AIType.Aggressive:
                entity.AddTag("Hostile");
                entity.AddTag("CanAttack");
                entity.AddTag("CanChase");
                break;
                
            case AIType.Defensive:
                entity.AddTag("CanAttack");
                entity.AddTag("CanFlee");
                break;
                
            case AIType.Guard:
                entity.AddTag("CanAttack");
                entity.AddTag("CanChase");
                entity.AddTag("Guardian");
                entity.Set("GuardPosition", transform.position);
                break;
                
            case AIType.Passive:
                entity.AddTag("Peaceful");
                break;
        }

        if (_behaviorSettings.enablePatrol)
        {
            entity.AddTag("CanPatrol");
        }
    }

    void OnDrawGizmosSelected()
    {
        if (!_drawGizmos) return;

        Gizmos.color = _gizmoColor;

        // Draw aggro range
        Gizmos.DrawWireeSphere(transform.position, _behaviorSettings.aggroRange);

        // Draw attack range
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(transform.position, _behaviorSettings.attackRange);

        // Draw view distance and angle
        Gizmos.color = Color.yellow;
        Vector3 viewAngleA = DirectionFromAngle(-_behaviorSettings.viewAngle / 2, false);
        Vector3 viewAngleB = DirectionFromAngle(_behaviorSettings.viewAngle / 2, false);

        Gizmos.DrawLine(transform.position, transform.position + viewAngleA * _behaviorSettings.viewDistance);
        Gizmos.DrawLine(transform.position, transform.position + viewAngleB * _behaviorSettings.viewDistance);

        // Draw patrol points
        if (_behaviorSettings.enablePatrol && _behaviorSettings.patrolPoints != null)
        {
            Gizmos.color = Color.blue;
            for (int i = 0; i < _behaviorSettings.patrolPoints.Length; i++)
            {
                if (_behaviorSettings.patrolPoints[i] != null)
                {
                    Gizmos.DrawWireSphere(_behaviorSettings.patrolPoints[i].position, 0.5f);
                    
                    if (i < _behaviorSettings.patrolPoints.Length - 1)
                    {
                        Gizmos.DrawLine(_behaviorSettings.patrolPoints[i].position, 
                                      _behaviorSettings.patrolPoints[i + 1].position);
                    }
                }
            }
        }
    }

    private Vector3 DirectionFromAngle(float angleInDegrees, bool angleIsGlobal)
    {
        if (!angleIsGlobal)
        {
            angleInDegrees += transform.eulerAngles.y;
        }
        return new Vector3(Mathf.Sin(angleInDegrees * Mathf.Deg2Rad), 0, Mathf.Cos(angleInDegrees * Mathf.Deg2Rad));
    }
}

public enum AIType
{
    Passive, Defensive, Aggressive, Guard
}

public enum AIState
{
    Idle, Patrol, Chase, Attack, Flee, Guard, Search
}
```

### Interactive Object Scene Installer

```csharp
using UnityEngine;
using Atomic.Entities;
using UnityEngine.Events;

[System.Serializable]
public class InteractionSettings
{
    public string interactionPrompt = "Press E to interact";
    public float interactionRange = 2f;
    public bool requireLineOfSight = true;
    public LayerMask interactionLayers = -1;
    public bool oneTimeUse = false;
    public float cooldownTime = 0f;
}

public class SceneInteractableInstaller : SceneEntityInstaller
{
    [Header("Interaction Configuration")]
    [SerializeField] private InteractionSettings _interactionSettings = new InteractionSettings();
    
    [Header("Visual Feedback")]
    [SerializeField] private GameObject _highlightEffect;
    [SerializeField] private AudioClip _interactionSound;
    [SerializeField] private ParticleSystem _interactionParticles;
    
    [Header("Events")]
    [SerializeField] private UnityEvent _onInteraction;
    [SerializeField] private UnityEvent _onHighlight;
    [SerializeField] private UnityEvent _onUnhighlight;

    public override void Install(IEntity entity)
    {
        // Install interaction settings
        entity.Set("InteractionPrompt", _interactionSettings.interactionPrompt);
        entity.Set("InteractionRange", _interactionSettings.interactionRange);
        entity.Set("RequireLineOfSight", _interactionSettings.requireLineOfSight);
        entity.Set("InteractionLayers", (int)_interactionSettings.interactionLayers);
        entity.Set("OneTimeUse", _interactionSettings.oneTimeUse);
        entity.Set("CooldownTime", _interactionSettings.cooldownTime);
        entity.Set("LastInteractionTime", 0f);
        entity.Set("TimesInteracted", 0);

        // Install visual components references
        if (_highlightEffect != null)
        {
            entity.Set("HighlightEffect", _highlightEffect);
            _highlightEffect.SetActive(false);
        }

        if (_interactionSound != null)
        {
            entity.Set("InteractionSound", _interactionSound);
        }

        if (_interactionParticles != null)
        {
            entity.Set("InteractionParticles", _interactionParticles);
        }

        // Install interaction tags
        entity.AddTag("Interactable");
        entity.AddTag("CanHighlight");

        if (_interactionSettings.oneTimeUse)
        {
            entity.AddTag("OneTimeUse");
        }

        // Store Unity Events (would need custom serialization in real implementation)
        RegisterInteractionEvents(entity);

        Debug.Log($"Interactable configuration installed for entity: {entity.Name}");
    }

    private void RegisterInteractionEvents(IEntity entity)
    {
        // In a real implementation, you'd need a system to handle Unity Events
        // This is a simplified example showing the concept
        entity.Set("OnInteractionCallback", new System.Action(() => 
        {
            _onInteraction?.Invoke();
            PlayInteractionEffects(entity);
        }));

        entity.Set("OnHighlightCallback", new System.Action(() => 
        {
            _onHighlight?.Invoke();
            ShowHighlight(entity);
        }));

        entity.Set("OnUnhighlightCallback", new System.Action(() => 
        {
            _onUnhighlight?.Invoke();
            HideHighlight(entity);
        }));
    }

    private void PlayInteractionEffects(IEntity entity)
    {
        if (entity.HasValue("InteractionSound"))
        {
            var audioSource = GetComponent<AudioSource>();
            if (audioSource != null)
            {
                audioSource.PlayOneShot(entity.Get<AudioClip>("InteractionSound"));
            }
        }

        if (entity.HasValue("InteractionParticles"))
        {
            var particles = entity.Get<ParticleSystem>("InteractionParticles");
            particles?.Play();
        }
    }

    private void ShowHighlight(IEntity entity)
    {
        if (entity.HasValue("HighlightEffect"))
        {
            var highlight = entity.Get<GameObject>("HighlightEffect");
            highlight?.SetActive(true);
        }
    }

    private void HideHighlight(IEntity entity)
    {
        if (entity.HasValue("HighlightEffect"))
        {
            var highlight = entity.Get<GameObject>("HighlightEffect");
            highlight?.SetActive(false);
        }
    }

    void OnDrawGizmosSelected()
    {
        // Draw interaction range
        Gizmos.color = Color.green;
        Gizmos.DrawWireSphere(transform.position, _interactionSettings.interactionRange);
    }
}
```

### Level Design Scene Installer

```csharp
using UnityEngine;
using Atomic.Entities;
using System.Collections.Generic;

[System.Serializable]
public class SpawnConfiguration
{
    public string entityType;
    public GameObject prefab;
    public int spawnCount = 1;
    public float spawnRadius = 5f;
    public bool randomizePosition = true;
    public Vector3 spawnOffset = Vector3.zero;
}

public class SceneLevelInstaller : SceneEntityInstaller
{
    [Header("Level Configuration")]
    [SerializeField] private string _levelName = "Untitled Level";
    [SerializeField] private int _levelDifficulty = 1;
    [SerializeField] private float _levelScale = 1f;
    
    [Header("Spawning")]
    [SerializeField] private SpawnConfiguration[] _spawnConfigurations;
    [SerializeField] private Transform _spawnParent;
    
    [Header("Environment")]
    [SerializeField] private Color _ambientColor = Color.white;
    [SerializeField] private float _fogDensity = 0.01f;
    [SerializeField] private Material _skyboxMaterial;

    public override void Install(IEntity entity)
    {
        // Install level metadata
        entity.Set("LevelName", _levelName);
        entity.Set("LevelDifficulty", _levelDifficulty);
        entity.Set("LevelScale", _levelScale);
        entity.Set("SpawnedEntities", new List<IEntity>());

        // Install environment settings
        entity.Set("AmbientColor", _ambientColor);
        entity.Set("FogDensity", _fogDensity);
        if (_skyboxMaterial != null)
        {
            entity.Set("SkyboxMaterial", _skyboxMaterial);
        }

        // Apply environment settings immediately
        ApplyEnvironmentSettings();

        // Process spawn configurations
        ProcessSpawnConfigurations(entity);

        // Add level tags
        entity.AddTag("Level");
        entity.AddTag($"Difficulty_{_levelDifficulty}");

        Debug.Log($"Level '{_levelName}' installed with difficulty {_levelDifficulty}");
    }

    private void ApplyEnvironmentSettings()
    {
        RenderSettings.ambientLight = _ambientColor;
        RenderSettings.fogDensity = _fogDensity;
        
        if (_skyboxMaterial != null)
        {
            RenderSettings.skybox = _skyboxMaterial;
        }
    }

    private void ProcessSpawnConfigurations(IEntity levelEntity)
    {
        var spawnedEntities = new List<IEntity>();

        foreach (var config in _spawnConfigurations)
        {
            for (int i = 0; i < config.spawnCount; i++)
            {
                var spawnedEntity = CreateSpawnedEntity(config, i);
                if (spawnedEntity != null)
                {
                    spawnedEntities.Add(spawnedEntity);
                }
            }
        }

        levelEntity.Set("SpawnedEntities", spawnedEntities);
    }

    private IEntity CreateSpawnedEntity(SpawnConfiguration config, int index)
    {
        Vector3 spawnPosition = CalculateSpawnPosition(config, index);
        
        // In a real implementation, you'd use an entity factory here
        // This is a simplified example
        var entity = new Entity($"{config.entityType}_{index}");
        entity.Set("Position", spawnPosition);
        entity.Set("EntityType", config.entityType);
        entity.AddTag("SpawnedByLevel");
        entity.AddTag(config.entityType);

        // Apply level scaling
        ApplyLevelScaling(entity);

        return entity;
    }

    private Vector3 CalculateSpawnPosition(SpawnConfiguration config, int index)
    {
        Vector3 basePosition = transform.position + config.spawnOffset;

        if (config.randomizePosition)
        {
            Vector2 randomCircle = Random.insideUnitCircle * config.spawnRadius;
            return basePosition + new Vector3(randomCircle.x, 0, randomCircle.y);
        }
        else
        {
            float angle = (360f / config.spawnCount) * index;
            float x = Mathf.Cos(angle * Mathf.Deg2Rad) * config.spawnRadius;
            float z = Mathf.Sin(angle * Mathf.Deg2Rad) * config.spawnRadius;
            return basePosition + new Vector3(x, 0, z);
        }
    }

    private void ApplyLevelScaling(IEntity entity)
    {
        if (entity.HasValue("Health"))
        {
            float scaledHealth = entity.Get<float>("Health") * _levelScale;
            entity.Set("Health", scaledHealth);
            entity.Set("MaxHealth", scaledHealth);
        }

        if (entity.HasValue("Damage"))
        {
            float scaledDamage = entity.Get<float>("Damage") * _levelScale;
            entity.Set("Damage", scaledDamage);
        }
    }

    void OnDrawGizmosSelected()
    {
        if (_spawnConfigurations == null) return;

        foreach (var config in _spawnConfigurations)
        {
            Gizmos.color = Color.cyan;
            Vector3 spawnCenter = transform.position + config.spawnOffset;
            Gizmos.DrawWireSphere(spawnCenter, config.spawnRadius);

            // Draw spawn points preview
            Gizmos.color = Color.magenta;
            for (int i = 0; i < config.spawnCount; i++)
            {
                Vector3 spawnPos = CalculateSpawnPosition(config, i);
                Gizmos.DrawWireCube(spawnPos, Vector3.one * 0.5f);
            }
        }
    }
}
```

## Integration with Unity Editor

### Custom Inspector Integration

```csharp
#if UNITY_EDITOR
using UnityEditor;

[CustomEditor(typeof(SceneStatsInstaller))]
public class SceneStatsInstallerEditor : Editor
{
    public override void OnInspectorGUI()
    {
        DrawDefaultInspector();
        
        SceneStatsInstaller installer = (SceneStatsInstaller)target;
        
        GUILayout.Space(10);
        
        if (GUILayout.Button("Preview Installation"))
        {
            PreviewInstallation(installer);
        }
        
        if (GUILayout.Button("Test Install on Temporary Entity"))
        {
            TestInstallation(installer);
        }
    }
    
    private void PreviewInstallation(SceneStatsInstaller installer)
    {
        Debug.Log("Preview of installation configuration:");
        Debug.Log($"Health will be set to: {installer._stats.health}");
        Debug.Log($"Tags to be added: {string.Join(", ", installer._tags)}");
    }
    
    private void TestInstallation(SceneStatsInstaller installer)
    {
        var testEntity = new Entity("Test Entity");
        installer.Install(testEntity);
        Debug.Log($"Test installation completed for: {testEntity.Name}");
    }
}
#endif
```

## Implementation Notes

### Unity Integration
- Inherits from MonoBehaviour for scene integration
- OnValidate provides immediate editor feedback
- Conditional compilation ensures editor-only features

### Editor Refresh System
- Internal refresh callback for custom tooling
- Safe execution outside play mode and compilation
- Exception handling for robust editor experience

### Type Safety
- Generic version eliminates manual casting
- Sealed override ensures consistent behavior
- Protected abstract method for implementation focus

## Best Practices

### Scene Organization
- Group related installers on parent GameObjects
- Use meaningful names for installer components
- Document complex configurations in component headers

### Inspector Configuration
- Use SerializeField for private configuration fields
- Implement OnDrawGizmosSelected for visual debugging
- Add header attributes to organize inspector layout

### Performance Considerations
- Minimize work in OnValidate to avoid editor lag
- Cache expensive calculations during installation
- Consider impact of large numbers of installers in scenes

## Common Patterns

### Multi-Component Installation

```csharp
public class SceneMultiComponentInstaller : SceneEntityInstaller
{
    [SerializeField] private SceneEntityInstaller[] _subInstallers;
    
    public override void Install(IEntity entity)
    {
        foreach (var installer in _subInstallers)
        {
            if (installer != null)
            {
                installer.Install(entity);
            }
        }
    }
}
```

### Conditional Scene Installation

```csharp
public class SceneConditionalInstaller : SceneEntityInstaller
{
    [SerializeField] private bool _installOnlyInPlayMode = true;
    [SerializeField] private string[] _requiredTags;
    [SerializeField] private SceneEntityInstaller _installer;
    
    public override void Install(IEntity entity)
    {
        if (_installOnlyInPlayMode && !Application.isPlaying)
            return;
            
        if (_requiredTags.Any(tag => !entity.HasTag(tag)))
            return;
            
        _installer?.Install(entity);
    }
}
```

The `SceneEntityInstaller` provides powerful Unity Editor integration for visual entity configuration, enabling designers and developers to set up entities directly in scenes with immediate feedback and robust tooling support within the Atomic framework.