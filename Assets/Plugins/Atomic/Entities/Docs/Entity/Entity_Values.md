# 💎 Entity_Values

`Entity_Values` is a partial class implementation that manages value storage for entities using a high-performance hash table with boxing optimization for value types. It provides efficient storage, retrieval, and management of entity properties with strong typing and memory optimization.

## Key Features

- **High-Performance Hash Table** – Uses prime-sized hash tables for optimal distribution
- **Boxing Optimization** – Minimizes boxing/unboxing for value types
- **Unsafe Access** – Zero-allocation reference access for structs
- **Event System** – Comprehensive value change notifications
- **Type Safety** – Generic methods with compile-time type checking
- **Memory Efficient** – Optimized storage with slot-based allocation

---

## Value Storage Architecture

### Internal Structure

```csharp
internal struct ValueSlot
{
    public int key;
    public object value;
    public bool primitive;    // True if value is boxed struct
    public bool exists;       // Slot validity flag
    public int next;          // Collision chain pointer
}
```

### Boxing System

```csharp
private sealed class Boxing<T> : IBoxing
{
    public T value;           // Actual struct value
    object IBoxing.Value => value;
    Type IBoxing.Type => typeof(T);
}
```

### Storage Fields

```csharp
private ValueSlot[] _valueSlots;      // Hash table slots
private int[] _valueBuckets;          // Hash bucket array
private int _valueCapacity;           // Current capacity
private int _valueCount;              // Number of stored values
private int _valueFreeList;           // Free slot chain
private int _valueLastIndex;          // Last used slot index
private int _valuePrimeIndex;         // Current prime table index
```

---

## Public API

### Events

```csharp
public event Action<IEntity, int> OnValueAdded;     // Value added event
public event Action<IEntity, int> OnValueDeleted;   // Value deleted event  
public event Action<IEntity, int> OnValueChanged;   // Value changed event
```

### Properties

```csharp
public int ValueCount { get; }                      // Total value count
```

### Value Retrieval

```csharp
public T GetValue<T>(int key)
public object GetValue(int key)
public bool TryGetValue<T>(int key, out T value)
public bool TryGetValue(int key, out object value)
```

### Unsafe Value Access

```csharp
public ref T GetValueUnsafe<T>(int key)
public bool TryGetValueUnsafe<T>(int key, out T value)
```

### Value Management

```csharp
public void AddValue<T>(int key, T value) where T : struct
public void AddValue(int key, object value)
public void SetValue<T>(int key, T value) where T : struct  
public void SetValue(int key, object value)
public bool DelValue(int key)
public bool HasValue(int key)
```

### Bulk Operations

```csharp
public void ClearValues()
public KeyValuePair<int, object>[] GetValues()
public int CopyValues(KeyValuePair<int, object>[] results)
```

### Enumeration

```csharp
public ValueEnumerator GetValueEnumerator()
IEnumerator<KeyValuePair<int, object>> IEntity.GetValueEnumerator()
```

---

## Usage Patterns

### Basic Value Management

```csharp
public class PlayerEntity : Entity
{
    public void Initialize()
    {
        // Add struct values (optimized storage)
        AddValue(EntityNames.NameToId("Health"), 100);
        AddValue(EntityNames.NameToId("Position"), Vector3.zero);
        AddValue(EntityNames.NameToId("Speed"), 5.5f);
        
        // Add reference type values
        AddValue(EntityNames.NameToId("Inventory"), new List<Item>());
        AddValue(EntityNames.NameToId("PlayerData"), new PlayerData());
        
        Debug.Log($"Entity now has {ValueCount} values");
    }
    
    public void UpdateHealth(int newHealth)
    {
        SetValue(EntityNames.NameToId("Health"), newHealth);
    }
    
    public Vector3 GetPosition()
    {
        return GetValue<Vector3>(EntityNames.NameToId("Position"));
    }
}
```

### High-Performance Value Access

```csharp
public class OptimizedMovementSystem
{
    private readonly int positionId = EntityNames.NameToId("Position");
    private readonly int velocityId = EntityNames.NameToId("Velocity");
    
    public void UpdateEntityPosition(Entity entity, float deltaTime)
    {
        // Use unsafe access for zero-allocation performance
        if (entity.TryGetValueUnsafe<Vector3>(positionId, out var position) &&
            entity.TryGetValueUnsafe<Vector3>(velocityId, out var velocity))
        {
            // Direct reference modification (no boxing/unboxing)
            ref var positionRef = ref entity.GetValueUnsafe<Vector3>(positionId);
            positionRef += velocity * deltaTime;
            
            // Notify about change (triggers events)
            entity.SetValue(positionId, positionRef);
        }
    }
}
```

### Event-Driven Value Monitoring

```csharp
public class ValueChangeMonitor
{
    private readonly Dictionary<int, ValueHistory> valueHistories = new();
    
    public void MonitorEntity(Entity entity)
    {
        entity.OnValueAdded += OnValueAdded;
        entity.OnValueChanged += OnValueChanged;
        entity.OnValueDeleted += OnValueDeleted;
    }
    
    private void OnValueAdded(IEntity entity, int key)
    {
        string valueName = EntityNames.IdToName(key);
        var value = entity.GetValue(key);
        
        Debug.Log($"Value added: {valueName} = {value}");
        
        valueHistories[key] = new ValueHistory
        {
            InitialValue = value,
            Changes = new List<ValueChange>()
        };
    }
    
    private void OnValueChanged(IEntity entity, int key)
    {
        string valueName = EntityNames.IdToName(key);
        var newValue = entity.GetValue(key);
        
        if (valueHistories.TryGetValue(key, out var history))
        {
            history.Changes.Add(new ValueChange
            {
                Timestamp = DateTime.Now,
                NewValue = newValue
            });
        }
        
        Debug.Log($"Value changed: {valueName} = {newValue}");
    }
    
    private void OnValueDeleted(IEntity entity, int key)
    {
        string valueName = EntityNames.IdToName(key);
        Debug.Log($"Value deleted: {valueName}");
        
        valueHistories.Remove(key);
    }
}
```

### Type-Safe Value Operations

```csharp
public static class EntityValueExtensions
{
    public static void SetHealth(this Entity entity, int health)
    {
        entity.SetValue(EntityNames.NameToId("Health"), health);
    }
    
    public static int GetHealth(this Entity entity)
    {
        return entity.GetValue<int>(EntityNames.NameToId("Health"));
    }
    
    public static bool TryGetHealth(this Entity entity, out int health)
    {
        return entity.TryGetValue(EntityNames.NameToId("Health"), out health);
    }
    
    public static void ModifyHealth(this Entity entity, int delta)
    {
        var healthId = EntityNames.NameToId("Health");
        if (entity.TryGetValue<int>(healthId, out var currentHealth))
        {
            entity.SetValue(healthId, currentHealth + delta);
        }
    }
    
    public static void SetPosition(this Entity entity, Vector3 position)
    {
        entity.SetValue(EntityNames.NameToId("Position"), position);
    }
    
    public static Vector3 GetPosition(this Entity entity)
    {
        return entity.GetValue<Vector3>(EntityNames.NameToId("Position"));
    }
    
    public static bool IsAtPosition(this Entity entity, Vector3 targetPosition, float threshold = 0.1f)
    {
        if (entity.TryGetValue<Vector3>(EntityNames.NameToId("Position"), out var position))
        {
            return Vector3.Distance(position, targetPosition) <= threshold;
        }
        return false;
    }
}
```

### Value Validation System

```csharp
public class ValueValidationSystem
{
    private readonly Dictionary<int, IValueValidator> validators = new();
    
    public interface IValueValidator
    {
        bool IsValid(object value);
        string GetErrorMessage(object value);
    }
    
    public void RegisterValidator(string valueName, IValueValidator validator)
    {
        int id = EntityNames.NameToId(valueName);
        validators[id] = validator;
    }
    
    public void ValidateEntity(Entity entity)
    {
        entity.OnValueAdded += (e, key) => ValidateValue(e, key);
        entity.OnValueChanged += (e, key) => ValidateValue(e, key);
    }
    
    private void ValidateValue(IEntity entity, int key)
    {
        if (validators.TryGetValue(key, out var validator))
        {
            var value = entity.GetValue(key);
            if (!validator.IsValid(value))
            {
                string valueName = EntityNames.IdToName(key);
                string error = validator.GetErrorMessage(value);
                Debug.LogError($"Invalid value for {valueName}: {error}");
            }
        }
    }
}

// Example validators
public class HealthValidator : ValueValidationSystem.IValueValidator
{
    public bool IsValid(object value) => value is int health && health >= 0 && health <= 100;
    public string GetErrorMessage(object value) => $"Health must be between 0 and 100, got {value}";
}

public class PositionValidator : ValueValidationSystem.IValueValidator
{
    public bool IsValid(object value) => value is Vector3 pos && !float.IsNaN(pos.x) && !float.IsNaN(pos.y) && !float.IsNaN(pos.z);
    public string GetErrorMessage(object value) => $"Position contains NaN values: {value}";
}
```

### Bulk Value Operations

```csharp
public class EntitySerializer
{
    public Dictionary<string, object> SerializeValues(Entity entity)
    {
        var result = new Dictionary<string, object>();
        var values = entity.GetValues();
        
        foreach (var kvp in values)
        {
            string name = EntityNames.IdToName(kvp.Key);
            result[name] = kvp.Value;
        }
        
        return result;
    }
    
    public void DeserializeValues(Entity entity, Dictionary<string, object> data)
    {
        entity.ClearValues(); // Clear existing values
        
        foreach (var kvp in data)
        {
            int id = EntityNames.NameToId(kvp.Key);
            entity.AddValue(id, kvp.Value);
        }
    }
    
    public void CopyValues(Entity source, Entity destination)
    {
        var values = source.GetValues();
        destination.ClearValues();
        
        foreach (var kvp in values)
        {
            if (kvp.Value is ICloneable cloneable)
            {
                destination.AddValue(kvp.Key, cloneable.Clone());
            }
            else
            {
                destination.AddValue(kvp.Key, kvp.Value);
            }
        }
    }
}
```

### Performance Monitoring

```csharp
public class ValuePerformanceMonitor
{
    private readonly Dictionary<int, ValueMetrics> metrics = new();
    
    public struct ValueMetrics
    {
        public int AccessCount;
        public int ModificationCount;
        public DateTime LastAccess;
        public DateTime LastModification;
        public Type ValueType;
        public int SizeEstimate;
    }
    
    public void MonitorEntity(Entity entity)
    {
        entity.OnValueAdded += (e, key) => RecordValueAdded(key, e.GetValue(key));
        entity.OnValueChanged += (e, key) => RecordValueModified(key, e.GetValue(key));
        entity.OnValueDeleted += (e, key) => RecordValueDeleted(key);
        
        // Monitor value access (requires custom wrapper)
        WrapValueAccess(entity);
    }
    
    private void RecordValueAdded(int key, object value)
    {
        metrics[key] = new ValueMetrics
        {
            AccessCount = 0,
            ModificationCount = 1,
            LastAccess = DateTime.Now,
            LastModification = DateTime.Now,
            ValueType = value.GetType(),
            SizeEstimate = EstimateSize(value)
        };
    }
    
    private void RecordValueModified(int key, object value)
    {
        if (metrics.TryGetValue(key, out var metric))
        {
            metric.ModificationCount++;
            metric.LastModification = DateTime.Now;
            metric.SizeEstimate = EstimateSize(value);
            metrics[key] = metric;
        }
    }
    
    private int EstimateSize(object value)
    {
        // Rough size estimation logic
        return value?.GetType().IsValueType == true 
            ? System.Runtime.InteropServices.Marshal.SizeOf(value) 
            : 32; // Reference type overhead estimate
    }
    
    public void PrintMetrics()
    {
        Debug.Log("=== Value Performance Metrics ===");
        foreach (var kvp in metrics)
        {
            string name = EntityNames.IdToName(kvp.Key);
            var metric = kvp.Value;
            
            Debug.Log($"{name}:");
            Debug.Log($"  Type: {metric.ValueType.Name}");
            Debug.Log($"  Access Count: {metric.AccessCount}");
            Debug.Log($"  Modification Count: {metric.ModificationCount}");
            Debug.Log($"  Size Estimate: {metric.SizeEstimate} bytes");
            Debug.Log($"  Last Access: {metric.LastAccess:HH:mm:ss}");
            Debug.Log($"  Last Modified: {metric.LastModification:HH:mm:ss}");
        }
    }
}
```

## Performance Characteristics

### Time Complexity
- **Get/Set/Has**: O(1) average, O(n) worst case (hash collision)
- **Add/Delete**: O(1) average, O(n) worst case  
- **Clear**: O(n) - must notify about all deletions
- **Resize**: O(n) - rehashing all existing values

### Space Complexity
- **Memory overhead**: ~40 bytes per value slot
- **Hash table**: Prime-sized for optimal distribution
- **Boxing optimization**: Struct values boxed once, reused
- **Free list**: Efficient slot reuse after deletion

### Boxing Behavior
- **Struct values**: Boxed in `Boxing<T>` wrapper (optimized)
- **Reference types**: Stored directly (no boxing)
- **Type changes**: Re-boxing when struct type changes
- **Unsafe access**: Zero-allocation for struct access

## Implementation Details

### Hash Table Design
- **Prime-sized capacity** for optimal hash distribution
- **Separate chaining** for collision resolution  
- **Free list management** for deleted slot reuse
- **Automatic resizing** when capacity exceeded

### Boxing Optimization
- **Type-specific boxing** preserves struct identity
- **Interface-based unboxing** for generic access
- **Reuse optimization** avoids repeated boxing
- **Memory efficiency** minimizes allocation overhead

### Event System Integration
- **State change notifications** trigger `OnStateChanged`
- **Specific value events** for fine-grained monitoring
- **Batch notifications** during clear operations
- **Exception safety** ensures consistent state

## Best Practices

### Performance Optimization
1. **Cache key IDs** – Pre-calculate IDs for frequently used values
2. **Use unsafe access** for performance-critical struct operations
3. **Batch modifications** to minimize event overhead
4. **Pre-allocate capacity** if value count is known

### Memory Management
1. **Clear unused values** to free memory
2. **Avoid frequent type changes** to minimize re-boxing
3. **Use appropriate value types** for memory efficiency
4. **Monitor value sizes** in memory-constrained environments

### Type Safety
1. **Use generic methods** for compile-time type checking
2. **Validate value types** before storage
3. **Handle type mismatches** gracefully
4. **Document value contracts** for shared values

## Thread Safety

- **Not thread-safe** – All operations must be on main thread
- **Event callbacks** execute synchronously on calling thread
- **Unsafe methods** require additional synchronization
- **Hash table modifications** not atomic across operations