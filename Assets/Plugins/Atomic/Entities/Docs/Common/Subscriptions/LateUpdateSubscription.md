# 🔚 LateUpdateSubscription

`LateUpdateSubscription` is a disposable subscription handle that automatically unregisters a callback from an `IUpdatable`'s `OnLateUpdated` event when disposed. It provides safe event subscription management for post-processing update events that occur after all regular Update calls.

## Key Features

- **Post-Processing Focus** – Handles LateUpdate cycle events
- **Camera and Animation** – Perfect for camera tracking and final animations
- **Execution Order** – Runs after all Update calls complete
- **Automatic Cleanup** – Unsubscribes callback on disposal
- **Delta Time Support** – Callbacks receive deltaTime parameter

---

## Structure Definition

```csharp
public readonly struct LateUpdateSubscription : IDisposable
{
    private readonly IUpdatable _source;
    private readonly Action<float> _callback;
}
```

### Constructor

```csharp
internal LateUpdateSubscription(IUpdatable source, Action<float> callback)
```

- **source**: The updatable object to subscribe to
- **callback**: The action to invoke on each late update (receives deltaTime)

### Disposal

```csharp
public void Dispose()
```

Safely unsubscribes the callback from the source's `OnLateUpdated` event.

---

## Usage Patterns

### Camera Following System

```csharp
public class CameraFollowSystem
{
    private readonly Dictionary<IUpdatable, (Camera camera, Transform target, Vector3 offset, LateUpdateSubscription subscription)> cameraFollows = new();
    
    public void RegisterCameraFollow(IUpdatable system, Camera camera, Transform target, Vector3 offset)
    {
        var subscription = system.SubscribeToLateUpdate(deltaTime => 
            UpdateCameraFollow(camera, target, offset, deltaTime));
            
        cameraFollows[system] = (camera, target, offset, subscription);
    }
    
    private void UpdateCameraFollow(Camera camera, Transform target, Vector3 offset, float deltaTime)
    {
        if (target == null || camera == null) return;
        
        // Smooth camera following after all entity updates
        Vector3 targetPosition = target.position + offset;
        camera.transform.position = Vector3.Lerp(camera.transform.position, targetPosition, 5f * deltaTime);
        
        // Optional look-at behavior
        Vector3 lookDirection = target.position - camera.transform.position;
        if (lookDirection != Vector3.zero)
        {
            Quaternion targetRotation = Quaternion.LookRotation(lookDirection);
            camera.transform.rotation = Quaternion.Lerp(camera.transform.rotation, targetRotation, 2f * deltaTime);
        }
    }
    
    public void UnregisterCameraFollow(IUpdatable system)
    {
        if (cameraFollows.TryGetValue(system, out var data))
        {
            data.subscription.Dispose();
            cameraFollows.Remove(system);
        }
    }
}
```

### UI Update System

```csharp
public class UILateUpdateSystem
{
    private readonly Dictionary<IUpdatable, (List<UIElement> elements, LateUpdateSubscription subscription)> uiSystems = new();
    
    public struct UIElement
    {
        public RectTransform RectTransform;
        public Transform WorldTarget;
        public Vector3 Offset;
        public bool FollowTarget;
    }
    
    public void RegisterUISystem(IUpdatable system, List<UIElement> uiElements)
    {
        var subscription = system.SubscribeToLateUpdate(deltaTime => 
            UpdateUIElements(uiElements, deltaTime));
            
        uiSystems[system] = (uiElements, subscription);
    }
    
    private void UpdateUIElements(List<UIElement> elements, float deltaTime)
    {
        foreach (var element in elements)
        {
            if (element.FollowTarget && element.WorldTarget != null)
            {
                // Update UI element to follow world position after all transforms have been updated
                Vector3 screenPosition = Camera.main.WorldToScreenPoint(element.WorldTarget.position + element.Offset);
                element.RectTransform.position = screenPosition;
            }
            
            // Additional UI updates that depend on final world state
            UpdateUIElementState(element, deltaTime);
        }
    }
    
    private void UpdateUIElementState(UIElement element, float deltaTime)
    {
        // Update health bars, name tags, etc. based on final entity states
    }
    
    public void UnregisterUISystem(IUpdatable system)
    {
        if (uiSystems.TryGetValue(system, out var data))
        {
            data.subscription.Dispose();
            uiSystems.Remove(system);
        }
    }
}
```

### Animation Finalization System

```csharp
public class AnimationFinalizationSystem
{
    private readonly Dictionary<IUpdatable, (List<AnimationState> animations, LateUpdateSubscription subscription)> animationSystems = new();
    
    public struct AnimationState
    {
        public Animator Animator;
        public Transform Transform;
        public string CurrentState;
        public float BlendWeight;
        public bool RequiresFinalization;
    }
    
    public void RegisterAnimationSystem(IUpdatable system, List<AnimationState> animations)
    {
        var subscription = system.SubscribeToLateUpdate(deltaTime => 
            FinalizeAnimations(animations, deltaTime));
            
        animationSystems[system] = (animations, subscription);
    }
    
    private void FinalizeAnimations(List<AnimationState> animations, float deltaTime)
    {
        foreach (var animState in animations)
        {
            if (!animState.RequiresFinalization) continue;
            
            // Apply final animation adjustments after all other updates
            FinalizeAnimationState(animState, deltaTime);
        }
    }
    
    private void FinalizeAnimationState(AnimationState animState, float deltaTime)
    {
        // Apply root motion corrections
        ApplyRootMotionCorrections(animState);
        
        // Blend between animation states
        BlendAnimationLayers(animState, deltaTime);
        
        // Apply inverse kinematics or other post-processing
        ApplyInverseKinematics(animState);
    }
    
    private void ApplyRootMotionCorrections(AnimationState animState)
    {
        // Correct root motion based on final entity positions
    }
    
    private void BlendAnimationLayers(AnimationState animState, float deltaTime)
    {
        // Final blending between animation layers
    }
    
    private void ApplyInverseKinematics(AnimationState animState)
    {
        // IK adjustments based on final world positions
    }
}
```

### Rendering Finalization System

```csharp
public class RenderingFinalizationSystem
{
    private readonly Dictionary<IUpdatable, (List<Renderer> renderers, LateUpdateSubscription subscription)> renderingSystems = new();
    
    public void RegisterRenderingSystem(IUpdatable system, List<Renderer> renderers)
    {
        var subscription = system.SubscribeToLateUpdate(deltaTime => 
            FinalizeRendering(renderers, deltaTime));
            
        renderingSystems[system] = (renderers, subscription);
    }
    
    private void FinalizeRendering(List<Renderer> renderers, float deltaTime)
    {
        foreach (var renderer in renderers)
        {
            if (renderer == null) continue;
            
            // Final rendering adjustments after all transforms are updated
            UpdateMaterialProperties(renderer, deltaTime);
            UpdateLevelOfDetail(renderer);
            UpdateCullingMask(renderer);
        }
    }
    
    private void UpdateMaterialProperties(Renderer renderer, float deltaTime)
    {
        // Update shader properties based on final entity states
        var material = renderer.material;
        
        // Example: Update world position-based effects
        Vector3 worldPos = renderer.transform.position;
        material.SetVector("_WorldPosition", worldPos);
        
        // Example: Update time-based effects
        material.SetFloat("_Time", Time.time);
    }
    
    private void UpdateLevelOfDetail(Renderer renderer)
    {
        // Adjust LOD based on final camera distance
        float distanceToCamera = Vector3.Distance(renderer.transform.position, Camera.main.transform.position);
        
        // Apply LOD logic
        ApplyLODLevel(renderer, distanceToCamera);
    }
    
    private void UpdateCullingMask(Renderer renderer)
    {
        // Update culling based on final visibility state
    }
}
```

### Debug Visualization System

```csharp
public class DebugVisualizationSystem
{
    private readonly List<LateUpdateSubscription> subscriptions = new();
    private readonly Dictionary<IUpdatable, DebugSettings> debugSettings = new();
    
    public struct DebugSettings
    {
        public bool ShowVelocityVectors;
        public bool ShowBoundingBoxes;
        public bool ShowEntityInfo;
        public Color DebugColor;
    }
    
    public void RegisterDebugSystem(IUpdatable system, DebugSettings settings)
    {
        debugSettings[system] = settings;
        
        var subscription = system.SubscribeToLateUpdate(deltaTime => 
            DrawDebugVisualization(system, deltaTime));
        subscriptions.Add(subscription);
    }
    
    private void DrawDebugVisualization(IUpdatable system, float deltaTime)
    {
        if (!debugSettings.TryGetValue(system, out var settings)) return;
        
        // Draw debug information after all updates are complete
        if (settings.ShowVelocityVectors)
        {
            DrawVelocityVectors(system, settings.DebugColor);
        }
        
        if (settings.ShowBoundingBoxes)
        {
            DrawBoundingBoxes(system, settings.DebugColor);
        }
        
        if (settings.ShowEntityInfo)
        {
            DrawEntityInformation(system);
        }
    }
    
    private void DrawVelocityVectors(IUpdatable system, Color color)
    {
        // Draw velocity vectors for entities in the system
        var entities = GetSystemEntities(system);
        foreach (var entity in entities)
        {
            if (TryGetVelocity(entity, out var velocity) && velocity.magnitude > 0.1f)
            {
                Vector3 position = GetEntityPosition(entity);
                Debug.DrawRay(position, velocity, color);
            }
        }
    }
    
    private void DrawBoundingBoxes(IUpdatable system, Color color)
    {
        // Draw bounding boxes for entities
        var entities = GetSystemEntities(system);
        foreach (var entity in entities)
        {
            var bounds = GetEntityBounds(entity);
            DrawWireCube(bounds.center, bounds.size, color);
        }
    }
    
    private void DrawEntityInformation(IUpdatable system)
    {
        // Draw on-screen entity information
        var entities = GetSystemEntities(system);
        foreach (var entity in entities)
        {
            Vector3 worldPos = GetEntityPosition(entity);
            Vector3 screenPos = Camera.main.WorldToScreenPoint(worldPos);
            
            if (screenPos.z > 0) // In front of camera
            {
                DrawEntityInfoText(screenPos, entity);
            }
        }
    }
}
```

### Late Update Performance Monitor

```csharp
public class LateUpdatePerformanceMonitor
{
    private readonly Dictionary<IUpdatable, LateUpdateMetrics> systemMetrics = new();
    private readonly List<LateUpdateSubscription> subscriptions = new();
    
    public struct LateUpdateMetrics
    {
        public int LateUpdateCount;
        public float TotalLateUpdateTime;
        public float AverageLateUpdateTime;
        public float MaxLateUpdateTime;
        public float LastFrameLateUpdateTime;
    }
    
    public void StartMonitoring(IEnumerable<IUpdatable> systems)
    {
        foreach (var system in systems)
        {
            var subscription = system.SubscribeToLateUpdate(deltaTime => 
                MonitorLateUpdate(system, deltaTime));
            subscriptions.Add(subscription);
            
            systemMetrics[system] = new LateUpdateMetrics();
        }
    }
    
    private void MonitorLateUpdate(IUpdatable system, float deltaTime)
    {
        var startTime = Time.realtimeSinceStartup;
        
        // Monitor the late update phase
        var metrics = systemMetrics[system];
        
        metrics.LateUpdateCount++;
        metrics.LastFrameLateUpdateTime = deltaTime;
        metrics.TotalLateUpdateTime += deltaTime;
        metrics.AverageLateUpdateTime = metrics.TotalLateUpdateTime / metrics.LateUpdateCount;
        metrics.MaxLateUpdateTime = Mathf.Max(metrics.MaxLateUpdateTime, deltaTime);
        
        systemMetrics[system] = metrics;
        
        // Log performance warnings
        if (deltaTime > 0.020f) // 50 FPS threshold
        {
            Debug.LogWarning($"Slow late update frame: {deltaTime:F3}s for {system.GetType().Name}");
        }
        
        var processingTime = Time.realtimeSinceStartup - startTime;
        if (processingTime > 0.005f) // 5ms threshold
        {
            Debug.LogWarning($"Late update processing took {processingTime:F3}s for {system.GetType().Name}");
        }
    }
    
    public LateUpdateMetrics GetMetrics(IUpdatable system)
    {
        return systemMetrics.TryGetValue(system, out var metrics) ? metrics : default;
    }
    
    public void PrintLateUpdateReport()
    {
        Debug.Log("=== Late Update Performance Report ===");
        foreach (var kvp in systemMetrics)
        {
            var system = kvp.Key;
            var metrics = kvp.Value;
            
            Debug.Log($"{system.GetType().Name}:");
            Debug.Log($"  Late Updates: {metrics.LateUpdateCount}");
            Debug.Log($"  Average Time: {metrics.AverageLateUpdateTime:F4}s");
            Debug.Log($"  Max Time: {metrics.MaxLateUpdateTime:F4}s");
            Debug.Log($"  Last Frame: {metrics.LastFrameLateUpdateTime:F4}s");
        }
    }
}
```

### Post-Processing Effects System

```csharp
public class PostProcessingEffectsSystem
{
    private readonly Dictionary<IUpdatable, (List<PostProcessingEffect> effects, LateUpdateSubscription subscription)> effectSystems = new();
    
    public abstract class PostProcessingEffect
    {
        public abstract void ApplyEffect(float deltaTime);
        public bool Enabled { get; set; } = true;
    }
    
    public void RegisterEffectSystem(IUpdatable system, List<PostProcessingEffect> effects)
    {
        var subscription = system.SubscribeToLateUpdate(deltaTime => 
            ApplyPostProcessingEffects(effects, deltaTime));
            
        effectSystems[system] = (effects, subscription);
    }
    
    private void ApplyPostProcessingEffects(List<PostProcessingEffect> effects, float deltaTime)
    {
        // Apply effects in order after all rendering is complete
        foreach (var effect in effects)
        {
            if (effect.Enabled)
            {
                try
                {
                    effect.ApplyEffect(deltaTime);
                }
                catch (Exception e)
                {
                    Debug.LogError($"Error applying post-processing effect: {e}");
                }
            }
        }
    }
    
    public void UnregisterEffectSystem(IUpdatable system)
    {
        if (effectSystems.TryGetValue(system, out var data))
        {
            data.subscription.Dispose();
            effectSystems.Remove(system);
        }
    }
}

// Example post-processing effect
public class ScreenShakeEffect : PostProcessingEffectsSystem.PostProcessingEffect
{
    private float intensity;
    private float duration;
    private float remainingTime;
    
    public void TriggerShake(float intensity, float duration)
    {
        this.intensity = intensity;
        this.duration = duration;
        this.remainingTime = duration;
        this.Enabled = true;
    }
    
    public override void ApplyEffect(float deltaTime)
    {
        if (remainingTime <= 0)
        {
            Enabled = false;
            return;
        }
        
        // Apply screen shake to camera
        float shakeAmount = intensity * (remainingTime / duration);
        Vector3 shakeOffset = Random.insideUnitSphere * shakeAmount;
        shakeOffset.z = 0; // Keep camera on same Z plane
        
        Camera.main.transform.localPosition += shakeOffset;
        
        remainingTime -= deltaTime;
    }
}
```

## Integration with Unity Systems

### Transform Hierarchy Updates

```csharp
public class TransformHierarchyFinalizer
{
    private readonly List<LateUpdateSubscription> subscriptions = new();
    
    public void RegisterHierarchySystem(IUpdatable system, Transform[] rootTransforms)
    {
        var subscription = system.SubscribeToLateUpdate(deltaTime => 
            FinalizeTransformHierarchies(rootTransforms, deltaTime));
        subscriptions.Add(subscription);
    }
    
    private void FinalizeTransformHierarchies(Transform[] rootTransforms, float deltaTime)
    {
        foreach (var root in rootTransforms)
        {
            if (root == null) continue;
            
            // Apply final transform corrections after all updates
            ProcessTransformHierarchy(root, deltaTime);
        }
    }
    
    private void ProcessTransformHierarchy(Transform transform, float deltaTime)
    {
        // Apply any final transform adjustments
        ApplyConstraints(transform);
        
        // Process children
        foreach (Transform child in transform)
        {
            ProcessTransformHierarchy(child, deltaTime);
        }
    }
    
    private void ApplyConstraints(Transform transform)
    {
        // Apply position, rotation, or scale constraints
    }
}
```

## Best Practices

### Execution Order Considerations
1. **Post-processing focus** – Use for operations that depend on final state
2. **Camera updates** – Perfect for camera following and effects
3. **UI updates** – Update UI elements based on final world positions
4. **Final adjustments** – Apply corrections and finalizations

### Performance Optimization
1. **Lightweight operations** – Keep late update callbacks fast
2. **Batch processing** – Group related late update operations
3. **Avoid heavy calculations** – Late update runs every frame
4. **Cache references** – Store frequently accessed components

### Use Case Alignment
1. **Animation finalization** – Apply final animation adjustments
2. **Rendering preparation** – Prepare for rendering pipeline
3. **Debug visualization** – Draw debug information after all updates
4. **Post-processing effects** – Apply visual effects and filters

## Common Use Cases

- **Camera systems** that follow or track entities
- **UI elements** that need to track world positions
- **Animation finalization** and blending
- **Debug visualization** and development tools
- **Post-processing effects** and visual enhancements
- **Final transform adjustments** and constraints

## Thread Safety

- **Main thread only** – All operations on Unity's main thread
- **Render thread preparation** – Prepares data for rendering
- **No synchronization needed** – Single-threaded execution
- **Unity integration** – Follows Unity's update cycle timing