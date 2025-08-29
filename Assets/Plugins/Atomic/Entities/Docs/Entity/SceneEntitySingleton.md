# 🔍 SceneEntitySingleton

`SceneEntitySingleton<E>` is an abstract Unity-specific base class that extends `SceneEntity` to provide singleton pattern functionality. It ensures only one instance of a specific entity type exists per scene or globally, with support for cross-scene persistence and flexible resolution strategies.

## Key Features

- **Singleton Pattern** – Guarantees single instance per type
- **Global or Scene-Local** – Configurable singleton scope
- **Cross-Scene Persistence** – Optional persistence across scene loads
- **Multiple Resolution Methods** – Access by global instance or scene-specific lookup
- **Automatic Cleanup** – Self-managing instance references
- **Unity Integration** – Full MonoBehaviour lifecycle support
- **Inspector Configuration** – Configure singleton behavior via Unity Editor

---

## Class Definition

```csharp
public abstract class SceneEntitySingleton<E> : SceneEntity 
    where E : SceneEntitySingleton<E>
{
    // Global singleton instance
    public static E Instance { get; }
    
    // Configuration
    [SerializeField] private bool _isGlobal = true;
    [SerializeField] private bool _dontDestroyOnLoad = false;
    
    // Scene-specific resolution
    public static E Resolve(Component component);
    public static E Resolve(GameObject gameObject);
    public static E Resolve(Scene scene);
}
```

## Configuration Properties

### _isGlobal
- **Default**: `true`
- **Description**: Makes this instance accessible via the global `Instance` property
- **Usage**: Enable for globally accessible singletons, disable for scene-local instances

### _dontDestroyOnLoad
- **Default**: `false`
- **Description**: Prevents destruction when loading new scenes
- **Usage**: Enable for singletons that should persist across scenes

## Access Patterns

### Global Instance Access
```csharp
// Direct access to global singleton
E singleton = E.Instance;
```
- Throws exception if no global instance found
- Returns the first instance marked as global
- Cached for performance after first access

### Scene-Specific Resolution
```csharp
// Resolve singleton from component's scene
E singleton = E.Resolve(someComponent);

// Resolve singleton from GameObject's scene
E singleton = E.Resolve(someGameObject);

// Resolve singleton from specific scene
E singleton = E.Resolve(someScene);
```
- Searches specific scenes for singleton instances
- Caches results per scene for performance
- Throws exception if no instance found in target scene

## Example Usage

### Game Manager Singleton

```csharp
[AddComponentMenu("Game/Game Manager")]
public class GameManager : SceneEntitySingleton<GameManager>
{
    [Header("Game Settings")]
    [SerializeField] private int maxLives = 3;
    [SerializeField] private float gameSpeed = 1.0f;
    
    // Game state
    public int Score { get; private set; }
    public int Lives { get; private set; }
    public bool IsGameRunning { get; private set; }
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Initialize game values
        this.AddValue(EntityNames.SCORE, 0);
        this.AddValue(EntityNames.LIVES, maxLives);
        this.AddValue(EntityNames.GAME_SPEED, gameSpeed);
        
        // Add game management behaviours
        this.AddBehaviour(new GameStateBehaviour());
        this.AddBehaviour(new ScoreManagerBehaviour());
        
        Lives = maxLives;
    }
    
    public void StartGame()
    {
        IsGameRunning = true;
        this.SetValue(EntityNames.GAME_STATE, GameState.Playing);
        
        // Notify other systems
        this.OnStateChanged?.Invoke();
    }
    
    public void AddScore(int points)
    {
        Score += points;
        this.SetValue(EntityNames.SCORE, Score);
    }
    
    public void LoseLife()
    {
        Lives--;
        this.SetValue(EntityNames.LIVES, Lives);
        
        if (Lives <= 0)
        {
            GameOver();
        }
    }
    
    private void GameOver()
    {
        IsGameRunning = false;
        this.SetValue(EntityNames.GAME_STATE, GameState.GameOver);
    }
}

// Usage from any script
public class PlayerController : MonoBehaviour
{
    void Start()
    {
        // Access game manager globally
        var gameManager = GameManager.Instance;
        
        if (!gameManager.IsGameRunning)
        {
            gameManager.StartGame();
        }
    }
    
    void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Coin"))
        {
            // Add score through singleton
            GameManager.Instance.AddScore(100);
            Destroy(other.gameObject);
        }
    }
}
```

### Audio Manager with Persistence

```csharp
[AddComponentMenu("Audio/Audio Manager")]
public class AudioManager : SceneEntitySingleton<AudioManager>
{
    [Header("Audio Settings")]
    [SerializeField] private AudioSource musicSource;
    [SerializeField] private AudioSource sfxSource;
    [SerializeField] private AudioClip[] musicTracks;
    [SerializeField] private AudioClip[] soundEffects;
    
    // Configuration: This should persist across scenes
    [SerializeField] private bool _isGlobal = true;
    [SerializeField] private bool _dontDestroyOnLoad = true;
    
    public float MasterVolume { get; private set; } = 1.0f;
    public float MusicVolume { get; private set; } = 0.8f;
    public float SFXVolume { get; private set; } = 1.0f;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Setup audio values
        this.AddValue(EntityNames.MASTER_VOLUME, MasterVolume);
        this.AddValue(EntityNames.MUSIC_VOLUME, MusicVolume);
        this.AddValue(EntityNames.SFX_VOLUME, SFXVolume);
        
        // Add audio behaviours
        this.AddBehaviour(new MusicManagerBehaviour());
        this.AddBehaviour(new SFXManagerBehaviour());
        
        // Load saved settings
        LoadAudioSettings();
    }
    
    public void PlayMusic(string trackName)
    {
        var track = Array.Find(musicTracks, t => t.name == trackName);
        if (track != null)
        {
            musicSource.clip = track;
            musicSource.Play();
        }
    }
    
    public void PlaySFX(string effectName)
    {
        var effect = Array.Find(soundEffects, e => e.name == effectName);
        if (effect != null)
        {
            sfxSource.PlayOneShot(effect);
        }
    }
    
    public void SetMasterVolume(float volume)
    {
        MasterVolume = Mathf.Clamp01(volume);
        this.SetValue(EntityNames.MASTER_VOLUME, MasterVolume);
        UpdateAudioSources();
    }
    
    private void UpdateAudioSources()
    {
        musicSource.volume = MasterVolume * MusicVolume;
        sfxSource.volume = MasterVolume * SFXVolume;
    }
    
    private void LoadAudioSettings()
    {
        MasterVolume = PlayerPrefs.GetFloat("MasterVolume", 1.0f);
        MusicVolume = PlayerPrefs.GetFloat("MusicVolume", 0.8f);
        SFXVolume = PlayerPrefs.GetFloat("SFXVolume", 1.0f);
        UpdateAudioSources();
    }
}

// Usage from anywhere
public class UIManager : MonoBehaviour
{
    [SerializeField] private Slider masterVolumeSlider;
    
    void Start()
    {
        // Access persistent audio manager
        var audioManager = AudioManager.Instance;
        masterVolumeSlider.value = audioManager.MasterVolume;
        
        masterVolumeSlider.onValueChanged.AddListener(OnMasterVolumeChanged);
    }
    
    private void OnMasterVolumeChanged(float value)
    {
        AudioManager.Instance.SetMasterVolume(value);
    }
    
    public void PlayButtonClickSound()
    {
        AudioManager.Instance.PlaySFX("ButtonClick");
    }
}
```

### Scene-Local UI Manager

```csharp
[AddComponentMenu("UI/Scene UI Manager")]
public class SceneUIManager : SceneEntitySingleton<SceneUIManager>
{
    [Header("UI References")]
    [SerializeField] private Canvas mainCanvas;
    [SerializeField] private GameObject pauseMenu;
    [SerializeField] private GameObject gameOverMenu;
    [SerializeField] private GameObject hudPanel;
    
    // Configuration: Scene-local, not persistent
    [SerializeField] private bool _isGlobal = false;  // Scene-specific instance
    [SerializeField] private bool _dontDestroyOnLoad = false;  // Destroyed with scene
    
    public bool IsPaused { get; private set; }
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Setup UI values
        this.AddValue(EntityNames.MAIN_CANVAS, mainCanvas);
        this.AddValue(EntityNames.IS_PAUSED, false);
        
        // Add UI behaviours
        this.AddBehaviour(new PauseMenuBehaviour());
        this.AddBehaviour(new HUDUpdateBehaviour());
        
        // Initialize UI state
        ShowHUD();
    }
    
    public void ShowPauseMenu()
    {
        IsPaused = true;
        pauseMenu.SetActive(true);
        hudPanel.SetActive(false);
        Time.timeScale = 0f;
        
        this.SetValue(EntityNames.IS_PAUSED, true);
    }
    
    public void HidePauseMenu()
    {
        IsPaused = false;
        pauseMenu.SetActive(false);
        hudPanel.SetActive(true);
        Time.timeScale = 1f;
        
        this.SetValue(EntityNames.IS_PAUSED, false);
    }
    
    public void ShowGameOverMenu()
    {
        gameOverMenu.SetActive(true);
        hudPanel.SetActive(false);
    }
    
    public void ShowHUD()
    {
        hudPanel.SetActive(true);
        pauseMenu.SetActive(false);
        gameOverMenu.SetActive(false);
    }
}

// Usage: Scene-specific access
public class PlayerInput : MonoBehaviour
{
    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Escape))
        {
            // Access scene-specific UI manager
            var uiManager = SceneUIManager.Resolve(this);
            
            if (uiManager.IsPaused)
                uiManager.HidePauseMenu();
            else
                uiManager.ShowPauseMenu();
        }
    }
}
```

### Multi-Scene Singleton System

```csharp
// Base class for multi-scene singletons
public abstract class PersistentSingleton<E> : SceneEntitySingleton<E> 
    where E : PersistentSingleton<E>
{
    protected virtual void Awake()
    {
        // Check for existing instance
        if (Instance != null && Instance != this)
        {
            Debug.LogWarning($"Duplicate {typeof(E).Name} found! Destroying duplicate.");
            Destroy(gameObject);
            return;
        }
        
        base.Awake();
        
        // Make persistent
        _dontDestroyOnLoad = true;
        DontDestroyOnLoad(gameObject);
    }
}

// Player data manager that persists across scenes
public class PlayerDataManager : PersistentSingleton<PlayerDataManager>
{
    [Header("Player Data")]
    public string PlayerName { get; private set; } = "Player";
    public int Level { get; private set; } = 1;
    public int Experience { get; private set; } = 0;
    public int Currency { get; private set; } = 0;
    
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Setup persistent data
        this.AddValue(EntityNames.PLAYER_NAME, PlayerName);
        this.AddValue(EntityNames.PLAYER_LEVEL, Level);
        this.AddValue(EntityNames.EXPERIENCE, Experience);
        this.AddValue(EntityNames.CURRENCY, Currency);
        
        LoadPlayerData();
    }
    
    public void AddExperience(int exp)
    {
        Experience += exp;
        this.SetValue(EntityNames.EXPERIENCE, Experience);
        
        // Check for level up
        CheckLevelUp();
        SavePlayerData();
    }
    
    public void AddCurrency(int amount)
    {
        Currency += amount;
        this.SetValue(EntityNames.CURRENCY, Currency);
        SavePlayerData();
    }
    
    private void CheckLevelUp()
    {
        int expForNext = Level * 100; // Simple formula
        if (Experience >= expForNext)
        {
            Level++;
            Experience -= expForNext;
            this.SetValue(EntityNames.PLAYER_LEVEL, Level);
            
            // Trigger level up event
            this.GetBehaviour<LevelUpBehaviour>()?.OnLevelUp();
        }
    }
    
    private void LoadPlayerData()
    {
        PlayerName = PlayerPrefs.GetString("PlayerName", "Player");
        Level = PlayerPrefs.GetInt("PlayerLevel", 1);
        Experience = PlayerPrefs.GetInt("Experience", 0);
        Currency = PlayerPrefs.GetInt("Currency", 0);
    }
    
    private void SavePlayerData()
    {
        PlayerPrefs.SetString("PlayerName", PlayerName);
        PlayerPrefs.SetInt("PlayerLevel", Level);
        PlayerPrefs.SetInt("Experience", Experience);
        PlayerPrefs.SetInt("Currency", Currency);
        PlayerPrefs.Save();
    }
}
```

### Scene-Aware Singleton Resolution

```csharp
public class MultiSceneExample : MonoBehaviour
{
    void Start()
    {
        DemonstrateResolutionMethods();
    }
    
    private void DemonstrateResolutionMethods()
    {
        // Method 1: Global instance access
        try
        {
            var globalManager = GameManager.Instance;
            Debug.Log($"Global GameManager found: {globalManager.name}");
        }
        catch (Exception e)
        {
            Debug.LogError($"Global GameManager not found: {e.Message}");
        }
        
        // Method 2: Scene-specific resolution via component
        try
        {
            var sceneManager = GameManager.Resolve(this);
            Debug.Log($"Scene GameManager found in {gameObject.scene.name}: {sceneManager.name}");
        }
        catch (Exception e)
        {
            Debug.LogError($"Scene GameManager not found: {e.Message}");
        }
        
        // Method 3: Scene-specific resolution via GameObject
        try
        {
            var sceneManager = GameManager.Resolve(gameObject);
            Debug.Log($"Scene GameManager found via GameObject: {sceneManager.name}");
        }
        catch (Exception e)
        {
            Debug.LogError($"Scene GameManager not found via GameObject: {e.Message}");
        }
        
        // Method 4: Explicit scene resolution
        try
        {
            Scene currentScene = SceneManager.GetActiveScene();
            var sceneManager = GameManager.Resolve(currentScene);
            Debug.Log($"Scene GameManager found in explicit scene: {sceneManager.name}");
        }
        catch (Exception e)
        {
            Debug.LogError($"Scene GameManager not found in explicit scene: {e.Message}");
        }
    }
}
```

### Singleton Factory Pattern

```csharp
public class SingletonFactory : MonoBehaviour
{
    [SerializeField] private GameObject gameManagerPrefab;
    [SerializeField] private GameObject audioManagerPrefab;
    [SerializeField] private GameObject uiManagerPrefab;
    
    void Start()
    {
        EnsureSingletonsExist();
    }
    
    private void EnsureSingletonsExist()
    {
        // Create GameManager if it doesn't exist
        try
        {
            var gameManager = GameManager.Instance;
            Debug.Log("GameManager already exists");
        }
        catch
        {
            Debug.Log("Creating GameManager singleton");
            var gm = Instantiate(gameManagerPrefab);
            gm.name = "GameManager (Singleton)";
        }
        
        // Create AudioManager if it doesn't exist
        try
        {
            var audioManager = AudioManager.Instance;
            Debug.Log("AudioManager already exists");
        }
        catch
        {
            Debug.Log("Creating AudioManager singleton");
            var am = Instantiate(audioManagerPrefab);
            am.name = "AudioManager (Singleton)";
        }
        
        // Create scene-local UIManager
        try
        {
            var uiManager = SceneUIManager.Resolve(this);
            Debug.Log("SceneUIManager already exists in current scene");
        }
        catch
        {
            Debug.Log("Creating SceneUIManager for current scene");
            var ui = Instantiate(uiManagerPrefab);
            ui.name = "UIManager (Scene Singleton)";
        }
    }
}
```

### Development and Debug Tools

```csharp
#if UNITY_EDITOR
[CustomEditor(typeof(SceneEntitySingleton<>), true)]
public class SceneEntitySingletonEditor : Editor
{
    public override void OnInspectorGUI()
    {
        base.OnInspectorGUI();
        
        var singleton = target as SceneEntity;
        if (singleton == null) return;
        
        EditorGUILayout.Space();
        EditorGUILayout.LabelField("Singleton Info", EditorStyles.boldLabel);
        
        // Use reflection to access singleton properties
        var singletonType = singleton.GetType();
        var instanceProperty = singletonType.GetProperty("Instance", 
            System.Reflection.BindingFlags.Public | System.Reflection.BindingFlags.Static);
        
        GUI.enabled = false;
        
        if (instanceProperty != null)
        {
            try
            {
                var instance = instanceProperty.GetValue(null);
                EditorGUILayout.ObjectField("Global Instance", 
                    instance as UnityEngine.Object, singletonType, true);
            }
            catch (Exception e)
            {
                EditorGUILayout.HelpBox($"No global instance: {e.Message}", MessageType.Info);
            }
        }
        
        GUI.enabled = true;
        
        if (Application.isPlaying)
        {
            EditorGUILayout.Space();
            EditorGUILayout.LabelField("Runtime Actions", EditorStyles.boldLabel);
            
            if (GUILayout.Button("Test Global Access"))
            {
                TestGlobalAccess();
            }
            
            if (GUILayout.Button("Test Scene Resolution"))
            {
                TestSceneResolution();
            }
        }
    }
    
    private void TestGlobalAccess()
    {
        var singleton = target as SceneEntity;
        var singletonType = singleton.GetType();
        var instanceProperty = singletonType.GetProperty("Instance");
        
        try
        {
            var instance = instanceProperty.GetValue(null);
            Debug.Log($"Global access successful: {instance}");
        }
        catch (Exception e)
        {
            Debug.LogError($"Global access failed: {e.Message}");
        }
    }
    
    private void TestSceneResolution()
    {
        var singleton = target as SceneEntity;
        var singletonType = singleton.GetType();
        var resolveMethod = singletonType.GetMethod("Resolve", 
            new Type[] { typeof(Component) });
        
        try
        {
            var instance = resolveMethod.Invoke(null, new object[] { singleton });
            Debug.Log($"Scene resolution successful: {instance}");
        }
        catch (Exception e)
        {
            Debug.LogError($"Scene resolution failed: {e.Message}");
        }
    }
}
#endif
```

## Singleton Patterns

### Global Singleton
```csharp
// Single instance accessible globally
[SerializeField] private bool _isGlobal = true;
[SerializeField] private bool _dontDestroyOnLoad = false;
```

### Persistent Singleton
```csharp
// Single instance that persists across scenes
[SerializeField] private bool _isGlobal = true;
[SerializeField] private bool _dontDestroyOnLoad = true;
```

### Scene-Local Singleton
```csharp
// Single instance per scene
[SerializeField] private bool _isGlobal = false;
[SerializeField] private bool _dontDestroyOnLoad = false;
```

## Best Practices

1. **Choose Appropriate Scope** – Use global for game-wide systems, scene-local for level-specific managers
2. **Handle Missing Instances** – Always wrap singleton access in try-catch blocks
3. **Avoid Persistence Overuse** – Only persist truly cross-scene singletons
4. **Use Scene Resolution** – Access scene-specific instances when working with multiple scenes
5. **Initialize Early** – Configure singletons with appropriate execution order
6. **Clean Dependencies** – Ensure other systems handle singleton destruction gracefully

## Unity-Specific Considerations

### Scene Management
- Global instances survive scene changes if persistent
- Scene-local instances are destroyed with their scenes
- Resolution methods search specific scenes efficiently

### Performance
- Instance property is cached after first access
- Scene resolution uses pooled collections for efficiency
- Static dictionaries maintain per-scene caches

### Thread Safety
- Not thread-safe by design (Unity main thread only)
- Access only from Unity's main thread

## Common Issues and Solutions

### Duplicate Instances
- Check _isGlobal settings across prefabs
- Implement duplicate detection in Awake()
- Use consistent prefab configurations

### Missing Singleton
- Verify singleton exists in target scene
- Check _isGlobal flag configuration
- Ensure singleton survives scene transitions if needed

### Initialization Order
- Use DefaultExecutionOrder attribute for critical singletons
- Implement lazy initialization patterns
- Handle dependencies in OnInstall() rather than Awake()

## Integration with Atomic Systems

### With Entity Behaviours
```csharp
public class SingletonBehaviour : IEntityBehaviour
{
    public void OnInstall(IEntity entity)
    {
        // Access singleton from behaviour
        var gameManager = GameManager.Instance;
        gameManager.RegisterEntity(entity);
    }
}
```

### With Entity Values
```csharp
public class GameManager : SceneEntitySingleton<GameManager>
{
    protected override void OnInstall()
    {
        base.OnInstall();
        
        // Share singleton reference as entity value
        this.AddValue(EntityNames.GAME_MANAGER, this);
        this.AddValue(EntityNames.SINGLETON_INSTANCE, Instance);
    }
}
```

## Notes

- SceneEntitySingleton requires UNITY_5_3_OR_NEWER
- Supports Odin Inspector attributes for enhanced editor experience
- Generic type constraint ensures proper singleton typing
- Uses Unity's object pool for efficient scene searches
- Exception-based error handling for missing instances
- Automatic cleanup of static references on destruction