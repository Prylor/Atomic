# ⚖️ FixedUpdateSubscription

`FixedUpdateSubscription` is a disposable subscription handle that automatically unregisters a callback from an `IUpdatable`'s `OnFixedUpdated` event when disposed. It provides safe event subscription management for physics-timestep update events.

## Key Features

- **Fixed Timestep Updates** – Handles FixedUpdate cycle events
- **Physics Integration** – Perfect for physics-related calculations
- **Consistent Timing** – Uses Unity's fixed timestep for deterministic updates
- **Automatic Cleanup** – Unsubscribes callback on disposal
- **Delta Time Support** – Callbacks receive fixed deltaTime parameter

---

## Structure Definition

```csharp
public readonly struct FixedUpdateSubscription : IDisposable
{
    private readonly IUpdatable _source;
    private readonly Action<float> _callback;
}
```

### Constructor

```csharp
internal FixedUpdateSubscription(IUpdatable source, Action<float> callback)
```

- **source**: The updatable object to subscribe to
- **callback**: The action to invoke on each fixed update (receives fixed deltaTime)

### Disposal

```csharp
public void Dispose()
```

Safely unsubscribes the callback from the source's `OnFixedUpdated` event.

---

## Usage Patterns

### Physics Update Monitoring

```csharp
public class PhysicsMonitor
{
    private FixedUpdateSubscription subscription;
    private Vector3 lastPosition;
    private float totalDistance;
    
    public void StartMonitoring(IUpdatable physicsSystem)
    {
        subscription = physicsSystem.SubscribeToFixedUpdate(OnPhysicsUpdate);
    }
    
    public void StopMonitoring()
    {
        subscription.Dispose();
    }
    
    private void OnPhysicsUpdate(float fixedDeltaTime)
    {
        // Monitor physics consistency
        Debug.Log($"Physics update with fixed deltaTime: {fixedDeltaTime:F4}");
        
        // Track physics calculations
        ProcessPhysicsStep(fixedDeltaTime);
    }
}
```

### Movement System Integration

```csharp
public class MovementController
{
    private readonly Dictionary<IUpdatable, (Rigidbody rb, Vector3 targetVelocity, FixedUpdateSubscription subscription)> controlledObjects = new();
    
    public void RegisterMovingObject(IUpdatable system, Rigidbody rigidbody)
    {
        var subscription = system.SubscribeToFixedUpdate(fixedDeltaTime => 
            UpdateMovement(rigidbody, fixedDeltaTime));
            
        controlledObjects[system] = (rigidbody, Vector3.zero, subscription);
    }
    
    public void SetTargetVelocity(IUpdatable system, Vector3 velocity)
    {
        if (controlledObjects.TryGetValue(system, out var data))
        {
            controlledObjects[system] = (data.rb, velocity, data.subscription);
        }
    }
    
    private void UpdateMovement(Rigidbody rb, float fixedDeltaTime)
    {
        // Find the target velocity for this rigidbody
        var targetVelocity = Vector3.zero;
        foreach (var data in controlledObjects.Values)
        {
            if (data.rb == rb)
            {
                targetVelocity = data.targetVelocity;
                break;
            }
        }
        
        // Apply movement using fixed timestep
        rb.velocity = Vector3.MoveTowards(rb.velocity, targetVelocity, 10f * fixedDeltaTime);
    }
    
    public void UnregisterMovingObject(IUpdatable system)
    {
        if (controlledObjects.TryGetValue(system, out var data))
        {
            data.subscription.Dispose();
            controlledObjects.Remove(system);
        }
    }
}
```

### Force Application System

```csharp
public class ForceApplicationSystem
{
    private readonly Dictionary<IUpdatable, (List<ForceData> forces, FixedUpdateSubscription subscription)> systemForces = new();
    
    public struct ForceData
    {
        public Rigidbody Target;
        public Vector3 Force;
        public ForceMode Mode;
        public float Duration;
        public float RemainingTime;
    }
    
    public void RegisterSystem(IUpdatable system)
    {
        var forces = new List<ForceData>();
        var subscription = system.SubscribeToFixedUpdate(fixedDeltaTime => 
            ProcessForces(system, fixedDeltaTime));
            
        systemForces[system] = (forces, subscription);
    }
    
    public void AddForce(IUpdatable system, Rigidbody target, Vector3 force, ForceMode mode, float duration = -1f)
    {
        if (systemForces.TryGetValue(system, out var data))
        {
            data.forces.Add(new ForceData
            {
                Target = target,
                Force = force,
                Mode = mode,
                Duration = duration,
                RemainingTime = duration
            });
        }
    }
    
    private void ProcessForces(IUpdatable system, float fixedDeltaTime)
    {
        if (!systemForces.TryGetValue(system, out var data)) return;
        
        for (int i = data.forces.Count - 1; i >= 0; i--)
        {
            var forceData = data.forces[i];
            
            // Apply force
            forceData.Target.AddForce(forceData.Force, forceData.Mode);
            
            // Update duration
            if (forceData.Duration > 0)
            {
                forceData.RemainingTime -= fixedDeltaTime;
                
                if (forceData.RemainingTime <= 0)
                {
                    // Force expired
                    data.forces.RemoveAt(i);
                }
                else
                {
                    // Update remaining time
                    forceData.RemainingTime = forceData.RemainingTime;
                    data.forces[i] = forceData;
                }
            }
        }
    }
    
    public void UnregisterSystem(IUpdatable system)
    {
        if (systemForces.TryGetValue(system, out var data))
        {
            data.subscription.Dispose();
            systemForces.Remove(system);
        }
    }
}
```

### Physics Performance Profiler

```csharp
public class PhysicsProfiler
{
    private readonly Dictionary<IUpdatable, PhysicsMetrics> systemMetrics = new();
    private readonly List<FixedUpdateSubscription> subscriptions = new();
    
    public struct PhysicsMetrics
    {
        public int FixedUpdateCount;
        public float TotalFixedTime;
        public float LastFixedDeltaTime;
        public float AverageFixedDeltaTime;
        public int MissedFrames;
    }
    
    public void StartProfiling(IEnumerable<IUpdatable> physicsSystems)
    {
        foreach (var system in physicsSystems)
        {
            var subscription = system.SubscribeToFixedUpdate(fixedDeltaTime => 
                ProfileFixedUpdate(system, fixedDeltaTime));
            subscriptions.Add(subscription);
            
            systemMetrics[system] = new PhysicsMetrics();
        }
    }
    
    private void ProfileFixedUpdate(IUpdatable system, float fixedDeltaTime)
    {
        var metrics = systemMetrics[system];
        
        metrics.FixedUpdateCount++;
        metrics.TotalFixedTime += fixedDeltaTime;
        metrics.LastFixedDeltaTime = fixedDeltaTime;
        metrics.AverageFixedDeltaTime = metrics.TotalFixedTime / metrics.FixedUpdateCount;
        
        // Check for timing consistency
        float expectedDeltaTime = Time.fixedDeltaTime;
        if (Mathf.Abs(fixedDeltaTime - expectedDeltaTime) > 0.001f)
        {
            metrics.MissedFrames++;
            Debug.LogWarning($"Fixed update timing inconsistency: expected {expectedDeltaTime:F4}, got {fixedDeltaTime:F4}");
        }
        
        systemMetrics[system] = metrics;
    }
    
    public PhysicsMetrics GetMetrics(IUpdatable system)
    {
        return systemMetrics.TryGetValue(system, out var metrics) ? metrics : default;
    }
    
    public void PrintPhysicsReport()
    {
        Debug.Log("=== Physics Performance Report ===");
        foreach (var kvp in systemMetrics)
        {
            var system = kvp.Key;
            var metrics = kvp.Value;
            
            Debug.Log($"{system.GetType().Name}:");
            Debug.Log($"  Fixed Updates: {metrics.FixedUpdateCount}");
            Debug.Log($"  Total Fixed Time: {metrics.TotalFixedTime:F3}s");
            Debug.Log($"  Average Delta: {metrics.AverageFixedDeltaTime:F4}s");
            Debug.Log($"  Missed Frames: {metrics.MissedFrames}");
        }
    }
}
```

### Physics Simulation Controller

```csharp
public class PhysicsSimulationController
{
    private readonly Dictionary<IUpdatable, (bool enabled, FixedUpdateSubscription subscription)> simulationStates = new();
    
    public void RegisterSimulation(IUpdatable system, Action<float> simulationCallback)
    {
        var subscription = system.SubscribeToFixedUpdate(fixedDeltaTime => 
        {
            if (IsSimulationEnabled(system))
            {
                simulationCallback(fixedDeltaTime);
            }
        });
        
        simulationStates[system] = (true, subscription);
    }
    
    public void EnableSimulation(IUpdatable system, bool enabled)
    {
        if (simulationStates.TryGetValue(system, out var state))
        {
            simulationStates[system] = (enabled, state.subscription);
            
            if (enabled)
            {
                Debug.Log($"Physics simulation enabled for {system.GetType().Name}");
            }
            else
            {
                Debug.Log($"Physics simulation disabled for {system.GetType().Name}");
            }
        }
    }
    
    private bool IsSimulationEnabled(IUpdatable system)
    {
        return simulationStates.TryGetValue(system, out var state) && state.enabled;
    }
    
    public void PauseAllSimulations()
    {
        foreach (var system in simulationStates.Keys.ToList())
        {
            EnableSimulation(system, false);
        }
    }
    
    public void ResumeAllSimulations()
    {
        foreach (var system in simulationStates.Keys.ToList())
        {
            EnableSimulation(system, true);
        }
    }
    
    public void UnregisterSimulation(IUpdatable system)
    {
        if (simulationStates.TryGetValue(system, out var state))
        {
            state.subscription.Dispose();
            simulationStates.Remove(system);
        }
    }
}
```

### Deterministic Physics Logger

```csharp
public class DeterministicPhysicsLogger
{
    private readonly Dictionary<IUpdatable, List<PhysicsSnapshot>> physicsHistory = new();
    private readonly List<FixedUpdateSubscription> subscriptions = new();
    
    public struct PhysicsSnapshot
    {
        public float FixedTime;
        public float DeltaTime;
        public Vector3[] Positions;
        public Vector3[] Velocities;
        public Quaternion[] Rotations;
        public int FrameNumber;
    }
    
    public void StartLogging(IUpdatable system, Rigidbody[] trackedBodies)
    {
        var history = new List<PhysicsSnapshot>();
        physicsHistory[system] = history;
        
        var subscription = system.SubscribeToFixedUpdate(fixedDeltaTime => 
            LogPhysicsState(system, fixedDeltaTime, trackedBodies));
        subscriptions.Add(subscription);
    }
    
    private void LogPhysicsState(IUpdatable system, float fixedDeltaTime, Rigidbody[] bodies)
    {
        var snapshot = new PhysicsSnapshot
        {
            FixedTime = Time.fixedTime,
            DeltaTime = fixedDeltaTime,
            FrameNumber = Time.fixedStepCount,
            Positions = bodies.Select(b => b.position).ToArray(),
            Velocities = bodies.Select(b => b.velocity).ToArray(),
            Rotations = bodies.Select(b => b.rotation).ToArray()
        };
        
        physicsHistory[system].Add(snapshot);
        
        // Limit history size
        if (physicsHistory[system].Count > 1000)
        {
            physicsHistory[system].RemoveAt(0);
        }
        
        // Log significant changes
        if (snapshot.FrameNumber % 50 == 0) // Every 50 fixed updates
        {
            Debug.Log($"Physics snapshot at frame {snapshot.FrameNumber}: {bodies.Length} bodies tracked");
        }
    }
    
    public List<PhysicsSnapshot> GetHistory(IUpdatable system)
    {
        return physicsHistory.TryGetValue(system, out var history) ? history : new List<PhysicsSnapshot>();
    }
    
    public void ExportHistory(IUpdatable system, string filename)
    {
        if (physicsHistory.TryGetValue(system, out var history))
        {
            // Export physics history for analysis
            // Implementation depends on desired format (JSON, CSV, binary, etc.)
            Debug.Log($"Exporting {history.Count} physics snapshots to {filename}");
        }
    }
}
```

### Physics-Based Animation System

```csharp
public class PhysicsAnimationSystem
{
    private readonly Dictionary<IUpdatable, AnimationData> animationStates = new();
    
    public struct AnimationData
    {
        public Transform Target;
        public Vector3 StartPosition;
        public Vector3 EndPosition;
        public float Duration;
        public float ElapsedTime;
        public AnimationCurve Curve;
        public FixedUpdateSubscription Subscription;
    }
    
    public void StartAnimation(IUpdatable system, Transform target, Vector3 endPosition, float duration, AnimationCurve curve = null)
    {
        var subscription = system.SubscribeToFixedUpdate(fixedDeltaTime => 
            UpdateAnimation(system, fixedDeltaTime));
            
        animationStates[system] = new AnimationData
        {
            Target = target,
            StartPosition = target.position,
            EndPosition = endPosition,
            Duration = duration,
            ElapsedTime = 0f,
            Curve = curve ?? AnimationCurve.EaseInOut(0f, 0f, 1f, 1f),
            Subscription = subscription
        };
    }
    
    private void UpdateAnimation(IUpdatable system, float fixedDeltaTime)
    {
        if (!animationStates.TryGetValue(system, out var data)) return;
        
        data.ElapsedTime += fixedDeltaTime;
        float progress = Mathf.Clamp01(data.ElapsedTime / data.Duration);
        
        // Apply curve
        float curveValue = data.Curve.Evaluate(progress);
        
        // Interpolate position
        Vector3 currentPosition = Vector3.Lerp(data.StartPosition, data.EndPosition, curveValue);
        data.Target.position = currentPosition;
        
        // Check if animation complete
        if (progress >= 1f)
        {
            data.Target.position = data.EndPosition;
            CompleteAnimation(system);
        }
        else
        {
            // Update elapsed time
            data.ElapsedTime = data.ElapsedTime;
            animationStates[system] = data;
        }
    }
    
    private void CompleteAnimation(IUpdatable system)
    {
        if (animationStates.TryGetValue(system, out var data))
        {
            data.Subscription.Dispose();
            animationStates.Remove(system);
            
            OnAnimationComplete?.Invoke(system, data.Target);
        }
    }
    
    public event Action<IUpdatable, Transform> OnAnimationComplete;
    
    public void StopAnimation(IUpdatable system)
    {
        if (animationStates.TryGetValue(system, out var data))
        {
            data.Subscription.Dispose();
            animationStates.Remove(system);
        }
    }
}
```

## Integration with Unity Physics

### Rigidbody Monitoring

```csharp
public class RigidbodyMonitoringSystem
{
    private readonly List<FixedUpdateSubscription> subscriptions = new();
    
    public void MonitorRigidbodies(IUpdatable physicsSystem, IEnumerable<Rigidbody> rigidbodies)
    {
        var subscription = physicsSystem.SubscribeToFixedUpdate(fixedDeltaTime => 
        {
            foreach (var rb in rigidbodies)
            {
                MonitorRigidbody(rb, fixedDeltaTime);
            }
        });
        
        subscriptions.Add(subscription);
    }
    
    private void MonitorRigidbody(Rigidbody rb, float fixedDeltaTime)
    {
        // Check for physics anomalies
        if (rb.velocity.magnitude > 100f)
        {
            Debug.LogWarning($"High velocity detected on {rb.name}: {rb.velocity.magnitude:F2}");
        }
        
        if (rb.angularVelocity.magnitude > 50f)
        {
            Debug.LogWarning($"High angular velocity detected on {rb.name}: {rb.angularVelocity.magnitude:F2}");
        }
        
        // Log position changes
        LogRigidbodyState(rb, fixedDeltaTime);
    }
}
```

## Best Practices

### Physics Considerations
1. **Use for physics calculations** – FixedUpdate is designed for physics
2. **Consistent timing** – Takes advantage of fixed timestep
3. **Rigidbody interactions** – Perfect for force applications
4. **Deterministic behavior** – Ensures predictable physics

### Performance Optimization
1. **Minimize allocations** – Avoid creating objects in callbacks
2. **Cache references** – Store Rigidbody and Transform references
3. **Batch operations** – Group related physics calculations
4. **Profile physics load** – Monitor fixed update performance

### Timing Considerations
1. **Fixed deltaTime** – Use the provided fixedDeltaTime parameter
2. **Frame rate independence** – Physics runs at consistent rate
3. **Interpolation** – Consider render frame interpolation needs
4. **Synchronization** – Coordinate with regular Update when needed

## Common Use Cases

- **Physics simulations** and rigid body dynamics
- **Movement systems** with consistent timing
- **Force applications** and physics effects
- **Animation systems** requiring fixed timestep
- **Deterministic calculations** for reproducible results
- **Performance profiling** of physics systems

## Thread Safety

- **Physics thread** – Callbacks execute during Unity's fixed update
- **Main thread only** – All operations on Unity's main thread
- **Synchronization** – No additional synchronization needed
- **Thread safety** – Unity handles physics thread safety