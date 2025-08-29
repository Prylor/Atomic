# 🧩 EntityView

`EntityView` is the concrete MonoBehaviour implementation of `IEntityView` in the Atomic framework. It provides a complete solution for visualizing entities in Unity, featuring installer-based configuration, behaviour management, gizmo drawing, and GameObject lifecycle control.

## Key Features

- **MonoBehaviour Implementation** – Full Unity integration with GameObject lifecycle
- **Installer System** – Modular configuration through `EntityViewInstaller` components
- **Behaviour Management** – Dynamic entity behaviour attachment and removal
- **Gizmo Support** – Visual debugging tools with customizable drawing modes
- **GameObject Control** – Automatic activation/deactivation based on visibility
- **Pool-Friendly** – Designed to work with object pooling systems
- **Editor Integration** – Inspector-friendly with proper serialization

---

## Class Definition

```csharp
[AddComponentMenu("Atomic/Entities/Entity View")]
[DisallowMultipleComponent]
public class EntityView : EntityViewBase
{
    // Configuration
    [SerializeField] private List<EntityViewInstaller> _installers;
    
    // Runtime state
    private IEntityBehaviour[] _behaviours;
    private int _behaviourCount;
    private bool _installed;
    
    // Gizmo settings
    [SerializeField] private bool _onlySelectedGizmos;
    [SerializeField] private bool _onlyEditModeGizmos;
}
```

## Inheritance

- **EntityViewBase** – Provides base `IEntityView` implementation
- **MonoBehaviour** – Unity component integration
- **IEntityView** – Core view interface contract

## Key Properties

### Behaviours
```csharp
private IEntityBehaviour[] _behaviours;
private int _behaviourCount;
```
- **Description**: Collection of behaviours applied to the entity when view is shown
- **Management**: Added/removed dynamically, synchronized with entity state

### Installers
```csharp
[SerializeField] private List<EntityViewInstaller> _installers;
```
- **Description**: Configuration components that set up the view during first show
- **Usage**: Modular view configuration, reusable setup logic

## Core Methods

### Behaviour Management

#### AddBehaviour
```csharp
public void AddBehaviour(IEntityBehaviour behaviour);
```
- **Description**: Adds a behaviour to the view and applies it to the entity if visible
- **Parameters**: `behaviour` – The behaviour to add
- **Behavior**: 
  - Prevents duplicate behaviours
  - Immediately applies to entity if view is visible
  - Throws `ArgumentNullException` if behaviour is null

#### HasBehaviour
```csharp
public bool HasBehaviour(IEntityBehaviour behaviour);
```
- **Description**: Checks if the view contains the specified behaviour
- **Returns**: True if behaviour is present

#### DelBehaviour
```csharp
public void DelBehaviour(IEntityBehaviour behaviour);
```
- **Description**: Removes behaviour from view and entity
- **Behavior**: Immediately removes from entity if view is visible

#### GetBehaviourAt
```csharp
public IEntityBehaviour GetBehaviourAt(int index);
```
- **Description**: Retrieves behaviour at specified index
- **Throws**: `ArgumentOutOfRangeException` for invalid index

### Static Factory Method

#### Create
```csharp
public static EntityView Create(CreateArgs args = default);
```
- **Description**: Creates a new EntityView GameObject with specified configuration
- **Parameters**: `CreateArgs` structure with setup options
- **Returns**: Configured EntityView instance

## Example Usage

### Basic Entity View Setup

```csharp
using UnityEngine;
using Atomic.Entities;

public class PlayerView : MonoBehaviour
{
    [SerializeField] private EntityView entityView;
    [SerializeField] private Renderer playerRenderer;
    [SerializeField] private Animator playerAnimator;
    
    private void Start()
    {
        // Create player entity
        var player = new Entity("Player");
        player.SetValue("Health", 100);
        player.SetValue("Level", 1);
        
        // Add behaviours to view before showing
        entityView.AddBehaviour(new PlayerMovementBehaviour());
        entityView.AddBehaviour(new PlayerAnimationBehaviour(playerAnimator));
        entityView.AddBehaviour(new PlayerHealthBehaviour());
        
        // Show the view with the entity
        entityView.Show(player);
    }
    
    public void ChangePlayerColor(Color newColor)
    {
        if (entityView.IsVisible)
        {
            playerRenderer.material.color = newColor;
        }
    }
}
```

### Procedural View Management

Following Atomic's procedural pattern:

```csharp
// Static utility methods for EntityView management
public static class EntityViewUtils
{
    public static EntityView CreatePlayerView(Vector3 position)
    {
        var args = new EntityView.CreateArgs
        {
            name = "Player View",
            installers = new List<EntityViewInstaller> 
            { 
                // Add installers as needed
            },
            behaviours = new IEntityBehaviour[]
            {
                new PlayerMovementBehaviour(),
                new PlayerCombatBehaviour(),
                new PlayerInventoryBehaviour()
            },
            onlySelectedGizmos = true,
            onlyEditModeGizmos = false
        };
        
        EntityView view = EntityView.Create(args);
        view.transform.position = position;
        
        return view;
    }
    
    public static void SetupEnemyView(EntityView view, EnemyType enemyType)
    {
        // Add behaviours based on enemy type
        switch (enemyType)
        {
            case EnemyType.Melee:
                view.AddBehaviour(new MeleeAttackBehaviour());
                break;
            case EnemyType.Ranged:
                view.AddBehaviour(new RangedAttackBehaviour());
                break;
            case EnemyType.Flying:
                view.AddBehaviour(new FlyingMovementBehaviour());
                break;
        }
        
        // Common enemy behaviours
        view.AddBehaviour(new EnemyAIBehaviour());
        view.AddBehaviour(new EnemyHealthBehaviour());
    }
    
    public static void AttachWeapon(EntityView view, WeaponBehaviour weapon)
    {
        if (view.IsVisible && !view.HasBehaviour(weapon))
        {
            view.AddBehaviour(weapon);
            Debug.Log($"Attached weapon to {view.Entity.Name}");
        }
    }
    
    public static void DetachWeapon(EntityView view, WeaponBehaviour weapon)
    {
        if (view.HasBehaviour(weapon))
        {
            view.DelBehaviour(weapon);
            Debug.Log($"Detached weapon from {view.Entity.Name}");
        }
    }
}

// Usage
public class GameManager : MonoBehaviour
{
    public void SpawnPlayer(Vector3 spawnPoint)
    {
        EntityView playerView = EntityViewUtils.CreatePlayerView(spawnPoint);
        
        // Create and show entity
        var player = new Entity("Player");
        player.SetValue("Health", 100);
        player.SetValue("Mana", 50);
        
        playerView.Show(player);
    }
    
    public void SpawnEnemy(Vector3 position, EnemyType type)
    {
        EntityView enemyView = EntityViewUtils.CreateEnemyView(position);
        EntityViewUtils.SetupEnemyView(enemyView, type);
        
        var enemy = new Entity($"Enemy_{type}");
        enemy.SetValue("Health", 75);
        enemy.SetValue("Damage", 25);
        
        enemyView.Show(enemy);
    }
}
```

### Dynamic Behaviour Management

```csharp
public class WeaponSystem : MonoBehaviour
{
    [SerializeField] private EntityView playerView;
    
    private WeaponBehaviour currentWeapon;
    
    public void EquipWeapon(WeaponData weaponData)
    {
        // Remove current weapon
        if (currentWeapon != null)
        {
            playerView.DelBehaviour(currentWeapon);
        }
        
        // Create and add new weapon behaviour
        currentWeapon = new WeaponBehaviour(weaponData);
        playerView.AddBehaviour(currentWeapon);
        
        // Update entity stats
        if (playerView.IsVisible)
        {
            var entity = playerView.Entity;
            entity.SetValue("Damage", weaponData.baseDamage);
            entity.SetValue("AttackSpeed", weaponData.attackSpeed);
        }
    }
    
    public void UnequipWeapon()
    {
        if (currentWeapon != null)
        {
            playerView.DelBehaviour(currentWeapon);
            currentWeapon = null;
            
            // Reset entity stats
            if (playerView.IsVisible)
            {
                var entity = playerView.Entity;
                entity.SetValue("Damage", 10); // Default damage
                entity.SetValue("AttackSpeed", 1.0f); // Default speed
            }
        }
    }
}
```

### Installer-Based Configuration

```csharp
// Create a custom installer for specific view setups
[CreateAssetMenu(menuName = "Atomic/Installers/Player View Installer")]
public class PlayerViewInstaller : EntityViewInstaller
{
    [SerializeField] private GameObject weaponMount;
    [SerializeField] private ParticleSystem levelUpEffect;
    [SerializeField] private AudioClip[] soundEffects;
    
    public override void Install(EntityView view)
    {
        // Add visual components behaviours
        view.AddBehaviour(new WeaponMountBehaviour(weaponMount));
        view.AddBehaviour(new LevelUpEffectBehaviour(levelUpEffect));
        view.AddBehaviour(new SoundEffectBehaviour(soundEffects));
        
        // Add gameplay behaviours
        view.AddBehaviour(new PlayerInputBehaviour());
        view.AddBehaviour(new PlayerStatsDisplayBehaviour());
        
        Debug.Log($"Player view installer completed for {view.name}");
    }
}

// Usage in EntityView setup
public class PlayerSpawner : MonoBehaviour
{
    [SerializeField] private PlayerViewInstaller playerInstaller;
    
    public EntityView SpawnPlayer()
    {
        var args = new EntityView.CreateArgs
        {
            name = "Player",
            installers = new List<EntityViewInstaller> { playerInstaller }
        };
        
        EntityView view = EntityView.Create(args);
        
        // Installer will automatically run when entity is first shown
        var player = new Entity("Player");
        view.Show(player);
        
        return view;
    }
}
```

### Gizmo Integration

```csharp
// Custom behaviour with gizmo drawing
public class AttackRangeBehaviour : IEntityBehaviour, IEntityGizmos
{
    private float attackRange = 3f;
    
    public void OnSpawn(IEntity entity) { }
    public void OnDespawn(IEntity entity) { }
    public void OnActivate(IEntity entity) { }
    public void OnDeactivate(IEntity entity) { }
    
    public void OnGizmosDraw(IEntity entity)
    {
        // Draw attack range in scene view
        if (entity.TryGetValue("Position", out Vector3 position))
        {
            Gizmos.color = Color.red;
            Gizmos.DrawWireSphere(position, attackRange);
            
            Gizmos.color = Color.yellow;
            Gizmos.DrawLine(position, position + Vector3.forward * attackRange);
        }
    }
}

// EntityView with gizmo configuration
public class CombatEntityView : MonoBehaviour
{
    [SerializeField] private EntityView entityView;
    
    private void Start()
    {
        // Configure gizmo settings
        // entityView._onlySelectedGizmos = true; // Only when selected
        // entityView._onlyEditModeGizmos = true; // Only in edit mode
        
        // Add behaviour with gizmo support
        entityView.AddBehaviour(new AttackRangeBehaviour());
        
        var entity = new Entity("Combat Unit");
        entity.SetValue("Position", transform.position);
        entityView.Show(entity);
    }
}
```

## Best Practices

### View Configuration
- **Use Installers** – Leverage installer system for reusable, modular configuration
- **Behaviour Composition** – Add behaviours based on entity type and requirements
- **Gizmo Settings** – Configure gizmo visibility based on debugging needs

```csharp
// ✅ Good: Modular installer-based setup
[SerializeField] private List<EntityViewInstaller> installers = new()
{
    playerMovementInstaller,
    playerCombatInstaller,
    playerUIInstaller
};

// ❌ Bad: Hardcoded behaviour setup in view
private void Start()
{
    entityView.AddBehaviour(new MovementBehaviour());
    entityView.AddBehaviour(new CombatBehaviour());
    // ... many more hardcoded behaviours
}
```

### Behaviour Management
- **Check Duplicates** – EntityView automatically prevents duplicate behaviours
- **Lifecycle Sync** – Behaviours are automatically added/removed with view visibility
- **Dynamic Changes** – Add/remove behaviours at runtime based on gameplay state

```csharp
// ✅ Good: Dynamic behaviour management
public void EnterCombatMode()
{
    if (!entityView.HasBehaviour(combatBehaviour))
    {
        entityView.AddBehaviour(combatBehaviour);
    }
}

// ✅ Good: Safe behaviour removal
public void ExitCombatMode()
{
    entityView.DelBehaviour(combatBehaviour);
}
```

### Gizmo Usage
- **Performance** – Use `onlySelectedGizmos` for expensive gizmo operations
- **Context** – Use `onlyEditModeGizmos` for debug-only visualizations
- **Visual Clarity** – Different colors for different gizmo types

## Performance Considerations

### Behaviour Optimization
- **Minimal Behaviours** – Only add necessary behaviours to reduce update overhead
- **Behaviour Pooling** – Reuse behaviour instances when possible
- **Conditional Updates** – Use entity state to control behaviour execution

```csharp
// Efficient behaviour that checks conditions
public class OptimizedMovementBehaviour : IEntityBehaviour, IEntityUpdate
{
    public void OnUpdate(IEntity entity, float deltaTime)
    {
        // Only update if entity is active and has movement
        if (!entity.GetValue<bool>("IsMoving")) return;
        
        Vector3 velocity = entity.GetValue<Vector3>("Velocity");
        if (velocity == Vector3.zero) return;
        
        // Perform movement update
        UpdatePosition(entity, velocity, deltaTime);
    }
}
```

### Gizmo Performance
- **Selective Drawing** – Use gizmo flags to control when gizmos are drawn
- **LOD Gizmos** – Reduce gizmo complexity based on distance
- **Batch Operations** – Group multiple gizmo draws together

### Memory Management
- **View Pooling** – Reuse EntityView instances to reduce allocation
- **Behaviour Cleanup** – Behaviours are automatically cleaned up on Hide()
- **Installer Caching** – Installers run only once per view instance

## Integration with Unity Systems

### Prefab Integration
```csharp
// EntityView as prefab component
public class PrefabEntityView : MonoBehaviour
{
    [SerializeField] private EntityView entityView;
    [SerializeField] private EntityViewInstaller[] installers;
    
    private void Awake()
    {
        // Configure installers from prefab setup
        entityView._installers = new List<EntityViewInstaller>(installers);
    }
    
    public void Initialize(IEntity entity)
    {
        entityView.Show(entity);
    }
}
```

### Animation Integration
```csharp
public class AnimatedEntityView : MonoBehaviour
{
    [SerializeField] private EntityView entityView;
    [SerializeField] private Animator animator;
    
    private void Start()
    {
        // Add animation behaviour
        var animBehaviour = new AnimationSyncBehaviour(animator);
        entityView.AddBehaviour(animBehaviour);
        
        var entity = new Entity("Animated Character");
        entity.SetValue("AnimationState", "Idle");
        entityView.Show(entity);
    }
}
```

### UI Integration
```csharp
public class UIEntityView : MonoBehaviour
{
    [SerializeField] private EntityView entityView;
    [SerializeField] private Canvas uiCanvas;
    
    private void Start()
    {
        // Add UI update behaviour
        var uiBehaviour = new UIUpdateBehaviour(uiCanvas);
        entityView.AddBehaviour(uiBehaviour);
    }
}
```

## CreateArgs Structure

```csharp
[Serializable]
public struct CreateArgs
{
    [Tooltip("The name of the new GameObject to create for the EntityView.")]
    public string name;

    [Tooltip("Installers that will configure the EntityView upon creation.")]
    public List<EntityViewInstaller> installers;

    [Tooltip("Behaviours that will be added to the entity when the view is shown.")]
    public IEnumerable<IEntityBehaviour> behaviours;

    [Tooltip("If true, gizmos will be drawn only in Edit Mode.")]
    public bool onlyEditModeGizmos;

    [Tooltip("If true, gizmos will be drawn only when the object is selected.")]
    public bool onlySelectedGizmos;
}
```

## Notes

- **Unity Integration** – Full MonoBehaviour support with proper serialization
- **Modular Configuration** – Installer system enables reusable view setups
- **Behaviour Composition** – Dynamic behaviour management for flexible entity presentation
- **Visual Debugging** – Integrated gizmo system for development and debugging
- **Performance Optimized** – Efficient behaviour management and selective updates
- **Pool Compatible** – Designed to work with object pooling systems