# 🧩 ScriptableEntityInstaller

A Unity ScriptableObject-based installer that defines reusable entity configuration logic as data assets. Enables shared, persistent entity configuration that can be applied across multiple entities and scenes.

## Overview

`ScriptableEntityInstaller` allows developers to create reusable entity configuration assets using Unity's ScriptableObject system. Perfect for creating shared entity templates, configuration presets, and modular setup logic that can be easily managed, versioned, and distributed as assets.

## Class Definition

```csharp
#if UNITY_5_3_OR_NEWER
namespace Atomic.Entities
{
    /// <summary>
    /// A Unity ScriptableObject that defines reusable logic for installing or configuring an IEntity.
    /// </summary>
    public abstract class ScriptableEntityInstaller : ScriptableObject, IEntityInstaller
    {
        /// <summary>
        /// Applies configuration or data to the given entity.
        /// </summary>
        /// <param name="entity">The entity to configure or initialize.</param>
        public abstract void Install(IEntity entity);
    }

    /// <summary>
    /// A strongly-typed version of ScriptableEntityInstaller for installing entities of type T.
    /// </summary>
    /// <typeparam name="T">The specific entity type this installer supports.</typeparam>
    public abstract class ScriptableEntityInstaller<T> : ScriptableEntityInstaller where T : class, IEntity
    {
        /// <inheritdoc />
        public sealed override void Install(IEntity entity) => this.Install((T) entity);

        /// <summary>
        /// Applies configuration to a strongly-typed entity instance.
        /// </summary>
        /// <param name="entity">The entity to install.</param>
        protected abstract void Install(T entity);
    }
}
#endif
```

## Key Features

### Asset-Based Configuration
- Persistent ScriptableObject assets for reusable configuration
- Version control friendly entity templates
- Easy sharing and distribution of entity configurations

### Reusability
- Single installer can be applied to multiple entities
- Modular configuration system through asset composition
- Runtime and edit-time configuration support

### Unity Integration
- Native ScriptableObject integration with Unity's asset system
- Inspector-based configuration with immediate serialization
- Asset reference system for complex configuration hierarchies

## Usage Examples

### Basic Character Template Installer

```csharp
using UnityEngine;
using Atomic.Entities;

[CreateAssetMenu(fileName = "CharacterTemplate", menuName = "Entity/Installers/Character Template")]
public class CharacterTemplateInstaller : ScriptableEntityInstaller
{
    [Header("Base Stats")]
    [SerializeField] private float _health = 100f;
    [SerializeField] private float _mana = 50f;
    [SerializeField] private float _stamina = 75f;
    
    [Header("Combat Stats")]
    [SerializeField] private float _attack = 10f;
    [SerializeField] private float _defense = 5f;
    [SerializeField] private float _attackSpeed = 1f;
    
    [Header("Movement")]
    [SerializeField] private float _movementSpeed = 5f;
    [SerializeField] private float _jumpHeight = 2f;
    
    [Header("Character Tags")]
    [SerializeField] private string[] _tags = { "Character", "Living", "Combatant" };
    
    [Header("Visual Settings")]
    [SerializeField] private Color _characterColor = Color.white;
    [SerializeField] private float _characterScale = 1f;

    public override void Install(IEntity entity)
    {
        // Install base stats
        entity.Set("Health", _health);
        entity.Set("MaxHealth", _health);
        entity.Set("Mana", _mana);
        entity.Set("MaxMana", _mana);
        entity.Set("Stamina", _stamina);
        entity.Set("MaxStamina", _stamina);

        // Install combat stats
        entity.Set("Attack", _attack);
        entity.Set("Defense", _defense);
        entity.Set("AttackSpeed", _attackSpeed);

        // Install movement stats
        entity.Set("MovementSpeed", _movementSpeed);
        entity.Set("JumpHeight", _jumpHeight);

        // Install visual settings
        entity.Set("CharacterColor", _characterColor);
        entity.Set("CharacterScale", _characterScale);

        // Install tags
        foreach (string tag in _tags)
        {
            if (!string.IsNullOrEmpty(tag))
            {
                entity.AddTag(tag.Trim());
            }
        }

        Debug.Log($"Character template '{name}' installed for entity: {entity.Name}");
    }
}
```

### Equipment Set Installer

```csharp
using UnityEngine;
using Atomic.Entities;
using System.Collections.Generic;

[System.Serializable]
public class EquipmentItem
{
    public string slotName;
    public string itemName;
    public int itemId;
    public Sprite itemIcon;
    
    [Header("Stats")]
    public int attackBonus;
    public int defenseBonus;
    public int healthBonus;
    public int manaBonus;
    
    [Header("Properties")]
    public int durability = 100;
    public float weight = 1f;
    public int value = 100;
    public string rarity = "Common";
}

[CreateAssetMenu(fileName = "EquipmentSet", menuName = "Entity/Installers/Equipment Set")]
public class EquipmentSetInstaller : ScriptableEntityInstaller
{
    [Header("Equipment Set Configuration")]
    [SerializeField] private string _setName;
    [SerializeField] private string _setDescription;
    [SerializeField] private EquipmentItem[] _equipment;
    
    [Header("Set Bonuses")]
    [SerializeField] private int _setBonusThreshold = 3;
    [SerializeField] private int _setBonusAttack = 5;
    [SerializeField] private int _setBonusDefense = 5;
    [SerializeField] private float _setBonusHealthRegen = 1f;

    public override void Install(IEntity entity)
    {
        var installedItems = new List<string>();
        int totalAttackBonus = 0;
        int totalDefenseBonus = 0;
        int totalHealthBonus = 0;
        int totalManaBonus = 0;

        // Install individual equipment items
        foreach (var item in _equipment)
        {
            InstallEquipmentItem(entity, item);
            installedItems.Add(item.slotName);

            // Accumulate bonuses
            totalAttackBonus += item.attackBonus;
            totalDefenseBonus += item.defenseBonus;
            totalHealthBonus += item.healthBonus;
            totalManaBonus += item.manaBonus;
        }

        // Apply accumulated stat bonuses
        ApplyStatBonuses(entity, totalAttackBonus, totalDefenseBonus, totalHealthBonus, totalManaBonus);

        // Check and apply set bonuses
        if (installedItems.Count >= _setBonusThreshold)
        {
            ApplySetBonus(entity);
            entity.AddTag($"SetBonus_{_setName}");
        }

        // Store set information
        entity.Set("EquipmentSet", _setName);
        entity.Set("EquippedSlots", installedItems);

        Debug.Log($"Equipment set '{_setName}' installed for entity: {entity.Name} ({installedItems.Count} items)");
    }

    private void InstallEquipmentItem(IEntity entity, EquipmentItem item)
    {
        var itemData = new Dictionary<string, object>
        {
            ["Name"] = item.itemName,
            ["ItemId"] = item.itemId,
            ["Icon"] = item.itemIcon,
            ["AttackBonus"] = item.attackBonus,
            ["DefenseBonus"] = item.defenseBonus,
            ["HealthBonus"] = item.healthBonus,
            ["ManaBonus"] = item.manaBonus,
            ["Durability"] = item.durability,
            ["MaxDurability"] = item.durability,
            ["Weight"] = item.weight,
            ["Value"] = item.value,
            ["Rarity"] = item.rarity,
            ["SetName"] = _setName
        };

        entity.Set(item.slotName, itemData);
    }

    private void ApplyStatBonuses(IEntity entity, int attack, int defense, int health, int mana)
    {
        if (attack != 0)
        {
            float currentAttack = entity.HasValue("Attack") ? entity.Get<float>("Attack") : 0f;
            entity.Set("Attack", currentAttack + attack);
        }

        if (defense != 0)
        {
            float currentDefense = entity.HasValue("Defense") ? entity.Get<float>("Defense") : 0f;
            entity.Set("Defense", currentDefense + defense);
        }

        if (health != 0)
        {
            float currentMaxHealth = entity.HasValue("MaxHealth") ? entity.Get<float>("MaxHealth") : 100f;
            float newMaxHealth = currentMaxHealth + health;
            entity.Set("MaxHealth", newMaxHealth);
            
            // Scale current health proportionally
            if (entity.HasValue("Health"))
            {
                float currentHealth = entity.Get<float>("Health");
                float healthRatio = currentHealth / currentMaxHealth;
                entity.Set("Health", newMaxHealth * healthRatio);
            }
        }

        if (mana != 0)
        {
            float currentMaxMana = entity.HasValue("MaxMana") ? entity.Get<float>("MaxMana") : 50f;
            entity.Set("MaxMana", currentMaxMana + mana);
        }
    }

    private void ApplySetBonus(IEntity entity)
    {
        Debug.Log($"Applying set bonus for '{_setName}' to entity: {entity.Name}");

        // Apply set bonuses
        if (_setBonusAttack > 0)
        {
            float currentAttack = entity.Get<float>("Attack");
            entity.Set("Attack", currentAttack + _setBonusAttack);
        }

        if (_setBonusDefense > 0)
        {
            float currentDefense = entity.Get<float>("Defense");
            entity.Set("Defense", currentDefense + _setBonusDefense);
        }

        if (_setBonusHealthRegen > 0)
        {
            entity.Set("HealthRegenRate", _setBonusHealthRegen);
            entity.AddTag("HealthRegeneration");
        }

        // Store set bonus information
        entity.Set("SetBonusActive", true);
        entity.Set("SetBonusName", _setName);
    }
}
```

### Skill Configuration Installer

```csharp
using UnityEngine;
using Atomic.Entities;
using System.Collections.Generic;

[System.Serializable]
public class SkillData
{
    public string skillName;
    public string skillDescription;
    public int skillLevel = 1;
    public int maxSkillLevel = 10;
    public float cooldownTime = 1f;
    public float manaCost = 10f;
    public float damage = 0f;
    public float healing = 0f;
    public float duration = 0f;
    public string[] skillTags;
}

[CreateAssetMenu(fileName = "SkillSet", menuName = "Entity/Installers/Skill Set")]
public class SkillSetInstaller : ScriptableEntityInstaller
{
    [Header("Skill Set Configuration")]
    [SerializeField] private string _skillSetName;
    [SerializeField] private SkillData[] _skills;
    
    [Header("Skill System Settings")]
    [SerializeField] private int _maxActiveSkills = 4;
    [SerializeField] private bool _autoAssignToSlots = true;
    
    [Header("Learning Requirements")]
    [SerializeField] private int _minimumLevel = 1;
    [SerializeField] private string[] _prerequisiteTags;

    public override void Install(IEntity entity)
    {
        // Check prerequisites
        if (!CheckPrerequisites(entity))
        {
            Debug.LogWarning($"Entity {entity.Name} does not meet prerequisites for skill set '{_skillSetName}'");
            return;
        }

        var learnedSkills = new List<string>();
        var skillSlots = new Dictionary<int, string>();

        // Install skills
        foreach (var skill in _skills)
        {
            InstallSkill(entity, skill);
            learnedSkills.Add(skill.skillName);
        }

        // Auto-assign to skill slots if enabled
        if (_autoAssignToSlots)
        {
            AutoAssignSkillSlots(entity, learnedSkills, skillSlots);
        }

        // Store skill set information
        entity.Set("SkillSet", _skillSetName);
        entity.Set("LearnedSkills", learnedSkills);
        entity.Set("SkillSlots", skillSlots);
        entity.Set("MaxActiveSkills", _maxActiveSkills);

        // Add skill set tags
        entity.AddTag("HasSkills");
        entity.AddTag($"SkillSet_{_skillSetName}");

        Debug.Log($"Skill set '{_skillSetName}' installed for entity: {entity.Name} ({learnedSkills.Count} skills)");
    }

    private bool CheckPrerequisites(IEntity entity)
    {
        // Check level requirement
        if (entity.HasValue("Level"))
        {
            int entityLevel = entity.Get<int>("Level");
            if (entityLevel < _minimumLevel)
                return false;
        }

        // Check prerequisite tags
        foreach (string tag in _prerequisiteTags)
        {
            if (!entity.HasTag(tag))
                return false;
        }

        return true;
    }

    private void InstallSkill(IEntity entity, SkillData skill)
    {
        var skillInfo = new Dictionary<string, object>
        {
            ["Name"] = skill.skillName,
            ["Description"] = skill.skillDescription,
            ["Level"] = skill.skillLevel,
            ["MaxLevel"] = skill.maxSkillLevel,
            ["CooldownTime"] = skill.cooldownTime,
            ["ManaCost"] = skill.manaCost,
            ["Damage"] = skill.damage,
            ["Healing"] = skill.healing,
            ["Duration"] = skill.duration,
            ["LastUsedTime"] = 0f,
            ["TimesUsed"] = 0,
            ["IsOnCooldown"] = false
        };

        entity.Set($"Skill_{skill.skillName}", skillInfo);

        // Add skill-specific tags
        if (skill.skillTags != null)
        {
            foreach (string tag in skill.skillTags)
            {
                entity.AddTag($"Skill_{tag}");
            }
        }

        // Add general skill type tags
        if (skill.damage > 0)
            entity.AddTag("HasOffensiveSkills");
        
        if (skill.healing > 0)
            entity.AddTag("HasHealingSkills");
            
        if (skill.duration > 0)
            entity.AddTag("HasBuffSkills");
    }

    private void AutoAssignSkillSlots(IEntity entity, List<string> skills, Dictionary<int, string> skillSlots)
    {
        int slotIndex = 0;
        foreach (string skillName in skills)
        {
            if (slotIndex >= _maxActiveSkills)
                break;

            skillSlots[slotIndex] = skillName;
            slotIndex++;
        }
    }
}
```

### AI Behavior Tree Installer

```csharp
using UnityEngine;
using Atomic.Entities;
using System.Collections.Generic;

[System.Serializable]
public class BehaviorNode
{
    public string nodeName;
    public string nodeType;
    public Dictionary<string, object> nodeParameters = new Dictionary<string, object>();
    public BehaviorNode[] childNodes;
}

[CreateAssetMenu(fileName = "AIBehaviorTree", menuName = "Entity/Installers/AI Behavior Tree")]
public class AIBehaviorTreeInstaller : ScriptableEntityInstaller
{
    [Header("Behavior Tree Configuration")]
    [SerializeField] private string _behaviorTreeName;
    [SerializeField] private TextAsset _behaviorTreeJson;
    
    [Header("AI Settings")]
    [SerializeField] private float _tickRate = 10f;
    [SerializeField] private bool _debugMode = false;
    [SerializeField] private bool _visualizeTree = false;

    private BehaviorNode _rootNode;

    void OnEnable()
    {
        LoadBehaviorTree();
    }

    public override void Install(IEntity entity)
    {
        if (_rootNode == null)
        {
            Debug.LogError($"No behavior tree loaded for installer '{name}'");
            return;
        }

        // Install behavior tree structure
        entity.Set("BehaviorTree", _rootNode);
        entity.Set("BehaviorTreeName", _behaviorTreeName);
        entity.Set("TickRate", _tickRate);
        entity.Set("DebugMode", _debugMode);
        entity.Set("VisualizeTree", _visualizeTree);

        // Install AI execution state
        entity.Set("CurrentNode", _rootNode);
        entity.Set("NodeStack", new Stack<BehaviorNode>());
        entity.Set("ExecutionState", "Running");
        entity.Set("LastTickTime", 0f);

        // Install blackboard for AI memory
        entity.Set("AIBlackboard", new Dictionary<string, object>());

        // Add AI tags
        entity.AddTag("HasAI");
        entity.AddTag("BehaviorTree");
        entity.AddTag($"BT_{_behaviorTreeName}");

        if (_debugMode)
            entity.AddTag("DebugAI");

        Debug.Log($"Behavior tree '{_behaviorTreeName}' installed for entity: {entity.Name}");
    }

    private void LoadBehaviorTree()
    {
        if (_behaviorTreeJson != null)
        {
            try
            {
                string json = _behaviorTreeJson.text;
                _rootNode = JsonUtility.FromJson<BehaviorNode>(json);
            }
            catch (System.Exception ex)
            {
                Debug.LogError($"Failed to load behavior tree from JSON: {ex.Message}");
            }
        }
    }

#if UNITY_EDITOR
    [UnityEditor.MenuItem("Assets/Create/Entity/Generate Behavior Tree Template")]
    public static void CreateBehaviorTreeTemplate()
    {
        var template = new BehaviorNode
        {
            nodeName = "Root",
            nodeType = "Sequence",
            childNodes = new BehaviorNode[]
            {
                new BehaviorNode
                {
                    nodeName = "CheckForTarget",
                    nodeType = "Condition",
                    nodeParameters = new Dictionary<string, object> { ["range"] = 10f }
                },
                new BehaviorNode
                {
                    nodeName = "MoveToTarget",
                    nodeType = "Action",
                    nodeParameters = new Dictionary<string, object> { ["speed"] = 5f }
                }
            }
        };

        string json = JsonUtility.ToJson(template, true);
        string path = UnityEditor.EditorUtility.SaveFilePanel("Save Behavior Tree Template", "Assets", "BehaviorTreeTemplate", "json");
        
        if (!string.IsNullOrEmpty(path))
        {
            System.IO.File.WriteAllText(path, json);
            UnityEditor.AssetDatabase.Refresh();
        }
    }
#endif
}
```

### Configuration Preset System

```csharp
using UnityEngine;
using Atomic.Entities;
using System.Collections.Generic;

[System.Serializable]
public class ConfigurationPreset
{
    public string presetName;
    public string description;
    public ScriptableEntityInstaller[] installers;
    public bool enabled = true;
}

[CreateAssetMenu(fileName = "EntityPresetCollection", menuName = "Entity/Installers/Preset Collection")]
public class EntityPresetCollection : ScriptableEntityInstaller
{
    [Header("Preset Collection")]
    [SerializeField] private string _collectionName;
    [SerializeField] private ConfigurationPreset[] _presets;
    
    [Header("Installation Options")]
    [SerializeField] private bool _installAllPresets = true;
    [SerializeField] private string[] _selectedPresets;
    [SerializeField] private bool _ignoreDisabledPresets = true;

    public override void Install(IEntity entity)
    {
        var installedPresets = new List<string>();
        var appliedInstallers = new List<ScriptableEntityInstaller>();

        foreach (var preset in _presets)
        {
            if (!preset.enabled && _ignoreDisabledPresets)
                continue;

            if (!_installAllPresets && !System.Array.Exists(_selectedPresets, p => p == preset.presetName))
                continue;

            Debug.Log($"Installing preset '{preset.presetName}' for entity: {entity.Name}");

            foreach (var installer in preset.installers)
            {
                if (installer != null)
                {
                    try
                    {
                        installer.Install(entity);
                        appliedInstallers.Add(installer);
                    }
                    catch (System.Exception ex)
                    {
                        Debug.LogError($"Error installing '{installer.name}' from preset '{preset.presetName}': {ex.Message}");
                    }
                }
            }

            installedPresets.Add(preset.presetName);
        }

        // Store installation metadata
        entity.Set("PresetCollection", _collectionName);
        entity.Set("InstalledPresets", installedPresets);
        entity.Set("AppliedInstallers", appliedInstallers.ConvertAll(i => i.name));

        // Add collection tags
        entity.AddTag("ConfiguredByPreset");
        entity.AddTag($"Preset_{_collectionName}");

        Debug.Log($"Preset collection '{_collectionName}' installed for entity: {entity.Name} ({installedPresets.Count} presets)");
    }

    public void InstallSpecificPreset(IEntity entity, string presetName)
    {
        var preset = System.Array.Find(_presets, p => p.presetName == presetName);
        if (preset != null)
        {
            foreach (var installer in preset.installers)
            {
                installer?.Install(entity);
            }
        }
        else
        {
            Debug.LogWarning($"Preset '{presetName}' not found in collection '{_collectionName}'");
        }
    }

    public string[] GetAvailablePresets()
    {
        var presetNames = new List<string>();
        foreach (var preset in _presets)
        {
            if (preset.enabled || !_ignoreDisabledPresets)
            {
                presetNames.Add(preset.presetName);
            }
        }
        return presetNames.ToArray();
    }

    public ConfigurationPreset GetPreset(string presetName)
    {
        return System.Array.Find(_presets, p => p.presetName == presetName);
    }
}
```

## Integration with Unity Workflow

### Asset Creation and Management

```csharp
#if UNITY_EDITOR
using UnityEditor;
using UnityEngine;

public static class EntityInstallerUtility
{
    [MenuItem("Assets/Create/Entity/Character Template")]
    public static void CreateCharacterTemplate()
    {
        CreateAssetOfType<CharacterTemplateInstaller>("CharacterTemplate");
    }

    [MenuItem("Assets/Create/Entity/Equipment Set")]
    public static void CreateEquipmentSet()
    {
        CreateAssetOfType<EquipmentSetInstaller>("EquipmentSet");
    }

    [MenuItem("Assets/Create/Entity/Skill Set")]
    public static void CreateSkillSet()
    {
        CreateAssetOfType<SkillSetInstaller>("SkillSet");
    }

    private static void CreateAssetOfType<T>(string defaultName) where T : ScriptableObject
    {
        T asset = ScriptableObject.CreateInstance<T>();
        string path = AssetDatabase.GetAssetPath(Selection.activeObject);
        
        if (string.IsNullOrEmpty(path))
        {
            path = "Assets";
        }
        else if (Path.GetExtension(path) != "")
        {
            path = path.Replace(Path.GetFileName(AssetDatabase.GetAssetPath(Selection.activeObject)), "");
        }

        string assetPathAndName = AssetDatabase.GenerateUniqueAssetPath($"{path}/{defaultName}.asset");
        AssetDatabase.CreateAsset(asset, assetPathAndName);
        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();
        EditorUtility.FocusProjectWindow();
        Selection.activeObject = asset;
    }

    [MenuItem("Entity/Validate All Installers")]
    public static void ValidateAllInstallers()
    {
        var installers = AssetDatabase.FindAssets("t:ScriptableEntityInstaller");
        int validCount = 0;
        int errorCount = 0;

        foreach (var guid in installers)
        {
            var path = AssetDatabase.GUIDToAssetPath(guid);
            var installer = AssetDatabase.LoadAssetAtPath<ScriptableEntityInstaller>(path);
            
            if (ValidateInstaller(installer))
            {
                validCount++;
            }
            else
            {
                errorCount++;
                Debug.LogError($"Validation failed for installer: {installer.name}", installer);
            }
        }

        Debug.Log($"Installer validation complete: {validCount} valid, {errorCount} errors");
    }

    private static bool ValidateInstaller(ScriptableEntityInstaller installer)
    {
        // Implement validation logic
        return installer != null;
    }
}
#endif
```

## Implementation Notes

### ScriptableObject Benefits
- Persistent asset-based configuration storage
- Inspector-based configuration with serialization
- Reusable across multiple entities and scenes

### Type Safety
- Generic version provides compile-time type checking
- Sealed override ensures consistent behavior
- Protected abstract method focuses implementation

### Asset Management
- Native Unity asset system integration
- Version control friendly configuration
- Easy sharing and distribution of entity templates

## Best Practices

### Asset Organization
- Use consistent naming conventions for installer assets
- Organize installers in logical folder structures
- Document complex configurations with asset descriptions

### Configuration Design
- Keep installers focused on specific configuration aspects
- Use composition over inheritance for complex setups
- Provide sensible default values for all configuration fields

### Performance Considerations
- Minimize allocations during installation process
- Cache expensive calculations in installer assets
- Consider impact of large numbers of asset references

## Common Patterns

### Installer Composition Pattern

```csharp
[CreateAssetMenu(fileName = "CompositeInstaller", menuName = "Entity/Installers/Composite")]
public class CompositeScriptableInstaller : ScriptableEntityInstaller
{
    [SerializeField] private ScriptableEntityInstaller[] _installers;
    
    public override void Install(IEntity entity)
    {
        foreach (var installer in _installers)
        {
            if (installer != null)
            {
                installer.Install(entity);
            }
        }
    }
}
```

### Conditional Installer Pattern

```csharp
[CreateAssetMenu(fileName = "ConditionalInstaller", menuName = "Entity/Installers/Conditional")]
public class ConditionalScriptableInstaller : ScriptableEntityInstaller
{
    [SerializeField] private string[] _requiredTags;
    [SerializeField] private string[] _forbiddenTags;
    [SerializeField] private ScriptableEntityInstaller _installer;
    
    public override void Install(IEntity entity)
    {
        bool canInstall = _requiredTags.All(tag => entity.HasTag(tag)) &&
                         !_forbiddenTags.Any(tag => entity.HasTag(tag));
                         
        if (canInstall && _installer != null)
        {
            _installer.Install(entity);
        }
    }
}
```

The `ScriptableEntityInstaller` provides a powerful asset-based approach to entity configuration, enabling reusable, persistent, and easily managed entity setup logic that integrates seamlessly with Unity's asset workflow within the Atomic framework.