# 🧩 IPrefabEntityPool

A Unity-specific pooling interface for managing SceneEntity instances created from prefabs, with integrated Transform management and Unity lifecycle support. Provides sophisticated prefab-based entity pooling with position, rotation, and parenting capabilities.

## Overview

`IPrefabEntityPool` extends basic entity pooling to support Unity's prefab system, enabling efficient management of scene-based entities with full Transform integration. Designed for Unity workflows requiring prefab instantiation with pooling optimization.

## Interface Definition

```csharp
#if UNITY_5_3_OR_NEWER
namespace Atomic.Entities
{
    /// <summary>
    /// Non-generic version for base SceneEntity types.
    /// </summary>
    public interface IPrefabEntityPool : IPrefabEntityPool<SceneEntity>
    {
    }

    /// <summary>
    /// Manages multiple scene entity pools associated with specific prefabs.
    /// </summary>
    /// <typeparam name="E">The type of scene entity being pooled</typeparam>
    public interface IPrefabEntityPool<E> : IDisposable where E : SceneEntity
    {
        /// <summary>
        /// Initializes the pool for a specific prefab with preallocated entities.
        /// </summary>
        void Init(E prefab, int count);

        /// <summary>
        /// Rents an entity instance from the pool for the specified prefab.
        /// </summary>
        E Rent(E prefab);

        /// <summary>
        /// Rents an entity and parents it under the specified transform.
        /// </summary>
        E Rent(E prefab, Transform parent);

        /// <summary>
        /// Rents an entity with specific position and rotation.
        /// </summary>
        E Rent(E prefab, Vector3 position, Quaternion rotation, Transform parent = null);

        /// <summary>
        /// Returns a previously rented entity to its corresponding pool.
        /// </summary>
        void Return(E entity);

        /// <summary>
        /// Clears the pool for a specific prefab.
        /// </summary>
        void Dispose(E prefab);
    }
}
#endif
```

## Key Features

### Unity Integration
- Native support for Unity prefabs and SceneEntity instances
- Transform management for position, rotation, and parenting
- Seamless integration with Unity's GameObject lifecycle

### Multi-Prefab Support
- Multiple pools managed by prefab reference
- Automatic pool creation based on prefab identity
- Efficient prefab-based entity organization

### Transform Management
- Built-in position and rotation handling during rental
- Flexible parenting support for scene hierarchy management
- Automatic Transform reset and positioning

## Usage Examples

### Basic Prefab Pool Setup

```csharp
public class PrefabPoolManager : MonoBehaviour
{
    [Header("Prefab Pool Configuration")]
    [SerializeField] private SceneEntity[] _prefabsToPool;
    [SerializeField] private int _initialPoolSize = 10;
    
    private IPrefabEntityPool _prefabPool;
    
    void Start()
    {
        // Initialize prefab pool
        _prefabPool = new PrefabEntityPool();
        
        // Pre-populate pools for each prefab
        foreach (var prefab in _prefabsToPool)
        {
            _prefabPool.Init(prefab, _initialPoolSize);
            Debug.Log($"Initialized pool for {prefab.name} with {_initialPoolSize} instances");
        }
    }
    
    public SceneEntity SpawnEntity(SceneEntity prefab, Vector3 position)
    {
        // Rent entity from prefab-specific pool
        var entity = _prefabPool.Rent(prefab, position, Quaternion.identity);
        
        // Configure spawned entity
        entity.Set("SpawnTime", Time.time);
        entity.Set("SpawnPosition", position);
        entity.AddTag("Active");
        
        return entity;
    }
    
    public SceneEntity SpawnEntityWithParent(SceneEntity prefab, Transform parent)
    {
        // Rent entity and set parent
        var entity = _prefabPool.Rent(prefab, parent);
        
        // Additional setup
        entity.Set("ParentName", parent.name);
        entity.Set("LocalSpawn", true);
        entity.AddTag("Parented");
        
        return entity;
    }
    
    public void DespawnEntity(SceneEntity entity)
    {
        // Clean up entity state
        entity.RemoveTag("Active");
        entity.RemoveTag("Parented");
        entity.UnsubscribeAll();
        
        // Return to pool
        _prefabPool.Return(entity);
    }
    
    void OnDestroy()
    {
        _prefabPool?.Dispose();
    }
}
```

### Advanced Enemy Spawning System

```csharp
public class EnemySpawningSystem : MonoBehaviour
{
    [System.Serializable]
    public struct EnemyPrefabConfig
    {
        public SceneEntity prefab;
        public int poolSize;
        public float spawnWeight;
        public Vector2 levelRange;
    }
    
    [Header("Enemy Configuration")]
    [SerializeField] private EnemyPrefabConfig[] _enemyConfigs;
    [SerializeField] private Transform _enemyContainer;
    
    [Header("Spawn Settings")]
    [SerializeField] private Transform[] _spawnPoints;
    [SerializeField] private float _spawnInterval = 2f;
    [SerializeField] private int _maxActiveEnemies = 20;
    
    private IPrefabEntityPool _enemyPool;
    private readonly List<SceneEntity> _activeEnemies = new();
    private int _currentPlayerLevel = 1;
    
    void Start()
    {
        SetupEnemyPools();
        StartCoroutine(SpawnEnemiesCoroutine());
    }
    
    private void SetupEnemyPools()
    {
        _enemyPool = new PrefabEntityPool();
        
        foreach (var config in _enemyConfigs)
        {
            _enemyPool.Init(config.prefab, config.poolSize);
        }
    }
    
    private IEnumerator SpawnEnemiesCoroutine()
    {
        while (true)
        {
            yield return new WaitForSeconds(_spawnInterval);
            
            if (_activeEnemies.Count < _maxActiveEnemies && ShouldSpawnEnemy())
            {
                SpawnRandomEnemy();
            }
            
            // Clean up destroyed enemies
            _activeEnemies.RemoveAll(enemy => enemy == null || !enemy.HasTag("Active"));
        }
    }
    
    private void SpawnRandomEnemy()
    {
        // Select appropriate enemy prefab based on level
        var availableConfigs = _enemyConfigs.Where(config => 
            _currentPlayerLevel >= config.levelRange.x && 
            _currentPlayerLevel <= config.levelRange.y
        ).ToArray();
        
        if (availableConfigs.Length == 0) return;
        
        // Weighted selection
        var selectedConfig = SelectWeightedRandom(availableConfigs);
        
        // Choose spawn point
        var spawnPoint = _spawnPoints[Random.Range(0, _spawnPoints.Length)];
        
        // Spawn enemy
        var enemy = _enemyPool.Rent(
            selectedConfig.prefab,
            spawnPoint.position,
            spawnPoint.rotation,
            _enemyContainer
        );
        
        // Configure enemy
        ConfigureEnemyForLevel(enemy, _currentPlayerLevel);
        
        // Track active enemy
        _activeEnemies.Add(enemy);
        
        // Subscribe to enemy death
        enemy.OnDestroyed += () => OnEnemyDestroyed(enemy);
    }
    
    private EnemyPrefabConfig SelectWeightedRandom(EnemyPrefabConfig[] configs)
    {
        float totalWeight = configs.Sum(c => c.spawnWeight);
        float randomValue = Random.Range(0f, totalWeight);
        float currentWeight = 0f;
        
        foreach (var config in configs)
        {
            currentWeight += config.spawnWeight;
            if (randomValue <= currentWeight)
            {
                return config;
            }
        }
        
        return configs[configs.Length - 1];
    }
    
    private void ConfigureEnemyForLevel(SceneEntity enemy, int playerLevel)
    {
        // Level-based stat scaling
        float levelMultiplier = 1f + (playerLevel - 1) * 0.2f;
        
        // Base stats from prefab
        float baseHealth = enemy.Get<float>("BaseHealth");
        float baseDamage = enemy.Get<float>("BaseDamage");
        float baseSpeed = enemy.Get<float>("BaseSpeed");
        
        // Apply level scaling
        enemy.Set("Health", baseHealth * levelMultiplier);
        enemy.Set("MaxHealth", baseHealth * levelMultiplier);
        enemy.Set("Damage", baseDamage * levelMultiplier);
        enemy.Set("Speed", baseSpeed * Mathf.Min(levelMultiplier, 2f)); // Cap speed scaling
        
        // Level-specific tags
        enemy.AddTag($"Level{playerLevel}");
        enemy.AddTag("Active");
        
        if (playerLevel >= 5)
        {
            enemy.AddTag("Veteran");
            enemy.Set("ExperienceReward", enemy.Get<int>("ExperienceReward") * 2);
        }
        
        // Visual scaling for higher levels
        if (playerLevel >= 3)
        {
            var scale = 1f + (playerLevel - 3) * 0.1f;
            enemy.transform.localScale = Vector3.one * scale;
        }
    }
    
    private void OnEnemyDestroyed(SceneEntity enemy)
    {
        // Remove from active list
        _activeEnemies.Remove(enemy);
        
        // Return to pool
        DespawnEnemy(enemy);
    }
    
    public void DespawnEnemy(SceneEntity enemy)
    {
        // Clean up enemy state
        enemy.RemoveTag("Active");
        enemy.RemoveTag("Veteran");
        enemy.transform.localScale = Vector3.one; // Reset scale
        
        // Reset stats to defaults
        ResetEnemyToDefaults(enemy);
        
        // Return to pool
        _enemyPool.Return(enemy);
    }
    
    private void ResetEnemyToDefaults(SceneEntity enemy)
    {
        // Reset to prefab defaults or stored base values
        float baseHealth = enemy.Get<float>("BaseHealth");
        float baseDamage = enemy.Get<float>("BaseDamage");
        float baseSpeed = enemy.Get<float>("BaseSpeed");
        
        enemy.Set("Health", baseHealth);
        enemy.Set("MaxHealth", baseHealth);
        enemy.Set("Damage", baseDamage);
        enemy.Set("Speed", baseSpeed);
    }
    
    private bool ShouldSpawnEnemy()
    {
        // Add logic for spawn conditions
        return Random.Range(0f, 1f) < 0.7f; // 70% chance
    }
    
    public void SetPlayerLevel(int level)
    {
        _currentPlayerLevel = level;
    }
    
    void OnDestroy()
    {
        _enemyPool?.Dispose();
    }
}
```

### Projectile Pool with Physics Integration

```csharp
public class ProjectilePoolSystem : MonoBehaviour
{
    [System.Serializable]
    public struct ProjectileConfig
    {
        public SceneEntity prefab;
        public int poolSize;
        public float defaultSpeed;
        public float maxLifetime;
        public LayerMask collisionMask;
    }
    
    [Header("Projectile Pools")]
    [SerializeField] private ProjectileConfig[] _projectileConfigs;
    
    [Header("Pool Management")]
    [SerializeField] private Transform _projectileContainer;
    [SerializeField] private bool _enablePoolStatistics = true;
    
    private IPrefabEntityPool _projectilePool;
    private readonly Dictionary<SceneEntity, ProjectileConfig> _activeProjectiles = new();
    private readonly Dictionary<SceneEntity, ProjectileConfig> _configLookup = new();
    
    // Statistics
    private int _totalProjectilesFired;
    private int _totalProjectilesHit;
    private int _totalProjectilesMissed;
    
    void Start()
    {
        SetupProjectilePools();
    }
    
    private void SetupProjectilePools()
    {
        _projectilePool = new PrefabEntityPool();
        
        foreach (var config in _projectileConfigs)
        {
            _projectilePool.Init(config.prefab, config.poolSize);
            _configLookup[config.prefab] = config;
        }
        
        Debug.Log($"Initialized {_projectileConfigs.Length} projectile pools");
    }
    
    public SceneEntity FireProjectile(SceneEntity prefab, Vector3 origin, Vector3 direction, float? customSpeed = null)
    {
        if (!_configLookup.TryGetValue(prefab, out var config))
        {
            Debug.LogError($"No pool configuration found for prefab: {prefab.name}");
            return null;
        }
        
        // Rent projectile from pool
        var projectile = _projectilePool.Rent(prefab, origin, Quaternion.LookRotation(direction), _projectileContainer);
        
        // Configure projectile
        float speed = customSpeed ?? config.defaultSpeed;
        ConfigureProjectile(projectile, direction, speed, config);
        
        // Track active projectile
        _activeProjectiles[projectile] = config;
        
        // Statistics
        _totalProjectilesFired++;
        
        // Schedule lifetime management
        StartCoroutine(ManageProjectileLifetime(projectile, config.maxLifetime));
        
        return projectile;
    }
    
    private void ConfigureProjectile(SceneEntity projectile, Vector3 direction, float speed, ProjectileConfig config)
    {
        // Physics setup
        projectile.Set("Direction", direction);
        projectile.Set("Speed", speed);
        projectile.Set("MaxLifetime", config.maxLifetime);
        projectile.Set("CollisionMask", config.collisionMask.value);
        projectile.Set("LaunchTime", Time.time);
        
        // Tags
        projectile.AddTag("Active");
        projectile.AddTag("Projectile");
        
        // Get Rigidbody if present
        var rigidbody = projectile.GetComponent<Rigidbody>();
        if (rigidbody != null)
        {
            rigidbody.velocity = direction * speed;
            rigidbody.isKinematic = false;
        }
        
        // Subscribe to collision events
        var collider = projectile.GetComponent<Collider>();
        if (collider != null)
        {
            collider.enabled = true;
        }
        
        // Add collision behavior
        projectile.OnCollisionEnter += (other) => OnProjectileCollision(projectile, other);
    }
    
    private IEnumerator ManageProjectileLifetime(SceneEntity projectile, float maxLifetime)
    {
        yield return new WaitForSeconds(maxLifetime);
        
        // Timeout - return projectile if still active
        if (projectile != null && projectile.HasTag("Active"))
        {
            _totalProjectilesMissed++;
            ReturnProjectile(projectile, "Timeout");
        }
    }
    
    private void OnProjectileCollision(SceneEntity projectile, Collider other)
    {
        // Check collision validity
        if (!_activeProjectiles.TryGetValue(projectile, out var config))
            return;
        
        // Check collision mask
        int layerMask = 1 << other.gameObject.layer;
        if ((config.collisionMask.value & layerMask) == 0)
            return;
        
        // Handle collision
        Vector3 impactPoint = projectile.transform.position;
        HandleProjectileImpact(projectile, other, impactPoint);
        
        _totalProjectilesHit++;
        ReturnProjectile(projectile, "Hit");
    }
    
    private void HandleProjectileImpact(SceneEntity projectile, Collider target, Vector3 impactPoint)
    {
        // Create impact effects
        CreateImpactEffect(impactPoint, projectile.transform.forward);
        
        // Apply damage if target has health
        var targetEntity = target.GetComponent<SceneEntity>();
        if (targetEntity != null && targetEntity.HasValue("Health"))
        {
            float damage = projectile.Get<float>("Damage");
            float currentHealth = targetEntity.Get<float>("Health");
            targetEntity.Set("Health", currentHealth - damage);
            
            // Trigger damage events
            GameEvents.OnDamageDealt?.Invoke(targetEntity, damage, impactPoint);
        }
        
        // Apply physics impulse
        var targetRigidbody = target.GetComponent<Rigidbody>();
        if (targetRigidbody != null)
        {
            Vector3 impulse = projectile.Get<Vector3>("Direction") * projectile.Get<float>("ImpactForce");
            targetRigidbody.AddForceAtPosition(impulse, impactPoint, ForceMode.Impulse);
        }
    }
    
    private void CreateImpactEffect(Vector3 position, Vector3 normal)
    {
        // Create impact particle effect, sound, etc.
        // Could also use pooled effect entities
    }
    
    public void ReturnProjectile(SceneEntity projectile, string reason = "Manual")
    {
        if (!_activeProjectiles.ContainsKey(projectile))
            return;
        
        // Clean up projectile state
        CleanProjectileForReturn(projectile);
        
        // Return to pool
        _projectilePool.Return(projectile);
        
        // Remove from active tracking
        _activeProjectiles.Remove(projectile);
        
        Debug.Log($"Projectile returned to pool: {reason}");
    }
    
    private void CleanProjectileForReturn(SceneEntity projectile)
    {
        // Stop physics
        var rigidbody = projectile.GetComponent<Rigidbody>();
        if (rigidbody != null)
        {
            rigidbody.velocity = Vector3.zero;
            rigidbody.isKinematic = true;
        }
        
        // Disable collider
        var collider = projectile.GetComponent<Collider>();
        if (collider != null)
        {
            collider.enabled = false;
        }
        
        // Clean entity state
        projectile.RemoveTag("Active");
        projectile.UnsubscribeAll();
        
        // Reset transform
        projectile.transform.localScale = Vector3.one;
        projectile.transform.rotation = Quaternion.identity;
    }
    
    // Public API for common projectile types
    public SceneEntity FireBullet(Vector3 origin, Vector3 direction, float speed = 20f)
    {
        var bulletPrefab = _projectileConfigs[0].prefab; // Assume first is bullet
        return FireProjectile(bulletPrefab, origin, direction, speed);
    }
    
    public SceneEntity FireRocket(Vector3 origin, Vector3 direction, float speed = 15f)
    {
        var rocketPrefab = _projectileConfigs[1].prefab; // Assume second is rocket
        return FireProjectile(rocketPrefab, origin, direction, speed);
    }
    
    // Statistics and monitoring
    void OnGUI()
    {
        if (!_enablePoolStatistics) return;
        
        int y = 10;
        GUI.Label(new Rect(10, y, 300, 20), "Projectile Pool Statistics:");
        y += 25;
        
        GUI.Label(new Rect(10, y, 200, 20), $"Total Fired: {_totalProjectilesFired}");
        y += 20;
        GUI.Label(new Rect(10, y, 200, 20), $"Hits: {_totalProjectilesHit}");
        y += 20;
        GUI.Label(new Rect(10, y, 200, 20), $"Misses: {_totalProjectilesMissed}");
        y += 20;
        GUI.Label(new Rect(10, y, 200, 20), $"Active: {_activeProjectiles.Count}");
        
        float hitRate = _totalProjectilesFired > 0 ? (float)_totalProjectilesHit / _totalProjectilesFired * 100f : 0f;
        y += 20;
        GUI.Label(new Rect(10, y, 200, 20), $"Hit Rate: {hitRate:F1}%");
    }
    
    void OnDestroy()
    {
        // Return all active projectiles
        foreach (var projectile in _activeProjectiles.Keys.ToArray())
        {
            ReturnProjectile(projectile, "Shutdown");
        }
        
        _projectilePool?.Dispose();
    }
}
```

## Integration with Atomic Framework

### Reactive Prefab Pool Management

```csharp
public class ReactivePrefabPoolSystem : MonoBehaviour
{
    private IPrefabEntityPool _prefabPool;
    
    [Header("Reactive Pool Configuration")]
    private readonly ReactiveInt _totalActiveEntities = new(0);
    private readonly ReactiveFloat _poolEfficiency = new(1f);
    private readonly ReactiveBool _poolOverloaded = new(false);
    
    [SerializeField] private SceneEntity[] _managedPrefabs;
    [SerializeField] private int _warningThreshold = 15;
    [SerializeField] private int _overloadThreshold = 20;
    
    void Start()
    {
        SetupReactivePool();
        
        // React to pool state changes
        _totalActiveEntities.Subscribe(OnActiveEntitiesChanged);
        _poolEfficiency.Subscribe(OnEfficiencyChanged);
        _poolOverloaded.Subscribe(OnOverloadStateChanged);
    }
    
    private void SetupReactivePool()
    {
        _prefabPool = new PrefabEntityPool();
        
        foreach (var prefab in _managedPrefabs)
        {
            _prefabPool.Init(prefab, 10);
        }
        
        // Start monitoring
        InvokeRepeating(nameof(UpdatePoolMetrics), 1f, 1f);
    }
    
    public SceneEntity SpawnEntity(SceneEntity prefab, Vector3 position, Quaternion rotation)
    {
        var entity = _prefabPool.Rent(prefab, position, rotation);
        
        // Track entity lifecycle
        entity.OnDestroyed += () => OnEntityDestroyed(entity);
        
        _totalActiveEntities.Value++;
        
        return entity;
    }
    
    public void DespawnEntity(SceneEntity entity)
    {
        _prefabPool.Return(entity);
        _totalActiveEntities.Value--;
    }
    
    private void OnEntityDestroyed(SceneEntity entity)
    {
        _totalActiveEntities.Value = Mathf.Max(0, _totalActiveEntities.Value - 1);
    }
    
    private void UpdatePoolMetrics()
    {
        // Calculate pool efficiency (would need pool to expose statistics)
        float efficiency = CalculatePoolEfficiency();
        _poolEfficiency.Value = efficiency;
        
        // Check overload condition
        bool isOverloaded = _totalActiveEntities.Value > _overloadThreshold;
        _poolOverloaded.Value = isOverloaded;
    }
    
    private float CalculatePoolEfficiency()
    {
        // Implementation would calculate based on cache hit rate, etc.
        return Random.Range(0.7f, 1f); // Placeholder
    }
    
    private void OnActiveEntitiesChanged(int count)
    {
        Debug.Log($"Active entities: {count}");
        
        if (count > _warningThreshold)
        {
            Debug.LogWarning($"High entity count: {count}");
        }
        
        // Update UI or other systems
        GameEvents.OnActiveEntityCountChanged?.Invoke(count);
    }
    
    private void OnEfficiencyChanged(float efficiency)
    {
        Debug.Log($"Pool efficiency: {efficiency:P}");
        
        if (efficiency < 0.5f)
        {
            Debug.LogWarning("Low pool efficiency detected");
        }
    }
    
    private void OnOverloadStateChanged(bool overloaded)
    {
        if (overloaded)
        {
            Debug.LogError("Pool overloaded - consider increasing pool sizes");
            GameEvents.OnPoolOverloaded?.Invoke();
        }
        else
        {
            Debug.Log("Pool load returned to normal");
            GameEvents.OnPoolLoadNormalized?.Invoke();
        }
    }
}
```

## Implementation Notes

### Unity Integration
- Requires Unity 5.3 or newer for full compatibility
- Integrates with Unity's Transform, GameObject, and prefab systems
- Supports Unity's serialization and Inspector integration

### Prefab Management
- Uses prefab references as pool keys for organization
- Supports automatic pool creation on demand
- Handles prefab name collision detection

### Transform Handling
- Built-in position and rotation management during rental
- Flexible parenting support for scene hierarchy
- Automatic Transform reset during return process

## Best Practices

### Prefab Organization
- Use consistent prefab naming conventions for pool identification
- Pre-initialize pools for frequently spawned prefabs
- Monitor pool utilization to optimize sizes

### Performance Optimization
- Pool commonly used prefabs at application start
- Use appropriate initial pool sizes based on usage patterns
- Consider pool expansion strategies for dynamic scenarios

### Memory Management
- Properly dispose pools and clear references
- Monitor total GameObject count across all pools
- Implement pool cleanup strategies for scene transitions

## Common Patterns

### Prefab Pool Factory

```csharp
public class PrefabPoolFactory : MonoBehaviour
{
    [System.Serializable]
    public struct PoolDefinition
    {
        public SceneEntity prefab;
        public int size;
        public Transform container;
    }
    
    public static IPrefabEntityPool CreatePrefabPool(PoolDefinition[] definitions)
    {
        var pool = new PrefabEntityPool();
        
        foreach (var def in definitions)
        {
            pool.Init(def.prefab, def.size);
        }
        
        return pool;
    }
}
```

### Scene-Specific Pool Management

```csharp
public class ScenePoolManager : MonoBehaviour
{
    private IPrefabEntityPool _scenePool;
    
    void Start()
    {
        // Create pool specific to current scene
        _scenePool = new PrefabEntityPool();
        
        // Initialize with scene-specific prefabs
        var scenePrefabs = Resources.LoadAll<SceneEntity>($"Prefabs/{UnityEngine.SceneManagement.SceneManager.GetActiveScene().name}");
        
        foreach (var prefab in scenePrefabs)
        {
            _scenePool.Init(prefab, 5);
        }
    }
    
    void OnDestroy()
    {
        _scenePool?.Dispose();
    }
}
```

The `IPrefabEntityPool` interface provides sophisticated Unity integration for entity pooling, enabling efficient management of prefab-based entities with full Transform and lifecycle support within the Atomic framework.