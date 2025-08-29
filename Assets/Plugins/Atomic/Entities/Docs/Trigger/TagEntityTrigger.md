# 🧩 TagEntityTrigger

A specialized entity trigger that monitors tag-related changes on entities, responding to tag additions and deletions. Provides reactive monitoring for entity tag state changes, enabling automatic system responses to entity classification updates.

## Overview

`TagEntityTrigger` automatically tracks when tags are added to or removed from entities, triggering callbacks for reactive system updates. Essential for building systems that respond to entity state classification changes, role updates, or behavioral flag modifications.

## Class Definition

```csharp
namespace Atomic.Entities
{
    /// <summary>
    /// A non-generic shortcut for TagEntityTrigger{IEntity}.
    /// Subscribes to tag-related events (OnTagAdded, OnTagDeleted) on basic IEntity instances.
    /// </summary>
    public class TagEntityTrigger : TagEntityTrigger<IEntity>
    {
    }
    
    /// <summary>
    /// A trigger that responds to tag changes (added or removed) on entities of type E.
    /// </summary>
    /// <typeparam name="E">The entity type, which must implement IEntity.</typeparam>
    public class TagEntityTrigger<E> : EntityTriggerBase<E> where E : IEntity
    {
        private readonly bool _added;
        private readonly bool _deleted;

        public TagEntityTrigger(bool added = true, bool deleted = true)
        {
            _added = added;
            _deleted = deleted;
        }

        public override void Track(E entity)
        {
            if (_added) entity.OnTagAdded += this.OnTagAdded;
            if (_deleted) entity.OnTagDeleted += this.OnTagDeleted;
        }

        public override void Untrack(E entity)
        {
            if (_added) entity.OnTagAdded -= this.OnTagAdded;
            if (_deleted) entity.OnTagDeleted -= this.OnTagDeleted;
        }

        private void OnTagDeleted(IEntity entity, int tag) => _action.Invoke((E) entity);
        private void OnTagAdded(IEntity entity, int tag) => _action.Invoke((E) entity);
    }
}
```

## Key Features

### Selective Monitoring
- Configurable tracking of tag additions only
- Configurable tracking of tag deletions only
- Full monitoring of both operations by default

### Automatic Event Management
- Automatic subscription to entity tag events
- Clean unsubscription for resource management
- Type-safe entity casting in callbacks

### Lightweight Operation
- Minimal overhead per tracked entity
- Event-driven architecture avoids polling
- Efficient for high-frequency tag changes

## Usage Examples

### Basic Tag Change Monitoring

```csharp
// Monitor all tag changes (additions and deletions)
var allTagTrigger = new TagEntityTrigger();
allTagTrigger.SetAction(entity =>
{
    Debug.Log($"Tag change detected on entity: {entity.Name}");
    // Re-evaluate entity in all relevant filters
    EntityManager.Instance.ReEvaluateEntity(entity);
});

// Track entities
allTagTrigger.Track(playerEntity);
allTagTrigger.Track(enemyEntity);

// When tags change on entities, the trigger automatically responds
playerEntity.AddTag("PoweredUp");      // Triggers callback
enemyEntity.RemoveTag("Aggressive");   // Triggers callback
```

### Addition-Only Tag Monitoring

```csharp
// Monitor only when tags are added
var additionTrigger = new TagEntityTrigger(added: true, deleted: false);
additionTrigger.SetAction(entity =>
{
    Debug.Log($"New tag added to entity: {entity.Name}");
    
    // Check for specific tag combinations
    if (entity.HasTag("Player") && entity.HasTag("Armed") && entity.HasTag("Ready"))
    {
        GameEvents.OnPlayerReadyForCombat?.Invoke(entity);
    }
});

// This will only trigger when tags are added, not removed
additionTrigger.Track(playerEntity);
```

### Deletion-Only Tag Monitoring

```csharp
// Monitor only when tags are removed
var deletionTrigger = new TagEntityTrigger(added: false, deleted: true);
deletionTrigger.SetAction(entity =>
{
    Debug.Log($"Tag removed from entity: {entity.Name}");
    
    // Check if entity lost critical tags
    if (!entity.HasTag("Active") || !entity.HasTag("Alive"))
    {
        GameEvents.OnEntityDeactivated?.Invoke(entity);
    }
});

deletionTrigger.Track(npcEntity);
```

### Combat State Management System

```csharp
public class CombatStateManager : MonoBehaviour
{
    private readonly TagEntityTrigger _combatTagTrigger = new TagEntityTrigger();
    private readonly List<IEntity> _combatReadyEntities = new();
    private readonly List<IEntity> _engagedEntities = new();

    void Start()
    {
        _combatTagTrigger.SetAction(OnCombatTagChanged);
        
        // Register to track all combat entities
        GameEvents.OnEntityCreated += entity =>
        {
            if (entity.HasTag("Combatant"))
            {
                _combatTagTrigger.Track(entity);
            }
        };
    }

    private void OnCombatTagChanged(IEntity entity)
    {
        UpdateCombatReadiness(entity);
        UpdateEngagementStatus(entity);
        EvaluateCombatState(entity);
    }

    private void UpdateCombatReadiness(IEntity entity)
    {
        bool isReady = entity.HasTag("Armed") && 
                      entity.HasTag("Healthy") && 
                      entity.HasTag("Active");

        bool wasReady = _combatReadyEntities.Contains(entity);

        if (isReady && !wasReady)
        {
            _combatReadyEntities.Add(entity);
            Debug.Log($"Entity {entity.Name} is now combat ready");
            GameEvents.OnEntityCombatReady?.Invoke(entity);
        }
        else if (!isReady && wasReady)
        {
            _combatReadyEntities.Remove(entity);
            Debug.Log($"Entity {entity.Name} is no longer combat ready");
            GameEvents.OnEntityCombatNotReady?.Invoke(entity);
        }
    }

    private void UpdateEngagementStatus(IEntity entity)
    {
        bool isEngaged = entity.HasTag("InCombat") || entity.HasTag("Targeting");
        bool wasEngaged = _engagedEntities.Contains(entity);

        if (isEngaged && !wasEngaged)
        {
            _engagedEntities.Add(entity);
            StartCombatBehavior(entity);
        }
        else if (!isEngaged && wasEngaged)
        {
            _engagedEntities.Remove(entity);
            StopCombatBehavior(entity);
        }
    }

    private void EvaluateCombatState(IEntity entity)
    {
        // Evaluate complex combat state combinations
        if (entity.HasTag("Player"))
        {
            EvaluatePlayerCombatState(entity);
        }
        else if (entity.HasTag("Enemy"))
        {
            EvaluateEnemyCombatState(entity);
        }
    }

    private void EvaluatePlayerCombatState(IEntity player)
    {
        bool hasWeapon = player.HasTag("Armed");
        bool isHealthy = player.HasTag("Healthy");
        bool hasStamina = player.HasTag("EnoughStamina");
        bool inCombat = player.HasTag("InCombat");

        if (inCombat && (!hasWeapon || !isHealthy || !hasStamina))
        {
            // Player is in combat but not properly equipped
            GameEvents.OnPlayerCombatDisadvantage?.Invoke(player);
        }
        else if (!inCombat && hasWeapon && isHealthy && hasStamina)
        {
            // Player is well-equipped and ready for combat
            GameEvents.OnPlayerOptimalCombatState?.Invoke(player);
        }
    }

    private void EvaluateEnemyCombatState(IEntity enemy)
    {
        bool isAggressive = enemy.HasTag("Aggressive");
        bool canSeePlayer = enemy.HasTag("PlayerVisible");
        bool isAlerted = enemy.HasTag("Alerted");

        if (isAggressive && canSeePlayer && !isAlerted)
        {
            // Enemy should become alerted
            enemy.AddTag("Alerted");
            GameEvents.OnEnemyBecomeAlert?.Invoke(enemy);
        }
        else if (!canSeePlayer && isAlerted)
        {
            // Enemy should start searching
            enemy.AddTag("Searching");
            GameEvents.OnEnemyStartSearching?.Invoke(enemy);
        }
    }

    private void StartCombatBehavior(IEntity entity)
    {
        Debug.Log($"Starting combat behavior for {entity.Name}");
        // Initialize combat AI, animations, etc.
    }

    private void StopCombatBehavior(IEntity entity)
    {
        Debug.Log($"Stopping combat behavior for {entity.Name}");
        // Cleanup combat state, return to normal behavior
    }

    void OnDestroy()
    {
        // Clean up all tracked entities
        foreach (var entity in _combatReadyEntities.Concat(_engagedEntities).Distinct())
        {
            _combatTagTrigger.Untrack(entity);
        }
    }
}
```

### Role-Based Permission System

```csharp
public class EntityPermissionSystem : MonoBehaviour
{
    [System.Serializable]
    public struct RolePermission
    {
        public string roleName;
        public string[] allowedActions;
        public string[] restrictedAreas;
    }

    [SerializeField] private RolePermission[] _rolePermissions;
    
    private readonly TagEntityTrigger _roleTrigger = new TagEntityTrigger();
    private readonly Dictionary<string, RolePermission> _permissionMap = new();
    private readonly Dictionary<IEntity, HashSet<string>> _entityRoles = new();

    void Start()
    {
        // Build permission map
        foreach (var permission in _rolePermissions)
        {
            _permissionMap[permission.roleName] = permission;
        }

        _roleTrigger.SetAction(OnEntityRoleChanged);
        
        // Track all entities with roles
        EntityRegistry.ForEach(entity =>
        {
            if (HasAnyRole(entity))
            {
                _roleTrigger.Track(entity);
            }
        });
    }

    private void OnEntityRoleChanged(IEntity entity)
    {
        UpdateEntityPermissions(entity);
        ValidateEntityAccess(entity);
        NotifyRoleChange(entity);
    }

    private void UpdateEntityPermissions(IEntity entity)
    {
        var currentRoles = GetEntityRoles(entity);
        var previousRoles = _entityRoles.ContainsKey(entity) 
            ? _entityRoles[entity] 
            : new HashSet<string>();

        _entityRoles[entity] = currentRoles;

        // Find added and removed roles
        var addedRoles = currentRoles.Except(previousRoles).ToList();
        var removedRoles = previousRoles.Except(currentRoles).ToList();

        foreach (string role in addedRoles)
        {
            GrantRolePermissions(entity, role);
            Debug.Log($"Entity {entity.Name} gained role: {role}");
        }

        foreach (string role in removedRoles)
        {
            RevokeRolePermissions(entity, role);
            Debug.Log($"Entity {entity.Name} lost role: {role}");
        }
    }

    private HashSet<string> GetEntityRoles(IEntity entity)
    {
        var roles = new HashSet<string>();
        
        foreach (var permission in _permissionMap.Keys)
        {
            if (entity.HasTag(permission))
            {
                roles.Add(permission);
            }
        }

        return roles;
    }

    private void GrantRolePermissions(IEntity entity, string role)
    {
        if (_permissionMap.TryGetValue(role, out var permission))
        {
            // Grant allowed actions
            foreach (string action in permission.allowedActions)
            {
                entity.AddTag($"CanDo_{action}");
            }

            // Grant area access
            foreach (string area in permission.restrictedAreas)
            {
                entity.AddTag($"CanAccess_{area}");
            }

            GameEvents.OnEntityRoleGranted?.Invoke(entity, role);
        }
    }

    private void RevokeRolePermissions(IEntity entity, string role)
    {
        if (_permissionMap.TryGetValue(role, out var permission))
        {
            // Revoke actions (only if no other roles grant them)
            foreach (string action in permission.allowedActions)
            {
                if (!HasActionPermissionFromOtherRoles(entity, action, role))
                {
                    entity.RemoveTag($"CanDo_{action}");
                }
            }

            // Revoke area access (only if no other roles grant them)
            foreach (string area in permission.restrictedAreas)
            {
                if (!HasAreaPermissionFromOtherRoles(entity, area, role))
                {
                    entity.RemoveTag($"CanAccess_{area}");
                }
            }

            GameEvents.OnEntityRoleRevoked?.Invoke(entity, role);
        }
    }

    private bool HasActionPermissionFromOtherRoles(IEntity entity, string action, string excludeRole)
    {
        return _entityRoles[entity]
            .Where(role => role != excludeRole)
            .Any(role => _permissionMap[role].allowedActions.Contains(action));
    }

    private bool HasAreaPermissionFromOtherRoles(IEntity entity, string area, string excludeRole)
    {
        return _entityRoles[entity]
            .Where(role => role != excludeRole)
            .Any(role => _permissionMap[role].restrictedAreas.Contains(area));
    }

    private void ValidateEntityAccess(IEntity entity)
    {
        // Check if entity is in a restricted area without permission
        if (entity.HasValue<string>("CurrentArea"))
        {
            string currentArea = entity.Get<string>("CurrentArea");
            if (!entity.HasTag($"CanAccess_{currentArea}"))
            {
                GameEvents.OnEntityUnauthorizedAccess?.Invoke(entity, currentArea);
            }
        }
    }

    private void NotifyRoleChange(IEntity entity)
    {
        var roles = GetEntityRoles(entity).ToArray();
        GameEvents.OnEntityRolesChanged?.Invoke(entity, roles);
    }

    private bool HasAnyRole(IEntity entity)
    {
        return _permissionMap.Keys.Any(role => entity.HasTag(role));
    }

    public bool CanEntityPerformAction(IEntity entity, string action)
    {
        return entity.HasTag($"CanDo_{action}");
    }

    public bool CanEntityAccessArea(IEntity entity, string area)
    {
        return entity.HasTag($"CanAccess_{area}");
    }

    public string[] GetEntityRoles(IEntity entity)
    {
        return _entityRoles.ContainsKey(entity) 
            ? _entityRoles[entity].ToArray() 
            : new string[0];
    }
}
```

### Dynamic Group Management System

```csharp
public class EntityGroupManager : MonoBehaviour
{
    [System.Serializable]
    public struct GroupConfig
    {
        public string groupName;
        public string[] requiredTags;
        public string[] exclusiveTags; // Tags that remove from group
        public int maxMembers;
    }

    [SerializeField] private GroupConfig[] _groupConfigs;
    
    private readonly Dictionary<string, List<IEntity>> _groups = new();
    private readonly Dictionary<IEntity, HashSet<string>> _entityGroups = new();
    private readonly TagEntityTrigger _groupTrigger = new TagEntityTrigger();

    void Start()
    {
        // Initialize groups
        foreach (var config in _groupConfigs)
        {
            _groups[config.groupName] = new List<IEntity>();
        }

        _groupTrigger.SetAction(OnEntityTagChanged);
        
        // Track all entities
        EntityRegistry.ForEach(entity => _groupTrigger.Track(entity));
        EntityRegistry.OnEntityCreated += _groupTrigger.Track;
    }

    private void OnEntityTagChanged(IEntity entity)
    {
        var currentGroups = _entityGroups.ContainsKey(entity)
            ? new HashSet<string>(_entityGroups[entity])
            : new HashSet<string>();

        var newGroups = new HashSet<string>();

        // Check which groups the entity should belong to
        foreach (var config in _groupConfigs)
        {
            bool shouldBelongToGroup = ShouldEntityBelongToGroup(entity, config);

            if (shouldBelongToGroup)
            {
                newGroups.Add(config.groupName);
            }
        }

        // Update group memberships
        var groupsToJoin = newGroups.Except(currentGroups).ToList();
        var groupsToLeave = currentGroups.Except(newGroups).ToList();

        foreach (string groupName in groupsToJoin)
        {
            AddEntityToGroup(entity, groupName);
        }

        foreach (string groupName in groupsToLeave)
        {
            RemoveEntityFromGroup(entity, groupName);
        }

        _entityGroups[entity] = newGroups;
    }

    private bool ShouldEntityBelongToGroup(IEntity entity, GroupConfig config)
    {
        // Check required tags
        foreach (string requiredTag in config.requiredTags)
        {
            if (!entity.HasTag(requiredTag))
                return false;
        }

        // Check exclusive tags (these exclude from group)
        foreach (string exclusiveTag in config.exclusiveTags)
        {
            if (entity.HasTag(exclusiveTag))
                return false;
        }

        // Check member limit
        if (_groups[config.groupName].Count >= config.maxMembers && 
            !_groups[config.groupName].Contains(entity))
        {
            return false;
        }

        return true;
    }

    private void AddEntityToGroup(IEntity entity, string groupName)
    {
        if (!_groups[groupName].Contains(entity))
        {
            _groups[groupName].Add(entity);
            entity.AddTag($"MemberOf_{groupName}");
            
            Debug.Log($"Entity {entity.Name} joined group: {groupName}");
            GameEvents.OnEntityJoinedGroup?.Invoke(entity, groupName);
            
            CheckGroupSpecialStates(groupName);
        }
    }

    private void RemoveEntityFromGroup(IEntity entity, string groupName)
    {
        if (_groups[groupName].Remove(entity))
        {
            entity.RemoveTag($"MemberOf_{groupName}");
            
            Debug.Log($"Entity {entity.Name} left group: {groupName}");
            GameEvents.OnEntityLeftGroup?.Invoke(entity, groupName);
            
            CheckGroupSpecialStates(groupName);
        }
    }

    private void CheckGroupSpecialStates(string groupName)
    {
        var groupMembers = _groups[groupName];
        
        // Check for special group states
        if (groupMembers.Count == 0)
        {
            GameEvents.OnGroupBecameEmpty?.Invoke(groupName);
        }
        else if (groupMembers.Count == 1)
        {
            GameEvents.OnGroupBecameSingle?.Invoke(groupName, groupMembers[0]);
        }
        
        // Check for group configuration maximums
        var config = _groupConfigs.First(c => c.groupName == groupName);
        if (groupMembers.Count >= config.maxMembers)
        {
            GameEvents.OnGroupReachedMaxCapacity?.Invoke(groupName);
        }
    }

    public IReadOnlyList<IEntity> GetGroupMembers(string groupName)
    {
        return _groups.TryGetValue(groupName, out var members) ? members : new List<IEntity>();
    }

    public string[] GetEntityGroups(IEntity entity)
    {
        return _entityGroups.TryGetValue(entity, out var groups) 
            ? groups.ToArray() 
            : new string[0];
    }

    public bool IsEntityInGroup(IEntity entity, string groupName)
    {
        return _entityGroups.TryGetValue(entity, out var groups) && groups.Contains(groupName);
    }

    public int GetGroupSize(string groupName)
    {
        return _groups.TryGetValue(groupName, out var members) ? members.Count : 0;
    }

    public void ForceUpdateAllGroups()
    {
        EntityRegistry.ForEach(OnEntityTagChanged);
    }

    void OnDestroy()
    {
        EntityRegistry.OnEntityCreated -= _groupTrigger.Track;
        
        // Clean up tracking
        foreach (var entityGroups in _entityGroups.Keys)
        {
            _groupTrigger.Untrack(entityGroups);
        }
    }
}
```

## Integration with Atomic Framework

### Reactive Filter Integration

```csharp
public class TagAwareEntityFilter : MonoBehaviour
{
    private readonly TagEntityTrigger _filterTrigger = new TagEntityTrigger();
    private readonly List<IEntity> _matchedEntities = new();
    
    [SerializeField] private string[] _requiredTags;
    [SerializeField] private string[] _forbiddenTags;

    void Start()
    {
        _filterTrigger.SetAction(ReEvaluateEntity);
        
        // Start tracking all entities
        EntityRegistry.ForEach(_filterTrigger.Track);
        EntityRegistry.OnEntityCreated += _filterTrigger.Track;
    }

    private void ReEvaluateEntity(IEntity entity)
    {
        bool shouldMatch = DoesEntityMatch(entity);
        bool currentlyMatches = _matchedEntities.Contains(entity);

        if (shouldMatch && !currentlyMatches)
        {
            _matchedEntities.Add(entity);
            OnEntityAdded?.Invoke(entity);
        }
        else if (!shouldMatch && currentlyMatches)
        {
            _matchedEntities.Remove(entity);
            OnEntityRemoved?.Invoke(entity);
        }
    }

    private bool DoesEntityMatch(IEntity entity)
    {
        // Check required tags
        foreach (string tag in _requiredTags)
        {
            if (!entity.HasTag(tag))
                return false;
        }

        // Check forbidden tags
        foreach (string tag in _forbiddenTags)
        {
            if (entity.HasTag(tag))
                return false;
        }

        return true;
    }

    public IReadOnlyList<IEntity> MatchedEntities => _matchedEntities;
    public event Action<IEntity> OnEntityAdded;
    public event Action<IEntity> OnEntityRemoved;
}
```

## Implementation Notes

### Event Management
- Automatic subscription to OnTagAdded and OnTagDeleted events
- Clean unsubscription prevents memory leaks
- Type-safe casting in event handlers

### Configuration Flexibility
- Independent control over addition and deletion monitoring
- Default behavior monitors both operations
- Minimal overhead when monitoring only one operation type

### Performance Characteristics
- Event-driven operation avoids polling overhead
- Efficient for entities with frequent tag changes
- Minimal memory overhead per tracked entity

## Best Practices

### Monitoring Configuration
- Use specific monitoring (added/deleted only) when possible
- Consider system requirements for trigger frequency
- Balance between reactivity and performance

### Integration Patterns
- Combine with other trigger types for complex conditions
- Use in conjunction with entity filters for reactive systems
- Implement proper cleanup in disposal patterns

### Event Handler Design
- Keep trigger callbacks lightweight and fast
- Avoid heavy operations in immediate callback execution
- Consider queuing heavy operations for batch processing

## Common Patterns

### Tag Combination Monitor

```csharp
public class TagCombinationMonitor
{
    private readonly TagEntityTrigger _trigger = new TagEntityTrigger();
    private readonly string[] _requiredTags;
    private readonly Action<IEntity> _onCombinationComplete;

    public TagCombinationMonitor(string[] requiredTags, Action<IEntity> onComplete)
    {
        _requiredTags = requiredTags;
        _onCombinationComplete = onComplete;
        
        _trigger.SetAction(CheckTagCombination);
    }

    private void CheckTagCombination(IEntity entity)
    {
        bool hasAllTags = _requiredTags.All(tag => entity.HasTag(tag));
        
        if (hasAllTags)
        {
            _onCombinationComplete(entity);
        }
    }

    public void Track(IEntity entity) => _trigger.Track(entity);
    public void Untrack(IEntity entity) => _trigger.Untrack(entity);
}
```

### State Machine Integration

```csharp
public class TagStateMachine
{
    private readonly Dictionary<string, HashSet<string>> _stateTransitions = new();
    private readonly TagEntityTrigger _stateTrigger = new TagEntityTrigger();

    public TagStateMachine()
    {
        _stateTrigger.SetAction(OnStateTagChanged);
    }

    public void DefineTransition(string fromState, string toState)
    {
        if (!_stateTransitions.ContainsKey(fromState))
            _stateTransitions[fromState] = new HashSet<string>();
        
        _stateTransitions[fromState].Add(toState);
    }

    private void OnStateTagChanged(IEntity entity)
    {
        var currentStates = GetEntityStates(entity);
        ValidateStateTransitions(entity, currentStates);
    }

    private HashSet<string> GetEntityStates(IEntity entity)
    {
        // Implementation to get current state tags
        return new HashSet<string>();
    }

    private void ValidateStateTransitions(IEntity entity, HashSet<string> states)
    {
        // Validate that state transitions are legal
        // Remove invalid states or trigger warnings
    }
}
```

The `TagEntityTrigger` provides efficient, reactive monitoring of entity tag changes, enabling sophisticated tag-based systems and automatic responses to entity state classification updates within the Atomic framework.