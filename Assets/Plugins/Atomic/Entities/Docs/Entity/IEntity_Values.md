# 🧩 IEntity_Values

`IEntity_Values` is a partial interface of `IEntity` that provides key-value storage functionality. It enables entities to store dynamic runtime state as typed values accessible by integer keys.

## Key Features

- **Dynamic State Storage** – Add, update, and remove values at runtime
- **Type-Safe Access** – Generic methods for type-safe value retrieval
- **Performance Options** – Unsafe methods for zero-allocation access to structs
- **Change Notifications** – Events for tracking value additions, deletions, and changes
- **Bulk Operations** – Methods for working with multiple values at once

---

## Events

```csharp
event Action<IEntity, int> OnValueAdded;
event Action<IEntity, int> OnValueDeleted;
event Action<IEntity, int> OnValueChanged;
```

### OnValueAdded
- **Triggered**: When a new value is added to the entity
- **Parameters**: The entity and the key of the added value

### OnValueDeleted  
- **Triggered**: When a value is removed from the entity
- **Parameters**: The entity and the key of the deleted value

### OnValueChanged
- **Triggered**: When an existing value is updated
- **Parameters**: The entity and the key of the changed value

## Properties

```csharp
int ValueCount { get; }
```
- **Description**: Returns the number of values currently stored in the entity
- **Access**: Read-only

## Core Methods

### Getting Values

```csharp
T GetValue<T>(int key)
```
- **Description**: Retrieves a value by key and casts it to type T
- **Throws**: Exception if key doesn't exist or cast fails

```csharp
ref T GetValueUnsafe<T>(int key)
```
- **Description**: Gets a reference to a value without boxing (structs only)
- **Performance**: Zero allocations for struct types
- **Warning**: Direct reference - modifications affect stored value

```csharp
object GetValue(int key)
```
- **Description**: Retrieves a value as object (with boxing for value types)

### Safe Value Access

```csharp
bool TryGetValue<T>(int key, out T value)
```
- **Description**: Attempts to get a value of type T
- **Returns**: `true` if value exists and cast succeeds

```csharp
bool TryGetValueUnsafe<T>(int key, out T value)
```
- **Description**: Attempts to get value reference without boxing
- **Performance**: Optimal for struct types

```csharp
bool TryGetValue(int key, out object value)
```
- **Description**: Attempts to get value as object

### Setting Values

```csharp
void SetValue(int key, object value)
```
- **Description**: Sets or updates a value
- **Behavior**: Adds if doesn't exist, updates if exists

```csharp
void SetValue<T>(int key, T value) where T : struct
```
- **Description**: Sets or updates a struct value
- **Performance**: Optimized for value types

### Adding Values

```csharp
void AddValue(int key, object value)
```
- **Description**: Adds a new value with the given key
- **Throws**: Exception if key already exists

```csharp
void AddValue<T>(int key, T value) where T : struct
```
- **Description**: Adds a new struct value
- **Performance**: Optimized for value types

### Checking & Removing

```csharp
bool HasValue(int key)
```
- **Description**: Checks if a value with the given key exists

```csharp
bool DelValue(int key)
```
- **Description**: Removes a value by key
- **Returns**: `true` if value was removed, `false` if not found

```csharp
void ClearValues()
```
- **Description**: Removes all values from the entity

### Bulk Operations

```csharp
KeyValuePair<int, object>[] GetValues()
```
- **Description**: Returns all key-value pairs as an array

```csharp
int CopyValues(KeyValuePair<int, object>[] results)
```
- **Description**: Copies values into provided array
- **Returns**: Number of values copied

```csharp
IEnumerator<KeyValuePair<int, object>> GetValueEnumerator()
```
- **Description**: Returns enumerator for iterating over values

## Example Usage

### Basic Value Management

```csharp
// Add values to entity
entity.AddValue(EntityNames.HEALTH, 100);
entity.AddValue(EntityNames.POSITION, new Vector3(0, 0, 0));
entity.AddValue(EntityNames.SPEED, 5.5f);

// Get values
int health = entity.GetValue<int>(EntityNames.HEALTH);
Vector3 position = entity.GetValue<Vector3>(EntityNames.POSITION);

// Safe access
if (entity.TryGetValue<float>(EntityNames.SPEED, out float speed))
{
    Debug.Log($"Speed: {speed}");
}

// Update values
entity.SetValue(EntityNames.HEALTH, 75);

// Check and remove
if (entity.HasValue(EntityNames.TEMP_BUFF))
{
    entity.DelValue(EntityNames.TEMP_BUFF);
}
```

### Performance-Optimized Access

```csharp
// Zero-allocation struct access
ref Vector3 posRef = ref entity.GetValueUnsafe<Vector3>(EntityNames.POSITION);
posRef.x += 10; // Directly modifies stored value

// Bulk operations
var allValues = entity.GetValues();
foreach (var kvp in allValues)
{
    Debug.Log($"Key: {kvp.Key}, Value: {kvp.Value}");
}
```

### Reactive Value Changes

```csharp
// Subscribe to value events
entity.OnValueAdded += (e, key) => 
{
    Debug.Log($"Value added: {key}");
};

entity.OnValueChanged += (e, key) =>
{
    if (key == EntityNames.HEALTH)
    {
        int newHealth = e.GetValue<int>(key);
        UpdateHealthBar(newHealth);
    }
};

entity.OnValueDeleted += (e, key) =>
{
    Debug.Log($"Value removed: {key}");
};
```

### Procedural Approach

Following Atomic's procedural pattern:

```csharp
public static class EntityInventory
{
    private const int INVENTORY_KEY = 100;
    
    public static void InitializeInventory(IEntity entity)
    {
        entity.AddValue(INVENTORY_KEY, new List<Item>());
    }
    
    public static void AddItem(IEntity entity, Item item)
    {
        var inventory = entity.GetValue<List<Item>>(INVENTORY_KEY);
        inventory.Add(item);
        entity.OnStateChanged?.Invoke();
    }
    
    public static bool HasItem(IEntity entity, int itemId)
    {
        if (!entity.TryGetValue<List<Item>>(INVENTORY_KEY, out var inventory))
            return false;
            
        return inventory.Any(item => item.Id == itemId);
    }
    
    public static void ClearInventory(IEntity entity)
    {
        if (entity.HasValue(INVENTORY_KEY))
        {
            var inventory = entity.GetValue<List<Item>>(INVENTORY_KEY);
            inventory.Clear();
            entity.OnStateChanged?.Invoke();
        }
    }
}
```

## Best Practices

1. **Use Integer Keys** – Define constants for value keys to avoid magic numbers
2. **Prefer TryGetValue** – Use safe access methods when value existence is uncertain
3. **Leverage Unsafe Methods** – Use `GetValueUnsafe` for frequently accessed structs
4. **Batch Operations** – Use bulk methods when working with multiple values
5. **Type Consistency** – Always use the same type for a given key
6. **Event Subscription** – Remember to unsubscribe from events to prevent memory leaks

## Performance Notes

- **Boxing/Unboxing** – Regular `GetValue<T>` boxes value types; use `GetValueUnsafe<T>` to avoid
- **Dictionary Storage** – Values are stored in a dictionary, providing O(1) access
- **Event Overhead** – Events add minimal overhead; batch changes when possible
- **Memory Allocation** – `GetValues()` allocates a new array; use `CopyValues()` to reuse arrays