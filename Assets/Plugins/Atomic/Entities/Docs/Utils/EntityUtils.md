# 🛠️ EntityUtils

`EntityUtils` provides low-level utility methods for internal use within the Atomic.Entities framework. It contains optimized helpers for delegate management, primitive operations, array manipulation, and Unity editor integration.

## Key Features

- **Prime Number Operations** – Efficient prime number calculations for hashing
- **Dynamic Array Management** – Optimized array expansion and manipulation
- **Unity Integration** – Play mode and edit mode detection
- **Memory Efficient** – Aggressive inlining for performance-critical operations
- **Type Safety** – Generic methods with proper constraint handling

---

## Static Methods

### Play Mode Detection

```csharp
public static bool IsPlayMode()
public static bool IsEditMode()
```

- **IsPlayMode()**: Returns true if application is in Unity Play Mode
- **IsEditMode()**: Returns true if application is in Unity Edit Mode and not compiling

### Array Operations

#### Add Item
```csharp
internal static void Add<T>(ref T[] array, ref int count, T item)
```
- Adds item to dynamically managed array
- Expands array if needed
- Updates count automatically

#### Add If Absent
```csharp
internal static bool AddIfAbsent<T>(ref T[] array, ref int count, T item, 
    IEqualityComparer<T> comparer, int initialCapacity = 1)
```
- Adds item only if not already present
- Uses custom comparer for equality checking
- Returns true if item was added

#### Contains Check
```csharp
internal static bool Contains<T>(T[] array, T item, int count, IEqualityComparer<T> comparer)
```
- Checks if array contains specified item
- Uses custom comparer for equality checking
- Respects current count, not array length

#### Remove Item
```csharp
internal static bool Remove<T>(ref T[] array, ref int count, T item, IEqualityComparer<T> comparer)
```
- Removes item from array
- Shifts subsequent elements left
- Updates count and clears removed slot
- Returns true if item was found and removed

#### Expand Array
```csharp
internal static void Expand<T>(ref T[] array)
```
- Doubles array capacity
- Handles edge cases and overflow protection
- Preserves existing elements

### Prime Number Utilities

#### Ceiling to Prime
```csharp
internal static int CeilToPrime(int value, out int index)
```
- Finds the smallest prime number >= value
- Returns index in internal prime table
- Used for optimal hash table sizing

### Edit Mode Support

#### RunInEditMode Detection
```csharp
internal static bool IsRunInEditModeDefined(object obj)
```
- Checks if object's type has RunInEditModeAttribute
- Used for lifecycle management in editor

---

## Internal Prime Table

```csharp
internal static readonly int[] PrimeTable =
{
    2, 3, 7, 17, 29, 53, 97, 193, 389, 769, 1543, 3079,
    6151, 12289, 24593, 49157, 98317, 196613, 393241, 786433,
    1572869, 3145739, 6291469, 12582917, 25165843, 50331653,
    100663319, 201326611, 402653189, 805306457, 1610612741
};
```

Pre-calculated prime numbers for efficient hash table sizing and mathematical operations.

## Example Usage

### Dynamic Array Management

```csharp
public class DynamicList<T>
{
    private T[] items;
    private int count;
    
    public void Add(T item)
    {
        EntityUtils.Add(ref items, ref count, item);
    }
    
    public bool Contains(T item)
    {
        return EntityUtils.Contains(items, item, count, EqualityComparer<T>.Default);
    }
    
    public bool Remove(T item)
    {
        return EntityUtils.Remove(ref items, ref count, item, EqualityComparer<T>.Default);
    }
}
```

### Unity Mode Detection

```csharp
public class EntitySystem
{
    private bool shouldUpdate;
    
    public void Initialize()
    {
        // Only update in play mode
        shouldUpdate = EntityUtils.IsPlayMode();
        
        // Special initialization for edit mode
        if (EntityUtils.IsEditMode())
        {
            InitializeEditModeFeatures();
        }
    }
    
    public void Update()
    {
        if (!shouldUpdate && !EntityUtils.IsPlayMode())
            return;
            
        ProcessEntities();
    }
}
```

### Prime-Based Hash Tables

```csharp
public class OptimizedHashTable<T>
{
    private T[] buckets;
    
    public OptimizedHashTable(int capacity)
    {
        // Size to optimal prime
        int primeCapacity = EntityUtils.CeilToPrime(capacity, out int index);
        buckets = new T[primeCapacity];
    }
    
    private int GetBucketIndex(int hash)
    {
        return Math.Abs(hash) % buckets.Length;
    }
}
```

### Unique Set Implementation

```csharp
public class EntitySet<T>
{
    private T[] items;
    private int count;
    private readonly IEqualityComparer<T> comparer;
    
    public EntitySet(IEqualityComparer<T> comparer = null)
    {
        this.comparer = comparer ?? EqualityComparer<T>.Default;
    }
    
    public bool Add(T item)
    {
        return EntityUtils.AddIfAbsent(ref items, ref count, item, comparer);
    }
    
    public bool Contains(T item)
    {
        return EntityUtils.Contains(items, item, count, comparer);
    }
    
    public bool Remove(T item)
    {
        return EntityUtils.Remove(ref items, ref count, item, comparer);
    }
}
```

### Editor Integration

```csharp
public class EditorEntityBehaviour : IEntityBehaviour
{
    public void Initialize(Entity entity)
    {
        // Check if this behaviour should run in edit mode
        bool runInEditMode = EntityUtils.IsRunInEditModeDefined(this);
        
        if (!runInEditMode && EntityUtils.IsEditMode())
        {
            // Skip initialization in edit mode
            return;
        }
        
        SetupBehaviour(entity);
    }
}
```

## Performance Considerations

- **Aggressive Inlining** – All public methods use MethodImpl(AggressiveInlining)
- **Minimal Allocations** – Array operations minimize garbage collection
- **Branch Prediction** – Methods structured for optimal CPU prediction
- **Memory Layout** – Contiguous array storage for cache efficiency

## Internal Usage

EntityUtils is primarily used internally by:

- **Entity collections** for dynamic storage
- **EntityRegistry** for efficient lookup tables
- **Behaviour management** for component arrays
- **Unity integration** for editor/runtime differences
- **Hash-based systems** for optimal bucket sizing

## Thread Safety

- **Static methods are thread-safe** for read operations
- **Array operations are NOT thread-safe** – caller must synchronize
- **Unity mode detection is thread-safe** on main thread only

## Best Practices

1. **Use for Internal APIs** – Not intended for public consumption
2. **Batch Operations** – Group array modifications to minimize overhead
3. **Pre-allocate When Possible** – Use initial capacity parameters
4. **Respect Thread Boundaries** – All operations on main thread
5. **Handle Edge Cases** – Always check for null arrays and bounds

## Implementation Notes

- Uses `MethodImpl(MethodImplOptions.AggressiveInlining)` for performance
- Handles array overflow protection in expansion
- Prime table optimized for common hash table sizes
- Editor attributes only included in Unity builds
- Generic constraints ensure type safety without boxing