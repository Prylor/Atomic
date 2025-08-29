# 💾 SceneEntity_Values

The `SceneEntity_Values` partial class provides comprehensive value storage and management functionality for `SceneEntity` instances. It implements a high-performance hash table-based system for storing, retrieving, and managing key-value pairs with support for any object type and specialized handling for primitive types.

## Key Features

- **Type-Safe Storage** – Generic methods for type-safe value operations
- **High-Performance Hash Table** – O(1) operations with prime-sized buckets
- **Primitive Optimization** – Special handling for value types to minimize boxing
- **Event-Driven Architecture** – Events for value addition, deletion, and changes
- **Flexible Value Types** – Support for any object type including primitives and references
- **Memory Efficient** – Optimized boxing system for primitive types

---

## Core Properties and Events

### Properties
```csharp
// Value count
public int ValueCount { get; }

// Events
public event Action<IEntity, int> OnValueAdded;
public event Action<IEntity, int> OnValueDeleted;
public event Action<IEntity, int> OnValueChanged;
```

### Internal Storage Structure
```csharp
internal struct ValueSlot
{
    public int key;        // Value identifier  
    public object value;   // Stored value
    public bool primitive; // Primitive type flag
    public bool exists;    // Slot validity flag
    public int next;       // Next slot in hash chain
}
```

## Core Methods

### Value Query Operations
```csharp
// Check if entity has specific value
public bool HasValue(int key)

// Get value with exception on missing key
public T GetValue<T>(int key)

// Get value with default fallback
public T GetValue<T>(int key, T fallback)

// Try get value safely
public bool TryGetValue<T>(int key, out T value)

// Get raw object value
public object GetValue(int key)
public object GetValue(int key, object fallback)
public bool TryGetValue(int key, out object value)
```

### Value Modification Operations
```csharp
// Add new value
public bool AddValue<T>(int key, T value)
public bool AddValue(int key, object value)

// Set value (add or update)
public bool SetValue<T>(int key, T value)
public bool SetValue(int key, object value)

// Remove value
public bool DelValue(int key)

// Clear all values
public void ClearValues()
```

### Value Retrieval Operations
```csharp
// Get all value keys
public int[] GetValueKeys()

// Copy value keys to array
public int CopyValueKeys(int[] results)

// Get value enumerator
public ValueEnumerator GetValueEnumerator()
```

### Batch Operations
```csharp
// Add multiple values
public void AddValues(IReadOnlyDictionary<int, object> values)

// Remove multiple values  
public void DelValues(IEnumerable<int> keys)
```

## Example Usage

### Basic Value Management

```csharp
public class ValueStorageEntity : SceneEntity
{
    [Header("Initial Values")]
    [SerializeField] private int health = 100;
    [SerializeField] private float speed = 5.0f;
    [SerializeField] private string entityName = "Entity";
    [SerializeField] private bool isActive = true;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Subscribe to value events
        this.OnValueAdded += OnValueAdded;
        this.OnValueDeleted += OnValueDeleted;
        this.OnValueChanged += OnValueChanged;
        
        // Initialize with typed values
        this.AddValue(EntityNames.HEALTH, health);
        this.AddValue(EntityNames.MAX_HEALTH, health);
        this.AddValue(EntityNames.SPEED, speed);
        this.AddValue(EntityNames.ENTITY_NAME, entityName);
        this.AddValue(EntityNames.IS_ACTIVE, isActive);
        
        // Add Unity-specific values
        this.AddValue(EntityNames.TRANSFORM, transform);
        this.AddValue(EntityNames.GAME_OBJECT, gameObject);
        this.AddValue(EntityNames.POSITION, transform.position);
        this.AddValue(EntityNames.ROTATION, transform.rotation);
        
        Debug.Log($"Entity initialized with {ValueCount} values");
    }
    
    private void OnValueAdded(IEntity entity, int key)
    {
        string keyName = EntityNames.IdToName(key);
        object value = this.GetValue(key);
        Debug.Log($"Value added: {keyName} = {value} ({value?.GetType().Name})");
    }
    
    private void OnValueDeleted(IEntity entity, int key)
    {
        string keyName = EntityNames.IdToName(key);
        Debug.Log($"Value deleted: {keyName}");
    }
    
    private void OnValueChanged(IEntity entity, int key)
    {
        string keyName = EntityNames.IdToName(key);
        object value = this.GetValue(key);
        Debug.Log($"Value changed: {keyName} = {value}");
    }
    
    // Convenience methods for common values
    public int Health
    {
        get => this.GetValue<int>(EntityNames.HEALTH, 0);
        set => this.SetValue(EntityNames.HEALTH, value);
    }
    
    public float Speed
    {
        get => this.GetValue<float>(EntityNames.SPEED, 0f);
        set => this.SetValue(EntityNames.SPEED, value);
    }
    
    public Vector3 Position
    {
        get => this.GetValue<Vector3>(EntityNames.POSITION, Vector3.zero);
        set
        {
            this.SetValue(EntityNames.POSITION, value);
            transform.position = value;
        }
    }
}
```

### Advanced Value System

```csharp
public class AdvancedValueSystem : SceneEntity
{
    [Header("Value Configuration")]
    [SerializeField] private bool enableValueValidation = true;
    [SerializeField] private bool logValueChanges = true;
    [SerializeField] private int maxValuesPerCategory = 20;
    
    [Header("Value Categories")]
    [SerializeField] private ValueCategory[] valueCategories;
    
    [System.Serializable]
    public class ValueCategory
    {
        public string categoryName;
        public List<string> valueNames;
        public System.Type expectedType;
        public bool readOnly;
        public object defaultValue;
        public Color debugColor = Color.white;
    }
    
    private Dictionary<int, ValueCategory> valueToCategory;
    private Dictionary<int, object> originalValues;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        InitializeValueSystem();
        
        // Enhanced event handling
        this.OnValueAdded += ValidateValueAddition;
        this.OnValueChanged += ValidateValueChange;
        this.OnValueDeleted += HandleValueDeletion;
    }
    
    private void InitializeValueSystem()
    {
        valueToCategory = new Dictionary<int, ValueCategory>();
        originalValues = new Dictionary<int, object>();
        
        // Build category mappings
        foreach (var category in valueCategories)
        {
            foreach (var valueName in category.valueNames)
            {
                int valueId = EntityNames.NameToId(valueName);
                valueToCategory[valueId] = category;
                
                // Set default values
                if (category.defaultValue != null)
                {
                    this.AddValue(valueId, category.defaultValue);
                    originalValues[valueId] = category.defaultValue;
                }
            }
        }
    }
    
    public bool AddValidatedValue<T>(int key, T value)
    {
        if (enableValueValidation && !ValidateValue(key, value))
        {
            Debug.LogWarning($"Value validation failed for key {EntityNames.IdToName(key)}");
            return false;
        }
        
        return this.AddValue(key, value);
    }
    
    public bool SetValidatedValue<T>(int key, T value)
    {
        if (enableValueValidation && !ValidateValue(key, value))
        {
            Debug.LogWarning($"Value validation failed for key {EntityNames.IdToName(key)}");
            return false;
        }
        
        return this.SetValue(key, value);
    }
    
    private bool ValidateValue<T>(int key, T value)
    {
        if (!valueToCategory.TryGetValue(key, out var category))
        {
            return true; // Allow uncategorized values
        }
        
        // Check if category is read-only
        if (category.readOnly && this.HasValue(key))
        {
            Debug.LogWarning($"Attempted to modify read-only value: {EntityNames.IdToName(key)}");
            return false;
        }
        
        // Check type compatibility
        if (category.expectedType != null && value != null)
        {
            Type valueType = typeof(T);
            if (!category.expectedType.IsAssignableFrom(valueType))
            {
                Debug.LogWarning($"Type mismatch for {EntityNames.IdToName(key)}: expected {category.expectedType.Name}, got {valueType.Name}");
                return false;
            }
        }
        
        // Check category limits
        int categoryValueCount = GetValueCountInCategory(category.categoryName);
        if (categoryValueCount >= maxValuesPerCategory && !this.HasValue(key))
        {
            Debug.LogWarning($"Category '{category.categoryName}' already has maximum values ({maxValuesPerCategory})");
            return false;
        }
        
        return true;
    }
    
    private int GetValueCountInCategory(string categoryName)
    {
        int count = 0;
        using (var enumerator = this.GetValueEnumerator())
        {
            while (enumerator.MoveNext())
            {
                var (key, _) = enumerator.Current;
                if (valueToCategory.TryGetValue(key, out var category) && 
                    category.categoryName == categoryName)
                {
                    count++;
                }
            }
        }
        return count;
    }
    
    public Dictionary<string, List<(int key, object value)>> GetValuesByCategory()
    {
        var result = new Dictionary<string, List<(int, object)>>();
        
        using (var enumerator = this.GetValueEnumerator())
        {
            while (enumerator.MoveNext())
            {
                var (key, value) = enumerator.Current;
                string categoryName = "Uncategorized";
                
                if (valueToCategory.TryGetValue(key, out var category))
                {
                    categoryName = category.categoryName;
                }
                
                if (!result.ContainsKey(categoryName))
                {
                    result[categoryName] = new List<(int, object)>();
                }
                
                result[categoryName].Add((key, value));
            }
        }
        
        return result;
    }
    
    public void ResetToDefaults()
    {
        foreach (var kvp in originalValues)
        {
            this.SetValue(kvp.Key, kvp.Value);
        }
        
        Debug.Log("Reset all values to defaults");
    }
    
    private void ValidateValueAddition(IEntity entity, int key)
    {
        if (logValueChanges)
        {
            string keyName = EntityNames.IdToName(key);
            string categoryName = valueToCategory.TryGetValue(key, out var cat) ? cat.categoryName : "Uncategorized";
            Debug.Log($"Added value '{keyName}' in category '{categoryName}'");
        }
    }
    
    private void ValidateValueChange(IEntity entity, int key)
    {
        if (logValueChanges)
        {
            string keyName = EntityNames.IdToName(key);
            object value = this.GetValue(key);
            Debug.Log($"Changed value '{keyName}' to {value}");
        }
    }
    
    private void HandleValueDeletion(IEntity entity, int key)
    {
        if (logValueChanges)
        {
            string keyName = EntityNames.IdToName(key);
            Debug.Log($"Deleted value '{keyName}'");
        }
    }
}
```

### Dynamic Value System

```csharp
public class DynamicValueEntity : SceneEntity
{
    [Header("Dynamic Values")]
    [SerializeField] private bool enableComputedValues = true;
    [SerializeField] private bool enableValueCaching = true;
    [SerializeField] private float cacheTimeout = 1.0f;
    
    private Dictionary<int, System.Func<object>> computedValueFunctions;
    private Dictionary<int, (object value, float timestamp)> valueCache;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        InitializeDynamicSystem();
        SetupComputedValues();
    }
    
    private void InitializeDynamicSystem()
    {
        computedValueFunctions = new Dictionary<int, System.Func<object>>();
        valueCache = new Dictionary<int, (object, float)>();
    }
    
    private void SetupComputedValues()
    {
        // Health percentage (computed from health and max health)
        RegisterComputedValue(EntityNames.HEALTH_PERCENTAGE, () =>
        {
            float health = this.GetValue<float>(EntityNames.HEALTH, 0f);
            float maxHealth = this.GetValue<float>(EntityNames.MAX_HEALTH, 100f);
            return maxHealth > 0 ? health / maxHealth : 0f;
        });
        
        // Distance from spawn point
        RegisterComputedValue(EntityNames.DISTANCE_FROM_SPAWN, () =>
        {
            if (!this.TryGetValue<Vector3>(EntityNames.SPAWN_POSITION, out var spawnPos))
                return 0f;
            
            return Vector3.Distance(transform.position, spawnPos);
        });
        
        // Current velocity magnitude
        RegisterComputedValue(EntityNames.CURRENT_SPEED, () =>
        {
            if (this.TryGetValue<Vector3>(EntityNames.VELOCITY, out var velocity))
                return velocity.magnitude;
            return 0f;
        });
        
        // Time since spawn
        RegisterComputedValue(EntityNames.TIME_SINCE_SPAWN, () =>
        {
            if (this.TryGetValue<float>(EntityNames.SPAWN_TIME, out var spawnTime))
                return Time.time - spawnTime;
            return 0f;
        });
    }
    
    public void RegisterComputedValue(int key, System.Func<object> computeFunction)
    {
        computedValueFunctions[key] = computeFunction;
    }
    
    public void UnregisterComputedValue(int key)
    {
        computedValueFunctions.Remove(key);
        valueCache.Remove(key);
    }
    
    public override T GetValue<T>(int key)
    {
        // Check for computed values first
        if (enableComputedValues && computedValueFunctions.ContainsKey(key))
        {
            return GetComputedValue<T>(key);
        }
        
        // Fall back to stored values
        return base.GetValue<T>(key);
    }
    
    public override T GetValue<T>(int key, T fallback)
    {
        // Check for computed values first
        if (enableComputedValues && computedValueFunctions.ContainsKey(key))
        {
            try
            {
                return GetComputedValue<T>(key);
            }
            catch
            {
                return fallback;
            }
        }
        
        // Fall back to stored values
        return base.GetValue<T>(key, fallback);
    }
    
    private T GetComputedValue<T>(int key)
    {
        // Check cache first
        if (enableValueCaching && valueCache.TryGetValue(key, out var cached))
        {
            if (Time.time - cached.timestamp < cacheTimeout)
            {
                return (T)cached.value;
            }
        }
        
        // Compute value
        var computeFunction = computedValueFunctions[key];
        object result = computeFunction();
        
        // Cache result
        if (enableValueCaching)
        {
            valueCache[key] = (result, Time.time);
        }
        
        return (T)result;
    }
    
    public void InvalidateCache(int key)
    {
        valueCache.Remove(key);
    }
    
    public void InvalidateAllCache()
    {
        valueCache.Clear();
    }
    
    // Update cached values periodically
    protected override void OnUpdate(float deltaTime)
    {
        base.OnUpdate(deltaTime);
        
        // Update position-based values
        this.SetValue(EntityNames.POSITION, transform.position);
        this.SetValue(EntityNames.ROTATION, transform.rotation);
        
        // Calculate and store velocity
        if (this.TryGetValue<Vector3>(EntityNames.LAST_POSITION, out var lastPos))
        {
            Vector3 velocity = (transform.position - lastPos) / deltaTime;
            this.SetValue(EntityNames.VELOCITY, velocity);
        }
        
        this.SetValue(EntityNames.LAST_POSITION, transform.position);
    }
}
```

### Value-Based State Machine

```csharp
public class ValueDrivenStateMachine : SceneEntity
{
    [Header("State Configuration")]
    [SerializeField] private StateDefinition[] stateDefinitions;
    [SerializeField] private bool logStateChanges = true;
    
    [System.Serializable]
    public class StateDefinition
    {
        public string stateName;
        public StateCondition[] enterConditions;
        public StateCondition[] exitConditions;
        public ValueAssignment[] stateValues;
        public float priority = 1.0f;
    }
    
    [System.Serializable]
    public class StateCondition
    {
        public string valueName;
        public ComparisonType comparison;
        public object expectedValue;
        
        public enum ComparisonType
        {
            Equals,
            NotEquals,
            GreaterThan,
            LessThan,
            GreaterOrEqual,
            LessOrEqual
        }
    }
    
    [System.Serializable]
    public class ValueAssignment
    {
        public string valueName;
        public object value;
    }
    
    private string currentState = "";
    private Dictionary<string, StateDefinition> stateMap;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        InitializeStateMachine();
        
        // Monitor value changes for state transitions
        this.OnValueChanged += CheckStateTransitions;
    }
    
    private void InitializeStateMachine()
    {
        stateMap = new Dictionary<string, StateDefinition>();
        foreach (var state in stateDefinitions)
        {
            stateMap[state.stateName] = state;
        }
        
        // Set initial state
        CheckStateTransitions(this, -1);
    }
    
    private void CheckStateTransitions(IEntity entity, int changedKey)
    {
        // Find the best matching state based on current values
        StateDefinition bestState = null;
        float bestPriority = float.MinValue;
        
        foreach (var state in stateDefinitions)
        {
            if (state.priority <= bestPriority) continue;
            
            if (EvaluateConditions(state.enterConditions))
            {
                // Check if we should exit current state
                if (!string.IsNullOrEmpty(currentState) && 
                    stateMap.TryGetValue(currentState, out var currentStateDef))
                {
                    if (!EvaluateConditions(currentStateDef.exitConditions))
                        continue;
                }
                
                bestState = state;
                bestPriority = state.priority;
            }
        }
        
        // Change state if needed
        if (bestState != null && bestState.stateName != currentState)
        {
            ChangeState(bestState);
        }
    }
    
    private bool EvaluateConditions(StateCondition[] conditions)
    {
        foreach (var condition in conditions)
        {
            int valueKey = EntityNames.NameToId(condition.valueName);
            
            if (!this.TryGetValue(valueKey, out object currentValue))
                return false;
                
            if (!EvaluateCondition(currentValue, condition))
                return false;
        }
        
        return true;
    }
    
    private bool EvaluateCondition(object currentValue, StateCondition condition)
    {
        if (currentValue == null || condition.expectedValue == null)
            return currentValue == condition.expectedValue;
        
        try
        {
            switch (condition.comparison)
            {
                case StateCondition.ComparisonType.Equals:
                    return currentValue.Equals(condition.expectedValue);
                    
                case StateCondition.ComparisonType.NotEquals:
                    return !currentValue.Equals(condition.expectedValue);
                    
                case StateCondition.ComparisonType.GreaterThan:
                    return ((IComparable)currentValue).CompareTo(condition.expectedValue) > 0;
                    
                case StateCondition.ComparisonType.LessThan:
                    return ((IComparable)currentValue).CompareTo(condition.expectedValue) < 0;
                    
                case StateCondition.ComparisonType.GreaterOrEqual:
                    return ((IComparable)currentValue).CompareTo(condition.expectedValue) >= 0;
                    
                case StateCondition.ComparisonType.LessOrEqual:
                    return ((IComparable)currentValue).CompareTo(condition.expectedValue) <= 0;
                    
                default:
                    return false;
            }
        }
        catch
        {
            return false;
        }
    }
    
    private void ChangeState(StateDefinition newState)
    {
        string previousState = currentState;
        currentState = newState.stateName;
        
        // Apply state values
        foreach (var assignment in newState.stateValues)
        {
            int valueKey = EntityNames.NameToId(assignment.valueName);
            this.SetValue(valueKey, assignment.value);
        }
        
        // Set current state value
        this.SetValue(EntityNames.CURRENT_STATE, currentState);
        
        if (logStateChanges)
        {
            Debug.Log($"State changed from '{previousState}' to '{currentState}'");
        }
    }
    
    public string GetCurrentState()
    {
        return currentState;
    }
    
    public bool IsInState(string stateName)
    {
        return currentState == stateName;
    }
}
```

## Performance Characteristics

### Time Complexity
- **GetValue**: O(1) average case
- **SetValue**: O(1) average case  
- **AddValue**: O(1) average case
- **DelValue**: O(1) average case
- **HasValue**: O(1) average case

### Memory Optimization
- **Primitive Boxing**: Specialized Boxing<T> class reduces boxing overhead
- **Hash Table**: Prime-sized buckets for optimal distribution
- **Slot-Based Storage**: Efficient memory layout with minimal overhead

### Type System Integration
- **Generic Methods**: Compile-time type safety
- **Boxing Optimization**: Reduced allocations for value types
- **Type Validation**: Runtime type checking when needed

## Best Practices

1. **Type Safety** – Use generic methods for compile-time type checking
2. **Performance** – Leverage O(1) operations for frequent value access
3. **Event Handling** – Subscribe to value events for reactive behavior
4. **Memory Efficiency** – Consider boxing overhead for frequently accessed primitives
5. **Error Handling** – Use TryGetValue for optional values
6. **Validation** – Implement value validation for critical game state

## Common Patterns

### Property Wrappers
```csharp
public int Health
{
    get => this.GetValue<int>(EntityNames.HEALTH, 0);
    set => this.SetValue(EntityNames.HEALTH, value);
}

public Vector3 Position
{
    get => this.GetValue<Vector3>(EntityNames.POSITION, Vector3.zero);
    set => this.SetValue(EntityNames.POSITION, value);
}
```

### Safe Value Access
```csharp
public float GetHealthPercent()
{
    if (this.TryGetValue<int>(EntityNames.HEALTH, out int health) &&
        this.TryGetValue<int>(EntityNames.MAX_HEALTH, out int maxHealth) &&
        maxHealth > 0)
    {
        return (float)health / maxHealth;
    }
    return 0f;
}
```

### Value Change Tracking
```csharp
private Dictionary<int, object> previousValues = new Dictionary<int, object>();

protected override void OnValueChanged(IEntity entity, int key)
{
    base.OnValueChanged(entity, key);
    
    object newValue = this.GetValue(key);
    
    if (previousValues.TryGetValue(key, out object oldValue))
    {
        HandleValueChange(key, oldValue, newValue);
    }
    
    previousValues[key] = newValue;
}
```

## Notes

- Values are stored in a high-performance hash table with O(1) operations
- Special boxing system reduces memory overhead for primitive types
- Events are fired synchronously after value operations complete
- Supports any object type including Unity objects and custom classes
- Custom enumerator provides efficient iteration over stored values
- Integration with EntityNames enables string-to-key mapping for convenience