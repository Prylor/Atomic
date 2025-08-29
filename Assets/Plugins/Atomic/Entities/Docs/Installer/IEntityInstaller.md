# 🧩 IEntityInstaller

The base interface for entity installation systems that define mechanisms for configuring, initializing, or injecting data into entity instances. Provides the foundation for building modular, reusable entity configuration systems.

## Overview

`IEntityInstaller` defines the contract for components that configure or initialize entities with specific data, behaviors, or settings. Essential for building modular entity configuration systems, dependency injection patterns, and reusable entity setup logic.

## Interface Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Defines a generic mechanism for configuring or injecting data into an IEntity instance.
    /// </summary>
    public interface IEntityInstaller
    {
        /// <summary>
        /// Installs data, configuration, or behaviors into the specified IEntity.
        /// </summary>
        /// <param name="entity">The entity to configure or initialize.</param>
        void Install(IEntity entity);
    }

    /// <summary>
    /// Represents a type-safe installer for entities of type T.
    /// </summary>
    /// <typeparam name="T">The specific type of entity this installer supports.</typeparam>
    public interface IEntityInstaller<in T> : IEntityInstaller where T : IEntity
    {
        /// <summary>
        /// Installs data, configuration, or behaviors into the specified entity of type T.
        /// </summary>
        /// <param name="entity">The strongly-typed entity to configure or initialize.</param>
        void Install(T entity);

        /// <inheritdoc />
        void IEntityInstaller.Install(IEntity entity) => this.Install((T) entity);
    }
}
```

## Key Features

### Configuration Abstraction
- Generic interface for entity configuration and initialization
- Modular approach to entity setup and data injection
- Standardized installation contract across all implementations

### Type Safety
- Generic version provides compile-time type checking
- Automatic casting handled by interface implementation
- Type-safe entity configuration without manual casts

### Modular Architecture
- Composable installation system for complex entity setups
- Reusable installation logic across different entity types
- Separation of concerns for entity configuration

## Usage Examples

### Basic Configuration Installer

```csharp
public class BasicStatsInstaller : IEntityInstaller
{
    private readonly float _health;
    private readonly float _mana;
    private readonly float _stamina;

    public BasicStatsInstaller(float health, float mana, float stamina)
    {
        _health = health;
        _mana = mana;
        _stamina = stamina;
    }

    public void Install(IEntity entity)
    {
        entity.Set("Health", _health);
        entity.Set("MaxHealth", _health);
        entity.Set("Mana", _mana);
        entity.Set("MaxMana", _mana);
        entity.Set("Stamina", _stamina);
        entity.Set("MaxStamina", _stamina);
        
        // Add basic status tags
        entity.AddTag("Alive");
        entity.AddTag("Active");
        
        Debug.Log($"Basic stats installed for entity: {entity.Name}");
    }
}

// Usage
var statsInstaller = new BasicStatsInstaller(100f, 50f, 75f);
statsInstaller.Install(playerEntity);
```

### Typed Equipment Installer

```csharp
public class EquipmentInstaller : IEntityInstaller<IEntity>
{
    private readonly Dictionary<string, object> _equipment;

    public EquipmentInstaller(Dictionary<string, object> equipment)
    {
        _equipment = equipment ?? throw new ArgumentNullException(nameof(equipment));
    }

    public void Install(IEntity entity)
    {
        foreach (var item in _equipment)
        {
            entity.Set(item.Key, item.Value);
            Debug.Log($"Equipped {item.Value} to {item.Key} slot for entity: {entity.Name}");
        }
        
        // Update equipment-related tags
        UpdateEquipmentTags(entity);
    }

    private void UpdateEquipmentTags(IEntity entity)
    {
        entity.SetTagState("Armed", entity.HasValue("MainHand"));
        entity.SetTagState("Armored", entity.HasValue("Chest"));
        entity.SetTagState("FullyEquipped", IsFullyEquipped(entity));
    }

    private bool IsFullyEquipped(IEntity entity)
    {
        string[] requiredSlots = { "MainHand", "Chest", "Legs", "Boots" };
        return requiredSlots.All(slot => entity.HasValue(slot));
    }
}

// Usage
var equipment = new Dictionary<string, object>
{
    ["MainHand"] = new Sword("Iron Sword", 25),
    ["Chest"] = new Armor("Chain Mail", 15),
    ["Legs"] = new Armor("Chain Leggings", 10),
    ["Boots"] = new Armor("Leather Boots", 5)
};

var equipmentInstaller = new EquipmentInstaller(equipment);
equipmentInstaller.Install(warriorEntity);
```

### Role-Based Installer

```csharp
public class RoleInstaller : IEntityInstaller
{
    private readonly EntityRole _role;

    public RoleInstaller(EntityRole role)
    {
        _role = role;
    }

    public void Install(IEntity entity)
    {
        switch (_role)
        {
            case EntityRole.Player:
                InstallPlayerRole(entity);
                break;
            case EntityRole.Enemy:
                InstallEnemyRole(entity);
                break;
            case EntityRole.NPC:
                InstallNPCRole(entity);
                break;
            case EntityRole.Item:
                InstallItemRole(entity);
                break;
        }
    }

    private void InstallPlayerRole(IEntity entity)
    {
        entity.AddTag("Player");
        entity.AddTag("Controllable");
        entity.AddTag("CanLevelUp");
        entity.AddTag("CanUseItems");
        
        entity.Set("ExperiencePoints", 0);
        entity.Set("Level", 1);
        entity.Set("SkillPoints", 0);
        
        Debug.Log($"Player role installed for entity: {entity.Name}");
    }

    private void InstallEnemyRole(IEntity entity)
    {
        entity.AddTag("Enemy");
        entity.AddTag("Hostile");
        entity.AddTag("CanDropLoot");
        
        entity.Set("ThreatLevel", 1);
        entity.Set("AggroRange", 5f);
        entity.Set("PatrolRadius", 10f);
        
        Debug.Log($"Enemy role installed for entity: {entity.Name}");
    }

    private void InstallNPCRole(IEntity entity)
    {
        entity.AddTag("NPC");
        entity.AddTag("Neutral");
        entity.AddTag("CanTalk");
        
        entity.Set("DialogueId", "default_npc");
        entity.Set("ShopId", "none");
        
        Debug.Log($"NPC role installed for entity: {entity.Name}");
    }

    private void InstallItemRole(IEntity entity)
    {
        entity.AddTag("Item");
        entity.AddTag("Pickupable");
        
        entity.Set("Value", 1);
        entity.Set("Weight", 0.1f);
        entity.Set("Stackable", true);
        
        Debug.Log($"Item role installed for entity: {entity.Name}");
    }
}

public enum EntityRole
{
    Player, Enemy, NPC, Item
}

// Usage
var playerRoleInstaller = new RoleInstaller(EntityRole.Player);
playerRoleInstaller.Install(playerEntity);

var enemyRoleInstaller = new RoleInstaller(EntityRole.Enemy);
enemyRoleInstaller.Install(goblinEntity);
```

### AI Configuration Installer

```csharp
public class AIConfigurationInstaller : IEntityInstaller
{
    private readonly AIConfiguration _config;

    public AIConfigurationInstaller(AIConfiguration config)
    {
        _config = config ?? throw new ArgumentNullException(nameof(config));
    }

    public void Install(IEntity entity)
    {
        // Install AI behavior settings
        entity.Set("AIType", _config.AIType.ToString());
        entity.Set("MovementSpeed", _config.MovementSpeed);
        entity.Set("AttackRange", _config.AttackRange);
        entity.Set("ViewDistance", _config.ViewDistance);
        entity.Set("ReactionTime", _config.ReactionTime);
        
        // Install AI state machine
        entity.Set("CurrentState", _config.InitialState.ToString());
        entity.Set("StateHistory", new List<string>());
        
        // Install AI tags based on configuration
        InstallAITags(entity);
        
        // Install AI behaviors
        InstallAIBehaviors(entity);
        
        Debug.Log($"AI configuration '{_config.AIType}' installed for entity: {entity.Name}");
    }

    private void InstallAITags(IEntity entity)
    {
        entity.AddTag("HasAI");
        entity.AddTag($"AI_{_config.AIType}");
        
        if (_config.CanPatrol)
            entity.AddTag("CanPatrol");
            
        if (_config.CanChase)
            entity.AddTag("CanChase");
            
        if (_config.CanAttack)
            entity.AddTag("CanAttack");
            
        if (_config.CanFlee)
            entity.AddTag("CanFlee");
    }

    private void InstallAIBehaviors(IEntity entity)
    {
        // Install behavior scripts or components based on AI type
        switch (_config.AIType)
        {
            case AIType.Aggressive:
                entity.Set("AggressionLevel", 0.8f);
                entity.Set("FleeThreshold", 0.1f);
                break;
                
            case AIType.Defensive:
                entity.Set("AggressionLevel", 0.3f);
                entity.Set("FleeThreshold", 0.4f);
                break;
                
            case AIType.Passive:
                entity.Set("AggressionLevel", 0.0f);
                entity.Set("FleeThreshold", 0.7f);
                break;
                
            case AIType.Guard:
                entity.Set("GuardPosition", entity.Get<Vector3>("Position"));
                entity.Set("GuardRadius", _config.GuardRadius);
                break;
        }
    }
}

[System.Serializable]
public class AIConfiguration
{
    public AIType AIType = AIType.Passive;
    public AIState InitialState = AIState.Idle;
    public float MovementSpeed = 3.5f;
    public float AttackRange = 2f;
    public float ViewDistance = 10f;
    public float ReactionTime = 0.5f;
    public float GuardRadius = 5f;
    public bool CanPatrol = true;
    public bool CanChase = true;
    public bool CanAttack = true;
    public bool CanFlee = false;
}

public enum AIType
{
    Passive, Defensive, Aggressive, Guard
}

public enum AIState
{
    Idle, Patrol, Chase, Attack, Flee, Guard
}

// Usage
var guardAI = new AIConfiguration
{
    AIType = AIType.Guard,
    InitialState = AIState.Guard,
    MovementSpeed = 2f,
    AttackRange = 3f,
    ViewDistance = 15f,
    GuardRadius = 8f,
    CanPatrol = false,
    CanFlee = false
};

var aiInstaller = new AIConfigurationInstaller(guardAI);
aiInstaller.Install(guardEntity);
```

### Composite Installer System

```csharp
public class CompositeInstaller : IEntityInstaller
{
    private readonly List<IEntityInstaller> _installers;

    public CompositeInstaller(params IEntityInstaller[] installers)
    {
        _installers = new List<IEntityInstaller>(installers ?? throw new ArgumentNullException(nameof(installers)));
    }

    public CompositeInstaller()
    {
        _installers = new List<IEntityInstaller>();
    }

    public void AddInstaller(IEntityInstaller installer)
    {
        if (installer != null)
        {
            _installers.Add(installer);
        }
    }

    public void RemoveInstaller(IEntityInstaller installer)
    {
        _installers.Remove(installer);
    }

    public void Install(IEntity entity)
    {
        Debug.Log($"Running composite installation for entity: {entity.Name}");
        
        foreach (var installer in _installers)
        {
            try
            {
                installer.Install(entity);
            }
            catch (Exception ex)
            {
                Debug.LogError($"Error in installer {installer.GetType().Name}: {ex.Message}");
                // Continue with remaining installers
            }
        }
        
        Debug.Log($"Composite installation completed for entity: {entity.Name}");
    }
}

// Usage
var playerSetup = new CompositeInstaller(
    new BasicStatsInstaller(100f, 50f, 75f),
    new RoleInstaller(EntityRole.Player),
    new EquipmentInstaller(startingEquipment),
    new SkillInstaller(startingSkills),
    new InventoryInstaller(startingItems)
);

playerSetup.Install(newPlayerEntity);
```

### Configuration-Driven Installer

```csharp
public class ConfigurationInstaller : IEntityInstaller
{
    private readonly EntityConfiguration _config;

    public ConfigurationInstaller(EntityConfiguration config)
    {
        _config = config ?? throw new ArgumentNullException(nameof(config));
    }

    public void Install(IEntity entity)
    {
        // Install values
        foreach (var value in _config.Values)
        {
            entity.Set(value.Key, ConvertValue(value.Value, value.Type));
        }
        
        // Install tags
        foreach (var tag in _config.Tags)
        {
            entity.AddTag(tag);
        }
        
        // Install behaviors
        InstallBehaviors(entity);
        
        // Run post-installation setup
        RunPostInstallation(entity);
        
        Debug.Log($"Configuration '{_config.Name}' installed for entity: {entity.Name}");
    }

    private object ConvertValue(string stringValue, string typeName)
    {
        switch (typeName.ToLower())
        {
            case "float":
                return float.Parse(stringValue);
            case "int":
                return int.Parse(stringValue);
            case "bool":
                return bool.Parse(stringValue);
            case "vector3":
                var parts = stringValue.Split(',');
                return new Vector3(float.Parse(parts[0]), float.Parse(parts[1]), float.Parse(parts[2]));
            default:
                return stringValue;
        }
    }

    private void InstallBehaviors(IEntity entity)
    {
        foreach (var behavior in _config.Behaviors)
        {
            // Install behavior components or scripts
            // This would integrate with behavior system
            Debug.Log($"Installing behavior: {behavior}");
        }
    }

    private void RunPostInstallation(IEntity entity)
    {
        if (!string.IsNullOrEmpty(_config.PostInstallationScript))
        {
            // Execute post-installation logic
            // This could run Lua scripts, execute methods, etc.
            Debug.Log($"Running post-installation: {_config.PostInstallationScript}");
        }
    }
}

[System.Serializable]
public class EntityConfiguration
{
    public string Name;
    public List<ConfigValue> Values = new();
    public List<string> Tags = new();
    public List<string> Behaviors = new();
    public string PostInstallationScript;
}

[System.Serializable]
public class ConfigValue
{
    public string Key;
    public string Value;
    public string Type;
}

// Usage with JSON configuration
var jsonConfig = @"{
    'Name': 'BasicWarrior',
    'Values': [
        {'Key': 'Health', 'Value': '100', 'Type': 'float'},
        {'Key': 'Damage', 'Value': '25', 'Type': 'int'},
        {'Key': 'Position', 'Value': '0,0,0', 'Type': 'vector3'}
    ],
    'Tags': ['Warrior', 'Melee', 'Heavy'],
    'Behaviors': ['MeleeCombat', 'HealthRegeneration'],
    'PostInstallationScript': 'InitializeCombatSystem'
}";

var config = JsonUtility.FromJson<EntityConfiguration>(jsonConfig);
var configInstaller = new ConfigurationInstaller(config);
configInstaller.Install(warriorEntity);
```

### Factory Integration Pattern

```csharp
public class EntityFactory
{
    private readonly Dictionary<string, List<IEntityInstaller>> _installersRegistry = new();

    public void RegisterInstaller(string entityType, IEntityInstaller installer)
    {
        if (!_installersRegistry.ContainsKey(entityType))
        {
            _installersRegistry[entityType] = new List<IEntityInstaller>();
        }
        
        _installersRegistry[entityType].Add(installer);
    }

    public IEntity CreateEntity(string entityType, string entityName)
    {
        var entity = new Entity(entityName);
        
        if (_installersRegistry.TryGetValue(entityType, out var installers))
        {
            foreach (var installer in installers)
            {
                installer.Install(entity);
            }
        }
        
        return entity;
    }
}

// Usage
var factory = new EntityFactory();

// Register installers for different entity types
factory.RegisterInstaller("Player", new BasicStatsInstaller(100f, 50f, 75f));
factory.RegisterInstaller("Player", new RoleInstaller(EntityRole.Player));
factory.RegisterInstaller("Player", new InventoryInstaller(startingItems));

factory.RegisterInstaller("Enemy", new BasicStatsInstaller(80f, 0f, 60f));
factory.RegisterInstaller("Enemy", new RoleInstaller(EntityRole.Enemy));
factory.RegisterInstaller("Enemy", new AIConfigurationInstaller(enemyAI));

// Create entities with automatic installation
var player = factory.CreateEntity("Player", "Hero");
var enemy = factory.CreateEntity("Enemy", "Goblin");
```

## Integration with Atomic Framework

### Installation Pipeline System

```csharp
public class EntityInstallationPipeline : MonoBehaviour
{
    [System.Serializable]
    public struct InstallationStage
    {
        public string stageName;
        public int priority;
        public UnityEvent<IEntity> onStageComplete;
    }

    [SerializeField] private InstallationStage[] _stages;
    
    private readonly Dictionary<string, List<IEntityInstaller>> _stageInstallers = new();
    private readonly Dictionary<int, string> _stagePriorities = new();

    void Start()
    {
        InitializeStages();
    }

    private void InitializeStages()
    {
        foreach (var stage in _stages)
        {
            _stageInstallers[stage.stageName] = new List<IEntityInstaller>();
            _stagePriorities[stage.priority] = stage.stageName;
        }
    }

    public void RegisterInstaller(string stageName, IEntityInstaller installer)
    {
        if (_stageInstallers.ContainsKey(stageName))
        {
            _stageInstallers[stageName].Add(installer);
        }
        else
        {
            Debug.LogWarning($"Stage '{stageName}' not found in installation pipeline");
        }
    }

    public void InstallEntity(IEntity entity)
    {
        Debug.Log($"Starting installation pipeline for entity: {entity.Name}");
        
        // Execute stages in priority order
        var sortedPriorities = _stagePriorities.Keys.OrderBy(p => p);
        
        foreach (var priority in sortedPriorities)
        {
            string stageName = _stagePriorities[priority];
            ExecuteStage(entity, stageName);
        }
        
        Debug.Log($"Installation pipeline completed for entity: {entity.Name}");
    }

    private void ExecuteStage(IEntity entity, string stageName)
    {
        Debug.Log($"Executing stage '{stageName}' for entity: {entity.Name}");
        
        if (_stageInstallers.TryGetValue(stageName, out var installers))
        {
            foreach (var installer in installers)
            {
                try
                {
                    installer.Install(entity);
                }
                catch (Exception ex)
                {
                    Debug.LogError($"Error in stage '{stageName}', installer {installer.GetType().Name}: {ex.Message}");
                }
            }
            
            // Trigger stage completion event
            var stage = _stages.First(s => s.stageName == stageName);
            stage.onStageComplete?.Invoke(entity);
        }
    }

    public void RegisterStandardInstallers()
    {
        // Register common installers to appropriate stages
        RegisterInstaller("Foundation", new RoleInstaller(EntityRole.Player));
        RegisterInstaller("Stats", new BasicStatsInstaller(100f, 50f, 75f));
        RegisterInstaller("Equipment", new EquipmentInstaller(defaultEquipment));
        RegisterInstaller("Finalization", new ValidationInstaller());
    }
}

public class ValidationInstaller : IEntityInstaller
{
    public void Install(IEntity entity)
    {
        bool isValid = ValidateEntity(entity);
        entity.SetTagState("Valid", isValid);
        
        if (!isValid)
        {
            Debug.LogWarning($"Entity {entity.Name} failed validation");
        }
    }

    private bool ValidateEntity(IEntity entity)
    {
        // Validation logic
        return entity.HasValue("Health") && entity.HasValue("MaxHealth");
    }
}
```

## Implementation Notes

### Interface Design
- Simple, focused interface with single installation method
- Generic version provides type safety without casting
- Explicit interface implementation handles type conversion

### Installation Patterns
- Modular design supports composition and reuse
- Error handling preserves system stability
- Flexible configuration through various installer types

### Resource Management
- Installers should handle resource creation responsibly
- Consider memory allocation impact during installation
- Support for cleanup or uninstallation if needed

## Best Practices

### Installer Design
- Keep installation logic focused and atomic
- Avoid dependencies between installers when possible
- Implement proper error handling and logging

### Configuration Management
- Use external configuration files for complex setups
- Support hot-reloading of configuration data
- Validate configuration before installation

### Performance Considerations
- Minimize allocations during installation
- Consider pooling for frequently created installers
- Batch installations when processing multiple entities

## Common Patterns

### Conditional Installation Pattern

```csharp
public class ConditionalInstaller : IEntityInstaller
{
    private readonly Func<IEntity, bool> _condition;
    private readonly IEntityInstaller _installer;

    public ConditionalInstaller(Func<IEntity, bool> condition, IEntityInstaller installer)
    {
        _condition = condition ?? throw new ArgumentNullException(nameof(condition));
        _installer = installer ?? throw new ArgumentNullException(nameof(installer));
    }

    public void Install(IEntity entity)
    {
        if (_condition(entity))
        {
            _installer.Install(entity);
        }
    }
}
```

### Template-Based Installation

```csharp
public class TemplateInstaller : IEntityInstaller
{
    private readonly EntityTemplate _template;

    public TemplateInstaller(EntityTemplate template)
    {
        _template = template ?? throw new ArgumentNullException(nameof(template));
    }

    public void Install(IEntity entity)
    {
        ApplyTemplate(entity, _template);
    }

    private void ApplyTemplate(IEntity entity, EntityTemplate template)
    {
        // Apply template data to entity
        foreach (var property in template.Properties)
        {
            entity.Set(property.Key, property.Value);
        }
        
        foreach (var tag in template.Tags)
        {
            entity.AddTag(tag);
        }
    }
}
```

The `IEntityInstaller` interface provides a clean, modular foundation for building sophisticated entity configuration and initialization systems, enabling reusable, composable entity setup logic within the Atomic framework.