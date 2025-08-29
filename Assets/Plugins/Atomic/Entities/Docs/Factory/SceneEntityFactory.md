# 🧩 SceneEntityFactory

An abstract Unity MonoBehaviour-based factory for creating entities with scene-based workflows. Provides design-time entity configuration, editor integration, and runtime entity creation optimized for Unity development patterns.

## Overview

`SceneEntityFactory` bridges Unity's MonoBehaviour system with the Atomic entity framework, enabling visual entity configuration in the Unity Editor. Supports precompilation for optimization, editor validation, and seamless integration with Unity's scene management and asset workflows.

## Class Definition

```csharp
#if UNITY_5_3_OR_NEWER
namespace Atomic.Entities
{
    /// <summary>
    /// Abstract base for Unity-based IEntity factories.
    /// </summary>
    public abstract class SceneEntityFactory : SceneEntityFactory<IEntity>, IEntityFactory
    {
        public sealed override IEntity Create()
        {
            var entity = new Entity(
                this.name,
                this.InitialTagCount,
                this.InitialValueCount,
                this.InitialBehaviourCount
            );
            this.Install(entity);
            return entity;
        }

        protected abstract void Install(IEntity entity);
    }

    /// <summary>
    /// Generic Unity-based factory for creating typed entities.
    /// </summary>
    /// <typeparam name="E">The type of entity to create</typeparam>
    public abstract class SceneEntityFactory<E> : MonoBehaviour, IEntityFactory<E> where E : IEntity
    {
        [SerializeField] protected int InitialTagCount;
        [SerializeField] protected int InitialValueCount;
        [SerializeField] protected int InitialBehaviourCount;

        public abstract E Create();
        protected virtual void OnValidate();
        protected virtual void Reset();
        protected virtual void Precompile();
    }
}
#endif
```

## Key Features

### Unity Editor Integration
- MonoBehaviour-based for direct placement in Unity scenes
- Serialized fields for entity initialization parameters
- OnValidate callback for real-time editor updates
- Context menu for manual precompilation

### Design-Time Optimization
- Precompile method extracts entity metadata at edit time
- Cached tag, value, and behavior counts for performance
- Editor-only validation and error handling
- Odin Inspector support for enhanced editor experience

### Scene-Based Workflows
- Direct placement in Unity scenes as GameObjects
- Integration with Unity's prefab system
- Scene baking and conversion support
- Asset management through Unity's asset database

## Usage Examples

### Basic Scene Entity Factory

```csharp
public class PlayerEntityFactory : SceneEntityFactory
{
    [Header("Player Configuration")]
    [SerializeField] private float _maxHealth = 100f;
    [SerializeField] private float _speed = 10f;
    [SerializeField] private int _level = 1;
    
    [Header("Equipment")]
    [SerializeField] private WeaponData _startingWeapon;
    [SerializeField] private ArmorData _startingArmor;
    
    protected override void Install(IEntity entity)
    {
        // Core stats
        entity.Set("Health", _maxHealth);
        entity.Set("MaxHealth", _maxHealth);
        entity.Set("Speed", _speed);
        entity.Set("Level", _level);
        
        // Equipment
        if (_startingWeapon != null)
        {
            entity.Set("WeaponId", _startingWeapon.id);
            entity.Set("WeaponDamage", _startingWeapon.damage);
        }
        
        if (_startingArmor != null)
        {
            entity.Set("ArmorId", _startingArmor.id);
            entity.Set("ArmorDefense", _startingArmor.defense);
        }
        
        // Tags
        entity.AddTag("Player");
        entity.AddTag("Controllable");
        entity.AddTag("Mortal");
        
        // Behaviors
        entity.AddBehaviour<PlayerMovementBehaviour>();
        entity.AddBehaviour<PlayerCombatBehaviour>();
    }
}
```

### Procedural Enemy Factory

```csharp
public class EnemyEntityFactory : SceneEntityFactory
{
    [System.Serializable]
    public struct EnemyTier
    {
        public string name;
        public float healthMultiplier;
        public float speedMultiplier;
        public float damageMultiplier;
        public Color tintColor;
    }
    
    [Header("Base Configuration")]
    [SerializeField] private float _baseHealth = 50f;
    [SerializeField] private float _baseSpeed = 5f;
    [SerializeField] private float _baseDamage = 10f;
    
    [Header("Tier System")]
    [SerializeField] private EnemyTier[] _tiers;
    [SerializeField] private AnimationCurve _tierProbability;
    
    [Header("Procedural Settings")]
    [SerializeField] private bool _randomizeStats = true;
    [SerializeField, Range(0.8f, 1.2f)] private float _statVariance = 0.1f;
    
    protected override void Install(IEntity entity)
    {
        EnemyTier selectedTier = SelectRandomTier();
        
        // Calculate final stats
        float health = _baseHealth * selectedTier.healthMultiplier;
        float speed = _baseSpeed * selectedTier.speedMultiplier;
        float damage = _baseDamage * selectedTier.damageMultiplier;
        
        // Apply stat randomization
        if (_randomizeStats)
        {
            health *= Random.Range(1f - _statVariance, 1f + _statVariance);
            speed *= Random.Range(1f - _statVariance, 1f + _statVariance);
            damage *= Random.Range(1f - _statVariance, 1f + _statVariance);
        }
        
        // Set entity data
        entity.Set("Health", health);
        entity.Set("MaxHealth", health);
        entity.Set("Speed", speed);
        entity.Set("Damage", damage);
        entity.Set("TierName", selectedTier.name);
        entity.Set("TintColor", selectedTier.tintColor);
        
        // Tags
        entity.AddTag("Enemy");
        entity.AddTag("Hostile");
        entity.AddTag(selectedTier.name);
        
        // Behaviors based on tier
        entity.AddBehaviour<EnemyMovementBehaviour>();
        entity.AddBehaviour<EnemyCombatBehaviour>();
        
        if (selectedTier.name.Contains("Elite"))
        {
            entity.AddBehaviour<EliteEnemyBehaviour>();
        }
    }
    
    private EnemyTier SelectRandomTier()
    {
        float randomValue = Random.value;
        float cumulativeProbability = 0f;
        
        for (int i = 0; i < _tiers.Length; i++)
        {
            float tierProbability = _tierProbability.Evaluate((float)i / (_tiers.Length - 1));
            cumulativeProbability += tierProbability;
            
            if (randomValue <= cumulativeProbability)
            {
                return _tiers[i];
            }
        }
        
        return _tiers[_tiers.Length - 1];
    }
}
```

### Scene-Based Entity Configuration

```csharp
public class ConfigurableEntityFactory : SceneEntityFactory<Entity>
{
    [System.Serializable]
    public struct TagConfig
    {
        public string tagName;
        public bool enabled;
    }
    
    [System.Serializable]
    public struct ValueConfig
    {
        public string key;
        public float value;
    }
    
    [System.Serializable]
    public struct BehaviourConfig
    {
        public string behaviourTypeName;
        public bool enabled;
    }
    
    [Header("Entity Configuration")]
    [SerializeField] private string _entityName = "ConfigurableEntity";
    
    [Header("Tags")]
    [SerializeField] private TagConfig[] _tags;
    
    [Header("Values")]
    [SerializeField] private ValueConfig[] _values;
    
    [Header("Behaviours")]
    [SerializeField] private BehaviourConfig[] _behaviours;
    
    public override Entity Create()
    {
        var entity = new Entity(
            _entityName,
            this.InitialTagCount,
            this.InitialValueCount,
            this.InitialBehaviourCount
        );
        
        // Apply tags
        foreach (var tagConfig in _tags)
        {
            if (tagConfig.enabled)
            {
                entity.AddTag(tagConfig.tagName);
            }
        }
        
        // Apply values
        foreach (var valueConfig in _values)
        {
            entity.Set(valueConfig.key, valueConfig.value);
        }
        
        // Apply behaviours
        foreach (var behaviourConfig in _behaviours)
        {
            if (behaviourConfig.enabled)
            {
                var behaviourType = Type.GetType(behaviourConfig.behaviourTypeName);
                if (behaviourType != null && typeof(IEntityBehaviour).IsAssignableFrom(behaviourType))
                {
                    var behaviour = (IEntityBehaviour)Activator.CreateInstance(behaviourType);
                    entity.AddBehaviour(behaviour);
                }
            }
        }
        
        return entity;
    }
    
    protected override void OnValidate()
    {
        base.OnValidate();
        
        // Validate configuration
        ValidateConfiguration();
    }
    
    private void ValidateConfiguration()
    {
        // Check for duplicate tag names
        var tagNames = _tags.Where(t => t.enabled).Select(t => t.tagName).ToArray();
        if (tagNames.Length != tagNames.Distinct().Count())
        {
            Debug.LogWarning("Duplicate tag names detected in ConfigurableEntityFactory", this);
        }
        
        // Check for duplicate value keys
        var valueKeys = _values.Select(v => v.key).ToArray();
        if (valueKeys.Length != valueKeys.Distinct().Count())
        {
            Debug.LogWarning("Duplicate value keys detected in ConfigurableEntityFactory", this);
        }
    }
}
```

## Integration with Atomic Framework

### Reactive Scene Factory

```csharp
public class ReactiveSceneFactory : SceneEntityFactory
{
    [Header("Reactive Configuration")]
    [SerializeField] private ReactiveFloat _healthMultiplier = new(1f);
    [SerializeField] private ReactiveFloat _speedMultiplier = new(1f);
    [SerializeField] private ReactiveInt _difficultyLevel = new(1);
    
    private readonly Dictionary<string, object> _cachedValues = new();
    
    void Start()
    {
        // React to configuration changes
        _healthMultiplier.Subscribe(OnHealthMultiplierChanged);
        _speedMultiplier.Subscribe(OnSpeedMultiplierChanged);
        _difficultyLevel.Subscribe(OnDifficultyChanged);
    }
    
    protected override void Install(IEntity entity)
    {
        // Base configuration
        float baseHealth = 100f;
        float baseSpeed = 10f;
        
        // Apply reactive multipliers
        entity.Set("Health", baseHealth * _healthMultiplier.Value);
        entity.Set("MaxHealth", baseHealth * _healthMultiplier.Value);
        entity.Set("Speed", baseSpeed * _speedMultiplier.Value);
        entity.Set("DifficultyLevel", _difficultyLevel.Value);
        
        // Apply difficulty-based modifications
        ApplyDifficultyModifications(entity);
        
        entity.AddTag("Reactive");
        entity.AddTag($"Difficulty{_difficultyLevel.Value}");
    }
    
    private void ApplyDifficultyModifications(IEntity entity)
    {
        int difficulty = _difficultyLevel.Value;
        
        // Scale entity properties based on difficulty
        entity.Set("ExperienceReward", 10 * difficulty);
        entity.Set("GoldReward", 5 * difficulty);
        
        if (difficulty >= 3)
        {
            entity.AddTag("Elite");
            entity.Set("CriticalChance", 0.15f);
        }
        
        if (difficulty >= 5)
        {
            entity.AddTag("Legendary");
            entity.Set("RegenerationRate", 2f);
        }
    }
    
    private void OnHealthMultiplierChanged(float newMultiplier)
    {
        Debug.Log($"Health multiplier changed to: {newMultiplier}");
        // Update existing entities if needed
    }
    
    private void OnSpeedMultiplierChanged(float newMultiplier)
    {
        Debug.Log($"Speed multiplier changed to: {newMultiplier}");
    }
    
    private void OnDifficultyChanged(int newDifficulty)
    {
        Debug.Log($"Difficulty level changed to: {newDifficulty}");
        // Potentially recreate or update entities
    }
}
```

### Scene Factory with Event Integration

```csharp
public class EventDrivenSceneFactory : SceneEntityFactory
{
    [Header("Event Configuration")]
    [SerializeField] private GameEvent[] _triggerEvents;
    [SerializeField] private bool _autoCreateOnStart = true;
    [SerializeField] private float _creationDelay = 0f;
    
    [Header("Creation Settings")]
    [SerializeField] private int _maxEntities = 10;
    [SerializeField] private float _cooldownTime = 1f;
    
    private readonly List<IEntity> _createdEntities = new();
    private float _lastCreationTime;
    
    void Start()
    {
        if (_autoCreateOnStart)
        {
            StartCoroutine(CreateEntityWithDelay());
        }
        
        // Subscribe to trigger events
        foreach (var gameEvent in _triggerEvents)
        {
            GameEvents.Subscribe(gameEvent, OnTriggerEvent);
        }
    }
    
    private IEnumerator CreateEntityWithDelay()
    {
        yield return new WaitForSeconds(_creationDelay);
        CreateEntityFromFactory();
    }
    
    private void OnTriggerEvent(GameEvent evt)
    {
        if (CanCreateEntity())
        {
            CreateEntityFromFactory();
        }
    }
    
    private bool CanCreateEntity()
    {
        return _createdEntities.Count < _maxEntities &&
               Time.time >= _lastCreationTime + _cooldownTime;
    }
    
    private void CreateEntityFromFactory()
    {
        IEntity entity = this.Create();
        entity.Set("FactorySource", this.name);
        entity.Set("CreationTime", Time.time);
        
        _createdEntities.Add(entity);
        _lastCreationTime = Time.time;
        
        // Subscribe to entity destruction
        entity.OnDestroyed += () => _createdEntities.Remove(entity);
        
        Debug.Log($"Created entity from {this.name}: {entity.Name}");
    }
    
    protected override void Install(IEntity entity)
    {
        // Base entity setup
        entity.Set("Health", 100f);
        entity.Set("Speed", 8f);
        entity.AddTag("EventDriven");
        entity.AddTag("SceneManaged");
        
        // Add event-specific data
        foreach (var gameEvent in _triggerEvents)
        {
            entity.AddTag($"Triggers_{gameEvent}");
        }
    }
}
```

## Implementation Notes

### Editor Integration
- OnValidate called when script is loaded or Inspector values change
- Precompile method extracts entity metadata for optimization
- Reset method initializes default values in Editor
- Odin Inspector attributes enhance the editing experience

### Performance Optimization
- InitialTagCount, InitialValueCount, InitialBehaviourCount cached for performance
- Precompile runs in Editor to avoid runtime introspection
- Thread-safe for Unity's main thread execution

### Unity Lifecycle
- MonoBehaviour lifecycle methods available for scene integration
- Seamless integration with Unity's serialization system
- Compatible with Unity's prefab workflow and scene management

## Best Practices

### Entity Configuration
- Use SerializeField attributes for configurable entity properties
- Implement Install method to apply entity-specific configuration
- Validate configuration in OnValidate for immediate feedback

### Performance Considerations
- Run Precompile in Editor to cache entity metadata
- Avoid heavy computation in Create method for runtime performance
- Use appropriate initial counts for entity collections

### Scene Management
- Place factories in logical scene locations for organization
- Use meaningful GameObject names for factory identification
- Consider factory lifetime and cleanup strategies

## Common Patterns

### Factory Inheritance Hierarchy

```csharp
public abstract class BaseUnitFactory : SceneEntityFactory
{
    [Header("Base Unit Stats")]
    [SerializeField] protected float _baseHealth = 100f;
    [SerializeField] protected float _baseSpeed = 5f;
    [SerializeField] protected float _baseDamage = 10f;
    
    protected virtual void InstallBaseUnit(IEntity entity)
    {
        entity.Set("Health", _baseHealth);
        entity.Set("MaxHealth", _baseHealth);
        entity.Set("Speed", _baseSpeed);
        entity.Set("Damage", _baseDamage);
        entity.AddTag("Unit");
    }
    
    protected override void Install(IEntity entity)
    {
        InstallBaseUnit(entity);
        InstallSpecialization(entity);
    }
    
    protected abstract void InstallSpecialization(IEntity entity);
}

public class WarriorFactory : BaseUnitFactory
{
    [Header("Warrior Specialization")]
    [SerializeField] private float _armorBonus = 20f;
    [SerializeField] private float _strengthMultiplier = 1.5f;
    
    protected override void InstallSpecialization(IEntity entity)
    {
        entity.Set("Armor", _armorBonus);
        entity.Set("Damage", entity.Get<float>("Damage") * _strengthMultiplier);
        entity.AddTag("Warrior");
        entity.AddTag("Melee");
    }
}
```

The `SceneEntityFactory` provides a powerful Unity integration for entity creation, combining the flexibility of the Atomic entity system with Unity's design-time tools and scene-based workflows.