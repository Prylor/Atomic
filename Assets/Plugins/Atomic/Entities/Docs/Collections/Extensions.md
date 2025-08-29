# 🧩 Collection Extensions

Extension methods for `IEntityCollection<E>` that provide convenient operations for batch entity management, Unity integration, and common collection patterns. These extensions enhance the base collection functionality with practical, reusable operations.

## Overview

The Collection Extensions provide additional functionality for entity collections through static extension methods. These methods focus on common patterns like batch operations, Unity GameObject integration, and convenient entity management workflows.

## Extension Methods Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Provides extension methods for working with IEntityCollection{E}.
    /// </summary>
    public static partial class Extensions
    {
        /// <summary>
        /// Adds a range of entities to the collection.
        /// </summary>
        public static void AddRange<E>(this IEntityCollection<E> it, params E[] entities) where E : IEntity;
        
        /// <summary>
        /// Adds a range of entities from an enumerable to the collection.
        /// Ignores null entities.
        /// </summary>
        public static void AddRange<E>(this IEntityCollection<E> it, IEnumerable<E> entities) where E : IEntity;

#if UNITY_5_3_OR_NEWER
        /// <summary>
        /// Instantiates a new entity based on the given prefab and adds it to the collection.
        /// </summary>
        public static E CreateEntity<E>(
            this IEntityCollection<E> it,
            E prefab,
            Vector3 position,
            Quaternion rotation,
            Transform parent = null
        ) where E : SceneEntity;

        /// <summary>
        /// Removes the entity from the collection and destroys its game object.
        /// </summary>
        public static void DestroyEntity<E>(this IEntityCollection<E> it, E entity, float delay = 0)
            where E : SceneEntity;
#endif
    }
}
```

## Key Features

### Batch Operations
- Efficient bulk addition of multiple entities
- Support for arrays and enumerables
- Null entity filtering for safe operations

### Unity Integration
- GameObject instantiation with automatic collection addition
- Integrated destruction with collection removal
- Transform hierarchy support

### Performance Optimization
- Aggressive inlining for critical operations
- Minimal overhead over direct collection operations
- Exception handling for robustness

## Usage Examples

### Basic Batch Operations

```csharp
public class EntityBatchManager : MonoBehaviour
{
    private readonly EntityCollection<IEntity> _enemies = new();
    private readonly EntityCollection<IEntity> _allies = new();

    void Start()
    {
        // Create multiple entities
        var enemyEntities = new IEntity[]
        {
            new Entity("Goblin1"),
            new Entity("Goblin2"),
            new Entity("Orc1"),
            null, // This will be ignored
            new Entity("Troll1")
        };

        var allyEntities = new List<IEntity>
        {
            new Entity("Knight1"),
            new Entity("Archer1"),
            new Entity("Wizard1")
        };

        // Add all entities in one operation
        _enemies.AddRange(enemyEntities);  // Adds 4 entities, ignores null
        _allies.AddRange(allyEntities);    // Adds 3 entities

        Debug.Log($"Enemies: {_enemies.Count}, Allies: {_allies.Count}");
    }

    public void SpawnWave(IEnumerable<IEntity> waveEntities)
    {
        // Efficient batch addition from external source
        _enemies.AddRange(waveEntities);
        
        Debug.Log($"Wave spawned! Total enemies: {_enemies.Count}");
    }

    public void CreateTestEntities(int count)
    {
        var testEntities = new IEntity[count];
        
        for (int i = 0; i < count; i++)
        {
            testEntities[i] = new Entity($"TestEntity_{i}");
        }
        
        // Add all at once instead of individual Add calls
        _enemies.AddRange(testEntities);
    }
}
```

### Unity Scene Entity Management

```csharp
#if UNITY_5_3_OR_NEWER
public class SceneEntitySpawner : MonoBehaviour
{
    [Header("Prefab Configuration")]
    [SerializeField] private SceneEntity _enemyPrefab;
    [SerializeField] private SceneEntity _collectiblePrefab;
    [SerializeField] private Transform _spawnParent;
    
    [Header("Spawn Settings")]
    [SerializeField] private int _enemyCount = 5;
    [SerializeField] private int _collectibleCount = 10;
    [SerializeField] private float _spawnRadius = 10f;
    
    private readonly EntityCollection<SceneEntity> _spawnedEnemies = new();
    private readonly EntityCollection<SceneEntity> _spawnedCollectibles = new();

    void Start()
    {
        // Setup collection events
        _spawnedEnemies.OnAdded += OnEnemySpawned;
        _spawnedEnemies.OnRemoved += OnEnemyDespawned;
        
        _spawnedCollectibles.OnAdded += OnCollectibleSpawned;
        _spawnedCollectibles.OnRemoved += OnCollectibleDespawned;
        
        // Spawn initial entities
        SpawnInitialEnemies();
        SpawnInitialCollectibles();
    }

    private void SpawnInitialEnemies()
    {
        for (int i = 0; i < _enemyCount; i++)
        {
            Vector3 spawnPosition = GetRandomSpawnPosition();
            Quaternion spawnRotation = Quaternion.Euler(0, Random.Range(0, 360), 0);
            
            // CreateEntity extension handles instantiation + collection addition
            var enemy = _spawnedEnemies.CreateEntity(
                _enemyPrefab,
                spawnPosition,
                spawnRotation,
                _spawnParent
            );
            
            // Configure the spawned enemy
            ConfigureEnemy(enemy, i);
        }
        
        Debug.Log($"Spawned {_enemyCount} enemies");
    }

    private void SpawnInitialCollectibles()
    {
        for (int i = 0; i < _collectibleCount; i++)
        {
            Vector3 spawnPosition = GetRandomSpawnPosition();
            
            var collectible = _spawnedCollectibles.CreateEntity(
                _collectiblePrefab,
                spawnPosition,
                Quaternion.identity,
                _spawnParent
            );
            
            ConfigureCollectible(collectible, i);
        }
        
        Debug.Log($"Spawned {_collectibleCount} collectibles");
    }

    private void ConfigureEnemy(SceneEntity enemy, int index)
    {
        enemy.name = $"Enemy_{index}";
        enemy.Set("Health", Random.Range(50f, 100f));
        enemy.Set("Damage", Random.Range(10f, 25f));
        enemy.Set("MovementSpeed", Random.Range(2f, 5f));
        enemy.AddTag("Enemy");
        enemy.AddTag("Hostile");
    }

    private void ConfigureCollectible(SceneEntity collectible, int index)
    {
        collectible.name = $"Collectible_{index}";
        collectible.Set("Value", Random.Range(10, 50));
        collectible.Set("Type", Random.Range(0, 3)); // 0=Coin, 1=Gem, 2=PowerUp
        collectible.AddTag("Collectible");
        collectible.AddTag("Valuable");
    }

    private Vector3 GetRandomSpawnPosition()
    {
        Vector2 randomCircle = Random.insideUnitCircle * _spawnRadius;
        return transform.position + new Vector3(randomCircle.x, 0, randomCircle.y);
    }

    public void SpawnAdditionalEnemy()
    {
        Vector3 spawnPos = GetRandomSpawnPosition();
        var enemy = _spawnedEnemies.CreateEntity(_enemyPrefab, spawnPos, Quaternion.identity, _spawnParent);
        ConfigureEnemy(enemy, _spawnedEnemies.Count);
    }

    public void DestroyRandomEnemy()
    {
        if (_spawnedEnemies.Count == 0)
            return;
            
        var randomEnemy = _spawnedEnemies.ElementAt(Random.Range(0, _spawnedEnemies.Count));
        
        // DestroyEntity extension handles collection removal + GameObject destruction
        _spawnedEnemies.DestroyEntity(randomEnemy);
    }

    public void DestroyAllEnemies()
    {
        // Create a copy to avoid modification during enumeration
        var enemiesToDestroy = new List<SceneEntity>();
        _spawnedEnemies.CopyTo(enemiesToDestroy);
        
        foreach (var enemy in enemiesToDestroy)
        {
            _spawnedEnemies.DestroyEntity(enemy);
        }
    }

    public void DestroyAllEnemiesWithDelay(float delay)
    {
        var enemiesToDestroy = new List<SceneEntity>();
        _spawnedEnemies.CopyTo(enemiesToDestroy);
        
        foreach (var enemy in enemiesToDestroy)
        {
            _spawnedEnemies.DestroyEntity(enemy, delay);
        }
    }

    private void OnEnemySpawned(SceneEntity enemy)
    {
        Debug.Log($"Enemy spawned: {enemy.name} at {enemy.transform.position}");
        GameEvents.OnEnemySpawned?.Invoke(enemy);
    }

    private void OnEnemyDespawned(SceneEntity enemy)
    {
        Debug.Log($"Enemy despawned: {enemy.name}");
        GameEvents.OnEnemyDespawned?.Invoke(enemy);
    }

    private void OnCollectibleSpawned(SceneEntity collectible)
    {
        Debug.Log($"Collectible spawned: {collectible.name}");
        GameEvents.OnCollectibleSpawned?.Invoke(collectible);
    }

    private void OnCollectibleDespawned(SceneEntity collectible)
    {
        Debug.Log($"Collectible despawned: {collectible.name}");
        GameEvents.OnCollectibleDespawned?.Invoke(collectible);
    }

    void OnDestroy()
    {
        _spawnedEnemies.Dispose();
        _spawnedCollectibles.Dispose();
    }
}
#endif
```

### Advanced Batch Processing System

```csharp
public class EntityBatchProcessor : MonoBehaviour
{
    private readonly EntityCollection<IEntity> _processingQueue = new();
    
    [Header("Batch Configuration")]
    [SerializeField] private int _maxBatchSize = 20;
    [SerializeField] private float _batchProcessInterval = 1f;
    [SerializeField] private bool _enableBatching = true;
    
    private float _lastBatchTime;
    private readonly Queue<IEntity> _pendingEntities = new();

    void Start()
    {
        _processingQueue.OnAdded += OnEntityAddedToQueue;
        _processingQueue.OnRemoved += OnEntityRemovedFromQueue;
    }

    void Update()
    {
        if (_enableBatching && Time.time - _lastBatchTime >= _batchProcessInterval)
        {
            ProcessPendingBatch();
            _lastBatchTime = Time.time;
        }
    }

    public void QueueEntityForProcessing(IEntity entity)
    {
        if (entity != null)
        {
            _pendingEntities.Enqueue(entity);
            
            if (!_enableBatching || _pendingEntities.Count >= _maxBatchSize)
            {
                ProcessPendingBatch();
            }
        }
    }

    public void QueueEntitiesForProcessing(IEnumerable<IEntity> entities)
    {
        foreach (var entity in entities)
        {
            if (entity != null)
            {
                _pendingEntities.Enqueue(entity);
            }
        }
        
        if (!_enableBatching || _pendingEntities.Count >= _maxBatchSize)
        {
            ProcessPendingBatch();
        }
    }

    private void ProcessPendingBatch()
    {
        if (_pendingEntities.Count == 0)
            return;

        int batchSize = Math.Min(_pendingEntities.Count, _maxBatchSize);
        var batchEntities = new IEntity[batchSize];
        
        for (int i = 0; i < batchSize; i++)
        {
            batchEntities[i] = _pendingEntities.Dequeue();
        }
        
        Debug.Log($"Processing batch of {batchSize} entities");
        
        // Use AddRange extension for efficient batch addition
        _processingQueue.AddRange(batchEntities);
    }

    public void ProcessAllPending()
    {
        if (_pendingEntities.Count == 0)
            return;

        var allPending = new IEntity[_pendingEntities.Count];
        int index = 0;
        
        while (_pendingEntities.Count > 0)
        {
            allPending[index++] = _pendingEntities.Dequeue();
        }
        
        Debug.Log($"Processing all pending entities: {allPending.Length}");
        _processingQueue.AddRange(allPending);
    }

    public void ClearProcessingQueue()
    {
        var entitiesToRemove = new List<IEntity>();
        _processingQueue.CopyTo(entitiesToRemove);
        
        foreach (var entity in entitiesToRemove)
        {
            _processingQueue.Remove(entity);
        }
        
        Debug.Log($"Cleared {entitiesToRemove.Count} entities from processing queue");
    }

    private void OnEntityAddedToQueue(IEntity entity)
    {
        Debug.Log($"Entity added to processing queue: {entity.Name}");
        ProcessEntity(entity);
    }

    private void OnEntityRemovedFromQueue(IEntity entity)
    {
        Debug.Log($"Entity removed from processing queue: {entity.Name}");
        FinalizeEntity(entity);
    }

    private void ProcessEntity(IEntity entity)
    {
        // Simulate entity processing
        entity.AddTag("Processing");
        entity.Set("ProcessingStartTime", Time.time);
        
        // Add some processing logic here
        StartCoroutine(ProcessEntityCoroutine(entity));
    }

    private void FinalizeEntity(IEntity entity)
    {
        entity.RemoveTag("Processing");
        entity.AddTag("Processed");
        
        if (entity.HasValue("ProcessingStartTime"))
        {
            float processingTime = Time.time - entity.Get<float>("ProcessingStartTime");
            entity.Set("ProcessingDuration", processingTime);
            entity.Remove("ProcessingStartTime");
        }
    }

    private System.Collections.IEnumerator ProcessEntityCoroutine(IEntity entity)
    {
        // Simulate processing delay
        yield return new WaitForSeconds(Random.Range(0.5f, 2f));
        
        // Remove from processing queue when done
        if (_processingQueue.Contains(entity))
        {
            _processingQueue.Remove(entity);
        }
    }

    public void SetBatchingEnabled(bool enabled)
    {
        _enableBatching = enabled;
        
        if (!enabled)
        {
            ProcessAllPending(); // Process everything immediately
        }
    }

    public int GetQueuedCount() => _pendingEntities.Count;
    public int GetProcessingCount() => _processingQueue.Count;

    void OnDestroy()
    {
        _processingQueue.Dispose();
        _pendingEntities.Clear();
    }
}
```

### Collection Synchronization System

```csharp
public class EntityCollectionSynchronizer : MonoBehaviour
{
    [System.Serializable]
    public struct SyncConfiguration
    {
        public string sourceName;
        public string targetName;
        public bool bidirectional;
        public bool includeSubset;
        public string[] requiredTags;
    }

    [SerializeField] private SyncConfiguration[] _syncConfigurations;
    
    private readonly Dictionary<string, EntityCollection<IEntity>> _collections = new();
    private readonly List<SyncBinding> _syncBindings = new();

    void Start()
    {
        InitializeCollections();
        SetupSynchronization();
    }

    private void InitializeCollections()
    {
        // Create collections for each unique name in configurations
        var collectionNames = new HashSet<string>();
        
        foreach (var config in _syncConfigurations)
        {
            collectionNames.Add(config.sourceName);
            collectionNames.Add(config.targetName);
        }
        
        foreach (var name in collectionNames)
        {
            _collections[name] = new EntityCollection<IEntity>();
            Debug.Log($"Created collection: {name}");
        }
    }

    private void SetupSynchronization()
    {
        foreach (var config in _syncConfigurations)
        {
            if (!_collections.TryGetValue(config.sourceName, out var source) ||
                !_collections.TryGetValue(config.targetName, out var target))
            {
                Debug.LogWarning($"Could not setup sync: {config.sourceName} -> {config.targetName}");
                continue;
            }

            var binding = new SyncBinding(config, source, target);
            _syncBindings.Add(binding);
            
            Debug.Log($"Setup sync: {config.sourceName} -> {config.targetName} " +
                     $"(Bidirectional: {config.bidirectional})");
        }
    }

    public EntityCollection<IEntity> GetCollection(string name)
    {
        return _collections.GetValueOrDefault(name);
    }

    public void AddEntityToCollection(string collectionName, IEntity entity)
    {
        if (_collections.TryGetValue(collectionName, out var collection))
        {
            collection.Add(entity);
        }
    }

    public void AddEntitiesToCollection(string collectionName, IEnumerable<IEntity> entities)
    {
        if (_collections.TryGetValue(collectionName, out var collection))
        {
            // Use AddRange extension for efficient batch addition
            collection.AddRange(entities);
        }
    }

    public void SyncAllCollections()
    {
        foreach (var binding in _syncBindings)
        {
            binding.PerformSync();
        }
    }

    void OnDestroy()
    {
        foreach (var binding in _syncBindings)
        {
            binding.Dispose();
        }
        
        foreach (var collection in _collections.Values)
        {
            collection.Dispose();
        }
    }

    private class SyncBinding : IDisposable
    {
        private readonly SyncConfiguration _config;
        private readonly EntityCollection<IEntity> _source;
        private readonly EntityCollection<IEntity> _target;

        public SyncBinding(SyncConfiguration config, EntityCollection<IEntity> source, EntityCollection<IEntity> target)
        {
            _config = config;
            _source = source;
            _target = target;

            // Setup one-way sync
            _source.OnAdded += OnSourceEntityAdded;
            _source.OnRemoved += OnSourceEntityRemoved;

            // Setup bidirectional sync if enabled
            if (_config.bidirectional)
            {
                _target.OnAdded += OnTargetEntityAdded;
                _target.OnRemoved += OnTargetEntityRemoved;
            }
        }

        private void OnSourceEntityAdded(IEntity entity)
        {
            if (ShouldSyncEntity(entity))
            {
                _target.Add(entity);
            }
        }

        private void OnSourceEntityRemoved(IEntity entity)
        {
            _target.Remove(entity);
        }

        private void OnTargetEntityAdded(IEntity entity)
        {
            if (_config.bidirectional && ShouldSyncEntity(entity))
            {
                _source.Add(entity);
            }
        }

        private void OnTargetEntityRemoved(IEntity entity)
        {
            if (_config.bidirectional)
            {
                _source.Remove(entity);
            }
        }

        private bool ShouldSyncEntity(IEntity entity)
        {
            if (!_config.includeSubset)
                return true;

            // Check required tags
            foreach (string requiredTag in _config.requiredTags)
            {
                if (!entity.HasTag(requiredTag))
                    return false;
            }

            return true;
        }

        public void PerformSync()
        {
            // Sync all entities from source to target
            var sourceEntities = new List<IEntity>();
            _source.CopyTo(sourceEntities);
            
            var entitiesToAdd = sourceEntities.Where(ShouldSyncEntity).ToArray();
            _target.AddRange(entitiesToAdd);
        }

        public void Dispose()
        {
            _source.OnAdded -= OnSourceEntityAdded;
            _source.OnRemoved -= OnSourceEntityRemoved;

            if (_config.bidirectional)
            {
                _target.OnAdded -= OnTargetEntityAdded;
                _target.OnRemoved -= OnTargetEntityRemoved;
            }
        }
    }
}
```

## Implementation Notes

### Performance Optimizations
- All extension methods use `MethodImpl(MethodImplOptions.AggressiveInlining)`
- Batch operations minimize event firing frequency
- Null checking prevents exceptions in AddRange operations

### Unity Integration
- Conditional compilation ensures Unity-specific code only compiles in Unity
- SceneEntity integration provides seamless GameObject management
- Transform hierarchy support for organized scene structure

### Error Handling
- Null entity filtering in AddRange methods
- ArgumentNullException for null collection parameters
- Safe handling of Unity-specific operations

## Best Practices

### Batch Operations
- Use AddRange for multiple entities instead of individual Add calls
- Consider temporary event disabling for very large batches
- Filter null entities before batch operations when possible

### Unity Integration
- Use CreateEntity for GameObject-based entities needing collection management
- Prefer DestroyEntity over manual Remove + GameObject.Destroy
- Consider parent transforms for organized scene hierarchies

### Performance Considerations
- Batch operations are more efficient than individual calls
- Unity instantiation has overhead - consider pooling for frequent operations
- Event callbacks should be lightweight to avoid performance impact

## Extension Method Details

### AddRange Methods
```csharp
// Array version - direct iteration
collection.AddRange(entity1, entity2, entity3);

// Enumerable version - foreach iteration with null filtering
collection.AddRange(entityList.Where(e => e.IsValid()));
```

### Unity Integration Methods
```csharp
// CreateEntity - combines instantiation + Add
var newEntity = collection.CreateEntity(prefab, position, rotation, parent);

// DestroyEntity - combines Remove + Destroy
collection.DestroyEntity(entity, delay: 2f);
```

The Collection Extensions provide essential functionality for practical entity collection management, offering both performance optimizations through batch operations and seamless Unity integration for GameObject-based entity systems within the Atomic framework.