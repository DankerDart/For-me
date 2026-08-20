// =====================================================================
//  HorrorMainMenu.cs
//  Главное меню хоррора — один компонент, всё внутри:
//    • 3D-сцена за UI (или 2D-фон)
//    • авто-сборка Canvas/Title/Buttons/Vignette/Grain/ExitConfirm
//    • плавный вход, дрожь заголовка, моргание ламп, фейд
//    • список пунктов меню настраивается в инспекторе
//  Вешать на пустой GameObject в сцене меню. Бросил — нажал Play.
//  Зависимости: только стандартный UnityEngine.UI.
// =====================================================================

using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.SceneManagement;
using UnityEngine.UI;

[DisallowMultipleComponent]
[AddComponentMenu("Horror/Horror Main Menu")]
public class HorrorMainMenu : MonoBehaviour
{
    // ===================== ТИПЫ =====================
    public enum MenuAction { StartGame, Continue, OpenSettings, CloseSettings, Exit }
    public enum CanvasMode { ScreenSpaceOverlay, ScreenSpaceCamera }

    [Serializable]
    public class MenuItem
    {
        public string label = "New Game";
        public MenuAction action = MenuAction.StartGame;
        public bool enabled = true;
    }

    // ===================== ИНСПЕКТОР =====================

    [Header("Canvas")]
    public CanvasMode canvasMode = CanvasMode.ScreenSpaceOverlay;
    [Tooltip("Камера, через которую виден 3D-фон. Только для ScreenSpaceCamera")]
    public Camera uiCamera;
    public int canvasSortingOrder = 100;

    [Header("3D-сцена (опционально)")]
    public Camera mainCamera;
    [Tooltip("Лампы, которые будут моргать. Если пусто — найдём все Light в сцене автоматически")]
    public Light[] flickeringLights;
    public bool autoFindLights = true;
    public bool configureMainCamera = true;
    public Color cameraBackground = new Color(0.01f, 0.01f, 0.02f, 1f);
    public bool enableFog = true;
    public Color fogColor = new Color(0.02f, 0.02f, 0.03f, 1f);
    public float fogDensity = 0.035f;
    [Tooltip("Опционально: статичная текстура-фон (если 3D-сцены пока нет)")]
    public Texture2D backgroundTexture;
    [Range(0f, 1f)] public float backgroundTextureDim = 0.4f;

    [Header("Контент")]
    public string titleText = "AFTER HOURS";
    public Font titleFont;
    public int titleFontSize = 96;
    public Color titleColor = new Color(0.85f, 0.84f, 0.78f, 1f);
    public Vector2 titleOffset = new Vector2(0, 220);
    public bool showContinue = true;
    public bool showSettings = true;

    [Header("Пункты меню (в порядке сверху вниз)")]
    public List<MenuItem> menuItems = new()
    {
        new MenuItem { label = "New Game", action = MenuAction.StartGame, enabled = true },
        new MenuItem { label = "Continue", action = MenuAction.Continue,  enabled = true },
        new MenuItem { label = "Settings", action = MenuAction.OpenSettings, enabled = true },
        new MenuItem { label = "Exit",     action = MenuAction.Exit,      enabled = true },
    };

    [Header("Анимация входа")]
    [Range(0.1f, 5f)] public float entranceDuration = 1.4f;
    [Range(0f, 1f)]   public float buttonStagger    = 0.18f;
    [Range(0f, 400f)] public float buttonSlide      = 90f;

    [Header("Атмосфера — виньетка")]
    public bool enableVignette = true;
    [Range(0f, 1f)]   public float vignetteIntensity     = 0.65f;
    [Range(0.5f, 20f)] public float vignettePulseInterval = 5.5f;
    [Range(0f, 0.5f)] public float vignettePulseAmount   = 0.18f;

    [Header("Атмосфера — зернистость")]
    public bool enableGrain = true;
    [Range(0f, 1f)]   public float grainIntensity   = 0.18f;
    [Range(0f, 2f)]   public float grainScrollSpeed = 0.15f;
    [Range(32, 512)]  public int   grainTextureSize = 192;

    [Header("Атмосфера — заголовок")]
    public bool enableTitleShake = true;
    [Range(0f, 10f)] public float titleShakeIntensity = 1.8f;
    [Range(1f, 60f)] public float titleShakeSpeed     = 22f;
    [Range(0f, 1f)]  public float titleShakePulse     = 0.35f;
    public bool enableGlitch = true;
    [Range(0.5f, 20f)] public float glitchInterval = 5.5f;
    [Range(0.02f, 0.5f)] public float glitchDuration = 0.09f;
    [Range(0f, 30f)] public float glitchOffset = 10f;
    public bool enableTitleFlicker = true;
    [Range(0.5f, 20f)] public float titleFlickerInterval = 4.2f;
    [Range(0f, 1f)] public float titleFlickerDim = 0.35f;
    [Range(0f, 1f)] public float titleFlickerBlackout = 0.18f;
    public Color titleFlickerTint = new Color(0.95f, 0.85f, 0.55f, 1f);

    [Header("Атмосфера — затемнение 3D-фона")]
    public bool enableBackgroundDim = true;
    [Range(0f, 1f)] public float backgroundDim = 0.45f;

    [Header("Атмосфера — 3D-лампы")]
    public bool enableLightFlicker = true;
    [Range(0.5f, 20f)] public float lightFlickerInterval = 4.5f;
    [Range(0f, 1f)] public float lightBlackoutChance = 0.2f;
    [Range(0f, 1f)] public float lightMinIntensity = 0.05f;
    public Color lightFlickerTint = new Color(0.92f, 0.95f, 1f, 1f);

    [Header("Аудио")]
    public AudioSource audioSource;
    public AudioClip hoverSound;
    public AudioClip clickSound;
    public AudioClip startSound;
    public AudioClip exitSound;
    [Range(0f, 1f)] public float sfxVolume = 0.7f;

    [Header("Сцена и поведение")]
    public string gameSceneName = "GameScene";
    public bool showExitConfirm = true;
    public string exitConfirmTitle = "LEAVE THE HALLWAY?";
    public string exitConfirmYes = "YES";
    public string exitConfirmNo = "NO";
    [Range(0.1f, 3f)] public float fadeOutDuration = 0.7f;
    public bool autoBuildUI = true;
    public bool startAtmosphereAfterEntrance = true;
    [Tooltip("Отладка: пропустить анимацию входа и сразу показать меню")]
    public bool skipEntrance = true;
    public KeyCode exitShortcut = KeyCode.Escape;

    // ===================== РАНТАЙМ =====================

    Canvas _canvas;
    CanvasGroup _mainGroup;
    RawImage _grain;
    Image _vignette;
    Image _dimOverlay;
    Text _title;
    RectTransform _titleRect;
    RectTransform _buttonContainer;
    readonly List<Button> _buttons = new();
    GameObject _settingsPanel;
    GameObject _confirmExitPanel;
    Texture2D _noiseTex;
    bool _ownsUI;

    // кэш для FX
    Color _vignetteBase = Color.white;
    Color _titleBase = Color.white;
    Vector2 _titleBasePos;
    float _shakeT, _pulsePhase, _glitchTimer, _glitchRemaining;
    float _grainU, _grainV;
    float _vignetteTimer;
    float _titleFlickerTimer;
    bool _titleFlickerActive;

    // лампы
    class FlickerLight
    {
        public Light light;
        public float baseIntensity;
        public Color baseColor;
        public float nextFlickerAt;
        public Coroutine routine;
    }
    readonly List<FlickerLight> _lights = new();

    bool _ready;
    bool _atmosphere;

    // ===================== ЖИЗНЕННЫЙ ЦИКЛ =====================

    IEnumerator Start()
    {
        yield return null; // даём сцене прогрузиться (особенно если 3D-фон)
        ConfigureScene();
        if (autoBuildUI) BuildUI();
        Canvas.ForceUpdateCanvases(); // форсим autosize канваса ДО анимации входа
        Debug.Log("[HorrorMainMenu] Меню собрано. Canvas=" + (_canvas != null ? _canvas.name : "null")
                  + " buttons=" + _buttons.Count + " grain=" + (_grain != null));
        _atmosphere = !startAtmosphereAfterEntrance; // true = сразу, false = после входа
        StartCoroutine(EntranceSequence());
    }

    void OnDestroy()
    {
        foreach (var fl in _lights)
            if (fl.routine != null) StopCoroutine(fl.routine);
        if (_noiseTex != null) Destroy(_noiseTex);
        if (_ownsUI && _canvas != null) Destroy(_canvas.gameObject);
    }

    [ContextMenu("Show Immediately (debug)")]
    public void DebugShowImmediately()
    {
        if (_mainGroup != null) { _mainGroup.alpha = 1f; _mainGroup.blocksRaycasts = true; }
        if (_title != null)
        {
            var tcg = _title.GetComponent<CanvasGroup>();
            if (tcg != null) tcg.alpha = 1f;
        }
        foreach (var b in _buttons)
        {
            if (b == null) continue;
            var go = b.gameObject; if (go == null) continue;
            var cg = go.GetComponent<CanvasGroup>();
            if (cg != null) cg.alpha = 1f;
            var rt = go.GetComponent<RectTransform>();
            if (rt != null) rt.anchoredPosition += new Vector2(0, buttonSlide);
        }
        _ready = true;
        _atmosphere = true;
        Debug.Log("[HorrorMainMenu] Debug: меню показано принудительно.");
    }

    [ContextMenu("Rebuild UI")]
    public void DebugRebuildUI()
    {
        if (_ownsUI && _canvas != null) Destroy(_canvas.gameObject);
        _canvas = null; _mainGroup = null; _grain = null; _vignette = null;
        _dimOverlay = null; _title = null; _titleRect = null; _buttonContainer = null;
        _buttons.Clear(); _settingsPanel = null; _confirmExitPanel = null;
        _noiseTex = null; _ownsUI = false;
        ConfigureScene();
        BuildUI();
        Canvas.ForceUpdateCanvases();
        DebugShowImmediately();
    }

    void Update()
    {
        if (!_ready) return;

        if (_atmosphere)
        {
            if (enableTitleShake) UpdateTitleShake();
            if (enableGrain) UpdateGrain();
            if (enableVignette) UpdateVignette();
            if (enableTitleFlicker) UpdateTitleFlicker();
        }

        if (Input.GetKeyDown(exitShortcut))
        {
            if (_confirmExitPanel != null && _confirmExitPanel.activeSelf) OnExitCancel();
            else if (_settingsPanel != null && _settingsPanel.activeSelf) OnCloseSettings();
            else if (showExitConfirm) OnExit();
        }
    }

    // ===================== СЦЕНА =====================

    void ConfigureScene()
    {
        if (mainCamera == null) mainCamera = Camera.main;
        if (mainCamera == null && uiCamera != null) mainCamera = uiCamera;
        if (uiCamera == null) uiCamera = mainCamera;

        if (configureMainCamera && mainCamera != null)
        {
            mainCamera.clearFlags = CameraClearFlags.SolidColor;
            mainCamera.backgroundColor = cameraBackground;
        }

        if (enableFog)
        {
            RenderSettings.fog = true;
            RenderSettings.fogColor = fogColor;
            RenderSettings.fogMode = FogMode.ExponentialSquared;
            RenderSettings.fogDensity = fogDensity;
        }

        if (enableLightFlicker) CollectLights();
    }

    void CollectLights()
    {
        _lights.Clear();
        Light[] sources = flickeringLights;
        if ((sources == null || sources.Length == 0) && autoFindLights)
            sources = FindObjectsOfType<Light>();

        if (sources == null) return;
        foreach (var l in sources)
        {
            if (l == null) continue;
            _lights.Add(new FlickerLight
            {
                light = l,
                baseIntensity = l.intensity,
                baseColor = l.color,
                nextFlickerAt = Time.unscaledTime + UnityEngine.Random.Range(0.5f, lightFlickerInterval)
            });
        }

        foreach (var fl in _lights)
            fl.routine = StartCoroutine(LightFlickerRoutine(fl));
    }

    IEnumerator LightFlickerRoutine(FlickerLight fl)
    {
        var wait = new WaitForSeconds(0.02f);
        while (fl.light != null)
        {
            float waitTime = fl.nextFlickerAt - Time.unscaledTime;
            if (waitTime > 0f) yield return new WaitForSeconds(waitTime);

            bool blackout = UnityEngine.Random.value < lightBlackoutChance;
            int flicks = blackout ? UnityEngine.Random.Range(6, 14) : UnityEngine.Random.Range(2, 5);
            float total = blackout ? UnityEngine.Random.Range(0.4f, 0.9f) : UnityEngine.Random.Range(0.12f, 0.25f);
            float step = total / flicks;

            for (int i = 0; i < flicks; i++)
            {
                bool deepDim = UnityEngine.Random.value < 0.3f;
                float k = deepDim ? 0f : UnityEngine.Random.Range(lightMinIntensity, 1f);
                fl.light.intensity = fl.baseIntensity * k;
                fl.light.color = lightFlickerTint;
                yield return new WaitForSeconds(step * UnityEngine.Random.Range(0.5f, 1.5f));
            }
            fl.light.intensity = fl.baseIntensity;
            fl.light.color = fl.baseColor;

            fl.nextFlickerAt = Time.unscaledTime + UnityEngine.Random.Range(
                lightFlickerInterval * 0.4f, lightFlickerInterval * 1.6f);
        }
    }

    // ===================== UI — авто-сборка =====================

    void BuildUI()
    {
        try
        {
            BuildUIInternal();
        }
        catch (System.Exception e)
        {
            Debug.LogError("[HorrorMainMenu] BuildUI упал с ошибкой: " + e.GetType().Name
                + " — " + e.Message + "\n" + e.StackTrace);
        }
    }

    void BuildUIInternal()
    {
        // EventSystem
        if (FindObjectOfType<EventSystem>() == null)
        {
            var es = new GameObject("EventSystem", typeof(EventSystem), typeof(StandaloneInputModule));
            es.transform.SetParent(transform, false);
        }

        // Canvas
        var canvasGO = new GameObject("HorrorMenuCanvas",
            typeof(Canvas), typeof(CanvasScaler), typeof(GraphicRaycaster));
        canvasGO.transform.SetParent(transform, false);
        canvasGO.layer = LayerMask.NameToLayer("UI");

        _canvas = canvasGO.GetComponent<Canvas>();
        bool wantCamera = canvasMode == CanvasMode.ScreenSpaceCamera;
        if (wantCamera && uiCamera == null)
        {
            Debug.LogWarning("[HorrorMainMenu] CanvasMode=ScreenSpaceCamera, но UI Camera не назначена. Перехожу на ScreenSpaceOverlay.");
            wantCamera = false;
        }
        _canvas.renderMode = wantCamera ? RenderMode.ScreenSpaceCamera : RenderMode.ScreenSpaceOverlay;
        if (wantCamera) _canvas.worldCamera = uiCamera;
        _canvas.sortingOrder = canvasSortingOrder;

        var scaler = canvasGO.GetComponent<CanvasScaler>();
        scaler.uiScaleMode = CanvasScaler.ScaleMode.ScaleWithScreenSize;
        scaler.referenceResolution = new Vector2(1920, 1080);
        scaler.matchWidthOrHeight = 0.5f;

        // ДУБОВЫЙ ФИКС: принудительно задаём размер канвасу, иначе в момент
        // создания дочерних элементов его RectTransform = 0×0 и они схлопываются.
        var canvasRT = canvasGO.GetComponent<RectTransform>();
        if (_canvas.renderMode == RenderMode.ScreenSpaceOverlay)
        {
            canvasRT.sizeDelta = new Vector2(Screen.width, Screen.height);
        }
        else
        {
            canvasRT.sizeDelta = new Vector2(1920, 1080);
        }
        Debug.Log("[HorrorMainMenu] Canvas создан: sizeDelta=" + canvasRT.sizeDelta
            + " parent=" + (canvasRT.parent != null ? canvasRT.parent.name : "<root>")
            + " screen=" + Screen.width + "x" + Screen.height);

        _mainGroup = canvasGO.AddComponent<CanvasGroup>();
        _mainGroup.alpha = 0f;
        _mainGroup.blocksRaycasts = false;

        // опциональная фоновая текстура (если 3D-сцены пока нет)
        if (backgroundTexture != null)
        {
            var bgGO = new GameObject("BackgroundImage",
                typeof(RectTransform), typeof(CanvasRenderer), typeof(Image));
            bgGO.transform.SetParent(canvasGO.transform, false);
            StretchFull(bgGO.GetComponent<RectTransform>());
            var bgImg = bgGO.GetComponent<Image>();
            bgImg.sprite = Sprite.Create(backgroundTexture,
                new Rect(0, 0, backgroundTexture.width, backgroundTexture.height),
                new Vector2(0.5f, 0.5f));
            bgImg.color = new Color(1, 1, 1, 1f - backgroundTextureDim);
            bgImg.raycastTarget = false;
        }

        // затемняющий оверлей над 3D-сценой
        if (enableBackgroundDim)
        {
            _dimOverlay = CreateFullScreenImage(canvasGO.transform, "DimOverlay");
            _dimOverlay.color = new Color(0f, 0f, 0f, backgroundDim);
            _dimOverlay.raycastTarget = false;
        }

        // виньетка
        if (enableVignette)
        {
            _vignette = CreateFullScreenImage(canvasGO.transform, "Vignette");
            _vignette.color = new Color(0f, 0f, 0f, vignetteIntensity);
            _vignette.raycastTarget = false;
            _vignetteBase = _vignette.color;
        }

        // зерно
        if (enableGrain)
        {
            var grainGO = new GameObject("FilmGrain", typeof(RectTransform), typeof(CanvasRenderer), typeof(RawImage));
            grainGO.transform.SetParent(canvasGO.transform, false);
            StretchFull(grainGO.GetComponent<RectTransform>());
            _grain = grainGO.GetComponent<RawImage>();
            _noiseTex = GenerateNoiseTexture(grainTextureSize);
            _grain.texture = _noiseTex;
            _grain.color = new Color(1, 1, 1, grainIntensity);
            _grain.raycastTarget = false;
        }

        // заголовок
        var titleGO = new GameObject("Title", typeof(RectTransform), typeof(CanvasRenderer), typeof(Text));
        titleGO.transform.SetParent(canvasGO.transform, false);
        _titleRect = titleGO.GetComponent<RectTransform>();
        _titleRect.anchorMin = new Vector2(0.5f, 1f);
        _titleRect.anchorMax = new Vector2(0.5f, 1f);
        _titleRect.pivot = new Vector2(0.5f, 1f);
        _titleRect.anchoredPosition = new Vector2(titleOffset.x, -titleOffset.y);
        _title = titleGO.GetComponent<Text>();
        _title.text = titleText;
        _title.alignment = TextAnchor.MiddleCenter;
        _title.fontSize = titleFontSize;
        _title.color = titleColor;
        _title.raycastTarget = false;
        _title.supportRichText = false;
        _title.horizontalOverflow = HorizontalWrapMode.Overflow;
        if (titleFont != null) _title.font = titleFont; else _title.font = GetDefaultFont();
        // лёгкая обводка для читаемости поверх фона
        var outline = titleGO.AddComponent<Outline>();
        outline.effectColor = new Color(0, 0, 0, 0.8f);
        outline.effectDistance = new Vector2(2, -2);
        var titleCG = titleGO.AddComponent<CanvasGroup>();
        titleCG.alpha = 0f;
        titleCG.interactable = false;
        titleCG.blocksRaycasts = false;
        _titleBase = titleColor;
        _titleBasePos = _titleRect.anchoredPosition;

        // контейнер кнопок
        var contGO = new GameObject("Buttons", typeof(RectTransform), typeof(VerticalLayoutGroup));
        contGO.transform.SetParent(canvasGO.transform, false);
        _buttonContainer = contGO.GetComponent<RectTransform>();
        _buttonContainer.anchorMin = _buttonContainer.anchorMax = new Vector2(0.5f, 0.5f);
        _buttonContainer.pivot = new Vector2(0.5f, 0.5f);
        _buttonContainer.anchoredPosition = new Vector2(0, -40);
        _buttonContainer.sizeDelta = new Vector2(420, 50);
        var vlg = contGO.GetComponent<VerticalLayoutGroup>();
        vlg.childAlignment = TextAnchor.MiddleCenter;
        vlg.spacing = 18;
        vlg.childForceExpandHeight = false;
        vlg.childForceExpandWidth = true;
        vlg.childControlHeight = true;
        vlg.childControlWidth = true;

        // кнопки
        BuildButtons();

        Debug.Log("[HorrorMainMenu] Title pos=" + (_titleRect != null ? _titleRect.anchoredPosition.ToString() : "null")
            + " sizeDelta=" + (_titleRect != null ? _titleRect.sizeDelta.ToString() : "null")
            + " alpha=" + (_title != null ? _title.color.a.ToString("F2") : "null"));
        Debug.Log("[HorrorMainMenu] Buttons=" + _buttons.Count
            + " firstButtonPos=" + (_buttons.Count > 0 && _buttons[0] != null
                ? _buttons[0].GetComponent<RectTransform>().anchoredPosition.ToString() : "null")
            + " containerSize=" + (_buttonContainer != null ? _buttonContainer.sizeDelta.ToString() : "null"));

        // панель настроек (простая заглушка)
        _settingsPanel = CreateSubPanel(canvasGO.transform, "SettingsPanel", "Settings");
        _settingsPanel.SetActive(false);

        // панель подтверждения выхода
        _confirmExitPanel = CreateSubPanel(canvasGO.transform, "ConfirmExitPanel", exitConfirmTitle, exitConfirmYes, exitConfirmNo);
        _confirmExitPanel.SetActive(false);

        _ownsUI = true;
    }

    void BuildButtons()
    {
        _buttons.Clear();
        foreach (var item in menuItems)
        {
            if (!item.enabled) continue;
            var btn = CreateButton(_buttonContainer, item.label);
            var captured = item; // для замыкания
            btn.onClick.AddListener(() => HandleAction(captured.action));
            if (hoverSound != null) AttachHoverSound(btn);
            _buttons.Add(btn);
        }

        // Прячем кнопки (для анимации входа)
        foreach (var b in _buttons)
        {
            if (b == null) continue;
            var go = b.gameObject;
            if (go == null) continue;
            var cg = go.GetComponent<CanvasGroup>();
            if (cg == null) cg = go.AddComponent<CanvasGroup>();
            cg.alpha = 0f;
            cg.interactable = false;
            var rt = go.GetComponent<RectTransform>();
            if (rt != null) rt.anchoredPosition -= new Vector2(0, buttonSlide);
        }
    }

    void HandleAction(MenuAction action)
    {
        PlaySFX(clickSound);
        switch (action)
        {
            case MenuAction.StartGame:      OnStartGame(); break;
            case MenuAction.Continue:       OnContinue();  break;
            case MenuAction.OpenSettings:   OnOpenSettings(); break;
            case MenuAction.CloseSettings:  OnCloseSettings(); break;
            case MenuAction.Exit:           OnExit();      break;
        }
    }

    // ===================== UI — фабрика =====================

    Image CreateFullScreenImage(Transform parent, string name)
    {
        var go = new GameObject(name, typeof(RectTransform), typeof(CanvasRenderer), typeof(Image));
        go.transform.SetParent(parent, false);
        StretchFull(go.GetComponent<RectTransform>());
        return go.GetComponent<Image>();
    }

    void StretchFull(RectTransform rt)
    {
        if (rt == null) return;
        rt.anchorMin = Vector2.zero;
        rt.anchorMax = Vector2.one;
        rt.offsetMin = Vector2.zero;
        rt.offsetMax = Vector2.zero;
        rt.sizeDelta = Vector2.zero;            // ключ: заполняем родителя даже если он (0,0)
        rt.anchoredPosition = Vector2.zero;
    }

    Button CreateButton(Transform parent, string label)
    {
        var go = new GameObject(label, typeof(RectTransform), typeof(CanvasRenderer), typeof(Image), typeof(Button));
        go.transform.SetParent(parent, false);
        var img = go.GetComponent<Image>();
        img.color = new Color(1f, 1f, 1f, 0.06f);

        var rt = go.GetComponent<RectTransform>();
        rt.sizeDelta = new Vector2(420, 64);

        var btn = go.GetComponent<Button>();
        var colors = btn.colors;
        colors.normalColor = new Color(1, 1, 1, 0.08f);
        colors.highlightedColor = new Color(1, 1, 1, 0.18f);
        colors.pressedColor = new Color(0.85f, 0.85f, 0.85f, 0.30f);
        colors.selectedColor = colors.highlightedColor;
        colors.fadeDuration = 0.08f;
        btn.colors = colors;
        var outline = go.AddComponent<Outline>();
        outline.effectColor = new Color(1, 1, 1, 0.15f);
        outline.effectDistance = new Vector2(1, -1);

        var textGO = new GameObject("Text", typeof(RectTransform), typeof(CanvasRenderer), typeof(Text));
        textGO.transform.SetParent(go.transform, false);
        var trt = textGO.GetComponent<RectTransform>();
        StretchFull(trt);
        var txt = textGO.GetComponent<Text>();
        txt.text = label;
        txt.alignment = TextAnchor.MiddleCenter;
        txt.fontSize = 36;
        txt.color = new Color(0.92f, 0.90f, 0.85f);
        txt.font = titleFont != null ? titleFont : GetDefaultFont();
        txt.raycastTarget = false;

        return btn;
    }

    GameObject CreateSubPanel(Transform parent, string name, string title, string yesLabel = null, string noLabel = null)
    {
        var go = new GameObject(name, typeof(RectTransform), typeof(CanvasRenderer), typeof(Image), typeof(CanvasGroup));
        go.transform.SetParent(parent, false);
        StretchFull(go.GetComponent<RectTransform>());
        var img = go.GetComponent<Image>();
        img.color = new Color(0f, 0f, 0f, 0.85f);
        img.raycastTarget = true;
        var cg = go.GetComponent<CanvasGroup>();
        cg.interactable = true;
        cg.blocksRaycasts = true;

        // центральный бокс
        var box = new GameObject("Box", typeof(RectTransform), typeof(CanvasRenderer), typeof(Image), typeof(VerticalLayoutGroup));
        box.transform.SetParent(go.transform, false);
        var brt = box.GetComponent<RectTransform>();
        brt.anchorMin = brt.anchorMax = new Vector2(0.5f, 0.5f);
        brt.pivot = new Vector2(0.5f, 0.5f);
        brt.sizeDelta = new Vector2(560, 240);
        var bimg = box.GetComponent<Image>();
        bimg.color = new Color(0.05f, 0.05f, 0.07f, 0.95f);
        var boutline = box.AddComponent<Outline>();
        boutline.effectColor = new Color(1, 1, 1, 0.1f);
        boutline.effectDistance = new Vector2(1, -1);
        var vlg = box.GetComponent<VerticalLayoutGroup>();
        vlg.padding = new RectOffset(40, 40, 40, 40);
        vlg.spacing = 24;
        vlg.childAlignment = TextAnchor.MiddleCenter;
        vlg.childForceExpandHeight = false;
        vlg.childForceExpandWidth = true;
        vlg.childControlHeight = true;
        vlg.childControlWidth = true;

        // заголовок панели
        var titleGO = new GameObject("Title", typeof(RectTransform), typeof(CanvasRenderer), typeof(Text));
        titleGO.transform.SetParent(box.transform, false);
        var ttxt = titleGO.GetComponent<Text>();
        ttxt.text = title;
        ttxt.alignment = TextAnchor.MiddleCenter;
        ttxt.fontSize = 38;
        ttxt.color = new Color(0.92f, 0.90f, 0.82f);
        ttxt.font = titleFont != null ? titleFont : GetDefaultFont();
        var le = titleGO.GetComponent<RectTransform>();
        le.sizeDelta = new Vector2(0, 60);

        // кнопки
        if (yesLabel != null)
        {
            var row = new GameObject("Row", typeof(RectTransform), typeof(HorizontalLayoutGroup));
            row.transform.SetParent(box.transform, false);
            var hlg = row.GetComponent<HorizontalLayoutGroup>();
            hlg.spacing = 24;
            hlg.childAlignment = TextAnchor.MiddleCenter;
            hlg.childForceExpandHeight = false;
            hlg.childForceExpandWidth = true;
            hlg.childControlHeight = true;
            hlg.childControlWidth = true;
            var rrt = row.GetComponent<RectTransform>();
            rrt.sizeDelta = new Vector2(0, 64);

            var yesBtn = CreateButton(row.transform, yesLabel);
            yesBtn.GetComponent<Image>().color = new Color(0.6f, 0.1f, 0.1f, 0.25f);
            yesBtn.onClick.AddListener(OnExitConfirm);
            var noBtn  = CreateButton(row.transform, noLabel);
            noBtn.onClick.AddListener(OnExitCancel);
        }
        else
        {
            // для настроек — одна кнопка Close
            var close = CreateButton(box.transform, "Close");
            close.onClick.AddListener(OnCloseSettings);
        }

        return go;
    }

    void AttachHoverSound(Button btn)
    {
        if (btn == null) return;
        var go = btn.gameObject;
        if (go == null) return;
        var trig = go.GetComponent<EventTrigger>();
        if (trig == null) trig = go.AddComponent<EventTrigger>();
        var entry = new EventTrigger.Entry { eventID = EventTriggerType.PointerEnter };
        entry.callback.AddListener(_ => PlaySFX(hoverSound));
        trig.triggers.Add(entry);
    }

    Font GetDefaultFont()
    {
        // LegacyRuntime.ttf — Unity 2018+. Arial.ttf — более старые. Пробуем оба.
        Font f = Resources.GetBuiltinResource<Font>("LegacyRuntime.ttf");
        if (f == null) f = Resources.GetBuiltinResource<Font>("Arial.ttf");
        if (f == null) Debug.LogWarning("[HorrorMainMenu] Не найден встроенный шрифт (LegacyRuntime/Arial). Текст может не отображаться. Назначь Title Font вручную в инспекторе.");
        return f;
    }

    Texture2D GenerateNoiseTexture(int size)
    {
        var tex = new Texture2D(size, size, TextureFormat.RGBA32, false)
        {
            wrapMode = TextureWrapMode.Repeat,
            filterMode = FilterMode.Point,
            hideFlags = HideFlags.DontSave
        };
        var px = new Color[size * size];
        for (int i = 0; i < px.Length; i++)
        {
            float v = UnityEngine.Random.value;
            px[i] = new Color(v, v, v, 1f);
        }
        tex.SetPixels(px);
        tex.Apply(false, false);
        return tex;
    }

    // ===================== АНИМАЦИИ =====================

    IEnumerator EntranceSequence()
    {
        // Режим отладки: всё сразу видимо, без анимаций
        if (skipEntrance)
        {
            if (_mainGroup != null) { _mainGroup.alpha = 1f; _mainGroup.blocksRaycasts = true; }
            if (_title != null)
            {
                var tcg = _title.GetComponent<CanvasGroup>();
                if (tcg != null) tcg.alpha = 1f;
            }
            foreach (var b in _buttons)
            {
                if (b == null) continue;
                var go = b.gameObject; if (go == null) continue;
                var cg = go.GetComponent<CanvasGroup>();
                if (cg != null) cg.alpha = 1f;
                var rt = go.GetComponent<RectTransform>();
                if (rt != null) rt.anchoredPosition += new Vector2(0, buttonSlide);
            }
            _ready = true;
            _atmosphere = true;
            Debug.Log("[HorrorMainMenu] skipEntrance=true — меню показано мгновенно");
            yield break;
        }

        Debug.Log("[HorrorMainMenu] Entrance: фаза 1 — канвас проявляется");
        // 1. фон-канвас проявляется
        yield return FadeGroup(_mainGroup, 0f, 1f, entranceDuration);
        _mainGroup.blocksRaycasts = true;
        Debug.Log("[HorrorMainMenu] Entrance: фаза 2 — заголовок");

        // 2. заголовок
        if (_title != null)
        {
            var tcg = _title.GetComponent<CanvasGroup>();
            yield return FadeGroup(tcg, 0f, 1f, entranceDuration * 0.6f);
        }
        Debug.Log("[HorrorMainMenu] Entrance: фаза 3 — кнопки (" + _buttons.Count + " шт)");

        // 3. кнопки по очереди
        for (int i = 0; i < _buttons.Count; i++)
        {
            if (_buttons[i] == null) continue;
            StartCoroutine(AnimateButtonIn(_buttons[i], i * buttonStagger));
        }
        yield return new WaitForSeconds(_buttons.Count * buttonStagger + entranceDuration * 0.6f + 0.1f);

        _ready = true;
        _atmosphere = true; // атмосфера включается после входа
        Debug.Log("[HorrorMainMenu] Entrance завершён. Меню готово к работе.");
    }

    IEnumerator AnimateButtonIn(Button btn, float delay)
    {
        yield return new WaitForSeconds(delay);
        if (btn == null) yield break;
        var cg = btn.GetComponent<CanvasGroup>();
        if (cg == null) cg = btn.gameObject.AddComponent<CanvasGroup>();
        var rt = btn.GetComponent<RectTransform>();
        if (rt == null) yield break;
        Vector2 from = rt.anchoredPosition;
        Vector2 to = from + new Vector2(0, buttonSlide);
        float dur = Mathf.Max(0.01f, entranceDuration * 0.6f);
        float t = 0f;
        while (t < dur)
        {
            t += Time.unscaledDeltaTime;
            float k = Mathf.Clamp01(t / dur);
            float e = 1f - Mathf.Pow(1f - k, 3f);
            if (cg != null) cg.alpha = e;
            rt.anchoredPosition = Vector2.Lerp(from, to, e);
            yield return null;
        }
        if (cg != null) { cg.alpha = 1f; cg.interactable = true; }
        rt.anchoredPosition = to;
    }

    IEnumerator FadeGroup(CanvasGroup cg, float from, float to, float duration)
    {
        if (cg == null) yield break;
        float t = 0f;
        while (t < duration)
        {
            t += Time.unscaledDeltaTime;
            cg.alpha = Mathf.Lerp(from, to, Mathf.Clamp01(t / duration));
            yield return null;
        }
        cg.alpha = to;
    }

    IEnumerator FadeAndLoadScene(string scene)
    {
        _mainGroup.blocksRaycasts = false;
        yield return FadeGroup(_mainGroup, 1f, 0f, fadeOutDuration);
        if (!string.IsNullOrEmpty(scene)) SceneManager.LoadScene(scene);
    }

    IEnumerator FadeAndQuit()
    {
        _mainGroup.blocksRaycasts = false;
        yield return FadeGroup(_mainGroup, 1f, 0f, fadeOutDuration);
        Application.Quit();
#if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
#endif
    }

    // ===================== ОБНОВЛЕНИЯ ЭФФЕКТОВ =====================

    void UpdateTitleShake()
    {
        if (_titleRect == null) return;
        float dt = Time.unscaledDeltaTime;
        _shakeT += dt;
        _pulsePhase += dt * 0.55f * Mathf.PI * 2f;

        _glitchTimer += dt;
        if (enableGlitch && _glitchRemaining <= 0f && _glitchTimer >= glitchInterval)
        {
            _glitchRemaining = glitchDuration;
            _glitchTimer = 0f;
        }
        bool glitching = _glitchRemaining > 0f;
        if (glitching) _glitchRemaining -= dt;

        float pulse = 1f + Mathf.Sin(_pulsePhase) * titleShakePulse;
        float intensity = titleShakeIntensity * pulse;

        float nx = (Mathf.PerlinNoise(_shakeT * titleShakeSpeed, 13.37f) - 0.5f) * 2f;
        float ny = (Mathf.PerlinNoise(7.21f, _shakeT * titleShakeSpeed) - 0.5f) * 2f;
        float ox = nx * intensity;
        float oy = ny * intensity;
        if (glitching)
        {
            ox += UnityEngine.Random.Range(-glitchOffset, glitchOffset);
            oy += UnityEngine.Random.Range(-glitchOffset, glitchOffset);
        }
        _titleRect.anchoredPosition = _titleBasePos + new Vector2(ox, oy);
    }

    void UpdateGrain()
    {
        if (_grain == null) return;
        _grainU += Time.unscaledDeltaTime * grainScrollSpeed;
        _grainV += Time.unscaledDeltaTime * grainScrollSpeed * 0.73f;
        if (_grainU > 1f) _grainU -= 1f;
        if (_grainV > 1f) _grainV -= 1f;
        _grain.uvRect = new Rect(_grainU, _grainV, 1f, 1f);
    }

    void UpdateVignette()
    {
        if (_vignette == null) return;
        _vignetteTimer += Time.unscaledDeltaTime;
        if (_vignetteTimer >= vignettePulseInterval)
        {
            _vignetteTimer = 0f;
            _vignette.color = new Color(_vignetteBase.r, _vignetteBase.g, _vignetteBase.b,
                Mathf.Clamp01(_vignetteBase.a + vignettePulseAmount));
            StartCoroutine(ResetVignette());
        }
    }

    IEnumerator ResetVignette()
    {
        yield return new WaitForSeconds(0.08f);
        if (_vignette != null) _vignette.color = _vignetteBase;
    }

    void UpdateTitleFlicker()
    {
        if (_title == null) return;
        if (!_titleFlickerActive)
        {
            _titleFlickerTimer += Time.unscaledDeltaTime;
            if (_titleFlickerTimer >= titleFlickerInterval)
            {
                _titleFlickerTimer = 0f;
                _titleFlickerActive = true;
                StartCoroutine(RunTitleFlicker());
            }
        }
    }

    IEnumerator RunTitleFlicker()
    {
        bool blackout = UnityEngine.Random.value < titleFlickerBlackout;
        int flicks = blackout ? UnityEngine.Random.Range(5, 11) : UnityEngine.Random.Range(2, 4);
        float step = blackout ? 0.07f : 0.05f;
        for (int i = 0; i < flicks; i++)
        {
            float dim = (UnityEngine.Random.value < 0.3f) ? 0f : titleFlickerDim;
            _title.color = new Color(
                _titleBase.r * dim + titleFlickerTint.r * (1f - dim) * 0.15f,
                _titleBase.g * dim + titleFlickerTint.g * (1f - dim) * 0.15f,
                _titleBase.b * dim + titleFlickerTint.b * (1f - dim) * 0.15f,
                _titleBase.a);
            yield return new WaitForSeconds(step * UnityEngine.Random.Range(0.6f, 1.4f));
        }
        _title.color = _titleBase;
        _titleFlickerActive = false;
    }

    // ===================== ПУБЛИЧНЫЕ КОМАНДЫ МЕНЮ =====================

    public void OnStartGame()  { if (!_ready) return; PlaySFX(startSound); StartCoroutine(FadeAndLoadScene(gameSceneName)); }
    public void OnContinue()   { if (!_ready) return; /* TODO: load save */ PlaySFX(clickSound); }
    public void OnOpenSettings() { if (!_ready) return; PlaySFX(clickSound); if (_settingsPanel != null) _settingsPanel.SetActive(true); }
    public void OnCloseSettings(){ PlaySFX(clickSound); if (_settingsPanel != null) _settingsPanel.SetActive(false); }
    public void OnExit()       { if (!_ready) return; PlaySFX(clickSound); if (_confirmExitPanel != null) _confirmExitPanel.SetActive(true); }
    public void OnExitCancel() { PlaySFX(clickSound); if (_confirmExitPanel != null) _confirmExitPanel.SetActive(false); }
    public void OnExitConfirm(){ PlaySFX(exitSound); StartCoroutine(FadeAndQuit()); }

    void PlaySFX(AudioClip clip)
    {
        if (audioSource != null && clip != null)
            audioSource.PlayOneShot(clip, sfxVolume);
    }
}
