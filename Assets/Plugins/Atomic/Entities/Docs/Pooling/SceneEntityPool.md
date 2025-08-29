# 🧩 SceneEntityPool

A Unity MonoBehaviour-based entity pool implementation that manages SceneEntity instances with automatic GameObject lifecycle management, prefab instantiation, and Unity Inspector integration. Provides the foundation for single-prefab entity pooling within Unity scenes.

## Overview

`SceneEntityPool` delivers a complete Unity-integrated solution for pooling SceneEntity instances created from a single prefab, featuring automatic initialization, GameObject activation management, and Inspector configuration. Essential for Unity projects requiring efficient single-prefab entity reuse patterns.

## Class Definition

```csharp
#if UNITY_5_3_OR_NEWER
namespace Atomic.Entities
{
    /// <summary>
    /// Default implementation for base SceneEntity types.
    /// </summary>
    [AddComponentMenu("Atomic/Entities/Entity Pool")]
    [DisallowMultipleComponent]
    public class SceneEntityPool : SceneEntityPool<SceneEntity>, IEntityPool
    {
        IEntity IEntityPool<IEntity>.Rent() => this.Rent();
        void IEntityPool<IEntity>.Return(IEntity entity) => this.Return((SceneEntity)entity);
        
        public static SceneEntityPool Create(CreateArgs args) => Create<SceneEntityPool>(args);
    }

    /// <summary>
    /// Generic Unity MonoBehaviour-based entity pool for scene-bound entities.
    /// </summary>
    /// <typeparam name="E">The type of entity managed by this pool</typeparam>
    public abstract class SceneEntityPool<E> : MonoBehaviour, IEntityPool<E> where E : SceneEntity
    {
        [SerializeField] private bool _initOnAwake = true;
        [SerializeField] private int _initialCount;
        [SerializeField] private E _prefab;
        [SerializeField] private Transform _container;

        internal readonly Stack<E> _pooledEntities = new();
        internal readonly HashSet<E> _rentEntities = new();

        public void Init(int count);
        public E Rent();
        public void Return(E entity);
        public void Dispose();
        
        protected virtual void OnCreate(E entity);
        protected virtual void OnRent(E entity);
        protected virtual void OnReturn(E entity);
        protected virtual void OnDispose(E entity);
    }
}
#endif
```

## Key Features

### Unity Inspector Integration
- SerializeField attributes for design-time configuration
- Automatic initialization on Awake if enabled
- Container Transform assignment for pool organization
- Odin Inspector support for enhanced editing experience

### Prefab-Based Entity Creation
- Single prefab source for all pooled entities
- Automatic instantiation using SceneEntity.Create
- GameObject lifecycle management with activation/deactivation
- Transform parenting and positioning support

### Pool Lifecycle Management
- Stack-based entity storage for LIFO reuse patterns
- HashSet tracking for rented entity validation
- Virtual lifecycle hooks for customization
- Automatic cleanup and disposal support

## Usage Examples

### Basic Scene Entity Pool Setup

```csharp
public class BasicPoolExample : MonoBehaviour
{
    [Header("Pool References")]
    [SerializeField] private SceneEntityPool _enemyPool;
    [SerializeField] private SceneEntityPool _projectilePool;
    [SerializeField] private SceneEntityPool _effectPool;
    
    [Header("Spawn Configuration")]
    [SerializeField] private Transform[] _spawnPoints;
    [SerializeField] private float _spawnInterval = 2f;
    
    void Start()
    {
        // Pools are automatically initialized if _initOnAwake is true
        // Manual initialization can be done if needed
        if (!_enemyPool._initOnAwake)
        {
            _enemyPool.Init(10);
        }
        
        StartCoroutine(SpawnLoop());
    }
    
    private IEnumerator SpawnLoop()
    {
        while (true)
        {
            yield return new WaitForSeconds(_spawnInterval);
            
            if (ShouldSpawnEnemy())
            {
                SpawnRandomEnemy();
            }
        }
    }
    
    private void SpawnRandomEnemy()
    {
        var spawnPoint = _spawnPoints[Random.Range(0, _spawnPoints.Length)];
        
        // Rent enemy from pool
        var enemy = _enemyPool.Rent();
        
        // Position enemy at spawn point
        enemy.transform.position = spawnPoint.position;
        enemy.transform.rotation = spawnPoint.rotation;
        
        // Configure enemy
        ConfigureEnemy(enemy, Random.Range(1, 5));
        
        // Schedule automatic return
        StartCoroutine(ReturnEnemyAfterLifetime(enemy, 10f));
    }
    
    private void ConfigureEnemy(SceneEntity enemy, int level)
    {
        // Set enemy properties
        enemy.Set("Level", level);
        enemy.Set("Health", 50f + (level * 20f));
        enemy.Set("Speed", 3f + (level * 0.5f));
        enemy.Set("Damage", 10f + (level * 5f));
        enemy.Set("SpawnTime", Time.time);
        
        // Add tags
        enemy.AddTag("Enemy");
        enemy.AddTag("Active");
        enemy.AddTag($"Level{level}");
        
        // Subscribe to death event
        enemy.OnHealthDepleted += () => OnEnemyDeath(enemy);
    }
    
    private void OnEnemyDeath(SceneEntity enemy)
    {
        // Create death effect
        CreateDeathEffect(enemy.transform.position);
        
        // Award experience
        int level = enemy.Get<int>("Level");
        GameManager.Instance.AwardExperience(level * 10);
        
        // Return enemy to pool
        ReturnEnemy(enemy);
    }
    
    private void CreateDeathEffect(Vector3 position)
    {
        var effect = _effectPool.Rent();
        effect.transform.position = position;
        effect.Set("EffectType", "Death");
        effect.Set("Duration", 2f);
        
        StartCoroutine(ReturnEffectAfterDuration(effect, 2f));
    }
    
    private IEnumerator ReturnEnemyAfterLifetime(SceneEntity enemy, float lifetime)
    {
        yield return new WaitForSeconds(lifetime);
        
        if (enemy != null && enemy.gameObject.activeInHierarchy)
        {
            ReturnEnemy(enemy);
        }
    }
    
    private IEnumerator ReturnEffectAfterDuration(SceneEntity effect, float duration)
    {
        yield return new WaitForSeconds(duration);
        
        if (effect != null)
        {
            _effectPool.Return(effect);
        }
    }
    
    private void ReturnEnemy(SceneEntity enemy)
    {
        // Clean enemy state
        enemy.RemoveTag("Active");
        enemy.RemoveTag("Damaged");
        
        // Remove level tags
        for (int i = 1; i <= 10; i++)
        {
            enemy.RemoveTag($"Level{i}");
        }
        
        enemy.UnsubscribeAll();
        
        // Return to pool
        _enemyPool.Return(enemy);
    }
    
    private bool ShouldSpawnEnemy()
    {
        // Check if we should spawn based on game state
        return Random.Range(0f, 1f) < 0.7f; // 70% chance
    }
    
    void OnDestroy()
    {
        _enemyPool?.Dispose();
        _projectilePool?.Dispose();
        _effectPool?.Dispose();
    }
}
```

### Advanced Projectile Pool with Physics

```csharp
public class ProjectilePoolManager : MonoBehaviour
{
    [Header("Projectile Pool")]
    [SerializeField] private SceneEntityPool _projectilePool;
    
    [Header("Projectile Configuration")]
    [SerializeField] private LayerMask _targetLayers;
    [SerializeField] private float _defaultSpeed = 20f;
    [SerializeField] private float _defaultLifetime = 5f;
    [SerializeField] private float _defaultDamage = 25f;
    
    [Header("Pool Statistics")]
    [SerializeField] private bool _enableStatistics = true;
    
    // Statistics tracking
    private int _totalProjectilesFired;
    private int _totalProjectilesHit;
    private int _totalProjectilesExpired;
    private readonly Dictionary<SceneEntity, float> _activeProjectiles = new();
    
    void Start()
    {
        // Pool should be pre-configured in Inspector
        if (_enableStatistics)
        {
            InvokeRepeating(nameof(LogStatistics), 10f, 10f);
        }
    }
    
    public SceneEntity FireProjectile(Vector3 origin, Vector3 direction, float? speed = null, float? damage = null)
    {
        // Rent projectile from pool
        var projectile = _projectilePool.Rent();
        
        // Configure projectile
        ConfigureProjectile(projectile, origin, direction, speed ?? _defaultSpeed, damage ?? _defaultDamage);
        
        // Track projectile
        _activeProjectiles[projectile] = Time.time;
        _totalProjectilesFired++;
        
        // Start projectile physics
        StartProjectilePhysics(projectile);
        
        return projectile;
    }
    
    private void ConfigureProjectile(SceneEntity projectile, Vector3 origin, Vector3 direction, float speed, float damage)
    {
        // Position and orient projectile
        projectile.transform.position = origin;
        projectile.transform.rotation = Quaternion.LookRotation(direction);
        
        // Set projectile properties
        projectile.Set("Direction", direction);
        projectile.Set("Speed", speed);
        projectile.Set("Damage", damage);
        projectile.Set("Lifetime", _defaultLifetime);
        projectile.Set("LaunchTime", Time.time);
        projectile.Set("DistanceTraveled", 0f);
        
        // Add tags
        projectile.AddTag("Projectile");
        projectile.AddTag("Active");
        
        // Enable physics components
        var rigidbody = projectile.GetComponent<Rigidbody>();
        if (rigidbody != null)
        {
            rigidbody.velocity = direction * speed;
            rigidbody.isKinematic = false;
        }
        
        var collider = projectile.GetComponent<Collider>();
        if (collider != null)
        {
            collider.enabled = true;
            collider.isTrigger = true; // For trigger-based collision detection
        }
        
        // Subscribe to collision events
        projectile.OnTriggerEnter += (other) => OnProjectileCollision(projectile, other);
    }
    
    private void StartProjectilePhysics(SceneEntity projectile)
    {
        StartCoroutine(ManageProjectileLifetime(projectile));
    }
    
    private IEnumerator ManageProjectileLifetime(SceneEntity projectile)
    {
        float lifetime = projectile.Get<float>("Lifetime");
        float launchTime = projectile.Get<float>("LaunchTime");
        Vector3 lastPosition = projectile.transform.position;
        
        while (Time.time - launchTime < lifetime && projectile.gameObject.activeInHierarchy)
        {
            // Update distance traveled
            float distanceDelta = Vector3.Distance(projectile.transform.position, lastPosition);
            float totalDistance = projectile.Get<float>("DistanceTraveled") + distanceDelta;
            projectile.Set("DistanceTraveled", totalDistance);
            lastPosition = projectile.transform.position;
            
            yield return new WaitForFixedUpdate();
        }
        
        // Projectile expired
        if (projectile.gameObject.activeInHierarchy)
        {
            _totalProjectilesExpired++;
            ReturnProjectile(projectile, "Expired");
        }
    }
    
    private void OnProjectileCollision(SceneEntity projectile, Collider other)
    {
        // Check if target is valid
        if ((_targetLayers.value & (1 << other.gameObject.layer)) == 0)
            return;
        
        // Get impact information
        Vector3 impactPoint = other.ClosestPoint(projectile.transform.position);
        Vector3 impactNormal = (projectile.transform.position - impactPoint).normalized;
        
        // Handle collision
        HandleProjectileImpact(projectile, other, impactPoint, impactNormal);
        
        _totalProjectilesHit++;
        ReturnProjectile(projectile, "Hit");
    }
    
    private void HandleProjectileImpact(SceneEntity projectile, Collider target, Vector3 impactPoint, Vector3 impactNormal)
    {
        float damage = projectile.Get<float>("Damage");
        
        // Apply damage to target if it has a SceneEntity
        var targetEntity = target.GetComponent<SceneEntity>();
        if (targetEntity != null && targetEntity.HasValue("Health"))
        {
            ApplyDamage(targetEntity, damage, impactPoint);
        }
        
        // Apply physics impulse
        var targetRigidbody = target.GetComponent<Rigidbody>();
        if (targetRigidbody != null)
        {
            Vector3 impulseDirection = projectile.Get<Vector3>("Direction");
            float impulseForce = projectile.Get<float>("Speed") * 0.5f; // Scale impulse
            targetRigidbody.AddForceAtPosition(impulseDirection * impulseForce, impactPoint, ForceMode.Impulse);
        }
        
        // Create impact effects
        CreateImpactEffects(impactPoint, impactNormal);
        
        // Log impact details
        Debug.Log($"Projectile hit {target.name} at {impactPoint} for {damage} damage");
    }
    
    private void ApplyDamage(SceneEntity target, float damage, Vector3 impactPoint)
    {
        float currentHealth = target.Get<float>("Health");
        float newHealth = currentHealth - damage;
        
        target.Set("Health", newHealth);
        target.Set("LastDamageTime", Time.time);
        target.Set("LastImpactPoint", impactPoint);
        
        // Check for death
        if (newHealth <= 0)
        {
            target.AddTag("Dead");
            GameEvents.OnEntityKilled?.Invoke(target);
        }
        
        // Trigger damage events
        GameEvents.OnDamageDealt?.Invoke(target, damage, impactPoint);
    }
    
    private void CreateImpactEffects(Vector3 position, Vector3 normal)
    {
        // Create particle effect at impact point
        // This could use another pool for effect entities
        
        // Play impact sound
        AudioManager.Instance?.PlaySFX("ProjectileImpact", position);
        
        // Create visual effect
        EffectManager.Instance?.CreateImpactEffect(position, normal);
    }
    
    public void ReturnProjectile(SceneEntity projectile, string reason = "Manual")
    {
        if (!_activeProjectiles.ContainsKey(projectile))
            return;
        
        // Log projectile statistics
        if (_enableStatistics)
        {
            float activeTime = Time.time - _activeProjectiles[projectile];
            float distanceTraveled = projectile.Get<float>("DistanceTraveled");
            Debug.Log($"Projectile returned ({reason}): {activeTime:F2}s active, {distanceTraveled:F1}m traveled");
        }
        
        // Clean projectile state
        CleanProjectileForReturn(projectile);
        
        // Return to pool
        _projectilePool.Return(projectile);
        
        // Remove from tracking
        _activeProjectiles.Remove(projectile);
    }
    
    private void CleanProjectileForReturn(SceneEntity projectile)
    {
        // Stop physics
        var rigidbody = projectile.GetComponent<Rigidbody>();
        if (rigidbody != null)
        {
            rigidbody.velocity = Vector3.zero;
            rigidbody.angularVelocity = Vector3.zero;
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
        projectile.transform.rotation = Quaternion.identity;
        projectile.transform.localScale = Vector3.one;
    }
    
    // Public API for specific projectile types
    public SceneEntity FireBullet(Vector3 origin, Vector3 direction)
    {
        return FireProjectile(origin, direction, 30f, 15f);
    }
    
    public SceneEntity FireSlowProjectile(Vector3 origin, Vector3 direction)
    {
        return FireProjectile(origin, direction, 10f, 50f);
    }
    
    public SceneEntity FireFastProjectile(Vector3 origin, Vector3 direction)
    {
        return FireProjectile(origin, direction, 50f, 10f);
    }
    
    private void LogStatistics()
    {
        float hitRate = _totalProjectilesFired > 0 ? (float)_totalProjectilesHit / _totalProjectilesFired * 100f : 0f;
        
        Debug.Log($"Projectile Statistics - Fired: {_totalProjectilesFired}, " +
                 $"Hit: {_totalProjectilesHit}, Expired: {_totalProjectilesExpired}, " +
                 $"Hit Rate: {hitRate:F1}%, Active: {_activeProjectiles.Count}");
    }
    
    // Inspector debugging
    void OnGUI()
    {
        if (!_enableStatistics) return;
        
        int y = 10;
        GUI.Label(new Rect(10, y, 300, 20), "Projectile Pool Statistics:");
        y += 25;
        
        GUI.Label(new Rect(10, y, 200, 20), $"Fired: {_totalProjectilesFired}");
        y += 20;
        GUI.Label(new Rect(10, y, 200, 20), $"Hit: {_totalProjectilesHit}");
        y += 20;
        GUI.Label(new Rect(10, y, 200, 20), $"Expired: {_totalProjectilesExpired}");
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

### Custom Entity Pool with Enhanced Features

```csharp
public class EnhancedEntityPool : SceneEntityPool<SceneEntity>
{
    [Header("Enhanced Pool Features")]
    [SerializeField] private bool _enableWarmup = true;
    [SerializeField] private bool _enableStatistics = true;
    [SerializeField] private bool _enableAutoExpansion = false;
    [SerializeField] private int _expansionAmount = 5;
    [SerializeField] private float _expansionThreshold = 0.8f;
    
    // Enhanced statistics
    private int _totalCreated;
    private int _totalRented;
    private int _totalReturned;
    private int _peakRented;
    private float _averageRentalTime;
    private readonly Dictionary<SceneEntity, float> _rentalTimes = new();
    
    protected override void Awake()
    {
        base.Awake();
        
        if (_enableWarmup)
        {
            StartCoroutine(WarmupPool());
        }
    }
    
    private IEnumerator WarmupPool()
    {
        // Gradually create and return entities to warm up the pool
        for (int i = 0; i < _initialCount; i++)
        {
            var entity = base.Rent();
            yield return new WaitForEndOfFrame();
            base.Return(entity);
            yield return new WaitForEndOfFrame();
        }
        
        Debug.Log($"Pool warmup completed for {_initialCount} entities");
    }
    
    public override SceneEntity Rent()
    {
        // Check for expansion need
        if (_enableAutoExpansion && ShouldExpandPool())
        {
            ExpandPool();
        }
        
        var entity = base.Rent();
        
        // Enhanced tracking
        if (_enableStatistics)
        {
            _totalRented++;
            _peakRented = Mathf.Max(_peakRented, _rentEntities.Count);
            _rentalTimes[entity] = Time.time;
        }
        
        return entity;
    }
    
    public override void Return(SceneEntity entity)
    {
        // Enhanced tracking
        if (_enableStatistics && _rentalTimes.TryGetValue(entity, out float rentalStart))
        {
            float rentalDuration = Time.time - rentalStart;
            UpdateAverageRentalTime(rentalDuration);
            _rentalTimes.Remove(entity);
            _totalReturned++;
        }
        
        base.Return(entity);
    }
    
    protected override void OnCreate(SceneEntity entity)
    {
        base.OnCreate(entity);
        
        // Enhanced entity initialization
        entity.Set("PoolId", this.GetInstanceID());
        entity.Set("CreationTime", Time.time);
        entity.Set("RentalCount", 0);
        entity.AddTag("Pooled");
        
        if (_enableStatistics)
        {
            _totalCreated++;
        }
        
        Debug.Log($"Enhanced pool created entity #{_totalCreated}");
    }
    
    protected override void OnRent(SceneEntity entity)
    {
        base.OnRent(entity);
        
        // Enhanced rental processing
        int rentalCount = entity.Get<int>("RentalCount") + 1;
        entity.Set("RentalCount", rentalCount);
        entity.Set("LastRentalTime", Time.time);
        entity.AddTag("Active");
        
        // Performance warning for high-frequency rentals
        if (rentalCount > 100)
        {
            Debug.LogWarning($"Entity has high rental count: {rentalCount}");
        }
    }
    
    protected override void OnReturn(SceneEntity entity)
    {
        // Enhanced return processing
        entity.Set("LastReturnTime", Time.time);
        entity.RemoveTag("Active");
        
        base.OnReturn(entity);
    }
    
    protected override void OnDispose(SceneEntity entity)
    {
        if (_enableStatistics)
        {
            int rentalCount = entity.Get<int>("RentalCount");
            float totalLifetime = Time.time - entity.Get<float>("CreationTime");
            Debug.Log($"Disposing entity: {rentalCount} rentals, {totalLifetime:F2}s lifetime");
        }
        
        base.OnDispose(entity);
    }
    
    private bool ShouldExpandPool()
    {
        if (_pooledEntities.Count == 0 && _rentEntities.Count > 0)
        {
            float utilization = (float)_rentEntities.Count / (_rentEntities.Count + _pooledEntities.Count + 1);
            return utilization > _expansionThreshold;
        }
        return false;
    }
    
    private void ExpandPool()
    {
        Debug.Log($"Expanding pool by {_expansionAmount} entities");
        
        for (int i = 0; i < _expansionAmount; i++)
        {
            var entity = CreateEntity();
            _pooledEntities.Push(entity);
        }
    }
    
    private void UpdateAverageRentalTime(float rentalDuration)
    {
        if (_totalReturned == 1)
        {
            _averageRentalTime = rentalDuration;
        }
        else
        {
            _averageRentalTime = ((_averageRentalTime * (_totalReturned - 1)) + rentalDuration) / _totalReturned;
        }
    }
    
    // Public statistics API
    public PoolStatistics GetStatistics()
    {
        return new PoolStatistics
        {
            totalCreated = _totalCreated,
            totalRented = _totalRented,
            totalReturned = _totalReturned,
            currentRented = _rentEntities.Count,
            currentPooled = _pooledEntities.Count,
            peakRented = _peakRented,
            averageRentalTime = _averageRentalTime
        };
    }
    
    [System.Serializable]
    public struct PoolStatistics
    {
        public int totalCreated;
        public int totalRented;
        public int totalReturned;
        public int currentRented;
        public int currentPooled;
        public int peakRented;
        public float averageRentalTime;
    }
    
    // Debug visualization
    void OnDrawGizmosSelected()
    {
        if (!_enableStatistics) return;
        
        // Visual representation of pool state
        Gizmos.color = Color.green;
        Gizmos.DrawWireSphere(transform.position, 0.5f + _pooledEntities.Count * 0.1f);
        
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(transform.position + Vector3.up, 0.5f + _rentEntities.Count * 0.1f);
    }
    
    // Context menu for testing
    [ContextMenu("Test Rent")]
    private void TestRent()
    {
        var entity = Rent();
        Debug.Log($"Test rented entity: {entity.name}");
    }
    
    [ContextMenu("Test Return All")]
    private void TestReturnAll()
    {
        foreach (var entity in _rentEntities.ToArray())
        {
            Return(entity);
        }
        Debug.Log("Returned all rented entities");
    }
    
    [ContextMenu("Log Statistics")]
    private void LogStatistics()
    {
        var stats = GetStatistics();
        Debug.Log($"Pool Statistics - Created: {stats.totalCreated}, " +
                 $"Rented: {stats.totalRented}, Returned: {stats.totalReturned}, " +
                 $"Current Rented: {stats.currentRented}, Pooled: {stats.currentPooled}, " +
                 $"Peak: {stats.peakRented}, Avg Rental: {stats.averageRentalTime:F2}s");
    }
}
```

## Integration with Atomic Framework

### Reactive Pool Monitoring

```csharp
public class ReactiveScenePoolMonitor : MonoBehaviour
{
    [SerializeField] private EnhancedEntityPool _monitoredPool;
    
    [Header("Reactive Monitoring")]
    private readonly ReactiveInt _currentRented = new(0);
    private readonly ReactiveInt _currentPooled = new(0);
    private readonly ReactiveFloat _poolUtilization = new(0f);
    
    void Start()
    {
        // React to pool state changes
        _currentRented.Subscribe(OnRentedCountChanged);
        _currentPooled.Subscribe(OnPooledCountChanged);
        _poolUtilization.Subscribe(OnUtilizationChanged);
        
        // Start monitoring
        InvokeRepeating(nameof(UpdatePoolMetrics), 1f, 1f);
    }
    
    private void UpdatePoolMetrics()
    {
        var stats = _monitoredPool.GetStatistics();
        
        _currentRented.Value = stats.currentRented;
        _currentPooled.Value = stats.currentPooled;
        
        float utilization = stats.currentRented + stats.currentPooled > 0 
            ? (float)stats.currentRented / (stats.currentRented + stats.currentPooled)
            : 0f;
        _poolUtilization.Value = utilization;
    }
    
    private void OnRentedCountChanged(int count)
    {
        Debug.Log($"Rented entities: {count}");
        
        if (count > 15)
        {
            Debug.LogWarning("High rental count - consider pool expansion");
        }
    }
    
    private void OnPooledCountChanged(int count)
    {
        Debug.Log($"Pooled entities: {count}");
        
        if (count < 2)
        {
            Debug.LogWarning("Pool running low");
        }
    }
    
    private void OnUtilizationChanged(float utilization)
    {
        Debug.Log($"Pool utilization: {utilization:P}");
        
        if (utilization > 0.9f)
        {
            GameEvents.OnPoolHighUtilization?.Invoke(_monitoredPool.name);
        }
    }
}
```

## Implementation Notes

### Unity Integration
- MonoBehaviour lifecycle provides Unity integration
- SerializeField attributes enable Inspector configuration
- Component menu integration for easy scene placement

### Prefab Management
- Single prefab source per pool instance
- Automatic instantiation using SceneEntity.Create method
- Container Transform organization for scene hierarchy

### GameObject Lifecycle
- Automatic activation/deactivation during rent/return
- Transform parenting and positioning support
- Clean disposal with GameObject destruction

## Best Practices

### Pool Configuration
- Size pools based on expected concurrent usage
- Use meaningful prefab assignments for pool identification
- Configure container transforms for scene organization

### Performance Optimization
- Enable initialization on Awake for predictable performance
- Monitor pool statistics to optimize sizing
- Use lifecycle hooks for performance profiling

### Unity Workflow Integration
- Place pools strategically in scene hierarchy
- Use Inspector configuration for design-time setup
- Leverage Unity's profiling tools for performance analysis

## Common Patterns

### Pool Manager Component

```csharp
public class PoolManagerComponent : MonoBehaviour
{
    [System.Serializable]
    public struct PoolDefinition
    {
        public string poolName;
        public SceneEntity prefab;
        public int initialSize;
        public Transform container;
    }
    
    [SerializeField] private PoolDefinition[] _poolDefinitions;
    private readonly Dictionary<string, SceneEntityPool> _pools = new();
    
    void Awake()
    {
        CreatePools();
    }
    
    private void CreatePools()
    {
        foreach (var definition in _poolDefinitions)
        {
            var poolGO = new GameObject($"Pool_{definition.poolName}");
            poolGO.transform.SetParent(transform);
            
            var pool = poolGO.AddComponent<SceneEntityPool>();
            // Configure pool with definition
            
            _pools[definition.poolName] = pool;
        }
    }
    
    public SceneEntity Rent(string poolName)
    {
        return _pools.TryGetValue(poolName, out var pool) ? pool.Rent() : null;
    }
    
    public void Return(string poolName, SceneEntity entity)
    {
        if (_pools.TryGetValue(poolName, out var pool))
        {
            pool.Return(entity);
        }
    }
}
```

### Static Pool Factory

```csharp
public static class PoolFactory
{
    public static SceneEntityPool CreatePool(SceneEntity prefab, int size, Transform parent = null)
    {
        var poolGO = new GameObject($"Pool_{prefab.name}");
        if (parent) poolGO.transform.SetParent(parent);
        
        var args = new SceneEntityPool.CreateArgs
        {
            name = poolGO.name,
            prefab = prefab,
            container = poolGO.transform,
            initOnAwake = true,
            initialCount = size
        };
        
        return SceneEntityPool.Create(args);
    }
}
```

The `SceneEntityPool` provides comprehensive Unity integration for single-prefab entity pooling, featuring sophisticated GameObject lifecycle management, Inspector configuration, and performance optimization capabilities within the Atomic framework's reactive architecture.