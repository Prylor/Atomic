# 🧩 IEntityCollection

`IEntityCollection` represents a mutable collection of entities with lifecycle management support. It extends standard collection interfaces while providing entity-specific functionality and event notifications.

## Key Features

- **Mutable Collection** – Add, remove, and modify entity collections
- **Lifecycle Integration** – Automatic entity lifecycle management
- **Event Notifications** – Track additions, removals, and state changes
- **Generic Support** – Type-safe collections for specific entity types
- **Standard Interfaces** – Implements ICollection<E> for compatibility
- **Disposal Support** – Proper cleanup with IDisposable

---

## Interface Definition

```csharp
public interface IEntityCollection : IEntityCollection<IEntity>, IReadOnlyEntityCollection
{
}

public interface IEntityCollection<E> : IReadOnlyEntityCollection<E>, ICollection<E>, IDisposable 
    where E : IEntity
{
    new int Count { get; }
    new bool Contains(E entity);
    new bool Add(E entity);
    new void CopyTo(E[] array, int arrayIndex);
}
```

## Inheritance Hierarchy

- **IReadOnlyEntityCollection<E>** – Read-only access and events
- **ICollection<E>** – Standard .NET collection interface
- **IDisposable** – Resource cleanup support

## Events (Inherited)

```csharp
event Action OnStateChanged;
event Action<E> OnAdded;
event Action<E> OnRemoved;
```

## Properties

### Count
```csharp
int Count { get; }
```
- **Description**: Number of entities in the collection
- **Access**: Read-only

## Core Methods

### Add
```csharp
bool Add(E entity)
```
- **Description**: Adds an entity to the collection
- **Returns**: `true` if added, `false` if already exists
- **Events**: Triggers `OnAdded` and `OnStateChanged`

### Remove
```csharp
bool Remove(E entity)
```
- **Description**: Removes an entity from the collection
- **Returns**: `true` if removed, `false` if not found
- **Events**: Triggers `OnRemoved` and `OnStateChanged`

### Contains
```csharp
bool Contains(E entity)
```
- **Description**: Checks if entity exists in collection
- **Returns**: `true` if entity is in collection

### Clear
```csharp
void Clear()
```
- **Description**: Removes all entities from collection
- **Events**: Triggers `OnRemoved` for each entity

### CopyTo
```csharp
void CopyTo(E[] array, int arrayIndex)
```
- **Description**: Copies entities to an array
- **Parameters**:
  - `array`: Destination array
  - `arrayIndex`: Starting index in destination

## Implementation: EntityCollection

The concrete `EntityCollection` class provides a full implementation:

```csharp
public class EntityCollection<E> : IEntityCollection<E> where E : IEntity
{
    private readonly HashSet<E> entities;
    
    public event Action OnStateChanged;
    public event Action<E> OnAdded;
    public event Action<E> OnRemoved;
    
    public int Count => entities.Count;
    
    public EntityCollection(int capacity = 0)
    {
        entities = new HashSet<E>(capacity);
    }
}
```

## Example Usage

### Basic Collection Management

```csharp
// Create a collection
var enemies = new EntityCollection<IEntity>();

// Add entities
var goblin = new Entity("Goblin");
var orc = new Entity("Orc");

enemies.Add(goblin);
enemies.Add(orc);

// Check and remove
if (enemies.Contains(goblin))
{
    enemies.Remove(goblin);
}

// Clear all
enemies.Clear();
```

### Reactive Collection

```csharp
public class EnemyManager
{
    private readonly IEntityCollection<IEntity> enemies;
    private int enemyCount;
    
    public EnemyManager()
    {
        enemies = new EntityCollection<IEntity>();
        
        // Subscribe to collection events
        enemies.OnAdded += OnEnemyAdded;
        enemies.OnRemoved += OnEnemyRemoved;
        enemies.OnStateChanged += UpdateUI;
    }
    
    private void OnEnemyAdded(IEntity enemy)
    {
        enemyCount++;
        enemy.Spawn();
        enemy.AddTag(EntityTags.ENEMY);
        
        Debug.Log($"Enemy spawned: {enemy.Name}");
    }
    
    private void OnEnemyRemoved(IEntity enemy)
    {
        enemyCount--;
        enemy.Despawn();
        enemy.Dispose();
        
        Debug.Log($"Enemy removed: {enemy.Name}");
    }
    
    private void UpdateUI()
    {
        UIManager.SetEnemyCount(enemyCount);
    }
}
```

### Typed Collections

```csharp
// Define a specific entity type
public interface ICharacter : IEntity
{
    int Level { get; }
}

// Use typed collection
public class PartyManager
{
    private readonly IEntityCollection<ICharacter> party;
    
    public PartyManager()
    {
        party = new EntityCollection<ICharacter>(capacity: 4);
    }
    
    public void AddToParty(ICharacter character)
    {
        if (party.Count >= 4)
        {
            Debug.Log("Party is full!");
            return;
        }
        
        party.Add(character);
        character.AddTag(EntityTags.PARTY_MEMBER);
    }
    
    public ICharacter GetHighestLevel()
    {
        ICharacter highest = null;
        foreach (var member in party)
        {
            if (highest == null || member.Level > highest.Level)
            {
                highest = member;
            }
        }
        return highest;
    }
}
```

### Collection Operations

```csharp
public static class CollectionOperations
{
    public static void SpawnAll(IEntityCollection<IEntity> collection)
    {
        foreach (var entity in collection)
        {
            if (!entity.Spawned)
            {
                entity.Spawn();
            }
        }
    }
    
    public static void DespawnAll(IEntityCollection<IEntity> collection)
    {
        foreach (var entity in collection)
        {
            if (entity.Spawned)
            {
                entity.Despawn();
            }
        }
    }
    
    public static List<IEntity> FindWithTag(IEntityCollection<IEntity> collection, int tag)
    {
        var result = new List<IEntity>();
        foreach (var entity in collection)
        {
            if (entity.HasTag(tag))
            {
                result.Add(entity);
            }
        }
        return result;
    }
    
    public static void ApplyToAll<T>(IEntityCollection<IEntity> collection, Action<T> action)
        where T : IEntityBehaviour
    {
        foreach (var entity in collection)
        {
            var behaviour = entity.GetBehaviour<T>();
            if (behaviour != null)
            {
                action(behaviour);
            }
        }
    }
}
```

### Collection Synchronization

```csharp
public class SynchronizedCollection
{
    private readonly IEntityCollection<IEntity> source;
    private readonly IEntityCollection<IEntity> mirror;
    
    public SynchronizedCollection(
        IEntityCollection<IEntity> source,
        IEntityCollection<IEntity> mirror)
    {
        this.source = source;
        this.mirror = mirror;
        
        // Sync on changes
        source.OnAdded += entity => mirror.Add(entity);
        source.OnRemoved += entity => mirror.Remove(entity);
    }
    
    public void SyncNow()
    {
        // Clear and rebuild mirror
        mirror.Clear();
        foreach (var entity in source)
        {
            mirror.Add(entity);
        }
    }
}
```

### Procedural Collection Management

Following Atomic's procedural approach:

```csharp
public static class EntityCollectionUtils
{
    public static int CountWithTag(IEntityCollection<IEntity> collection, int tag)
    {
        int count = 0;
        foreach (var entity in collection)
        {
            if (entity.HasTag(tag))
                count++;
        }
        return count;
    }
    
    public static void TransferEntity(
        IEntityCollection<IEntity> from,
        IEntityCollection<IEntity> to,
        IEntity entity)
    {
        if (from.Remove(entity))
        {
            to.Add(entity);
        }
    }
    
    public static void MergeCollections(
        IEntityCollection<IEntity> target,
        params IEntityCollection<IEntity>[] sources)
    {
        foreach (var source in sources)
        {
            foreach (var entity in source)
            {
                target.Add(entity);
            }
            source.Clear();
        }
    }
    
    public static IEntityCollection<IEntity> Clone(IEntityCollection<IEntity> source)
    {
        var clone = new EntityCollection<IEntity>(source.Count);
        foreach (var entity in source)
        {
            clone.Add(entity);
        }
        return clone;
    }
}
```

## Best Practices

1. **Pre-allocate Capacity** – Initialize with expected size to avoid resizing
2. **Unsubscribe Events** – Always unsubscribe when done to prevent leaks
3. **Batch Operations** – Group changes to minimize event triggers
4. **Type Safety** – Use generic collections for specific entity types
5. **Dispose Properly** – Call Dispose() when collection is no longer needed
6. **Thread Safety** – Collections are NOT thread-safe by default

## Performance Considerations

- **HashSet Storage** – O(1) add/remove/contains operations
- **Event Overhead** – Each operation triggers events
- **Iteration Cost** – O(n) for enumeration
- **Memory Allocation** – CopyTo allocates arrays

## Common Use Cases

- **Entity Groups** – Enemies, allies, projectiles
- **Scene Management** – Active entities, pooled entities
- **Team Systems** – Party members, squads
- **Filtering** – Dynamic subsets based on criteria
- **Lifecycle Management** – Batch spawn/despawn operations