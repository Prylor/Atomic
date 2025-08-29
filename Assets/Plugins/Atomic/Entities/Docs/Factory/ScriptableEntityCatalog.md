# 🧩 ScriptableEntityCatalog

A ScriptableObject-based catalog for organizing and accessing collections of entity factories through Unity's asset system. Provides design-time factory registration with runtime lookup capabilities for organized entity creation workflows.

## Overview

`ScriptableEntityCatalog` bridges Unity's asset management system with the Atomic entity framework, enabling visual organization of entity factories in the Unity Editor. Supports drag-and-drop factory assignment, asset-based distribution, and efficient runtime factory access through dictionary semantics.

## Class Definition

```csharp
#if UNITY_5_3_OR_NEWER
namespace Atomic.Entities
{
    /// <summary>
    /// Concrete catalog mapping ScriptableEntityFactory instances by asset name.
    /// </summary>
    [CreateAssetMenu(
        fileName = "EntityFactoryCatalog",
        menuName = "Atomic/Entities/New EntityFactoryCatalog"
    )]
    public class ScriptableEntityCatalog :
        ScriptableEntityCatalog<string, IEntity, ScriptableEntityFactory>,
        IEntityFactoryCatalog
    {
        protected override string GetKey(ScriptableEntityFactory factory) => factory.name;
    }

    /// <summary>
    /// Generic ScriptableObject catalog for entity factory collections.
    /// </summary>
    /// <typeparam name="TKey">The type of key used to identify factories</typeparam>
    /// <typeparam name="E">The type of entity each factory creates</typeparam>
    /// <typeparam name="F">The type of entity factory</typeparam>
    public abstract class ScriptableEntityCatalog<TKey, E, F> :
        ScriptableObject,
        IEntityFactoryCatalog<TKey, E>
        where E : IEntity
        where F : ScriptableEntityFactory<E>
    {
        [SerializeField] internal F[] _factories;
        private Dictionary<TKey, IEntityFactory<E>> _map;

        public int Count { get; }
        public IEnumerable<TKey> Keys { get; }
        public IEnumerable<IEntityFactory<E>> Values { get; }
        
        public bool ContainsKey(TKey key);
        public bool TryGetValue(TKey key, out IEntityFactory<E> value);
        public IEntityFactory<E> this[TKey key] { get; }
        
        protected abstract TKey GetKey(F factory);
    }
}
#endif
```

## Key Features

### Unity Asset Integration
- ScriptableObject-based for Unity's asset management system
- CreateAssetMenu attribute for easy catalog creation
- Serialized factory arrays with Inspector integration
- Odin Inspector support for enhanced editing experience

### Design-Time Factory Organization
- Visual drag-and-drop factory assignment in Inspector
- Automatic key generation from factory asset names
- Validation and duplicate detection in Editor
- Asset-based factory distribution and sharing

### Runtime Efficiency
- Lazy initialization of internal dictionary mapping
- O(1) factory lookup through dictionary interface
- IReadOnlyDictionary semantics for familiar API
- Thread-safe initialization with null checks

## Usage Examples

### Basic Catalog Creation and Usage

```csharp
// Create catalog asset through Unity menu: Atomic/Entities/New EntityFactoryCatalog
public class GameEntityManager : MonoBehaviour
{
    [Header("Entity Catalogs")]
    [SerializeField] private ScriptableEntityCatalog _enemyCatalog;
    [SerializeField] private ScriptableEntityCatalog _itemCatalog;
    [SerializeField] private ScriptableEntityCatalog _effectCatalog;
    
    void Start()
    {
        InitializeGameEntities();
    }
    
    private void InitializeGameEntities()
    {
        // Access factories from catalogs
        SpawnEnemies();
        CreateItems();
        GenerateEffects();
    }
    
    private void SpawnEnemies()
    {
        // Create different enemy types from catalog
        if (_enemyCatalog.TryGetValue("BasicEnemy", out var basicEnemyFactory))
        {
            for (int i = 0; i < 5; i++)
            {
                IEntity enemy = basicEnemyFactory.Create();
                enemy.Set("Position", GetRandomSpawnPosition());
                enemy.Set("SpawnTime", Time.time);
            }
        }
        
        if (_enemyCatalog.TryGetValue("EliteEnemy", out var eliteEnemyFactory))
        {
            IEntity elite = eliteEnemyFactory.Create();
            elite.Set("Position", GetBossSpawnPosition());
            elite.Set("EliteBonus", 2.0f);
        }
        
        // List all available enemy types
        Debug.Log("Available enemy types:");
        foreach (string enemyType in _enemyCatalog.Keys)
        {
            Debug.Log($"- {enemyType}");
        }
    }
    
    private void CreateItems()
    {
        // Create items from item catalog
        foreach (var kvp in _itemCatalog)
        {
            string itemType = kvp.Key;
            IEntityFactory<IEntity> factory = kvp.Value;
            
            IEntity item = factory.Create();
            item.Set("ItemType", itemType);
            item.Set("SpawnLocation", GetRandomItemLocation());
        }
    }
    
    private void GenerateEffects()
    {
        // Generate random effects
        var effectKeys = _effectCatalog.Keys.ToArray();
        for (int i = 0; i < 3; i++)
        {
            string randomEffectType = effectKeys[Random.Range(0, effectKeys.Length)];
            IEntity effect = _effectCatalog[randomEffectType].Create();
            effect.Set("Intensity", Random.Range(0.5f, 2f));
        }
    }
    
    private Vector3 GetRandomSpawnPosition() => new Vector3(
        Random.Range(-10f, 10f), 0f, Random.Range(-10f, 10f)
    );
    
    private Vector3 GetBossSpawnPosition() => new Vector3(0f, 0f, 15f);
    private Vector3 GetRandomItemLocation() => GetRandomSpawnPosition();
}
```

### Custom Catalog Implementation

```csharp
// Custom catalog for weapon factories with enum-based keys
public enum WeaponType
{
    Sword,
    Bow,
    Staff,
    Dagger,
    Hammer
}

[CreateAssetMenu(
    fileName = "WeaponCatalog",
    menuName = "Game/Weapons/New WeaponCatalog"
)]
public class WeaponEntityCatalog : ScriptableEntityCatalog<WeaponType, IEntity, WeaponEntityFactory>
{
    protected override WeaponType GetKey(WeaponEntityFactory factory)
    {
        // Extract weapon type from factory name or configuration
        if (System.Enum.TryParse<WeaponType>(factory.WeaponTypeName, out WeaponType weaponType))
        {
            return weaponType;
        }
        
        // Fallback to parsing from asset name
        string factoryName = factory.name.Replace("Factory", "").Replace("Entity", "");
        return System.Enum.TryParse<WeaponType>(factoryName, out weaponType) 
            ? weaponType 
            : WeaponType.Sword;
    }
}

public class WeaponSystem : MonoBehaviour
{
    [SerializeField] private WeaponEntityCatalog _weaponCatalog;
    
    public IEntity CreateWeapon(WeaponType type)
    {
        if (_weaponCatalog.TryGetValue(type, out var factory))
        {
            IEntity weapon = factory.Create();
            weapon.Set("WeaponType", type);
            weapon.Set("CreatedTime", Time.time);
            return weapon;
        }
        
        Debug.LogWarning($"No factory found for weapon type: {type}");
        return null;
    }
    
    public List<IEntity> CreateRandomWeapons(int count)
    {
        var weapons = new List<IEntity>();
        var availableTypes = System.Enum.GetValues(typeof(WeaponType)).Cast<WeaponType>();
        
        for (int i = 0; i < count; i++)
        {
            WeaponType randomType = availableTypes.ElementAt(Random.Range(0, availableTypes.Count()));
            IEntity weapon = CreateWeapon(randomType);
            if (weapon != null)
            {
                weapons.Add(weapon);
            }
        }
        
        return weapons;
    }
}
```

### Hierarchical Catalog System

```csharp
[CreateAssetMenu(fileName = "EntityMasterCatalog", menuName = "Atomic/Entities/Master Catalog")]
public class EntityMasterCatalog : ScriptableObject
{
    [Header("Specialized Catalogs")]
    [SerializeField] private ScriptableEntityCatalog _characterCatalog;
    [SerializeField] private ScriptableEntityCatalog _weaponCatalog;
    [SerializeField] private ScriptableEntityCatalog _armorCatalog;
    [SerializeField] private ScriptableEntityCatalog _consumableCatalog;
    [SerializeField] private ScriptableEntityCatalog _environmentCatalog;
    
    public enum EntityCategory
    {
        Characters,
        Weapons,
        Armor,
        Consumables,
        Environment
    }
    
    public IEntityFactoryCatalog GetCatalog(EntityCategory category)
    {
        return category switch
        {
            EntityCategory.Characters => _characterCatalog,
            EntityCategory.Weapons => _weaponCatalog,
            EntityCategory.Armor => _armorCatalog,
            EntityCategory.Consumables => _consumableCatalog,
            EntityCategory.Environment => _environmentCatalog,
            _ => null
        };
    }
    
    public IEntity CreateEntity(EntityCategory category, string factoryKey)
    {
        var catalog = GetCatalog(category);
        if (catalog != null && catalog.TryGetValue(factoryKey, out var factory))
        {
            IEntity entity = factory.Create();
            entity.Set("Category", category.ToString());
            entity.Set("FactoryKey", factoryKey);
            return entity;
        }
        
        return null;
    }
    
    public Dictionary<string, int> GetCatalogSummary()
    {
        var summary = new Dictionary<string, int>();
        
        if (_characterCatalog != null) summary["Characters"] = _characterCatalog.Count;
        if (_weaponCatalog != null) summary["Weapons"] = _weaponCatalog.Count;
        if (_armorCatalog != null) summary["Armor"] = _armorCatalog.Count;
        if (_consumableCatalog != null) summary["Consumables"] = _consumableCatalog.Count;
        if (_environmentCatalog != null) summary["Environment"] = _environmentCatalog.Count;
        
        return summary;
    }
}
```

## Integration with Atomic Framework

### Reactive Catalog Selection

```csharp
public class ReactiveCatalogSystem : MonoBehaviour
{
    [SerializeField] private ScriptableEntityCatalog[] _availableCatalogs;
    private readonly ReactiveValue<int> _selectedCatalogIndex = new(0);
    private readonly ReactiveValue<ScriptableEntityCatalog> _currentCatalog = new();
    
    void Start()
    {
        // React to catalog selection changes
        _selectedCatalogIndex.Subscribe(OnCatalogIndexChanged);
        _currentCatalog.Subscribe(OnCurrentCatalogChanged);
        
        // Initialize with first catalog
        if (_availableCatalogs.Length > 0)
        {
            _selectedCatalogIndex.Value = 0;
        }
    }
    
    private void OnCatalogIndexChanged(int index)
    {
        if (index >= 0 && index < _availableCatalogs.Length)
        {
            _currentCatalog.Value = _availableCatalogs[index];
        }
    }
    
    private void OnCurrentCatalogChanged(ScriptableEntityCatalog catalog)
    {
        if (catalog != null)
        {
            Debug.Log($"Active catalog: {catalog.name} with {catalog.Count} factories");
            
            // Update UI or other systems
            UpdateAvailableFactories(catalog);
        }
    }
    
    private void UpdateAvailableFactories(ScriptableEntityCatalog catalog)
    {
        // Notify other systems about available factory types
        var factoryNames = catalog.Keys.ToList();
        GameEvents.OnFactoryCatalogChanged?.Invoke(factoryNames);
    }
    
    public void SelectCatalog(int index)
    {
        _selectedCatalogIndex.Value = index;
    }
    
    public IEntity CreateFromCurrentCatalog(string key)
    {
        var catalog = _currentCatalog.Value;
        if (catalog != null && catalog.TryGetValue(key, out var factory))
        {
            return factory.Create();
        }
        return null;
    }
}
```

### Event-Driven Catalog Usage

```csharp
public class EventDrivenEntityCreation : MonoBehaviour
{
    [System.Serializable]
    public struct CatalogMapping
    {
        public GameEventType eventType;
        public ScriptableEntityCatalog catalog;
        public string[] factoryKeys;
    }
    
    [SerializeField] private CatalogMapping[] _catalogMappings;
    
    void Start()
    {
        // Subscribe to game events
        GameEvents.OnPlayerLevelUp += () => HandleEvent(GameEventType.PlayerLevelUp);
        GameEvents.OnEnemyWaveStart += () => HandleEvent(GameEventType.EnemyWaveStart);
        GameEvents.OnBossDefeated += () => HandleEvent(GameEventType.BossDefeated);
        GameEvents.OnTreasureFound += () => HandleEvent(GameEventType.TreasureFound);
    }
    
    private void HandleEvent(GameEventType eventType)
    {
        var mapping = _catalogMappings.FirstOrDefault(m => m.eventType == eventType);
        if (mapping.catalog != null)
        {
            CreateEntitiesFromMapping(mapping);
        }
    }
    
    private void CreateEntitiesFromMapping(CatalogMapping mapping)
    {
        foreach (string factoryKey in mapping.factoryKeys)
        {
            if (mapping.catalog.TryGetValue(factoryKey, out var factory))
            {
                IEntity entity = factory.Create();
                entity.Set("TriggeredBy", mapping.eventType.ToString());
                entity.Set("CreatedAt", Time.time);
                
                // Apply event-specific modifications
                ApplyEventModifications(entity, mapping.eventType);
            }
        }
    }
    
    private void ApplyEventModifications(IEntity entity, GameEventType eventType)
    {
        switch (eventType)
        {
            case GameEventType.PlayerLevelUp:
                entity.Set("BonusMultiplier", 1.5f);
                entity.AddTag("LevelUpReward");
                break;
                
            case GameEventType.EnemyWaveStart:
                entity.Set("WaveNumber", GameState.CurrentWave);
                entity.Set("DifficultyScaling", 1f + (GameState.CurrentWave * 0.1f));
                break;
                
            case GameEventType.BossDefeated:
                entity.Set("RarityBonus", 2.0f);
                entity.AddTag("BossLoot");
                break;
                
            case GameEventType.TreasureFound:
                entity.Set("TreasureValue", Random.Range(100, 500));
                entity.AddTag("Treasure");
                break;
        }
    }
}

public enum GameEventType
{
    PlayerLevelUp,
    EnemyWaveStart,
    BossDefeated,
    TreasureFound
}
```

## Implementation Notes

### Asset Management
- ScriptableObject lifecycle managed by Unity's asset system
- Serialized factory array persists between sessions
- CreateAssetMenu enables easy catalog creation in Project window

### Dictionary Initialization
- Lazy initialization on first access to avoid startup overhead
- EnsureInitialized() called before any dictionary operations
- Null factory references are skipped during initialization

### Key Generation
- Abstract GetKey method allows customizable key extraction
- Default implementation uses factory asset name as key
- Supports validation and duplicate key detection

### Performance Considerations
- Dictionary provides O(1) lookup after initialization
- Initialization cost is amortized across multiple accesses
- Consider factory count when designing catalog hierarchies

## Best Practices

### Catalog Organization
- Group related factories in focused catalogs by domain
- Use meaningful asset names that serve as factory keys
- Consider catalog size and loading performance

### Factory Management
- Validate factory assignments in Inspector
- Use consistent naming conventions for factories
- Document catalog purposes and expected factory types

### Runtime Usage
- Cache catalog references to avoid repeated asset loading
- Handle missing factories gracefully with fallbacks
- Monitor catalog access patterns for performance optimization

## Common Patterns

### Catalog Builder Pattern

```csharp
public class CatalogBuilder : MonoBehaviour
{
    [Header("Factory Sources")]
    [SerializeField] private ScriptableEntityFactory[] _factories;
    
    [Header("Output")]
    [SerializeField] private string _catalogName = "GeneratedCatalog";
    
    [ContextMenu("Generate Catalog")]
    private void GenerateCatalog()
    {
        var catalog = ScriptableObject.CreateInstance<ScriptableEntityCatalog>();
        catalog.name = _catalogName;
        catalog._factories = _factories;
        
        string assetPath = $"Assets/Catalogs/{_catalogName}.asset";
        AssetDatabase.CreateAsset(catalog, assetPath);
        AssetDatabase.SaveAssets();
        
        Debug.Log($"Generated catalog at: {assetPath}");
    }
}
```

### Dynamic Catalog Loading

```csharp
public class DynamicCatalogLoader : MonoBehaviour
{
    private readonly Dictionary<string, ScriptableEntityCatalog> _loadedCatalogs = new();
    
    public async Task<ScriptableEntityCatalog> LoadCatalogAsync(string catalogPath)
    {
        if (_loadedCatalogs.TryGetValue(catalogPath, out var cachedCatalog))
        {
            return cachedCatalog;
        }
        
        var request = Resources.LoadAsync<ScriptableEntityCatalog>(catalogPath);
        await Task.Run(() => { while (!request.isDone) { } });
        
        var catalog = request.asset as ScriptableEntityCatalog;
        if (catalog != null)
        {
            _loadedCatalogs[catalogPath] = catalog;
        }
        
        return catalog;
    }
    
    public void UnloadCatalog(string catalogPath)
    {
        if (_loadedCatalogs.Remove(catalogPath, out var catalog))
        {
            Resources.UnloadAsset(catalog);
        }
    }
}
```

The `ScriptableEntityCatalog` provides a powerful Unity-integrated solution for organizing and accessing entity factories, combining the flexibility of the Atomic framework with Unity's asset management and editor capabilities.