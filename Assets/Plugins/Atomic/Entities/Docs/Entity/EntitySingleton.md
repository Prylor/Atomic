# 🧩 EntitySingleton

`EntitySingleton` provides a singleton pattern implementation for entities, ensuring only one instance of a specific entity type exists globally. It's useful for managing unique game systems like player controllers, game managers, or global state holders.

## Key Features

- **Singleton Pattern** – Ensures single instance per type
- **Lazy Initialization** – Creates instance on first access
- **Type Safety** – Generic implementation for any entity type
- **Thread Safe** – Protected against race conditions
- **Auto-Registration** – Automatically registers with EntityRegistry

---

## Class Definition

```csharp
public class EntitySingleton<T> where T : IEntity, new()
{
    private static T instance;
    private static readonly object lockObject = new object();
    
    public static T Instance
    {
        get
        {
            if (instance == null)
            {
                lock (lockObject)
                {
                    if (instance == null)
                    {
                        instance = new T();
                        Initialize(instance);
                    }
                }
            }
            return instance;
        }
    }
    
    protected static void Initialize(T entity)
    {
        // Override for custom initialization
    }
}
```

## Implementation Examples

### Player Singleton

```csharp
public class PlayerEntity : Entity
{
    public static PlayerEntity Instance => PlayerSingleton.Instance;
    
    private class PlayerSingleton : EntitySingleton<PlayerEntity>
    {
        protected override void Initialize(PlayerEntity entity)
        {
            entity.Name = "Player";
            entity.AddTag(EntityTags.PLAYER);
            entity.AddTag(EntityTags.PERSISTENT);
            entity.AddValue(EntityNames.HEALTH, 100);
            entity.AddValue(EntityNames.LEVEL, 1);
            entity.AddBehaviour(new PlayerInputBehaviour());
        }
    }
}

// Usage anywhere in code
var player = PlayerEntity.Instance;
player.SetValue(EntityNames.SCORE, 1000);
```

### Game Manager Singleton

```csharp
public class GameManagerEntity : Entity
{
    private static GameManagerEntity instance;
    
    public static GameManagerEntity Instance
    {
        get
        {
            if (instance == null)
            {
                instance = new GameManagerEntity();
                instance.Initialize();
            }
            return instance;
        }
    }
    
    private GameManagerEntity()
    {
        Name = "GameManager";
    }
    
    private void Initialize()
    {
        AddValue(EntityNames.GAME_STATE, GameState.Menu);
        AddValue(EntityNames.DIFFICULTY, Difficulty.Normal);
        AddValue(EntityNames.PLAY_TIME, 0f);
        
        AddBehaviour(new GameStateBehaviour());
        AddBehaviour(new SaveSystemBehaviour());
        
        Spawn();
    }
    
    public void ResetGame()
    {
        SetValue(EntityNames.GAME_STATE, GameState.Playing);
        SetValue(EntityNames.PLAY_TIME, 0f);
    }
}
```

### Service Locator Pattern

```csharp
public static class EntityServices
{
    private static Dictionary<Type, IEntity> services = new();
    
    public static T GetService<T>() where T : IEntity, new()
    {
        var type = typeof(T);
        
        if (!services.TryGetValue(type, out var service))
        {
            service = new T();
            services[type] = service;
            InitializeService(service);
        }
        
        return (T)service;
    }
    
    private static void InitializeService(IEntity service)
    {
        service.AddTag(EntityTags.SERVICE);
        service.Spawn();
    }
}

// Usage
var audioManager = EntityServices.GetService<AudioManagerEntity>();
var saveManager = EntityServices.GetService<SaveManagerEntity>();
```

### Lazy Singleton with Reset

```csharp
public class SessionEntity : Entity
{
    private static SessionEntity instance;
    
    public static SessionEntity Instance => instance ??= CreateInstance();
    
    public static void Reset()
    {
        if (instance != null)
        {
            instance.Despawn();
            instance.Dispose();
            instance = null;
        }
    }
    
    private static SessionEntity CreateInstance()
    {
        var session = new SessionEntity();
        session.Name = "Session";
        session.AddValue(EntityNames.SESSION_ID, Guid.NewGuid());
        session.AddValue(EntityNames.START_TIME, DateTime.Now);
        session.Spawn();
        return session;
    }
}
```

## Procedural Singleton Operations

```csharp
public static class SingletonUtils
{
    private static Dictionary<Type, IEntity> singletons = new();
    
    public static T GetOrCreateSingleton<T>(Func<T> factory = null) 
        where T : IEntity, new()
    {
        var type = typeof(T);
        
        if (!singletons.TryGetValue(type, out var entity))
        {
            entity = factory != null ? factory() : new T();
            singletons[type] = entity;
        }
        
        return (T)entity;
    }
    
    public static void DestroySingleton<T>() where T : IEntity
    {
        var type = typeof(T);
        
        if (singletons.TryGetValue(type, out var entity))
        {
            if (entity is IDisposable disposable)
                disposable.Dispose();
                
            singletons.Remove(type);
        }
    }
    
    public static void DestroyAllSingletons()
    {
        foreach (var entity in singletons.Values)
        {
            if (entity is IDisposable disposable)
                disposable.Dispose();
        }
        singletons.Clear();
    }
    
    public static bool HasSingleton<T>() where T : IEntity
    {
        return singletons.ContainsKey(typeof(T));
    }
}
```

## Unity Integration

### SceneEntitySingleton

```csharp
public class SceneEntitySingleton<T> : SceneEntity where T : SceneEntitySingleton<T>
{
    private static T instance;
    
    public static T Instance
    {
        get
        {
            if (instance == null)
            {
                instance = FindObjectOfType<T>();
                
                if (instance == null)
                {
                    var go = new GameObject(typeof(T).Name);
                    instance = go.AddComponent<T>();
                    instance.Install();
                }
            }
            return instance;
        }
    }
    
    protected virtual void Awake()
    {
        if (instance != null && instance != this)
        {
            Destroy(gameObject);
            return;
        }
        
        instance = (T)this;
        DontDestroyOnLoad(gameObject);
    }
    
    protected virtual void OnDestroy()
    {
        if (instance == this)
        {
            instance = null;
        }
    }
}

// Usage
public class GameController : SceneEntitySingleton<GameController>
{
    protected override void OnInstall()
    {
        // Initialize game controller
    }
}
```

## Thread-Safe Implementation

```csharp
public class ThreadSafeSingleton<T> where T : IEntity, new()
{
    private static readonly Lazy<T> lazy = 
        new Lazy<T>(() => 
        {
            var entity = new T();
            OnCreate(entity);
            return entity;
        });
    
    public static T Instance => lazy.Value;
    
    private static void OnCreate(T entity)
    {
        // Initialization logic
        if (entity is Entity e)
        {
            e.Name = typeof(T).Name + "_Singleton";
            e.AddTag(EntityTags.SINGLETON);
        }
    }
}
```

## Best Practices

1. **Lazy Initialization** – Create only when needed
2. **Thread Safety** – Use locks or Lazy<T> for thread safety
3. **Cleanup** – Provide reset/cleanup methods
4. **Avoid Overuse** – Don't make everything a singleton
5. **Testing** – Provide ways to reset for tests

## Common Patterns

### Auto-Spawn Singleton
```csharp
public static T GetSpawnedSingleton<T>() where T : IEntity, new()
{
    var entity = GetOrCreateSingleton<T>();
    
    if (!entity.IsSpawned)
        entity.Spawn();
        
    return entity;
}
```

### Conditional Singleton
```csharp
public static T GetSingletonIf<T>(Func<bool> condition) 
    where T : IEntity, new()
{
    return condition() ? GetOrCreateSingleton<T>() : null;
}
```

## Anti-Patterns to Avoid

```csharp
// DON'T: Singleton with mutable global state
public class BadSingleton : Entity
{
    public static List<Enemy> AllEnemies = new(); // Bad!
}

// DO: Use entity values instead
public class GoodSingleton : Entity
{
    public static GoodSingleton Instance { get; }
    
    public List<Enemy> GetEnemies()
    {
        return GetValue<List<Enemy>>(EntityNames.ENEMIES);
    }
}
```

## Notes

- Singletons can make testing harder
- Consider dependency injection as alternative
- Be careful with singleton lifecycle in Unity
- Use sparingly for truly global objects
- Thread safety adds overhead