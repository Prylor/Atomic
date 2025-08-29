# 🧩 ScriptableEntityFactory

An abstract Unity ScriptableObject-based factory for creating entities through Unity's asset system. Provides design-time entity configuration with editor integration, asset-based distribution, and runtime optimization through precompilation.

## Overview

`ScriptableEntityFactory` enables entity creation through Unity's ScriptableObject system, allowing entity templates to be created, configured, and distributed as assets. Supports visual entity configuration, asset-based workflows, and editor-time optimization for production-ready entity creation pipelines.

## Class Definition

```csharp
#if UNITY_5_3_OR_NEWER
namespace Atomic.Entities
{
    /// <summary>
    /// Abstract base for Unity ScriptableObject-based IEntity factories.
    /// </summary>
    public abstract class ScriptableEntityFactory : ScriptableEntityFactory<IEntity>, IEntityFactory
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
    /// Generic ScriptableObject factory for creating typed entities.
    /// </summary>
    /// <typeparam name="E">The type of entity to create</typeparam>
    public abstract class ScriptableEntityFactory<E> : ScriptableObject, IEntityFactory<E> where E : IEntity
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

### Unity Asset Integration
- ScriptableObject-based for Unity's asset management system
- Asset-based entity configuration and distribution
- Integration with Unity's serialization and Inspector system
- Addressable Asset System compatibility

### Design-Time Optimization
- Precompile method for editor-time entity metadata extraction
- Cached initialization counts for runtime performance
- Editor validation through OnValidate callbacks
- Odin Inspector support for enhanced editing experience

### Flexible Entity Configuration
- Abstract Install method for custom entity setup
- Serialized fields for entity initialization parameters
- Support for complex entity configuration workflows
- Asset-based entity template sharing

## Usage Examples

### Basic Scriptable Entity Factory

```csharp
[CreateAssetMenu(fileName = "PlayerFactory", menuName = "Game/Entities/Player")]
public class PlayerEntityFactory : ScriptableEntityFactory
{
    [Header("Player Configuration")]
    [SerializeField] private float _maxHealth = 100f;
    [SerializeField] private float _speed = 10f;
    [SerializeField] private int _level = 1;
    
    [Header("Starting Equipment")]
    [SerializeField] private WeaponData _startingWeapon;
    [SerializeField] private ArmorData _startingArmor;
    
    [Header("Abilities")]
    [SerializeField] private AbilityData[] _startingAbilities;
    
    protected override void Install(IEntity entity)
    {
        // Core player stats
        entity.Set("Health", _maxHealth);
        entity.Set("MaxHealth", _maxHealth);
        entity.Set("Speed", _speed);
        entity.Set("Level", _level);
        entity.Set("Experience", 0);
        entity.Set("ExperienceToNext", CalculateExperienceRequired(_level));
        
        // Equipment
        if (_startingWeapon != null)
        {
            entity.Set("WeaponId", _startingWeapon.id);
            entity.Set("WeaponDamage", _startingWeapon.damage);
            entity.Set("WeaponRange", _startingWeapon.range);
        }
        
        if (_startingArmor != null)
        {
            entity.Set("ArmorId", _startingArmor.id);
            entity.Set("Defense", _startingArmor.defense);
            entity.Set("ArmorWeight", _startingArmor.weight);
        }
        
        // Abilities
        for (int i = 0; i < _startingAbilities.Length; i++)
        {
            entity.Set($"Ability_{i}", _startingAbilities[i].id);
            entity.Set($"AbilityCooldown_{i}", _startingAbilities[i].cooldown);
        }
        
        // Tags
        entity.AddTag("Player");
        entity.AddTag("Controllable");
        entity.AddTag("Mortal");
        entity.AddTag($"Level{_level}");
        
        // Behaviors
        entity.AddBehaviour<PlayerMovementBehaviour>();
        entity.AddBehaviour<PlayerCombatBehaviour>();
        entity.AddBehaviour<PlayerInventoryBehaviour>();
    }
    
    private int CalculateExperienceRequired(int level)
    {
        return level * 100 + (level - 1) * 50;
    }
}
```

### Configurable Enemy Factory

```csharp
[CreateAssetMenu(fileName = "EnemyFactory", menuName = "Game/Entities/Enemy")]
public class EnemyEntityFactory : ScriptableEntityFactory
{
    [System.Serializable]
    public struct StatBlock
    {
        public float health;
        public float speed;
        public float damage;
        public float defense;
        public float attackRange;
        public float detectionRange;
    }
    
    [System.Serializable]
    public struct LootTable
    {
        public ItemData[] items;
        public float[] dropChances;
        public int minGold;
        public int maxGold;
        public int experienceReward;
    }
    
    [Header("Enemy Identity")]
    [SerializeField] private string _enemyName = "Enemy";
    [SerializeField] private EnemyType _enemyType = EnemyType.Basic;
    [SerializeField] private EnemyTier _enemyTier = EnemyTier.Common;
    
    [Header("Combat Stats")]
    [SerializeField] private StatBlock _baseStats;
    [SerializeField] private float _criticalChance = 0.05f;
    [SerializeField] private float _criticalMultiplier = 2.0f;
    
    [Header("AI Behavior")]
    [SerializeField] private AIBehaviorType _behaviorType = AIBehaviorType.Aggressive;
    [SerializeField] private float _aggressionLevel = 0.7f;
    [SerializeField] private bool _canCallForHelp = false;
    [SerializeField] private float _helpRadius = 10f;
    
    [Header("Loot Configuration")]
    [SerializeField] private LootTable _lootTable;
    
    [Header("Special Abilities")]
    [SerializeField] private EnemyAbilityData[] _specialAbilities;
    
    protected override void Install(IEntity entity)
    {
        // Apply tier multipliers to base stats
        float tierMultiplier = GetTierMultiplier(_enemyTier);
        
        // Core combat stats
        entity.Set("Name", _enemyName);
        entity.Set("Health", _baseStats.health * tierMultiplier);
        entity.Set("MaxHealth", _baseStats.health * tierMultiplier);
        entity.Set("Speed", _baseStats.speed);
        entity.Set("Damage", _baseStats.damage * tierMultiplier);
        entity.Set("Defense", _baseStats.defense * tierMultiplier);
        entity.Set("AttackRange", _baseStats.attackRange);
        entity.Set("DetectionRange", _baseStats.detectionRange);
        
        // Combat modifiers
        entity.Set("CriticalChance", _criticalChance);
        entity.Set("CriticalMultiplier", _criticalMultiplier);
        
        // AI configuration
        entity.Set("BehaviorType", _behaviorType.ToString());
        entity.Set("AggressionLevel", _aggressionLevel);
        entity.Set("CanCallForHelp", _canCallForHelp);
        entity.Set("HelpRadius", _helpRadius);
        
        // Loot configuration
        entity.Set("ExperienceReward", _lootTable.experienceReward);
        entity.Set("MinGoldDrop", _lootTable.minGold);
        entity.Set("MaxGoldDrop", _lootTable.maxGold);
        
        // Store loot table for runtime use
        for (int i = 0; i < _lootTable.items.Length; i++)
        {
            entity.Set($"LootItem_{i}", _lootTable.items[i].id);
            entity.Set($"LootChance_{i}", _lootTable.dropChances[i]);
        }
        
        // Special abilities
        for (int i = 0; i < _specialAbilities.Length; i++)
        {
            entity.Set($"SpecialAbility_{i}", _specialAbilities[i].id);
            entity.Set($"AbilityCooldown_{i}", _specialAbilities[i].cooldown);
            entity.Set($"AbilityDamage_{i}", _specialAbilities[i].damage);
        }
        
        // Tags
        entity.AddTag("Enemy");
        entity.AddTag("Hostile");
        entity.AddTag(_enemyType.ToString());
        entity.AddTag(_enemyTier.ToString());
        entity.AddTag(_behaviorType.ToString());
        
        // Behaviors
        entity.AddBehaviour<EnemyAIBehaviour>();
        entity.AddBehaviour<EnemyCombatBehaviour>();
        entity.AddBehaviour<EnemyLootBehaviour>();
        
        if (_canCallForHelp)
        {
            entity.AddBehaviour<EnemyHelpCallingBehaviour>();
        }
        
        foreach (var ability in _specialAbilities)
        {
            if (ability.behaviorType != null)
            {
                entity.AddBehaviour(ability.behaviorType);
            }
        }
    }
    
    private float GetTierMultiplier(EnemyTier tier)
    {
        return tier switch
        {
            EnemyTier.Common => 1.0f,
            EnemyTier.Uncommon => 1.3f,
            EnemyTier.Rare => 1.7f,
            EnemyTier.Epic => 2.2f,
            EnemyTier.Legendary => 3.0f,
            _ => 1.0f
        };
    }
}

public enum EnemyType { Basic, Elite, Boss, Minion }
public enum EnemyTier { Common, Uncommon, Rare, Epic, Legendary }
public enum AIBehaviorType { Passive, Defensive, Aggressive, Berserker }
```

### Item Factory with Randomization

```csharp
[CreateAssetMenu(fileName = "ItemFactory", menuName = "Game/Entities/Item")]
public class ItemEntityFactory : ScriptableEntityFactory
{
    [System.Serializable]
    public struct ItemRarity
    {
        public string name;
        public Color color;
        public float statMultiplier;
        public float dropChance;
        public int minEnchantments;
        public int maxEnchantments;
    }
    
    [System.Serializable]
    public struct StatModifier
    {
        public string statName;
        public float minValue;
        public float maxValue;
        public ModifierType type;
    }
    
    [Header("Item Configuration")]
    [SerializeField] private string _itemName = "Item";
    [SerializeField] private ItemType _itemType = ItemType.Weapon;
    [SerializeField] private Sprite _itemIcon;
    [SerializeField] private string _description;
    
    [Header("Base Stats")]
    [SerializeField] private StatModifier[] _baseStats;
    [SerializeField] private int _baseValue = 100;
    [SerializeField] private float _durabilityMultiplier = 1.0f;
    
    [Header("Rarity System")]
    [SerializeField] private ItemRarity[] _rarities;
    [SerializeField] private AnimationCurve _rarityDistribution;
    
    [Header("Enchantments")]
    [SerializeField] private EnchantmentData[] _possibleEnchantments;
    [SerializeField] private bool _allowRandomEnchantments = true;
    
    protected override void Install(IEntity entity)
    {
        // Select random rarity
        ItemRarity selectedRarity = SelectRandomRarity();
        
        // Basic item properties
        entity.Set("Name", _itemName);
        entity.Set("Description", _description);
        entity.Set("ItemType", _itemType.ToString());
        entity.Set("BaseValue", _baseValue);
        entity.Set("Icon", _itemIcon ? _itemIcon.name : "");
        
        // Rarity properties
        entity.Set("Rarity", selectedRarity.name);
        entity.Set("RarityColor", ColorUtility.ToHtmlStringRGB(selectedRarity.color));
        entity.Set("StatMultiplier", selectedRarity.statMultiplier);
        
        // Calculate final stats with rarity multiplier
        foreach (var statMod in _baseStats)
        {
            float randomValue = Random.Range(statMod.minValue, statMod.maxValue);
            float finalValue = randomValue * selectedRarity.statMultiplier;
            
            entity.Set($"Stat_{statMod.statName}", finalValue);
            entity.Set($"StatType_{statMod.statName}", statMod.type.ToString());
        }
        
        // Durability
        float baseDurability = 100f * _durabilityMultiplier;
        entity.Set("Durability", baseDurability * selectedRarity.statMultiplier);
        entity.Set("MaxDurability", baseDurability * selectedRarity.statMultiplier);
        
        // Apply enchantments
        if (_allowRandomEnchantments && _possibleEnchantments.Length > 0)
        {
            ApplyRandomEnchantments(entity, selectedRarity);
        }
        
        // Calculate final value
        float rarityValueMultiplier = 1f + (selectedRarity.statMultiplier - 1f) * 2f;
        entity.Set("Value", Mathf.RoundToInt(_baseValue * rarityValueMultiplier));
        
        // Tags
        entity.AddTag("Item");
        entity.AddTag(_itemType.ToString());
        entity.AddTag(selectedRarity.name);
        
        // Add rarity-specific tags
        if (selectedRarity.statMultiplier >= 2.0f)
        {
            entity.AddTag("Legendary");
        }
        else if (selectedRarity.statMultiplier >= 1.5f)
        {
            entity.AddTag("Rare");
        }
        
        // Behaviors
        entity.AddBehaviour<ItemBehaviour>();
        entity.AddBehaviour<DurabilityBehaviour>();
        
        if (_itemType == ItemType.Weapon)
        {
            entity.AddBehaviour<WeaponBehaviour>();
        }
        else if (_itemType == ItemType.Armor)
        {
            entity.AddBehaviour<ArmorBehaviour>();
        }
    }
    
    private ItemRarity SelectRandomRarity()
    {
        float randomValue = Random.value;
        float cumulativeChance = 0f;
        
        for (int i = 0; i < _rarities.Length; i++)
        {
            float distributionValue = _rarityDistribution.Evaluate((float)i / (_rarities.Length - 1));
            cumulativeChance += distributionValue;
            
            if (randomValue <= cumulativeChance)
            {
                return _rarities[i];
            }
        }
        
        return _rarities[_rarities.Length - 1];
    }
    
    private void ApplyRandomEnchantments(IEntity entity, ItemRarity rarity)
    {
        int enchantmentCount = Random.Range(rarity.minEnchantments, rarity.maxEnchantments + 1);
        var appliedEnchantments = new HashSet<EnchantmentData>();
        
        for (int i = 0; i < enchantmentCount; i++)
        {
            // Select random enchantment that hasn't been applied
            var availableEnchantments = _possibleEnchantments.Where(e => !appliedEnchantments.Contains(e)).ToArray();
            if (availableEnchantments.Length == 0) break;
            
            var enchantment = availableEnchantments[Random.Range(0, availableEnchantments.Length)];
            appliedEnchantments.Add(enchantment);
            
            // Apply enchantment
            entity.Set($"Enchantment_{i}_Id", enchantment.id);
            entity.Set($"Enchantment_{i}_Power", Random.Range(enchantment.minPower, enchantment.maxPower));
            entity.AddTag($"Enchanted_{enchantment.name}");
        }
        
        entity.Set("EnchantmentCount", appliedEnchantments.Count);
        
        if (appliedEnchantments.Count > 0)
        {
            entity.AddTag("Enchanted");
            entity.AddBehaviour<EnchantmentBehaviour>();
        }
    }
}

public enum ItemType { Weapon, Armor, Accessory, Consumable, Material, Quest }
public enum ModifierType { Flat, Percentage, Multiplier }
```

## Integration with Atomic Framework

### Reactive Factory Configuration

```csharp
[CreateAssetMenu(fileName = "ReactiveFactory", menuName = "Game/Entities/Reactive")]
public class ReactiveEntityFactory : ScriptableEntityFactory
{
    [Header("Reactive Configuration")]
    [SerializeField] private ReactiveFloat _globalDifficultyMultiplier = new(1f);
    [SerializeField] private ReactiveInt _playerLevel = new(1);
    [SerializeField] private ReactiveBool _hardModeEnabled = new(false);
    
    [Header("Base Configuration")]
    [SerializeField] private float _baseHealth = 100f;
    [SerializeField] private float _baseSpeed = 5f;
    [SerializeField] private float _baseDamage = 10f;
    
    void OnEnable()
    {
        // React to global changes
        _globalDifficultyMultiplier.Subscribe(OnDifficultyChanged);
        _playerLevel.Subscribe(OnPlayerLevelChanged);
        _hardModeEnabled.Subscribe(OnHardModeChanged);
    }
    
    protected override void Install(IEntity entity)
    {
        // Apply reactive modifiers
        float difficultyMod = _globalDifficultyMultiplier.Value;
        float levelMod = 1f + (_playerLevel.Value - 1) * 0.1f;
        float hardModeMod = _hardModeEnabled.Value ? 1.5f : 1f;
        
        float finalHealthMultiplier = difficultyMod * levelMod * hardModeMod;
        
        // Configure entity with reactive values
        entity.Set("Health", _baseHealth * finalHealthMultiplier);
        entity.Set("MaxHealth", _baseHealth * finalHealthMultiplier);
        entity.Set("Speed", _baseSpeed * difficultyMod);
        entity.Set("Damage", _baseDamage * finalHealthMultiplier);
        
        // Store reactive references for runtime updates
        entity.Set("DifficultyMultiplier", difficultyMod);
        entity.Set("PlayerLevel", _playerLevel.Value);
        entity.Set("HardMode", _hardModeEnabled.Value);
        
        // Tags based on reactive state
        entity.AddTag("Reactive");
        entity.AddTag($"Difficulty_{difficultyMod:F1}");
        entity.AddTag($"PlayerLevel_{_playerLevel.Value}");
        
        if (_hardModeEnabled.Value)
        {
            entity.AddTag("HardMode");
        }
        
        // Behaviors
        entity.AddBehaviour<ReactiveEntityBehaviour>();
    }
    
    private void OnDifficultyChanged(float newDifficulty)
    {
        Debug.Log($"Difficulty changed to: {newDifficulty}");
        // Update existing entities if needed
    }
    
    private void OnPlayerLevelChanged(int newLevel)
    {
        Debug.Log($"Player level changed to: {newLevel}");
        // Potentially adjust entity spawn rates
    }
    
    private void OnHardModeChanged(bool enabled)
    {
        Debug.Log($"Hard mode: {enabled}");
        // Apply global modifiers
    }
}
```

## Implementation Notes

### Asset Lifecycle
- ScriptableObject lifecycle managed by Unity's asset system
- OnValidate called when asset properties change in Inspector
- Precompile runs in Editor to cache entity metadata
- Reset method provides default values for new assets

### Serialization
- Unity's serialization system handles field persistence
- SerializeField attributes control Inspector visibility
- Complex data structures supported through nested serializable classes

### Editor Integration
- CreateAssetMenu attribute enables asset creation from context menu
- Inspector customization through property attributes
- Odin Inspector support for enhanced editing capabilities

## Best Practices

### Asset Organization
- Use clear, descriptive asset names that serve as entity identifiers
- Organize factories in logical folder structures by category
- Version control factory assets for team collaboration

### Configuration Design
- Group related properties in clear sections with headers
- Use meaningful default values for rapid prototyping
- Validate configuration data in OnValidate method

### Performance Optimization
- Run Precompile to cache entity metadata in Editor
- Avoid complex operations in Create method
- Consider asset loading strategies for large factory collections

## Common Patterns

### Factory Template System

```csharp
[CreateAssetMenu(fileName = "TemplateFactory", menuName = "Game/Templates/Entity")]
public class EntityTemplate : ScriptableEntityFactory
{
    [Header("Template Configuration")]
    [SerializeField] private EntityConfigurationData _configData;
    
    protected override void Install(IEntity entity)
    {
        _configData.ApplyToEntity(entity);
    }
}

[System.Serializable]
public class EntityConfigurationData
{
    public string entityName;
    public StatBlock stats;
    public string[] tags;
    public KeyValuePair<string, object>[] properties;
    
    public void ApplyToEntity(IEntity entity)
    {
        entity.Set("Name", entityName);
        // Apply stats, tags, and properties
    }
}
```

The `ScriptableEntityFactory` provides a powerful Unity-integrated approach to entity creation, enabling asset-based entity configuration and distribution within the Atomic framework's procedural architecture.