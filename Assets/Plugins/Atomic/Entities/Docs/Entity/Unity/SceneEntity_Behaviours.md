# ⚙️ SceneEntity_Behaviours

The `SceneEntity_Behaviours` partial class provides comprehensive behaviour management functionality for `SceneEntity` instances. It implements the `IEntity_Behaviours` interface to handle addition, removal, querying, and lifecycle management of entity behaviours in Unity-specific contexts.

## Key Features

- **Dynamic Behaviour Management** – Add/remove behaviours at runtime
- **Type-Safe Queries** – Generic methods for behaviour access
- **Lifecycle Integration** – Automatic lifecycle events for behaviours
- **Performance Optimized** – Array pool usage and efficient storage
- **Event-Driven** – Events for behaviour addition/removal
- **Enumeration Support** – Custom enumerator for efficient iteration

---

## Core Properties and Events

### Properties
```csharp
// Behaviour count
public int BehaviourCount { get; }

// Events
public event Action<IEntity, IEntityBehaviour> OnBehaviourAdded;
public event Action<IEntity, IEntityBehaviour> OnBehaviourDeleted;
```

### Internal Storage
- Uses dynamic array with automatic resizing
- Leverages `ArrayPool<IEntityBehaviour>.Shared` for temporary operations
- Maintains behaviour count for efficient operations

## Behaviour Management Methods

### Adding Behaviours
```csharp
// Add a behaviour instance
public void AddBehaviour(IEntityBehaviour behaviour)

// Automatic lifecycle integration
// - Calls OnSpawn() if entity is spawned
// - Calls OnActivate() if entity is active
// - Fires OnBehaviourAdded event
```

### Removing Behaviours
```csharp
// Remove first behaviour of type T
public bool DelBehaviour<T>() where T : IEntityBehaviour

// Remove all behaviours of type T
public void DelBehaviours<T>() where T : IEntityBehaviour

// Remove specific behaviour instance
public bool DelBehaviour(IEntityBehaviour behaviour)

// Remove behaviour at index
public bool DelBehaviourAt(int index)

// Remove all behaviours
public void ClearBehaviours()
```

### Querying Behaviours
```csharp
// Check if behaviour exists
public bool HasBehaviour(IEntityBehaviour behaviour)
public bool HasBehaviour<T>() where T : IEntityBehaviour

// Get behaviour (throws if not found)
public T GetBehaviour<T>() where T : IEntityBehaviour

// Try to get behaviour (safe)
public bool TryGetBehaviour<T>(out T behaviour) where T : IEntityBehaviour

// Get behaviour by index
public IEntityBehaviour GetBehaviourAt(int index)
```

### Retrieving Collections
```csharp
// Get all behaviours as new array
public IEntityBehaviour[] GetBehaviours()

// Get behaviours of specific type
public T[] GetBehaviours<T>() where T : IEntityBehaviour

// Copy behaviours to existing array
public int CopyBehaviours(IEntityBehaviour[] results)
public int CopyBehaviours<T>(T[] results) where T : IEntityBehaviour
```

### Enumeration
```csharp
// Get custom enumerator
public BehaviourEnumerator GetBehaviourEnumerator()

// Support for IEntity interface
IEnumerator<IEntityBehaviour> IEntity.GetBehaviourEnumerator()
```

## Example Usage

### Basic Behaviour Management

```csharp
public class PlayerEntity : SceneEntity
{
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private int maxHealth = 100;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Add various behaviours
        this.AddBehaviour(new PlayerInputBehaviour());
        this.AddBehaviour(new MovementBehaviour(moveSpeed));
        this.AddBehaviour(new HealthBehaviour(maxHealth));
        this.AddBehaviour(new AnimationBehaviour());
        
        // Listen for behaviour changes
        this.OnBehaviourAdded += OnBehaviourAdded;
        this.OnBehaviourDeleted += OnBehaviourDeleted;
        
        Debug.Log($"Player initialized with {BehaviourCount} behaviours");
    }
    
    private void OnBehaviourAdded(IEntity entity, IEntityBehaviour behaviour)
    {
        Debug.Log($"Added behaviour: {behaviour.GetType().Name}");
        
        // React to specific behaviour types
        if (behaviour is PlayerInputBehaviour input)
        {
            input.SetMoveSpeed(moveSpeed);
        }
    }
    
    private void OnBehaviourDeleted(IEntity entity, IEntityBehaviour behaviour)
    {
        Debug.Log($"Removed behaviour: {behaviour.GetType().Name}");
    }
    
    public void EnableGodMode()
    {
        // Add temporary behaviour
        if (!this.HasBehaviour<GodModeBehaviour>())
        {
            this.AddBehaviour(new GodModeBehaviour());
        }
    }
    
    public void DisableGodMode()
    {
        // Remove specific behaviour type
        this.DelBehaviour<GodModeBehaviour>();
    }
}
```

### Dynamic Behaviour System

```csharp
public class AdaptiveEnemy : SceneEntity
{
    [SerializeField] private float difficultyLevel = 1.0f;
    [SerializeField] private float adaptationThreshold = 5.0f;
    
    private float timeSinceLastHit;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Start with basic AI
        this.AddBehaviour(new BasicEnemyAI());
        this.AddBehaviour(new PatrolBehaviour());
        
        // Add adaptation behaviour
        this.AddBehaviour(new AdaptationBehaviour());
    }
    
    protected override void OnUpdate(float deltaTime)
    {
        base.OnUpdate(deltaTime);
        
        timeSinceLastHit += deltaTime;
        
        // Adapt difficulty based on player performance
        if (timeSinceLastHit > adaptationThreshold)
        {
            AdaptDifficulty();
            timeSinceLastHit = 0;
        }
    }
    
    private void AdaptDifficulty()
    {
        difficultyLevel += 0.2f;
        
        // Remove old behaviours
        this.DelBehaviour<BasicEnemyAI>();
        this.DelBehaviour<PatrolBehaviour>();
        
        // Add advanced behaviours based on difficulty
        if (difficultyLevel > 2.0f)
        {
            this.AddBehaviour(new AdvancedEnemyAI());
            this.AddBehaviour(new AggressiveBehaviour());
        }
        
        if (difficultyLevel > 3.0f)
        {
            this.AddBehaviour(new PredictiveBehaviour());
            this.AddBehaviour(new FlankingBehaviour());
        }
        
        Debug.Log($"Enemy adapted to difficulty level: {difficultyLevel}");
        Debug.Log($"Active behaviours: {BehaviourCount}");
    }
    
    public void OnPlayerHit()
    {
        timeSinceLastHit = 0;
        
        // Remove aggressive behaviours when successful
        if (difficultyLevel > 2.0f)
        {
            this.DelBehaviour<AggressiveBehaviour>();
            difficultyLevel = Mathf.Max(1.0f, difficultyLevel - 0.1f);
        }
    }
}
```

### Behaviour State Management

```csharp
public class StateMachineEntity : SceneEntity
{
    public enum State
    {
        Idle,
        Moving,
        Attacking,
        Defending,
        Stunned
    }
    
    private State currentState = State.Idle;
    private Dictionary<State, List<Type>> stateBehaviours;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Define behaviours for each state
        stateBehaviours = new Dictionary<State, List<Type>>
        {
            [State.Idle] = new List<Type> { typeof(IdleBehaviour), typeof(LookAroundBehaviour) },
            [State.Moving] = new List<Type> { typeof(MovementBehaviour), typeof(NavigationBehaviour) },
            [State.Attacking] = new List<Type> { typeof(AttackBehaviour), typeof(TargetingBehaviour) },
            [State.Defending] = new List<Type> { typeof(DefenseBehaviour), typeof(BlockBehaviour) },
            [State.Stunned] = new List<Type> { typeof(StunnedBehaviour) }
        };
        
        // Start in idle state
        ChangeState(State.Idle);
    }
    
    public void ChangeState(State newState)
    {
        if (currentState == newState) return;
        
        Debug.Log($"Changing state from {currentState} to {newState}");
        
        // Remove behaviours of current state
        RemoveStateBehaviours(currentState);
        
        // Add behaviours of new state
        AddStateBehaviours(newState);
        
        currentState = newState;
    }
    
    private void RemoveStateBehaviours(State state)
    {
        if (!stateBehaviours.ContainsKey(state)) return;
        
        foreach (var behaviourType in stateBehaviours[state])
        {
            // Remove all behaviours of this type
            var behaviours = this.GetBehaviours();
            foreach (var behaviour in behaviours)
            {
                if (behaviour.GetType() == behaviourType)
                {
                    this.DelBehaviour(behaviour);
                }
            }
        }
    }
    
    private void AddStateBehaviours(State state)
    {
        if (!stateBehaviours.ContainsKey(state)) return;
        
        foreach (var behaviourType in stateBehaviours[state])
        {
            // Create and add behaviour instance
            var behaviour = CreateBehaviourInstance(behaviourType);
            if (behaviour != null)
            {
                this.AddBehaviour(behaviour);
            }
        }
    }
    
    private IEntityBehaviour CreateBehaviourInstance(Type behaviourType)
    {
        // Simple factory method - in real implementation, use DI or factory pattern
        return Activator.CreateInstance(behaviourType) as IEntityBehaviour;
    }
    
    // State transition methods
    public void StartMoving() => ChangeState(State.Moving);
    public void StartAttacking() => ChangeState(State.Attacking);
    public void StartDefending() => ChangeState(State.Defending);
    public void GetStunned() => ChangeState(State.Stunned);
    public void StopAction() => ChangeState(State.Idle);
}
```

### Behaviour Query and Analysis

```csharp
public class BehaviourAnalyzer : MonoBehaviour
{
    [SerializeField] private SceneEntity targetEntity;
    
    void Start()
    {
        if (targetEntity != null)
        {
            AnalyzeEntityBehaviours();
        }
    }
    
    private void AnalyzeEntityBehaviours()
    {
        Debug.Log($"=== Behaviour Analysis for {targetEntity.Name} ===");
        Debug.Log($"Total behaviours: {targetEntity.BehaviourCount}");
        
        // Get all behaviours
        var allBehaviours = targetEntity.GetBehaviours();
        
        // Categorize behaviours
        var updateBehaviours = new List<IEntityUpdate>();
        var spawnBehaviours = new List<IEntitySpawn>();
        var activeBehaviours = new List<IEntityActivate>();
        
        foreach (var behaviour in allBehaviours)
        {
            Debug.Log($"- {behaviour.GetType().Name}");
            
            if (behaviour is IEntityUpdate update)
                updateBehaviours.Add(update);
            
            if (behaviour is IEntitySpawn spawn)
                spawnBehaviours.Add(spawn);
            
            if (behaviour is IEntityActivate activate)
                activeBehaviours.Add(activate);
        }
        
        Debug.Log($"Update behaviours: {updateBehaviours.Count}");
        Debug.Log($"Spawn behaviours: {spawnBehaviours.Count}");
        Debug.Log($"Active behaviours: {activeBehaviours.Count}");
        
        // Test specific behaviour queries
        TestBehaviourQueries();
    }
    
    private void TestBehaviourQueries()
    {
        Debug.Log("=== Behaviour Query Tests ===");
        
        // Test HasBehaviour
        bool hasMovement = targetEntity.HasBehaviour<MovementBehaviour>();
        Debug.Log($"Has MovementBehaviour: {hasMovement}");
        
        // Test TryGetBehaviour
        if (targetEntity.TryGetBehaviour<HealthBehaviour>(out var healthBehaviour))
        {
            Debug.Log($"Found HealthBehaviour with max health: {healthBehaviour.MaxHealth}");
        }
        else
        {
            Debug.Log("No HealthBehaviour found");
        }
        
        // Test GetBehaviours<T>
        var inputBehaviours = targetEntity.GetBehaviours<IEntityUpdate>();
        Debug.Log($"Found {inputBehaviours.Length} update behaviours");
        
        // Test enumeration
        Debug.Log("All behaviours via enumerator:");
        var enumerator = targetEntity.GetBehaviourEnumerator();
        while (enumerator.MoveNext())
        {
            Debug.Log($"  - {enumerator.Current.GetType().Name}");
        }
        enumerator.Dispose();
    }
    
    [ContextMenu("Add Random Behaviour")]
    private void AddRandomBehaviour()
    {
        var behaviourTypes = new Type[]
        {
            typeof(MovementBehaviour),
            typeof(HealthBehaviour),
            typeof(AttackBehaviour),
            typeof(DefenseBehaviour),
            typeof(AnimationBehaviour)
        };
        
        var randomType = behaviourTypes[UnityEngine.Random.Range(0, behaviourTypes.Length)];
        var behaviour = Activator.CreateInstance(randomType) as IEntityBehaviour;
        
        targetEntity.AddBehaviour(behaviour);
        Debug.Log($"Added random behaviour: {randomType.Name}");
    }
    
    [ContextMenu("Remove Random Behaviour")]
    private void RemoveRandomBehaviour()
    {
        if (targetEntity.BehaviourCount == 0)
        {
            Debug.Log("No behaviours to remove");
            return;
        }
        
        int randomIndex = UnityEngine.Random.Range(0, targetEntity.BehaviourCount);
        var behaviour = targetEntity.GetBehaviourAt(randomIndex);
        
        targetEntity.DelBehaviourAt(randomIndex);
        Debug.Log($"Removed behaviour: {behaviour.GetType().Name}");
    }
}
```

### Performance Optimization with Behaviour Pooling

```csharp
public class BehaviourPool : MonoBehaviour
{
    private static readonly Dictionary<Type, Stack<IEntityBehaviour>> pools = 
        new Dictionary<Type, Stack<IEntityBehaviour>>();
    
    public static T GetBehaviour<T>() where T : IEntityBehaviour, new()
    {
        var type = typeof(T);
        
        if (pools.TryGetValue(type, out var pool) && pool.Count > 0)
        {
            return (T)pool.Pop();
        }
        
        return new T();
    }
    
    public static void ReturnBehaviour<T>(T behaviour) where T : IEntityBehaviour
    {
        var type = typeof(T);
        
        if (!pools.ContainsKey(type))
        {
            pools[type] = new Stack<IEntityBehaviour>();
        }
        
        // Reset behaviour state if needed
        if (behaviour is IResettable resettable)
        {
            resettable.Reset();
        }
        
        pools[type].Push(behaviour);
    }
}

public class PooledBehaviourEntity : SceneEntity
{
    private List<IEntityBehaviour> activeBehaviours = new List<IEntityBehaviour>();
    
    public void AddPooledBehaviour<T>() where T : IEntityBehaviour, new()
    {
        var behaviour = BehaviourPool.GetBehaviour<T>();
        this.AddBehaviour(behaviour);
        activeBehaviours.Add(behaviour);
    }
    
    public void RemovePooledBehaviour<T>() where T : IEntityBehaviour, new()
    {
        if (this.TryGetBehaviour<T>(out var behaviour))
        {
            this.DelBehaviour(behaviour);
            activeBehaviours.Remove(behaviour);
            BehaviourPool.ReturnBehaviour(behaviour);
        }
    }
    
    protected override void OnDespawn()
    {
        // Return all behaviours to pool
        foreach (var behaviour in activeBehaviours)
        {
            BehaviourPool.ReturnBehaviour(behaviour);
        }
        activeBehaviours.Clear();
        
        this.ClearBehaviours();
        base.OnDespawn();
    }
}
```

### Custom Behaviour Enumerator Usage

```csharp
public class BehaviourIterator : MonoBehaviour
{
    [SerializeField] private SceneEntity entity;
    
    void Start()
    {
        DemonstrateEnumeration();
    }
    
    private void DemonstrateEnumeration()
    {
        Debug.Log("=== Behaviour Enumeration Examples ===");
        
        // Method 1: Custom enumerator (most efficient)
        using (var enumerator = entity.GetBehaviourEnumerator())
        {
            while (enumerator.MoveNext())
            {
                var behaviour = enumerator.Current;
                ProcessBehaviour(behaviour);
            }
        }
        
        // Method 2: GetBehaviours() array
        var behaviours = entity.GetBehaviours();
        foreach (var behaviour in behaviours)
        {
            ProcessBehaviour(behaviour);
        }
        
        // Method 3: Index-based iteration
        for (int i = 0; i < entity.BehaviourCount; i++)
        {
            var behaviour = entity.GetBehaviourAt(i);
            ProcessBehaviour(behaviour);
        }
    }
    
    private void ProcessBehaviour(IEntityBehaviour behaviour)
    {
        Debug.Log($"Processing: {behaviour.GetType().Name}");
        
        // Example processing based on behaviour type
        switch (behaviour)
        {
            case IEntityUpdate updateBehaviour:
                Debug.Log("  - Has Update capability");
                break;
                
            case IEntityFixedUpdate fixedUpdateBehaviour:
                Debug.Log("  - Has FixedUpdate capability");
                break;
                
            case IEntitySpawn spawnBehaviour:
                Debug.Log("  - Has Spawn capability");
                break;
        }
    }
}
```

## Performance Considerations

### Memory Management
- Uses `ArrayPool<IEntityBehaviour>.Shared` for temporary operations
- Automatic array resizing with exponential growth
- Efficient memory usage with stackalloc for small operations

### Lifecycle Integration
- Behaviours automatically receive lifecycle events when added/removed
- Proper cleanup when entity is deactivated or despawned
- Event notifications for behaviour changes

### Query Optimization
- Linear search for type-based queries (O(n))
- Consider caching for frequently accessed behaviours
- Use TryGetBehaviour for optional behaviours

## Best Practices

1. **Lifecycle Awareness** – Add behaviours during OnInstall() when possible
2. **Type Safety** – Use generic methods for compile-time type checking
3. **Event Handling** – Subscribe to behaviour events for reactive systems
4. **Memory Efficiency** – Use TryGetBehaviour for optional access
5. **Performance** – Cache frequently accessed behaviours
6. **Cleanup** – Remove behaviours that are no longer needed
7. **Pooling** – Consider behaviour pooling for frequently created/destroyed behaviours

## Integration with Unity

### MonoBehaviour Lifecycle
- Behaviours receive entity lifecycle events
- Automatic handling of Unity's Start/OnEnable/OnDisable
- Proper cleanup on GameObject destruction

### Inspector Integration
- Behaviour count visible in inspector during runtime
- Custom editors can display behaviour information
- Debug methods for runtime behaviour manipulation

### Performance Profiling
- Track behaviour additions/removals in profiler
- Monitor memory allocations from behaviour arrays
- Use Unity's deep profiling to analyze behaviour performance

## Common Patterns

### Composite Behaviours
```csharp
public class CompositeBehaviour : IEntityBehaviour
{
    private List<IEntityBehaviour> subBehaviours = new List<IEntityBehaviour>();
    
    public void AddSubBehaviour(IEntityBehaviour behaviour)
    {
        subBehaviours.Add(behaviour);
    }
    
    public void OnInstall(IEntity entity)
    {
        foreach (var behaviour in subBehaviours)
            behaviour.OnInstall(entity);
    }
}
```

### Conditional Behaviours
```csharp
public void UpdateBehaviours()
{
    // Add behaviours based on conditions
    if (health < 50 && !this.HasBehaviour<LowHealthBehaviour>())
    {
        this.AddBehaviour(new LowHealthBehaviour());
    }
    
    // Remove behaviours when conditions change
    if (health >= 50 && this.HasBehaviour<LowHealthBehaviour>())
    {
        this.DelBehaviour<LowHealthBehaviour>();
    }
}
```

## Notes

- Behaviour management is thread-safe within Unity's main thread only
- Automatic lifecycle integration ensures proper behaviour state
- Events are fired synchronously during behaviour operations
- Array storage provides good performance for most use cases
- Custom enumerator avoids heap allocations during iteration
- Supports all standard entity behaviour interfaces