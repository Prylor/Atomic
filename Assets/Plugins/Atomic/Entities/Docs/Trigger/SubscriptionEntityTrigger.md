# 🧩 SubscriptionEntityTrigger

An abstract base trigger that manages entity monitoring through disposable subscriptions. Provides infrastructure for tracking entities using subscription-based patterns with automatic resource management and cleanup.

## Overview

`SubscriptionEntityTrigger` serves as a foundation for building triggers that use subscription-based monitoring patterns. It automatically manages `IDisposable` subscriptions for each tracked entity, ensuring proper resource cleanup when tracking stops. Essential for building triggers that integrate with reactive streams, observables, or other subscription-based monitoring systems.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// Base trigger for working with entities using subscriptions.
    /// Provides infrastructure for tracking entities and managing subscription resources.
    /// </summary>
    public abstract class SubscriptionEntityTrigger : SubscriptionEntityTrigger<IEntity>
    {
    }

    /// <summary>
    /// Abstract base trigger for entities that uses subscriptions.
    /// Maintains and manages IDisposable objects for each tracked entity
    /// and releases resources when tracking is stopped.
    /// </summary>
    /// <typeparam name="E">The type of entity implementing IEntity.</typeparam>
    public abstract class SubscriptionEntityTrigger<E> : EntityTriggerBase<E> where E : IEntity
    {
        private readonly Dictionary<IEntity, IDisposable> _subscriptions = new();

        public sealed override void Track(E entity)
        {
            if (!_subscriptions.ContainsKey(entity))
            {
                IDisposable subscription = this.Track(entity, _action);
                _subscriptions.Add(entity, subscription);
            }
        }

        public sealed override void Untrack(E entity)
        {
            if (_subscriptions.Remove(entity, out IDisposable subscription))
                subscription.Dispose();
        }

        protected abstract IDisposable Track(E entity, Action<E> callback);
    }
}
```

## Key Features

### Automatic Subscription Management
- Automatic storage and management of disposable subscriptions
- Guaranteed cleanup when entities are untracked
- Prevention of duplicate subscriptions for the same entity

### Resource Lifecycle Management
- Proper disposal of subscription resources
- Clean separation of subscription creation and management
- Thread-safe subscription handling

### Abstract Implementation Pattern
- Derived classes implement only subscription creation logic
- Base class handles all subscription management details
- Consistent resource management across all implementations

## Usage Examples

### Observable Value Change Trigger

```csharp
public class ObservableValueTrigger : SubscriptionEntityTrigger
{
    private readonly string _valueKey;
    private readonly IObservable<object> _valueStream;

    public ObservableValueTrigger(string valueKey)
    {
        _valueKey = valueKey ?? throw new ArgumentNullException(nameof(valueKey));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        // Create an observable stream for the specific value
        var valueObservable = Observable.Create<object>(observer =>
        {
            Action<IEntity, int> valueHandler = (e, key) =>
            {
                if (GetKeyName(key) == _valueKey)
                {
                    object newValue = e.Get<object>(_valueKey);
                    observer.OnNext(newValue);
                }
            };

            entity.OnValueChanged += valueHandler;

            return Disposable.Create(() =>
            {
                entity.OnValueChanged -= valueHandler;
            });
        });

        // Subscribe to the observable and return the subscription
        return valueObservable
            .DistinctUntilChanged()
            .Subscribe(value =>
            {
                callback(entity);
            });
    }

    private string GetKeyName(int key)
    {
        // Implementation would convert key ID to name
        // This is a simplified example
        return _valueKey;
    }
}

// Usage
var healthObservableTrigger = new ObservableValueTrigger("Health");
healthObservableTrigger.SetAction(entity =>
{
    Debug.Log($"Health changed for entity: {entity.Name}");
    float newHealth = entity.Get<float>("Health");
    GameEvents.OnHealthChanged?.Invoke(entity, newHealth);
});

healthObservableTrigger.Track(playerEntity);
```

### Timer-Based Polling Trigger

```csharp
public class TimerPollingTrigger : SubscriptionEntityTrigger
{
    private readonly TimeSpan _interval;
    private readonly Func<IEntity, bool> _condition;

    public TimerPollingTrigger(TimeSpan interval, Func<IEntity, bool> condition)
    {
        _interval = interval;
        _condition = condition ?? throw new ArgumentNullException(nameof(condition));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        var timer = new Timer(state =>
        {
            try
            {
                if (_condition(entity))
                {
                    callback(entity);
                }
            }
            catch (Exception ex)
            {
                Debug.LogError($"Error in timer polling trigger: {ex.Message}");
            }
        }, null, _interval, _interval);

        return timer;
    }
}

// Usage
var healthPollingTrigger = new TimerPollingTrigger(
    TimeSpan.FromSeconds(1.0), 
    entity => entity.HasValue("Health") && entity.Get<float>("Health") < 25f
);

healthPollingTrigger.SetAction(entity =>
{
    Debug.Log($"Entity {entity.Name} is critically low on health!");
    GameEvents.OnCriticalHealth?.Invoke(entity);
});

healthPollingTrigger.Track(playerEntity);
```

### Event Bus Integration Trigger

```csharp
public class EventBusTrigger : SubscriptionEntityTrigger
{
    private readonly IEventBus _eventBus;
    private readonly string _eventType;

    public EventBusTrigger(IEventBus eventBus, string eventType)
    {
        _eventBus = eventBus ?? throw new ArgumentNullException(nameof(eventBus));
        _eventType = eventType ?? throw new ArgumentNullException(nameof(eventType));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        // Subscribe to event bus for entity-specific events
        return _eventBus.Subscribe<EntityEvent>(_eventType, eventData =>
        {
            if (eventData.Entity == entity)
            {
                callback(entity);
            }
        });
    }
}

public class EntityEvent
{
    public IEntity Entity { get; set; }
    public string EventType { get; set; }
    public object EventData { get; set; }
}

// Usage
var combatEventTrigger = new EventBusTrigger(GameEventBus.Instance, "CombatEvent");
combatEventTrigger.SetAction(entity =>
{
    Debug.Log($"Combat event occurred for entity: {entity.Name}");
    GameEvents.OnEntityCombatEvent?.Invoke(entity);
});
```

### Reactive Extensions Integration

```csharp
public class RxValueStreamTrigger : SubscriptionEntityTrigger
{
    private readonly string[] _watchedKeys;
    private readonly TimeSpan _throttleTime;

    public RxValueStreamTrigger(TimeSpan throttleTime, params string[] watchedKeys)
    {
        _throttleTime = throttleTime;
        _watchedKeys = watchedKeys ?? throw new ArgumentNullException(nameof(watchedKeys));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        // Create observable streams for each watched key
        var streams = _watchedKeys.Select(key => CreateValueStream(entity, key));
        
        // Merge all streams and apply throttling
        return Observable.Merge(streams)
            .Throttle(_throttleTime)
            .Subscribe(_ => callback(entity));
    }

    private IObservable<object> CreateValueStream(IEntity entity, string key)
    {
        return Observable.Create<object>(observer =>
        {
            Action<IEntity, int> handler = (e, keyId) =>
            {
                if (GetKeyName(keyId) == key)
                {
                    observer.OnNext(e.Get<object>(key));
                }
            };

            entity.OnValueChanged += handler;

            return Disposable.Create(() =>
            {
                entity.OnValueChanged -= handler;
            });
        });
    }

    private string GetKeyName(int keyId)
    {
        // Implementation to convert key ID to name
        return "";
    }
}

// Usage
var throttledStatsTrigger = new RxValueStreamTrigger(
    TimeSpan.FromMilliseconds(500), 
    "Health", "Mana", "Stamina"
);

throttledStatsTrigger.SetAction(entity =>
{
    Debug.Log($"Stats updated for entity: {entity.Name} (throttled)");
    UpdateUI(entity);
});
```

### File System Watcher Integration

```csharp
public class ConfigFileWatcherTrigger : SubscriptionEntityTrigger
{
    private readonly string _configDirectory;

    public ConfigFileWatcherTrigger(string configDirectory)
    {
        _configDirectory = configDirectory ?? throw new ArgumentNullException(nameof(configDirectory));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        string configFile = Path.Combine(_configDirectory, $"{entity.Name}.config");
        
        if (!File.Exists(configFile))
        {
            // Return empty disposable if no config file exists
            return Disposable.Empty;
        }

        var watcher = new FileSystemWatcher(_configDirectory, $"{entity.Name}.config")
        {
            NotifyFilter = NotifyFilters.LastWrite | NotifyFilters.Size,
            EnableRaisingEvents = true
        };

        watcher.Changed += (sender, args) =>
        {
            Debug.Log($"Config file changed for entity: {entity.Name}");
            LoadEntityConfiguration(entity, args.FullPath);
            callback(entity);
        };

        return watcher;
    }

    private void LoadEntityConfiguration(IEntity entity, string configPath)
    {
        try
        {
            // Load configuration from file and apply to entity
            var config = File.ReadAllText(configPath);
            ApplyConfigurationToEntity(entity, config);
        }
        catch (Exception ex)
        {
            Debug.LogError($"Error loading configuration for {entity.Name}: {ex.Message}");
        }
    }

    private void ApplyConfigurationToEntity(IEntity entity, string config)
    {
        // Implementation would parse and apply configuration
        // This is a simplified example
    }
}

// Usage
var configWatcherTrigger = new ConfigFileWatcherTrigger("./EntityConfigs");
configWatcherTrigger.SetAction(entity =>
{
    Debug.Log($"Configuration reloaded for entity: {entity.Name}");
    GameEvents.OnEntityConfigurationUpdated?.Invoke(entity);
});
```

### Network Event Stream Trigger

```csharp
public class NetworkEventTrigger : SubscriptionEntityTrigger
{
    private readonly INetworkClient _networkClient;
    private readonly string _eventChannel;

    public NetworkEventTrigger(INetworkClient networkClient, string eventChannel)
    {
        _networkClient = networkClient ?? throw new ArgumentNullException(nameof(networkClient));
        _eventChannel = eventChannel ?? throw new ArgumentNullException(nameof(eventChannel));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        // Subscribe to network events for this specific entity
        string entityChannel = $"{_eventChannel}.{entity.Name}";
        
        return _networkClient.Subscribe(entityChannel, message =>
        {
            Debug.Log($"Network event received for entity {entity.Name}: {message}");
            ProcessNetworkMessage(entity, message);
            callback(entity);
        });
    }

    private void ProcessNetworkMessage(IEntity entity, string message)
    {
        // Process network message and update entity state
        try
        {
            var data = JsonUtility.FromJson<NetworkEntityUpdate>(message);
            ApplyNetworkUpdate(entity, data);
        }
        catch (Exception ex)
        {
            Debug.LogError($"Error processing network message for {entity.Name}: {ex.Message}");
        }
    }

    private void ApplyNetworkUpdate(IEntity entity, NetworkEntityUpdate data)
    {
        // Apply network update to entity
        foreach (var property in data.Properties)
        {
            entity.Set(property.Key, property.Value);
        }
    }
}

[System.Serializable]
public class NetworkEntityUpdate
{
    public string EntityId;
    public Dictionary<string, object> Properties;
}

// Usage
var networkTrigger = new NetworkEventTrigger(NetworkManager.Instance, "entity_updates");
networkTrigger.SetAction(entity =>
{
    Debug.Log($"Entity {entity.Name} updated from network");
    GameEvents.OnEntityNetworkUpdate?.Invoke(entity);
});
```

### Database Change Monitor Trigger

```csharp
public class DatabaseChangeMonitorTrigger : SubscriptionEntityTrigger
{
    private readonly IDatabaseConnection _database;
    private readonly string _tableName;

    public DatabaseChangeMonitorTrigger(IDatabaseConnection database, string tableName)
    {
        _database = database ?? throw new ArgumentNullException(nameof(database));
        _tableName = tableName ?? throw new ArgumentNullException(nameof(tableName));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        // Set up database change notification for this entity
        string entityId = entity.Name; // or entity.Id
        
        return _database.WatchForChanges(_tableName, entityId, changes =>
        {
            Debug.Log($"Database changes detected for entity {entity.Name}");
            ApplyDatabaseChanges(entity, changes);
            callback(entity);
        });
    }

    private void ApplyDatabaseChanges(IEntity entity, DatabaseChangeSet changes)
    {
        foreach (var change in changes.Changes)
        {
            switch (change.OperationType)
            {
                case DatabaseOperationType.Insert:
                case DatabaseOperationType.Update:
                    entity.Set(change.ColumnName, change.NewValue);
                    break;
                case DatabaseOperationType.Delete:
                    entity.Remove(change.ColumnName);
                    break;
            }
        }
    }
}

public class DatabaseChangeSet
{
    public List<DatabaseChange> Changes { get; set; } = new();
}

public class DatabaseChange
{
    public DatabaseOperationType OperationType { get; set; }
    public string ColumnName { get; set; }
    public object OldValue { get; set; }
    public object NewValue { get; set; }
}

public enum DatabaseOperationType
{
    Insert, Update, Delete
}

// Usage
var dbMonitorTrigger = new DatabaseChangeMonitorTrigger(DatabaseManager.Instance, "Entities");
dbMonitorTrigger.SetAction(entity =>
{
    Debug.Log($"Entity {entity.Name} synchronized from database");
    GameEvents.OnEntityDatabaseSync?.Invoke(entity);
});
```

## Integration with Atomic Framework

### Multi-Source Subscription Manager

```csharp
public class MultiSourceSubscriptionManager : MonoBehaviour
{
    [System.Serializable]
    public struct SubscriptionConfig
    {
        public string name;
        public SubscriptionSourceType sourceType;
        public float intervalSeconds;
        public string[] parameters;
    }

    public enum SubscriptionSourceType
    {
        Timer, Observable, EventBus, FileSystem, Network, Database
    }

    [SerializeField] private SubscriptionConfig[] _subscriptionConfigs;
    
    private readonly Dictionary<string, SubscriptionEntityTrigger> _triggers = new();
    private readonly Dictionary<IEntity, HashSet<string>> _entitySubscriptions = new();

    void Start()
    {
        foreach (var config in _subscriptionConfigs)
        {
            var trigger = CreateTriggerFromConfig(config);
            if (trigger != null)
            {
                trigger.SetAction(entity => OnSubscriptionTriggered(config.name, entity));
                _triggers[config.name] = trigger;
            }
        }
    }

    private SubscriptionEntityTrigger CreateTriggerFromConfig(SubscriptionConfig config)
    {
        switch (config.sourceType)
        {
            case SubscriptionSourceType.Timer:
                return CreateTimerTrigger(config);
            case SubscriptionSourceType.Observable:
                return CreateObservableTrigger(config);
            case SubscriptionSourceType.EventBus:
                return CreateEventBusTrigger(config);
            case SubscriptionSourceType.FileSystem:
                return CreateFileSystemTrigger(config);
            case SubscriptionSourceType.Network:
                return CreateNetworkTrigger(config);
            case SubscriptionSourceType.Database:
                return CreateDatabaseTrigger(config);
            default:
                Debug.LogWarning($"Unknown subscription source type: {config.sourceType}");
                return null;
        }
    }

    private SubscriptionEntityTrigger CreateTimerTrigger(SubscriptionConfig config)
    {
        var interval = TimeSpan.FromSeconds(config.intervalSeconds);
        return new TimerPollingTrigger(interval, entity => true); // Always trigger
    }

    private SubscriptionEntityTrigger CreateObservableTrigger(SubscriptionConfig config)
    {
        string valueKey = config.parameters.Length > 0 ? config.parameters[0] : "DefaultValue";
        return new ObservableValueTrigger(valueKey);
    }

    private SubscriptionEntityTrigger CreateEventBusTrigger(SubscriptionConfig config)
    {
        string eventType = config.parameters.Length > 0 ? config.parameters[0] : "DefaultEvent";
        return new EventBusTrigger(GameEventBus.Instance, eventType);
    }

    private SubscriptionEntityTrigger CreateFileSystemTrigger(SubscriptionConfig config)
    {
        string directory = config.parameters.Length > 0 ? config.parameters[0] : "./";
        return new ConfigFileWatcherTrigger(directory);
    }

    private SubscriptionEntityTrigger CreateNetworkTrigger(SubscriptionConfig config)
    {
        string channel = config.parameters.Length > 0 ? config.parameters[0] : "default";
        return new NetworkEventTrigger(NetworkManager.Instance, channel);
    }

    private SubscriptionEntityTrigger CreateDatabaseTrigger(SubscriptionConfig config)
    {
        string tableName = config.parameters.Length > 0 ? config.parameters[0] : "Entities";
        return new DatabaseChangeMonitorTrigger(DatabaseManager.Instance, tableName);
    }

    public void SubscribeEntity(IEntity entity, string triggerName)
    {
        if (_triggers.TryGetValue(triggerName, out var trigger))
        {
            trigger.Track(entity);
            
            if (!_entitySubscriptions.ContainsKey(entity))
                _entitySubscriptions[entity] = new HashSet<string>();
            
            _entitySubscriptions[entity].Add(triggerName);
        }
    }

    public void UnsubscribeEntity(IEntity entity, string triggerName)
    {
        if (_triggers.TryGetValue(triggerName, out var trigger))
        {
            trigger.Untrack(entity);
            
            if (_entitySubscriptions.TryGetValue(entity, out var subscriptions))
            {
                subscriptions.Remove(triggerName);
                
                if (subscriptions.Count == 0)
                    _entitySubscriptions.Remove(entity);
            }
        }
    }

    public void SubscribeEntityToAll(IEntity entity)
    {
        foreach (var triggerName in _triggers.Keys)
        {
            SubscribeEntity(entity, triggerName);
        }
    }

    public void UnsubscribeEntityFromAll(IEntity entity)
    {
        if (_entitySubscriptions.TryGetValue(entity, out var subscriptions))
        {
            foreach (var triggerName in subscriptions.ToArray())
            {
                UnsubscribeEntity(entity, triggerName);
            }
        }
    }

    private void OnSubscriptionTriggered(string triggerName, IEntity entity)
    {
        Debug.Log($"Subscription '{triggerName}' triggered for entity: {entity.Name}");
        GameEvents.OnEntitySubscriptionTriggered?.Invoke(triggerName, entity);
    }

    void OnDestroy()
    {
        // Clean up all subscriptions
        foreach (var entity in _entitySubscriptions.Keys.ToArray())
        {
            UnsubscribeEntityFromAll(entity);
        }
    }
}
```

## Implementation Notes

### Subscription Lifecycle
- Subscriptions automatically created during Track() calls
- Stored subscriptions disposed during Untrack() calls
- No duplicate subscriptions for same entity guaranteed

### Resource Management
- Automatic disposal prevents resource leaks
- Thread-safe subscription dictionary operations
- Proper cleanup of subscription resources

### Abstract Pattern Benefits
- Consistent subscription management across implementations
- Focus on subscription creation logic in derived classes
- Standardized resource cleanup patterns

## Best Practices

### Subscription Creation
- Return lightweight, disposable subscription objects
- Handle exceptions in subscription creation gracefully
- Consider subscription lifecycle and cleanup requirements

### Resource Cleanup
- Ensure IDisposable implementations properly cleanup resources
- Handle disposal exceptions gracefully
- Test subscription disposal under various conditions

### Error Handling
- Implement proper error handling in subscription callbacks
- Log subscription creation and disposal events for debugging
- Consider retry mechanisms for failed subscriptions

## Common Patterns

### Composite Subscription Pattern

```csharp
public class CompositeSubscriptionTrigger : SubscriptionEntityTrigger
{
    private readonly SubscriptionEntityTrigger[] _triggers;

    public CompositeSubscriptionTrigger(params SubscriptionEntityTrigger[] triggers)
    {
        _triggers = triggers ?? throw new ArgumentNullException(nameof(triggers));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        var subscriptions = new List<IDisposable>();

        foreach (var trigger in _triggers)
        {
            var subscription = trigger.Track(entity, callback);
            subscriptions.Add(subscription);
        }

        return new CompositeDisposable(subscriptions);
    }
}

public class CompositeDisposable : IDisposable
{
    private readonly List<IDisposable> _disposables;
    private bool _disposed = false;

    public CompositeDisposable(List<IDisposable> disposables)
    {
        _disposables = disposables ?? throw new ArgumentNullException(nameof(disposables));
    }

    public void Dispose()
    {
        if (!_disposed)
        {
            foreach (var disposable in _disposables)
            {
                try
                {
                    disposable?.Dispose();
                }
                catch (Exception ex)
                {
                    Debug.LogError($"Error disposing subscription: {ex.Message}");
                }
            }
            
            _disposed = true;
        }
    }
}
```

### Conditional Subscription Pattern

```csharp
public class ConditionalSubscriptionTrigger : SubscriptionEntityTrigger
{
    private readonly Func<IEntity, bool> _condition;
    private readonly SubscriptionEntityTrigger _innerTrigger;

    public ConditionalSubscriptionTrigger(
        Func<IEntity, bool> condition, 
        SubscriptionEntityTrigger innerTrigger)
    {
        _condition = condition ?? throw new ArgumentNullException(nameof(condition));
        _innerTrigger = innerTrigger ?? throw new ArgumentNullException(nameof(innerTrigger));
    }

    protected override IDisposable Track(IEntity entity, Action<IEntity> callback)
    {
        if (_condition(entity))
        {
            return _innerTrigger.Track(entity, callback);
        }

        return Disposable.Empty;
    }
}
```

The `SubscriptionEntityTrigger` provides a robust foundation for building sophisticated subscription-based entity monitoring systems with automatic resource management, enabling integration with reactive streams, external systems, and complex event sources within the Atomic framework.