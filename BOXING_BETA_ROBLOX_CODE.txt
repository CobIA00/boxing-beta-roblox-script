--[[
    Boxing Beta Autoplayer v3.0 - Universal (PC & Mobile)
    
    Este script unifica todos los componentes del autoplayer en un solo archivo
    para facilitar su implementación en Roblox. Incluye compatibilidad con PC y móvil,
    interfaz responsiva, controles táctiles, optimizaciones de rendimiento,
    configuración visual interactiva, persistencia de datos mejorada y 
    sistema anti-detección avanzado específico por plataforma.
    
    Desarrollado por: Manus
    Fecha: 06/06/2025
]]

--==============================================================================
-- CONFIGURACIÓN GLOBAL
--==============================================================================

local CONFIG = {
    -- Configuración general
    updateInterval = 0.06, -- Intervalo base de actualización (se ajusta por rendimiento)
    saveInterval = 300, -- segundos
    uiScale = 1.0, -- Factor de escala para la UI
    
    -- Configuración de aprendizaje
    learningRate = 0.1,
    discountFactor = 0.9,
    explorationRate = 0.2,
    maxMemorySize = 1000, -- Tamaño base de memoria (se ajusta por rendimiento)
    batchSize = 32,
    
    -- Configuración de percepción
    screenAnalysisInterval = 0.06, -- Intervalo base (se ajusta por rendimiento)
    detectionThreshold = 0.7,
    
    -- Configuración de decisiones
    reactionTimeMin = 0.1, -- Tiempo de reacción base (se ajusta por anti-detección)
    reactionTimeMax = 0.3, -- Tiempo de reacción base (se ajusta por anti-detección)
    comboProbability = 0.6,
    maxComboLength = 4,
    
    -- Configuración de ejecución
    delayMin = 0.04, -- Retraso base (se ajusta por anti-detección)
    delayMax = 0.18, -- Retraso base (se ajusta por anti-detección)
    humanVariation = 0.2, -- Variación base (se ajusta por anti-detección)
    inputHoldTime = 0.1, -- Tiempo base (se ajusta por anti-detección)
    
    -- Configuración anti-detección
    antiDetectionMeasures = true,
    humanPatterns = true,
    variableTimings = true,
    avoidPerfectTiming = true,
    randomMistakes = true,
    
    -- Configuración de rendimiento
    resourceOptimization = true,
    
    -- Configuración de persistencia
    useCompression = true,
    useEncryption = false, -- Desactivado por simplicidad en este ejemplo
    useBackups = true,
    maxBackups = 3,
    autoSaveInterval = 300,
    saveSlots = 3
}

--==============================================================================
-- SERVICIOS DE ROBLOX
--==============================================================================

local UserInputService = game:GetService("UserInputService")
local GuiService = game:GetService("GuiService")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local Stats = game:GetService("Stats")
local ContextActionService = game:GetService("ContextActionService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

-- Cargar módulos de aprendizaje
local LearningEngine = require("LearningEngine")
local LearningConfigUI = require("LearningConfigUI")

--==============================================================================
-- MÓDULO: PlatformDetection
--==============================================================================

--[[
    PlatformDetection.lua
    Parte del Boxing Beta Autoplayer v3.0 - Universal (PC & Mobile)
    
    Este módulo se encarga de detectar la plataforma y características del dispositivo,
    proporcionando información crucial para la adaptación de la experiencia.
    
    Mejoras implementadas:
    - Sistema de eventos para notificar cambios de orientación/resolución
    - Detección mejorada de dispositivos con validaciones robustas
    - Caché de información para reducir cálculos repetidos
    - Detección de capacidades de hardware
]]

-- Servicios de Roblox
local UserInputService = game:GetService("UserInputService")
local GuiService = game:GetService("GuiService")
local RunService = game:GetService("RunService")
local Stats = game:GetService("Stats")

-- Módulo EventBus (para implementar patrón de eventos)
local EventBus = require(script.Parent.EventBus)

local PlatformDetection = {}
PlatformDetection.__index = PlatformDetection

-- Constantes
local DEVICE_TYPES = {
    PC = "PC",
    MOBILE = "Mobile",
    TABLET = "Tablet",
    CONSOLE = "Console",
    UNKNOWN = "Unknown"
}

local ORIENTATIONS = {
    LANDSCAPE = "Landscape",
    PORTRAIT = "Portrait"
}

local OS_TYPES = {
    WINDOWS = "Windows",
    MACOS = "MacOS",
    IOS = "iOS",
    ANDROID = "Android",
    XBOX = "Xbox",
    PLAYSTATION = "PlayStation",
    UNKNOWN = "Unknown"
}

local TABLET_DIAGONAL_THRESHOLD = 1000 -- Umbral para considerar un dispositivo como tablet

function PlatformDetection.new(services)
    local self = setmetatable({}, PlatformDetection)
    
    -- Inyección de dependencias (servicios)
    self.userInputService = services and services.UserInputService or UserInputService
    self.guiService = services and services.GuiService or GuiService
    self.runService = services and services.RunService or RunService
    self.stats = services and services.Stats or Stats
    
    -- Eventos
    self.events = {
        orientationChanged = EventBus.new(),
        screenSizeChanged = EventBus.new(),
        deviceInfoUpdated = EventBus.new()
    }
    
    -- Información del dispositivo (con valores por defecto)
    self.deviceInfo = {
        deviceType = DEVICE_TYPES.UNKNOWN,
        os = OS_TYPES.UNKNOWN,
        touchEnabled = false,
        keyboardEnabled = false,
        mouseEnabled = false,
        gamepadEnabled = false,
        screenSize = Vector2.new(0, 0),
        orientation = ORIENTATIONS.LANDSCAPE,
        isMobile = false,
        isTablet = false,
        isConsole = false,
        isPC = false,
        -- Nuevos campos para capacidades de hardware
        memoryUsageMb = 0,
        cpuUsage = 0,
        gpuUsage = 0,
        networkLatencyMs = 0,
        fps = 0
    }
    
    -- Caché para reducir cálculos repetidos
    self.cache = {
        lastScreenSizeCheck = 0,
        lastHardwareCheck = 0,
        screenCheckInterval = 1, -- segundos
        hardwareCheckInterval = 5 -- segundos
    }
        isTablet = false,
        isConsole = false,
        isPC = false
    }
    
    -- Detectar información
    self:detectPlatform()
    
    -- Conectar eventos para cambios (ej. orientación)
    self:connectEvents()
    
    return self
end

-- Detectar plataforma y características
function PlatformDetection:detectPlatform()
    -- Usar pcall para manejar posibles errores en las APIs
    local success, result = pcall(function()
        -- Detectar tipo de dispositivo con validaciones
        if self.guiService:IsTenFootInterface() then
            self.deviceInfo.deviceType = DEVICE_TYPES.CONSOLE
            self.deviceInfo.isConsole = true
        elseif self.userInputService.TouchEnabled and not self.userInputService.KeyboardEnabled then
            self.deviceInfo.deviceType = DEVICE_TYPES.MOBILE
            self.deviceInfo.isMobile = true
        elseif self.userInputService.KeyboardEnabled and self.userInputService.MouseEnabled then
            self.deviceInfo.deviceType = DEVICE_TYPES.PC
            self.deviceInfo.isPC = true
        else
            -- Caso por defecto o desconocido
            self.deviceInfo.deviceType = DEVICE_TYPES.UNKNOWN
        end
        
        -- Detectar características con validaciones
        self.deviceInfo.touchEnabled = self.userInputService.TouchEnabled or false
        self.deviceInfo.keyboardEnabled = self.userInputService.KeyboardEnabled or false
        self.deviceInfo.mouseEnabled = self.userInputService.MouseEnabled or false
        self.deviceInfo.gamepadEnabled = self.userInputService.GamepadEnabled or false
    self.deviceInfo.keyboardEnabled = UserInputService.KeyboardEnabled
    self.deviceInfo.mouseEnabled = UserInputService.MouseEnabled
    self.deviceInfo.gamepadEnabled = UserInputService.GamepadEnabled
    
    -- Detectar OS (simplificado)
    local platform = UserInputService:GetPlatform()
    if platform == Enum.Platform.Windows then
        self.deviceInfo.os = "Windows"
    elseif platform == Enum.Platform.OSX then
        self.deviceInfo.os = "MacOS"
    elseif platform == Enum.Platform.IOS then
        self.deviceInfo.os = "iOS"
    elseif platform == Enum.Platform.Android then
        self.deviceInfo.os = "Android"
    elseif platform == Enum.Platform.XBoxOne then
        self.deviceInfo.os = "Xbox"
    elseif platform == Enum.Platform.PS4 then
        self.deviceInfo.os = "PlayStation"
    end
    
    -- Detectar tamaño de pantalla y orientación
    self:updateScreenInfo()
    
    -- Determinar si es tablet (heurística)
    if self.deviceInfo.isMobile then
        local screenSize = self.deviceInfo.screenSize
        local diagonal = math.sqrt(screenSize.X^2 + screenSize.Y^2)
        -- Asumir que pantallas táctiles grandes son tablets (ajustar umbral según sea necesario)
        if diagonal > 1000 then 
            self.deviceInfo.isTablet = true
            self.deviceInfo.deviceType = "Tablet"
        end
    end
    
    print("Plataforma detectada:", self.deviceInfo.deviceType, "OS:", self.deviceInfo.os)
end

-- Actualizar información de pantalla
function PlatformDetection:updateScreenInfo()
    local camera = workspace.CurrentCamera
    if camera then
        self.deviceInfo.screenSize = camera.ViewportSize
        
        if self.deviceInfo.screenSize.Y > self.deviceInfo.screenSize.X then
            self.deviceInfo.orientation = "Portrait"
        else
            self.deviceInfo.orientation = "Landscape"
        end
    end
end

-- Conectar eventos para cambios
function PlatformDetection:connectEvents()
    -- Actualizar tamaño de pantalla y orientación si la cámara cambia
    if workspace.CurrentCamera then
        workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(function()
            self:updateScreenInfo()
            -- Notificar cambio si es necesario
        end)
    end
end

-- Obtener información del dispositivo
function PlatformDetection:getDeviceInfo()
    -- Devolver una copia para evitar modificaciones externas
    local infoCopy = {}
    for k, v in pairs(self.deviceInfo) do
        infoCopy[k] = v
    end
    return infoCopy
end




--==============================================================================
-- MÓDULO: AdaptiveConfig
--==============================================================================

local AdaptiveConfig = {}
AdaptiveConfig.__index = AdaptiveConfig

function AdaptiveConfig.new(platformDetection)
    local self = setmetatable({}, AdaptiveConfig)
    
    -- Referencias a otros sistemas
    self.platformDetection = platformDetection
    
    -- Información del dispositivo
    self.deviceInfo = platformDetection:getDeviceInfo()
    
    -- Configuración base (copia de CONFIG global)
    self.config = {}
    for key, value in pairs(CONFIG) do
        self.config[key] = value
    end
    
    -- Ajustar configuración según la plataforma
    self:adjustConfigForPlatform()
    
    -- Callbacks para cambios de configuración
    self.onConfigChanged = nil
    
    return self
end

-- Ajustar configuración según la plataforma
function AdaptiveConfig:adjustConfigForPlatform()
    -- Ajustes específicos para móvil
    if self.deviceInfo.isMobile then
        -- Ajustes generales para móvil
        self.config.updateInterval = self.config.updateInterval * 1.2
        self.config.screenAnalysisInterval = self.config.screenAnalysisInterval * 1.2
        self.config.maxMemorySize = math.floor(self.config.maxMemorySize * 0.8)
        self.config.batchSize = math.floor(self.config.batchSize * 0.75)
        
        -- Ajustes específicos para tablet vs teléfono
        if self.deviceInfo.isTablet then
            -- Tablets: ajustes intermedios
            self.config.uiScale = 1.2
        else
            -- Teléfonos: ajustes más agresivos
            self.config.uiScale = 1.5
            self.config.updateInterval = self.config.updateInterval * 1.1
            self.config.screenAnalysisInterval = self.config.screenAnalysisInterval * 1.1
            self.config.maxMemorySize = math.floor(self.config.maxMemorySize * 0.9)
        end
        
        -- Ajustes según orientación
        if self.deviceInfo.orientation == "Portrait" then
            self.config.uiScale = self.config.uiScale * 1.1
        end
    else
        -- Ajustes para PC
        self.config.uiScale = 1.0
        
        -- Ajustes según resolución
        local screenSize = self.deviceInfo.screenSize
        if screenSize.X > 1920 then
            -- Pantallas de alta resolución
            self.config.uiScale = 1.2
        elseif screenSize.X < 1280 then
            -- Pantallas de baja resolución
            self.config.uiScale = 0.9
        end
    end
    
    print("Configuración adaptada para: " .. self.deviceInfo.deviceType)
end

-- Obtener configuración actual
function AdaptiveConfig:getConfig()
    -- Devolver una copia para evitar modificaciones externas
    local configCopy = {}
    for key, value in pairs(self.config) do
        configCopy[key] = value
    end
    return configCopy
end

-- Actualizar un valor de configuración
function AdaptiveConfig:updateConfigValue(key, value)
    if self.config[key] ~= nil then
        local oldValue = self.config[key]
        self.config[key] = value
        
        -- Llamar al callback si está definido
        if self.onConfigChanged then
            self.onConfigChanged(key, value, oldValue)
        end
        
        return true
    end
    return false
end

-- Establecer callback para cambios de configuración
function AdaptiveConfig:setConfigChangedCallback(callback)
    self.onConfigChanged = callback
end

--==============================================================================
-- MÓDULO: ResponsiveUI
--==============================================================================

local ResponsiveUI = {}
ResponsiveUI.__index = ResponsiveUI

function ResponsiveUI.new(platformDetection, adaptiveConfig)
    local self = setmetatable({}, ResponsiveUI)
    
    -- Referencias a otros sistemas
    self.platformDetection = platformDetection
    self.adaptiveConfig = adaptiveConfig
    
    -- Información del dispositivo
    self.deviceInfo = platformDetection:getDeviceInfo()
    
    -- Estado de la UI
    self.isVisible = false
    self.isEnabled = false
    self.selectedTab = "Estadísticas"
    self.logs = {}
    self.maxLogs = 100
    self.logLevel = "Info"
    self.logLevels = {
        Debug = 1,
        Info = 2,
        Warning = 3,
        Error = 4,
        Success = 5
    }
    
    -- Dimensiones de UI (se ajustarán según la plataforma)
    self.dimensions = {
        mainFrameWidth = 300,
        mainFrameHeight = 400,
        titleBarHeight = 30,
        tabHeight = 30,
        elementHeight = 25,
        elementSpacing = 5,
        padding = 10,
        fontSize = 14,
        buttonSize = 20
    }
    
    -- Estilos de UI
    self.style = {
        backgroundColor = Color3.fromRGB(30, 30, 30),
        titleBarColor = Color3.fromRGB(40, 40, 40),
        tabBarColor = Color3.fromRGB(35, 35, 35),
        buttonColor = Color3.fromRGB(60, 60, 60),
        buttonActiveColor = Color3.fromRGB(80, 80, 80),
        textColor = Color3.fromRGB(220, 220, 220),
        successColor = Color3.fromRGB(0, 180, 0),
        errorColor = Color3.fromRGB(180, 0, 0),
        warningColor = Color3.fromRGB(180, 180, 0),
        infoColor = Color3.fromRGB(0, 120, 215),
        font = Enum.Font.SourceSans
    }
    
    -- Elementos de UI
    self.ui = {
        mainFrame = nil,
        titleBar = nil,
        titleLabel = nil,
        closeButton = nil,
        minimizeButton = nil,
        tabsFrame = nil,
        tabButtons = {},
        contentFrame = nil,
        panels = {},
        toggleButton = nil,
        statLabels = {},
        logTexts = {},
        logContainer = nil,
        configElements = {}
    }
    
    -- Ajustar dimensiones según la plataforma
    self:adjustDimensions()
    
    return self
end

-- Ajustar dimensiones según la plataforma
function ResponsiveUI:adjustDimensions()
    -- Factor de escala base
    local scaleFactor = self.adaptiveConfig:getConfig().uiScale
    
    -- Ajustar dimensiones según el factor de escala
    self.dimensions.mainFrameWidth = math.floor(300 * scaleFactor)
    self.dimensions.mainFrameHeight = math.floor(400 * scaleFactor)
    self.dimensions.titleBarHeight = math.floor(30 * scaleFactor)
    self.dimensions.tabHeight = math.floor(30 * scaleFactor)
    self.dimensions.elementHeight = math.floor(25 * scaleFactor)
    self.dimensions.elementSpacing = math.floor(5 * scaleFactor)
    self.dimensions.padding = math.floor(10 * scaleFactor)
    self.dimensions.fontSize = math.floor(14 * scaleFactor)
    self.dimensions.buttonSize = math.floor(20 * scaleFactor)
    
    -- Ajustes específicos para móvil
    if self.deviceInfo.isMobile then
        -- En móvil, hacer elementos más grandes para facilitar el toque
        self.dimensions.elementHeight = math.max(30, self.dimensions.elementHeight)
        self.dimensions.buttonSize = math.max(25, self.dimensions.buttonSize)
        self.dimensions.elementSpacing = math.max(8, self.dimensions.elementSpacing)
        self.dimensions.padding = math.max(12, self.dimensions.padding)
        
        -- Ajustar tamaño general según orientación
        if self.deviceInfo.orientation == "Portrait" then
            -- En modo retrato, hacer la UI más estrecha pero más alta
            self.dimensions.mainFrameWidth = math.floor(self.dimensions.mainFrameWidth * 0.9)
            self.dimensions.mainFrameHeight = math.floor(self.dimensions.mainFrameHeight * 1.1)
        end
    end
end

-- Crear interfaz de usuario
function ResponsiveUI:createUI()
    -- Limpiar UI anterior si existe
    if self.ui.mainFrame then
        self.ui.mainFrame:Destroy()
    end
    
    -- Crear ScreenGui
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "BoxingBetaAutoplayerUI"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    
    -- Crear marco principal
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, self.dimensions.mainFrameWidth, 0, self.dimensions.mainFrameHeight)
    mainFrame.Position = UDim2.new(0, 10, 0, 10)
    mainFrame.BackgroundColor3 = self.style.backgroundColor
    mainFrame.BorderSizePixel = 0
    mainFrame.Visible = self.isVisible
    mainFrame.Parent = screenGui
    self.ui.mainFrame = mainFrame
    
    -- Añadir esquinas redondeadas
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 5)
    corner.Parent = mainFrame
    
    -- Crear barra de título
    self:createTitleBar()
    
    -- Crear barra de pestañas
    self:createTabBar()
    
    -- Crear marco de contenido
    self:createContentFrame()
    
    -- Crear paneles para cada pestaña
    self:createPanels()
    
    -- Crear botón de activar/desactivar
    self:createToggleButton()
    
    -- Conectar eventos
    self:connectEvents()
    
    -- Seleccionar pestaña por defecto
    self:selectTab(self.selectedTab)
    
    -- Añadir ScreenGui al PlayerGui
    local player = game:GetService("Players").LocalPlayer
    if player and player:FindFirstChild("PlayerGui") then
        screenGui.Parent = player.PlayerGui
    else
        -- Fallback si no se encuentra PlayerGui
        screenGui.Parent = game:GetService("CoreGui")
    end
    
    return screenGui
end

-- Crear barra de título
function ResponsiveUI:createTitleBar()
    local titleBar = Instance.new("Frame")
    titleBar.Name = "TitleBar"
    titleBar.Size = UDim2.new(1, 0, 0, self.dimensions.titleBarHeight)
    titleBar.Position = UDim2.new(0, 0, 0, 0)
    titleBar.BackgroundColor3 = self.style.titleBarColor
    titleBar.BorderSizePixel = 0
    titleBar.Parent = self.ui.mainFrame
    self.ui.titleBar = titleBar
    
    -- Añadir esquinas redondeadas (solo arriba)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 5)
    corner.Parent = titleBar
    
    -- Crear etiqueta de título
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Name = "TitleLabel"
    titleLabel.Size = UDim2.new(1, -self.dimensions.buttonSize*2 - self.dimensions.padding*2, 1, 0)
    titleLabel.Position = UDim2.new(0, self.dimensions.padding, 0, 0)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = "Boxing Beta Autoplayer v3.0"
    titleLabel.TextColor3 = self.style.textColor
    titleLabel.TextSize = self.dimensions.fontSize
    titleLabel.Font = self.style.font
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = titleBar
    self.ui.titleLabel = titleLabel
    
    -- Crear botón de cerrar
    local closeButton = Instance.new("TextButton")
    closeButton.Name = "CloseButton"
    closeButton.Size = UDim2.new(0, self.dimensions.buttonSize, 0, self.dimensions.buttonSize)
    closeButton.Position = UDim2.new(1, -self.dimensions.buttonSize - self.dimensions.padding, 0.5, -self.dimensions.buttonSize/2)
    closeButton.BackgroundColor3 = self.style.errorColor
    closeButton.Text = "X"
    closeButton.TextColor3 = self.style.textColor
    closeButton.TextSize = self.dimensions.fontSize
    closeButton.Font = self.style.font
    closeButton.Parent = titleBar
    self.ui.closeButton = closeButton
    
    -- Añadir esquinas redondeadas al botón de cerrar
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 3)
    closeCorner.Parent = closeButton
    
    -- Crear botón de minimizar
    local minimizeButton = Instance.new("TextButton")
    minimizeButton.Name = "MinimizeButton"
    minimizeButton.Size = UDim2.new(0, self.dimensions.buttonSize, 0, self.dimensions.buttonSize)
    minimizeButton.Position = UDim2.new(1, -self.dimensions.buttonSize*2 - self.dimensions.padding*2, 0.5, -self.dimensions.buttonSize/2)
    minimizeButton.BackgroundColor3 = self.style.buttonColor
    minimizeButton.Text = "-"
    minimizeButton.TextColor3 = self.style.textColor
    minimizeButton.TextSize = self.dimensions.fontSize
    minimizeButton.Font = self.style.font
    minimizeButton.Parent = titleBar
    self.ui.minimizeButton = minimizeButton
    
    -- Añadir esquinas redondeadas al botón de minimizar
    local minimizeCorner = Instance.new("UICorner")
    minimizeCorner.CornerRadius = UDim.new(0, 3)
    minimizeCorner.Parent = minimizeButton
end

-- Crear barra de pestañas
function ResponsiveUI:createTabBar()
    local tabsFrame = Instance.new("Frame")
    tabsFrame.Name = "TabsFrame"
    tabsFrame.Size = UDim2.new(1, 0, 0, self.dimensions.tabHeight)
    tabsFrame.Position = UDim2.new(0, 0, 0, self.dimensions.titleBarHeight)
    tabsFrame.BackgroundColor3 = self.style.tabBarColor
    tabsFrame.BorderSizePixel = 0
    tabsFrame.Parent = self.ui.mainFrame
    self.ui.tabsFrame = tabsFrame
    
    -- Definir pestañas
    local tabs = {"Estadísticas", "Configuración", "Logs"}
    local tabWidth = 1 / #tabs
    
    -- Crear botones de pestañas
    for i, tabName in ipairs(tabs) do
        local tabButton = Instance.new("TextButton")
        tabButton.Name = tabName .. "Tab"
        tabButton.Size = UDim2.new(tabWidth, 0, 1, 0)
        tabButton.Position = UDim2.new(tabWidth * (i-1), 0, 0, 0)
        tabButton.BackgroundColor3 = self.style.buttonColor
        tabButton.Text = tabName
        tabButton.TextColor3 = self.style.textColor
        tabButton.TextSize = self.dimensions.fontSize
        tabButton.Font = self.style.font
        tabButton.Parent = tabsFrame
        
        -- Guardar referencia al botón
        self.ui.tabButtons[tabName] = tabButton
    end
end

-- Crear marco de contenido
function ResponsiveUI:createContentFrame()
    local contentFrame = Instance.new("Frame")
    contentFrame.Name = "ContentFrame"
    contentFrame.Size = UDim2.new(1, 0, 1, -self.dimensions.titleBarHeight - self.dimensions.tabHeight)
    contentFrame.Position = UDim2.new(0, 0, 0, self.dimensions.titleBarHeight + self.dimensions.tabHeight)
    contentFrame.BackgroundColor3 = self.style.backgroundColor
    contentFrame.BorderSizePixel = 0
    contentFrame.Parent = self.ui.mainFrame
    self.ui.contentFrame = contentFrame
end

-- Crear paneles para cada pestaña
function ResponsiveUI:createPanels()
    -- Crear panel para cada pestaña
    local tabs = {"Estadísticas", "Configuración", "Logs"}
    
    for _, tabName in ipairs(tabs) do
        local panel = Instance.new("ScrollingFrame")
        panel.Name = tabName .. "Panel"
        panel.Size = UDim2.new(1, -self.dimensions.padding*2, 1, -self.dimensions.padding*2 - self.dimensions.elementHeight - self.dimensions.elementSpacing)
        panel.Position = UDim2.new(0, self.dimensions.padding, 0, self.dimensions.padding)
        panel.BackgroundTransparency = 1
        panel.BorderSizePixel = 0
        panel.ScrollBarThickness = 6
        panel.Visible = false -- Inicialmente oculto
        panel.Parent = self.ui.contentFrame
        
        -- Guardar referencia al panel
        self.ui.panels[tabName] = panel
    end
    
    -- Crear contenido específico para cada panel
    self:createStatsPanel()
    self:createConfigPanel()
    self:createLogsPanel()
end

-- Crear contenido del panel Estadísticas
function ResponsiveUI:createStatsPanel()
    local panel = self.ui.panels["Estadísticas"]
    if not panel then return end
    
    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, self.dimensions.elementSpacing)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = panel
    
    -- Definir estadísticas a mostrar
    local stats = {
        "Status",
        "Iteraciones",
        "Tasa Éxito",
        "Recompensa Promedio",
        "Tasa Exploración",
        "Tamaño Memoria",
        "Tamaño Tabla Q",
        "Victorias",
        "Derrotas",
        "Acciones Totales",
        "Tiempo Promedio Acción",
        "Runtime",
        "DeviceInfo"
    }
    
    -- Crear etiqueta para cada estadística
    for i, statName in ipairs(stats) do
        local statLabel = Instance.new("TextLabel")
        statLabel.Name = statName .. "Label"
        statLabel.Size = UDim2.new(1, 0, 0, self.dimensions.elementHeight)
        statLabel.BackgroundTransparency = 1
        statLabel.Text = statName .. ": N/A"
        statLabel.TextColor3 = self.style.textColor
        statLabel.TextSize = self.dimensions.fontSize
        statLabel.Font = self.style.font
        statLabel.TextXAlignment = Enum.TextXAlignment.Left
        statLabel.LayoutOrder = i
        statLabel.Parent = panel
        self.ui.statLabels[statName] = statLabel
    end
    
    panel.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + self.dimensions.padding)
end

-- Actualizar contenido del panel Estadísticas
function ResponsiveUI:updateStatsPanel()
    local panel = self.ui.panels["Estadísticas"]
    if not panel then return end
    
    -- Actualizar tamaños y fuentes
    for _, child in pairs(panel:GetChildren()) do
        if child:IsA("TextLabel") then
            child.Size = UDim2.new(1, -self.dimensions.padding, 0, self.dimensions.elementHeight)
            child.TextSize = self.dimensions.fontSize
        end
    end
    
    -- Actualizar tamaño del canvas
    local layout = panel:FindFirstChildOfClass("UIListLayout")
    if layout then
        panel.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + self.dimensions.padding)
    end
end

-- Crear contenido del panel Configuración (implementación simplificada)
function ResponsiveUI:createConfigPanel()
    local panel = self.ui.panels["Configuración"]
    if not panel then return end
    
    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, self.dimensions.elementSpacing)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = panel
    
    -- Ejemplo: Configuración de Tasa de Exploración
    local expRateLabel = Instance.new("TextLabel")
    expRateLabel.Size = UDim2.new(1, -self.dimensions.padding, 0, self.dimensions.elementHeight)
    expRateLabel.BackgroundTransparency = 1
    expRateLabel.Text = "Tasa de Exploración: " .. self.adaptiveConfig:getConfig().explorationRate
    expRateLabel.TextColor3 = self.style.textColor
    expRateLabel.TextSize = self.dimensions.fontSize
    expRateLabel.Font = self.style.font
    expRateLabel.TextXAlignment = Enum.TextXAlignment.Left
    expRateLabel.LayoutOrder = 1
    expRateLabel.Parent = panel
    self.ui.configElements["ExplorationRateLabel"] = expRateLabel
    
    local expRateSlider = Instance.new("Frame") -- Simulación de slider
    expRateSlider.Size = UDim2.new(1, -self.dimensions.padding, 0, self.dimensions.elementHeight * 0.5)
    expRateSlider.BackgroundColor3 = self.style.buttonColor
    expRateSlider.LayoutOrder = 2
    expRateSlider.Parent = panel
    
    -- Ejemplo: Checkbox para Errores Aleatorios
    local mistakesCheck = Instance.new("TextButton")
    mistakesCheck.Size = UDim2.new(1, -self.dimensions.padding, 0, self.dimensions.elementHeight)
    mistakesCheck.BackgroundColor3 = self.style.buttonColor
    mistakesCheck.Text = "Errores Aleatorios: " .. (self.adaptiveConfig:getConfig().randomMistakes and "ON" or "OFF")
    mistakesCheck.TextColor3 = self.style.textColor
    mistakesCheck.TextSize = self.dimensions.fontSize
    mistakesCheck.Font = self.style.font
    mistakesCheck.LayoutOrder = 3
    mistakesCheck.Parent = panel
    self.ui.configElements["RandomMistakesCheck"] = mistakesCheck
    
    -- Añadir más elementos de configuración según sea necesario...
    
    panel.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + self.dimensions.padding)
end

-- Actualizar contenido del panel Configuración
function ResponsiveUI:updateConfigPanel()
    local panel = self.ui.panels["Configuración"]
    if not panel then return end
    
    -- Actualizar tamaños y fuentes
    for _, child in pairs(panel:GetChildren()) do
        if child:IsA("TextLabel") or child:IsA("TextButton") then
            child.Size = UDim2.new(1, -self.dimensions.padding, 0, self.dimensions.elementHeight)
            child.TextSize = self.dimensions.fontSize
        elseif child:IsA("Frame") and child.Name ~= "UIListLayout" then
            child.Size = UDim2.new(1, -self.dimensions.padding, 0, self.dimensions.elementHeight * 0.5)
        end
    end
    
    -- Actualizar valores de configuración
    if self.ui.configElements["ExplorationRateLabel"] then
        self.ui.configElements["ExplorationRateLabel"].Text = "Tasa de Exploración: " .. self.adaptiveConfig:getConfig().explorationRate
    end
    
    if self.ui.configElements["RandomMistakesCheck"] then
        self.ui.configElements["RandomMistakesCheck"].Text = "Errores Aleatorios: " .. (self.adaptiveConfig:getConfig().randomMistakes and "ON" or "OFF")
    end
    
    -- Actualizar tamaño del canvas
    local layout = panel:FindFirstChildOfClass("UIListLayout")
    if layout then
        panel.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + self.dimensions.padding)
    end
end

-- Crear contenido del panel Logs
function ResponsiveUI:createLogsPanel()
    local panel = self.ui.panels["Logs"]
    if not panel then return end
    
    -- Crear contenedor para logs
    local logContainer = Instance.new("Frame")
    logContainer.Name = "LogContainer"
    logContainer.Size = UDim2.new(1, 0, 1, 0)
    logContainer.BackgroundTransparency = 1
    logContainer.Parent = panel
    self.ui.logContainer = logContainer
    
    -- Crear layout para logs
    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, self.dimensions.elementSpacing)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = logContainer
    
    -- Crear etiquetas para logs (reutilizables)
    for i = 1, 20 do
        local logText = Instance.new("TextLabel")
        logText.Name = "LogText" .. i
        logText.Size = UDim2.new(1, 0, 0, self.dimensions.elementHeight)
        logText.BackgroundTransparency = 1
        logText.Text = ""
        logText.TextColor3 = self.style.infoColor
        logText.TextSize = self.dimensions.fontSize
        logText.Font = self.style.font
        logText.TextXAlignment = Enum.TextXAlignment.Left
        logText.TextWrapped = true
        logText.LayoutOrder = i
        logText.Visible = false
        logText.Parent = logContainer
        
        table.insert(self.ui.logTexts, logText)
    end
end

-- Crear botón de activar/desactivar
function ResponsiveUI:createToggleButton()
    local toggleButton = Instance.new("TextButton")
    toggleButton.Name = "ToggleButton"
    toggleButton.Size = UDim2.new(1, -self.dimensions.padding*2, 0, self.dimensions.elementHeight)
    toggleButton.Position = UDim2.new(0, self.dimensions.padding, 1, -self.dimensions.elementHeight - self.dimensions.padding)
    toggleButton.BackgroundColor3 = self.isEnabled and self.style.successColor or self.style.errorColor
    toggleButton.Text = self.isEnabled and "ACTIVADO" or "DESACTIVADO"
    toggleButton.TextColor3 = self.style.textColor
    toggleButton.TextSize = self.dimensions.fontSize
    toggleButton.Font = self.style.font
    toggleButton.Parent = self.ui.contentFrame
    self.ui.toggleButton = toggleButton
    
    -- Añadir esquinas redondeadas
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 4)
    corner.Parent = toggleButton
end

-- Conectar eventos de la UI (implementación simplificada)
function ResponsiveUI:connectEvents()
    -- Botón de cerrar
    self.ui.closeButton.MouseButton1Click:Connect(function()
        self:hide()
    end)
    
    -- Botón de minimizar
    self.ui.minimizeButton.MouseButton1Click:Connect(function()
        self.ui.contentFrame.Visible = not self.ui.contentFrame.Visible
        self.ui.tabsFrame.Visible = not self.ui.tabsFrame.Visible
        self.ui.toggleButton.Visible = not self.ui.toggleButton.Visible
        self.ui.mainFrame.Size = self.ui.contentFrame.Visible and 
            UDim2.new(0, self.dimensions.mainFrameWidth, 0, self.dimensions.mainFrameHeight) or 
            UDim2.new(0, self.dimensions.mainFrameWidth, 0, self.dimensions.titleBarHeight)
    end)
    
    -- Botón de activar/desactivar
    self.ui.toggleButton.MouseButton1Click:Connect(function()
        self:toggleEnabled()
    end)
    
    -- Botones de pestañas
    for tabName, button in pairs(self.ui.tabButtons) do
        button.MouseButton1Click:Connect(function()
            self:selectTab(tabName)
        end)
    end
    
    -- Eventos de configuración (ejemplo)
    if self.ui.configElements["RandomMistakesCheck"] then
        self.ui.configElements["RandomMistakesCheck"].MouseButton1Click:Connect(function()
            local config = self.adaptiveConfig:getConfig()
            config.randomMistakes = not config.randomMistakes
            self.adaptiveConfig:updateConfigValue("randomMistakes", config.randomMistakes)
            self.ui.configElements["RandomMistakesCheck"].Text = "Errores Aleatorios: " .. (config.randomMistakes and "ON" or "OFF")
            self:addLog("Configuración actualizada: Errores Aleatorios = " .. (config.randomMistakes and "ON" or "OFF"), "Info")
        end)
    end
end

-- Resto de métodos de la UI (mostrar, ocultar, toggle, etc.) - implementación simplificada
function ResponsiveUI:show()
    self.isVisible = true
    if self.ui.mainFrame then
        self.ui.mainFrame.Visible = true
    end
end

function ResponsiveUI:hide()
    self.isVisible = false
    if self.ui.mainFrame then
        self.ui.mainFrame.Visible = false
    end
end

function ResponsiveUI:toggleEnabled()
    self.isEnabled = not self.isEnabled
    self:updateToggleButton()
    self:addLog("Autoplayer " .. (self.isEnabled and "ACTIVADO" or "DESACTIVADO"), "Info")
    
    if self.mainModule then
        self.mainModule:setEnabled(self.isEnabled)
    end
end

function ResponsiveUI:updateToggleButton()
    if self.ui.toggleButton then
        if self.isEnabled then
            self.ui.toggleButton.Text = "ACTIVADO"
            self.ui.toggleButton.BackgroundColor3 = self.style.successColor
            self.ui.statLabels["Status"].Text = "Estado: Activado"
            self.ui.statLabels["Status"].TextColor3 = self.style.successColor
        else
            self.ui.toggleButton.Text = "DESACTIVADO"
            self.ui.toggleButton.BackgroundColor3 = self.style.errorColor
            self.ui.statLabels["Status"].Text = "Estado: Desactivado"
            self.ui.statLabels["Status"].TextColor3 = self.style.errorColor
        end
    end
end

function ResponsiveUI:selectTab(tabName)
    self.selectedTab = tabName
    self:updateTabs()
end

function ResponsiveUI:updateTabs()
    for name, panel in pairs(self.ui.panels) do
        panel.Visible = (name == self.selectedTab)
    end
    
    for name, button in pairs(self.ui.tabButtons) do
        if name == self.selectedTab then
            button.BackgroundColor3 = self.style.buttonActiveColor
        else
            button.BackgroundColor3 = self.style.buttonColor
        end
    end
end

-- Añadir mensaje al log (implementación simplificada)
function ResponsiveUI:addLog(message, level)
    level = level or "Info"
    local levelNum = self.logLevels[level] or self.logLevels.Info
    local currentLevelNum = self.logLevels[self.logLevel] or self.logLevels.Info
    
    -- Filtrar por nivel de log
    if levelNum > currentLevelNum then
        return
    end
    
    -- Añadir timestamp y nivel al mensaje
    local timestamp = os.date("%H:%M:%S")
    local logEntry = string.format("[%s] [%s] %s", timestamp, level, message)
    
    table.insert(self.logs, {text = logEntry, level = level})
    
    -- Limitar tamaño del log
    if #self.logs > self.maxLogs then
        table.remove(self.logs, 1)
    end
    
    -- Actualizar UI si la pestaña de logs está visible
    if self.selectedTab == "Logs" then
        self:updateLogDisplay()
    end
end

-- Actualizar la visualización de logs (implementación simplificada)
function ResponsiveUI:updateLogDisplay()
    if not self.ui.logContainer then return end
    
    local logIndex = 1
    for i = #self.logs, 1, -1 do
        if logIndex > self.maxLogs then break end
        
        local logLabel = self.ui.logTexts[logIndex]
        local logData = self.logs[i]
        
        if logLabel then
            logLabel.Text = logData.text
            logLabel.Visible = true
            
            -- Asignar color según nivel
            if logData.level == "Error" then
                logLabel.TextColor3 = self.style.errorColor
            elseif logData.level == "Warning" then
                logLabel.TextColor3 = self.style.warningColor
            elseif logData.level == "Success" then
                logLabel.TextColor3 = self.style.successColor
            else
                logLabel.TextColor3 = self.style.infoColor
            end
            
            logIndex = logIndex + 1
        end
    end
    
    -- Ocultar etiquetas no usadas
    for i = logIndex, #self.ui.logTexts do
        self.ui.logTexts[i].Visible = false
    end
    
    -- Ajustar tamaño del canvas
    local layout = self.ui.logContainer:FindFirstChildOfClass("UIListLayout")
    if layout then
        self.ui.panels["Logs"].CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + self.dimensions.padding)
    end
end

-- Actualizar estadísticas en la UI (implementación simplificada)
function ResponsiveUI:updateStats(learningStats, executorStats)
    if not self.ui.statLabels then return end
    
    local function formatNum(num)
        if type(num) == "number" then
            return string.format("%.2f", num)
        end
        return tostring(num)
    end
    
    if learningStats then
        self.ui.statLabels["Iteraciones"].Text = "Iteraciones: " .. (learningStats.iterations or "N/A")
        self.ui.statLabels["Tasa Éxito"].Text = "Tasa Éxito: " .. formatNum(learningStats.performanceMetrics and learningStats.performanceMetrics.successRate or "N/A")
        self.ui.statLabels["Recompensa Promedio"].Text = "Recompensa Promedio: " .. formatNum(learningStats.performanceMetrics and learningStats.performanceMetrics.avgRewardPerAction or "N/A")
        self.ui.statLabels["Tasa Exploración"].Text = "Tasa de Exploración: " .. formatNum(learningStats.explorationRate or "N/A")
        self.ui.statLabels["Tamaño Memoria"].Text = "Tamaño Memoria: " .. (learningStats.memorySize or "N/A")
        self.ui.statLabels["Tamaño Tabla Q"].Text = "Tamaño Tabla Q: " .. (learningStats.qTableSize or "N/A")
        self.ui.statLabels["Victorias"].Text = "Victorias: " .. (learningStats.wins or "N/A")
        self.ui.statLabels["Derrotas"].Text = "Derrotas: " .. (learningStats.losses or "N/A")
    end
    
    if executorStats then
        self.ui.statLabels["Acciones Totales"].Text = "Acciones Totales: " .. (executorStats.totalActions or "N/A")
        self.ui.statLabels["Tiempo Promedio Acción"].Text = "Tiempo Promedio Acción: " .. formatNum(executorStats.avgTimeBetweenActions or "N/A") .. "s"
    end
    
    -- Actualizar tiempo de ejecución
    if self.mainModule and self.mainModule.startTime then
        local runtime = os.time() - self.mainModule.startTime
        self.ui.statLabels["Runtime"].Text = "Tiempo de ejecución: " .. runtime .. "s"
    end
    
    -- Actualizar información del dispositivo
    if self.ui.statLabels["DeviceInfo"] then
        self.ui.statLabels["DeviceInfo"].Text = "Dispositivo: " .. self.platformDetection:getDeviceInfo().deviceType
    end
end



--==============================================================================
-- MÓDULO: UnifiedInput
--==============================================================================

local UnifiedInput = {}
UnifiedInput.__index = UnifiedInput

-- Acciones lógicas del juego
local ACTIONS = {
    JAB = "Jab",
    UPPERCUT = "Uppercut",
    HOOK = "Hook",
    BLOCK = "Block",
    CLINCH = "Clinch",
    DODGE_LEFT = "DodgeLeft",
    DODGE_RIGHT = "DodgeRight",
    DODGE_BACK = "DodgeBack",
    MOVE_FORWARD = "MoveForward",
    MOVE_BACKWARD = "MoveBackward",
    MOVE_LEFT = "MoveLeft",
    MOVE_RIGHT = "MoveRight",
    TOGGLE_AUTOPLAYER = "ToggleAutoplayer",
    SWITCH_TAB_NEXT = "SwitchTabNext",
    SWITCH_TAB_PREV = "SwitchTabPrev",
    UI_INTERACT = "UIInteract" -- Acción genérica para interactuar con la UI
}

function UnifiedInput.new(platformDetection)
    local self = setmetatable({}, UnifiedInput)
    
    self.platformDetection = platformDetection
    self.deviceInfo = platformDetection:getDeviceInfo()
    
    -- Estado actual de las acciones (activo/inactivo)
    self.actionStates = {}
    for _, actionName in pairs(ACTIONS) do
        self.actionStates[actionName] = false
    end
    
    -- Mapeo de inputs a acciones
    self.keyMapPC = {
        -- Ataques
        [Enum.UserInputType.MouseButton1] = ACTIONS.JAB,
        [Enum.KeyCode.Q] = ACTIONS.UPPERCUT,
        [Enum.KeyCode.E] = ACTIONS.HOOK,
        -- Defensa
        [Enum.KeyCode.Space] = ACTIONS.BLOCK,
        [Enum.KeyCode.B] = ACTIONS.CLINCH,
        -- Esquivas
        [Enum.KeyCode.A] = ACTIONS.DODGE_LEFT, -- Podría ser movimiento también
        [Enum.KeyCode.D] = ACTIONS.DODGE_RIGHT, -- Podría ser movimiento también
        [Enum.KeyCode.S] = ACTIONS.DODGE_BACK, -- Podría ser movimiento también
        -- Movimiento (alternativo si A/D/S no son esquivas)
        [Enum.KeyCode.W] = ACTIONS.MOVE_FORWARD,
        -- [Enum.KeyCode.S] = ACTIONS.MOVE_BACKWARD, 
        -- [Enum.KeyCode.A] = ACTIONS.MOVE_LEFT,
        -- [Enum.KeyCode.D] = ACTIONS.MOVE_RIGHT,
        -- UI / Control
        [Enum.KeyCode.F1] = ACTIONS.TOGGLE_AUTOPLAYER, -- Ejemplo
        [Enum.KeyCode.Tab] = ACTIONS.SWITCH_TAB_NEXT, -- Ejemplo
        [Enum.KeyCode.LeftShift] = ACTIONS.SWITCH_TAB_PREV -- Ejemplo (Shift+Tab)
    }
    
    -- Mapeo de gestos táctiles a acciones (se configurará en setupMobileControls)
    self.touchMapMobile = {}
    
    -- Conexiones de eventos
    self.connections = {}
    
    -- Configurar inputs según la plataforma
    self:setupInputEvents()
    
    return self
end

-- Configurar los eventos de input según la plataforma
function UnifiedInput:setupInputEvents()
    -- Limpiar conexiones anteriores
    for _, conn in pairs(self.connections) do
        conn:Disconnect()
    end
    local inputType = input.UserInputType
    local keyCode = input.KeyCode
    
    -- Buscar acción mapeada para PC/Consola
    local actionName = nil
    if inputType == Enum.UserInputType.MouseButton1 or inputType == Enum.UserInputType.MouseButton2 then
        actionName = self.keyMapPC[inputType]
    table.insert(self.connections, UserInputService.InputEnded:Connect(function(input, gameProcessedEvent)
        self:handleInputEnded(input, gameProcessedEvent)
    end))
    
    -- Eventos específicos de móvil
    if self.deviceInfo.isMobile then
        self:setupMobileControls()
    end
    
    -- Eventos específicos de consola (si se implementa)
    -- if self.deviceInfo.isConsole then
    --     self:setupGamepadControls()
    -- end
end

-- Configurar controles táctiles para móvil usando ContextActionService
function UnifiedInput:setupMobileControls()
    print("Configurando controles táctiles...")
    
    -- Desvincular acciones anteriores si existen
    for _, actionName in pairs(ACTIONS) do
        ContextActionService:UnbindAction(actionName .. "_Mobile")
    end
    
    -- Mapear acciones a funciones
    local function createActionHandler(actionName, activate)
        return function(boundActionName, inputState, inputObject)
            if inputState == Enum.UserInputState.Begin then
                self.actionStates[actionName] = activate
                print(actionName .. ": " .. tostring(activate))
            elseif inputState == Enum.UserInputState.End then
                 -- Para acciones momentáneas (como tap), desactivar inmediatamente
                 -- Para acciones mantenidas, esperar al InputEnded general
                 if activate == true and (actionName == ACTIONS.JAB or actionName == ACTIONS.UI_INTERACT) then 
                     self.actionStates[actionName] = false
                 end
            end
            return Enum.ContextActionResult.Pass -- Permitir que otros scripts procesen el input
        end
    end
    
    -- Ejemplo de mapeo táctil (simplificado, se necesita UI real para botones)
    -- Esto es conceptual. En la práctica, se necesitarían botones en pantalla
    -- a los que vincular estas acciones.
    
    -- Simulación: Tap en diferentes áreas de la pantalla
    -- ContextActionService:BindActionAtPriority(ACTIONS.JAB .. "_Mobile", createActionHandler(ACTIONS.JAB, true), true, Enum.ContextActionPriority.High.Value, Enum.UserInputType.Touch)
    
    -- Simulación: Swipe para esquivar
    -- ContextActionService:BindActionAtPriority(ACTIONS.DODGE_LEFT .. "_Mobile", createActionHandler(ACTIONS.DODGE_LEFT, true), true, Enum.ContextActionPriority.High.Value, Enum.SwipeDirection.Left)
    -- ContextActionService:BindActionAtPriority(ACTIONS.DODGE_RIGHT .. "_Mobile", createActionHandler(ACTIONS.DODGE_RIGHT, true), true, Enum.ContextActionPriority.High.Value, Enum.SwipeDirection.Right)
    
    -- Conectar eventos TouchTap y TouchSwipe para gestos más específicos
    table.insert(self.connections, UserInputService.TouchTap:Connect(function(touchPositions, gameProcessedEvent)
        if gameProcessedEvent then return end
        -- Determinar qué acción corresponde al tap (depende de la UI)
        -- Ejemplo: Si toca el botón de Jab
        -- self.actionStates[ACTIONS.JAB] = true
        -- wait(0.1) -- Simular duración corta
        -- self.actionStates[ACTIONS.JAB] = false
        print("Touch Tap detectado en: ", touchPositions[1])
        -- Aquí iría la lógica para mapear el tap a una acción (ej. JAB, UI_INTERACT)
        -- Por ahora, solo lo registramos
        self:triggerActionBriefly(ACTIONS.UI_INTERACT) -- Simular interacción con UI
    end))
    
    table.insert(self.connections, UserInputService.TouchSwipe:Connect(function(swipeDirection, numberOfTouches, gameProcessedEvent)
        if gameProcessedEvent then return end
        print("Touch Swipe detectado: ", swipeDirection)
        if numberOfTouches == 1 then
            if swipeDirection == Enum.SwipeDirection.Left then
                self:triggerActionBriefly(ACTIONS.DODGE_LEFT)
            elseif swipeDirection == Enum.SwipeDirection.Right then
                self:triggerActionBriefly(ACTIONS.DODGE_RIGHT)
            elseif swipeDirection == Enum.SwipeDirection.Down then
                 self:triggerActionBriefly(ACTIONS.DODGE_BACK)
            elseif swipeDirection == Enum.SwipeDirection.Up then
                 self:triggerActionBriefly(ACTIONS.MOVE_FORWARD)
            end
        elseif numberOfTouches == 2 then -- Swipe con dos dedos para cambiar pestañas
             if swipeDirection == Enum.SwipeDirection.Left then
                self:triggerActionBriefly(ACTIONS.SWITCH_TAB_PREV)
            elseif swipeDirection == Enum.SwipeDirection.Right then
                self:triggerActionBriefly(ACTIONS.SWITCH_TAB_NEXT)
            end
        end
    end))
    
    -- Otros gestos (Pinch, LongPress) se pueden conectar de forma similar si es necesario
end

-- Manejar inicio de input (teclado, mouse, gamepad)
function UnifiedInput:handleInputBegan(input, gameProcessedEvent)
    if gameProcessedEvent then return end -- Ignorar si el juego ya lo procesó (ej. chat)
    
    local inputType = input.UserInputType
    local keyCode = input.KeyCode
    
    -- Buscar acción mapeada para PC/Consola
    local actionName = nil
    if inputType == Enum.UserInputType.MouseButton1 or inputType == Enum.UserInputType.MouseButton2 then
        actionName = self.keyMapPC[inputType]
    elseif inputType == Enum.UserInputType.Keyboard then
        actionName = self.keyMapPC[keyCode]
    elseif inputType == Enum.UserInputType.Gamepad1 then
        -- actionName = self.gamepadMap[keyCode] -- Si se implementa gamepad
    end
    
    if actionName then
        self.actionStates[actionName] = true
        -- print("Input Began: " .. actionName)
    end
end

-- Manejar fin de input (teclado, mouse, gamepad)
function UnifiedInput:handleInputEnded(input, gameProcessedEvent)
    -- gameProcessedEvent no aplica a InputEnded
    
    local inputType = input.UserInputType
    local keyCode = input.KeyCode
    
    -- Buscar acción mapeada para PC/Consola
    local actionName = nil
    if inputType == Enum.UserInputType.MouseButton1 or inputType == Enum.UserInputType.MouseButton2 then
        actionName = self.keyMapPC[inputType]
    elseif inputType == Enum.UserInputType.Keyboard then
        actionName = self.keyMapPC[keyCode]
    elseif inputType == Enum.UserInputType.Gamepad1 then
        -- actionName = self.gamepadMap[keyCode] -- Si se implementa gamepad
    end
    
    if actionName then
        self.actionStates[actionName] = false
        -- print("Input Ended: " .. actionName)
    end
end

-- Verificar si una acción específica está activa
function UnifiedInput:isActionActive(actionName)
    return self.actionStates[actionName] or false
end

-- Obtener el estado de todas las acciones
function UnifiedInput:getActionStates()
    -- Devolver una copia para evitar modificaciones externas
    local statesCopy = {}
    for name, isActive in pairs(self.actionStates) do
        statesCopy[name] = isActive
    end
    return statesCopy
end

-- Activar una acción por un corto período (para gestos como tap/swipe)
function UnifiedInput:triggerActionBriefly(actionName, duration)
    if not self.actionStates[actionName] then return end -- Verificar si la acción existe
    
    duration = duration or 0.1 -- Duración por defecto
    
    self.actionStates[actionName] = true
    print("Triggered Briefly: " .. actionName)
    
    -- Desactivar después de la duración
    delay(duration, function()
        self.actionStates[actionName] = false
        print("Ended Briefly: " .. actionName)
    end)
end

-- Limpiar conexiones al destruir
function UnifiedInput:destroy()
    for _, conn in pairs(self.connections) do
        conn:Disconnect()
    end
    self.connections = {}
    
    -- Desvincular acciones de ContextActionService
    for _, actionName in pairs(ACTIONS) do
        ContextActionService:UnbindAction(actionName .. "_Mobile")
    end
    print("Sistema de Input Unificado destruido.")
end

-- Devolver las constantes de acciones para uso externo
UnifiedInput.ACTIONS = ACTIONS

--==============================================================================
-- MÓDULO: TouchControls
--==============================================================================

local TouchControls = {}
TouchControls.__index = TouchControls

-- Constantes para gestos
local GESTURE_TYPES = {
    TAP = "Tap",
    DOUBLE_TAP = "DoubleTap",
    SWIPE = "Swipe",
    PINCH = "Pinch",
    LONG_PRESS = "LongPress",
    PAN = "Pan"
}

local SWIPE_DIRECTIONS = {
    UP = "Up",
    DOWN = "Down",
    LEFT = "Left",
    RIGHT = "Right"
}

function TouchControls.new(platformDetection, unifiedInput)
    local self = setmetatable({}, TouchControls)
    
    -- Referencias a otros sistemas
    self.platformDetection = platformDetection
    self.unifiedInput = unifiedInput
    
    -- Verificar si el dispositivo es táctil
    self.deviceInfo = platformDetection:getDeviceInfo()
    self.isTouchDevice = self.deviceInfo.touchEnabled
    
    -- Si no es un dispositivo táctil, no inicializar
    if not self.isTouchDevice then
        print("No es un dispositivo táctil. Controles táctiles no inicializados.")
        return self
    end
    
    -- Configuración de gestos
    self.config = {
        doubleTapMaxDelay = 0.3, -- segundos entre taps para considerarse double tap
        longPressMinDuration = 0.5, -- segundos para considerar long press
        swipeMinDistance = 50, -- píxeles mínimos para considerar swipe
        swipeMaxTime = 0.5, -- tiempo máximo para considerar swipe
        pinchMinChange = 30, -- cambio mínimo en distancia para considerar pinch
        tapMaxMovement = 10, -- movimiento máximo permitido para considerar tap
        controlsVisible = true -- visibilidad de los controles en pantalla
    }
    
    -- Estado de los gestos
    self.gestureState = {
        lastTapTime = 0,
        lastTapPosition = Vector2.new(0, 0),
        touchStartTime = 0,
        touchStartPositions = {},
        touchCurrentPositions = {},
        activeTouches = {},
        isLongPressing = false,
        isPanning = false,
        isPinching = false,
        initialPinchDistance = 0
    }
    
    -- Elementos de UI para controles táctiles
    self.ui = {
        screenGui = nil,
        controlsFrame = nil,
        buttons = {},
        joystick = nil,
        gestureArea = nil
    }
    
    -- Callbacks para gestos
    self.callbacks = {
        [GESTURE_TYPES.TAP] = {},
        [GESTURE_TYPES.DOUBLE_TAP] = {},
        [GESTURE_TYPES.SWIPE] = {},
        [GESTURE_TYPES.PINCH] = {},
        [GESTURE_TYPES.LONG_PRESS] = {},
        [GESTURE_TYPES.PAN] = {}
    }
    
    -- Conexiones de eventos
    self.connections = {}
    
    -- Inicializar controles táctiles
    self:initialize()
    
    return self
end

-- Inicializar controles táctiles
function TouchControls:initialize()
    if not self.isTouchDevice then return end
    
    print("Inicializando controles táctiles...")
    
    -- Crear UI de controles táctiles
    self:createTouchUI()
    
    -- Configurar eventos táctiles
    self:setupTouchEvents()
    
    -- Registrar callbacks predeterminados
    self:registerDefaultCallbacks()
    
    print("Controles táctiles inicializados.")
end

-- Crear UI de controles táctiles
function TouchControls:createTouchUI()
    -- Limpiar UI anterior si existe
    if self.ui.screenGui then
        self.ui.screenGui:Destroy()
    end
    
    -- Crear ScreenGui para controles táctiles
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "BoxingBetaTouchControls"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    screenGui.DisplayOrder = 10 -- Por encima de la UI principal
    self.ui.screenGui = screenGui
    
    -- Obtener tamaño de pantalla
    local screenSize = self.deviceInfo.screenSize
    
    -- Crear marco principal para controles
    local controlsFrame = Instance.new("Frame")
    controlsFrame.Name = "ControlsFrame"
    controlsFrame.Size = UDim2.new(1, 0, 1, 0)
    controlsFrame.BackgroundTransparency = 1
    controlsFrame.Visible = self.config.controlsVisible
    controlsFrame.Parent = screenGui
    self.ui.controlsFrame = controlsFrame
    
    -- Crear área para gestos (cubre toda la pantalla)
    local gestureArea = Instance.new("Frame")
    gestureArea.Name = "GestureArea"
    gestureArea.Size = UDim2.new(1, 0, 1, 0)
    gestureArea.BackgroundTransparency = 1 -- Invisible pero detecta inputs
    gestureArea.Parent = controlsFrame
    self.ui.gestureArea = gestureArea
    
    -- Crear botones táctiles para acciones principales
    self:createActionButtons()
    
    -- Crear joystick virtual para movimiento
    self:createVirtualJoystick()
    
    -- Añadir ScreenGui al PlayerGui
    local player = game:GetService("Players").LocalPlayer
    if player and player:FindFirstChild("PlayerGui") then
        screenGui.Parent = player.PlayerGui
    else
        -- Fallback si no se encuentra PlayerGui
        screenGui.Parent = game:GetService("CoreGui")
    end
end

-- Crear botones táctiles para acciones principales
function TouchControls:createActionButtons()
    -- Obtener tamaño de pantalla
    local screenSize = self.deviceInfo.screenSize
    local buttonSize = math.min(screenSize.X, screenSize.Y) * 0.12 -- 12% del lado más pequeño
    local padding = buttonSize * 0.2
    
    -- Definir botones de acción
    local actionButtons = {
        {name = "JabButton", text = "JAB", action = self.unifiedInput.ACTIONS.JAB, 
         position = UDim2.new(1, -buttonSize*2.2, 1, -buttonSize*2.2), color = Color3.fromRGB(255, 100, 100)},
        {name = "UppercutButton", text = "UP", action = self.unifiedInput.ACTIONS.UPPERCUT, 
         position = UDim2.new(1, -buttonSize*1.1, 1, -buttonSize*3.3), color = Color3.fromRGB(100, 100, 255)},
        {name = "HookButton", text = "HOOK", action = self.unifiedInput.ACTIONS.HOOK, 
         position = UDim2.new(1, -buttonSize*3.3, 1, -buttonSize*1.1), color = Color3.fromRGB(100, 255, 100)},
        {name = "BlockButton", text = "BLOCK", action = self.unifiedInput.ACTIONS.BLOCK, 
         position = UDim2.new(0, padding, 1, -buttonSize-padding), color = Color3.fromRGB(255, 255, 100)}
    }
    
    -- Crear cada botón
    for _, buttonInfo in ipairs(actionButtons) do
        local button = Instance.new("TextButton")
        button.Name = buttonInfo.name
        button.Size = UDim2.new(0, buttonSize, 0, buttonSize)
        button.Position = buttonInfo.position
        button.BackgroundColor3 = buttonInfo.color
        button.BackgroundTransparency = 0.5
        button.Text = buttonInfo.text
        button.TextColor3 = Color3.fromRGB(255, 255, 255)
        button.TextSize = buttonSize * 0.3
        button.Font = Enum.Font.SourceSansBold
        button.Parent = self.ui.controlsFrame
        
        -- Añadir esquinas redondeadas
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0.5, 0) -- Botón circular
        corner.Parent = button
        
        -- Guardar referencia al botón
        self.ui.buttons[buttonInfo.name] = button
        
        -- Conectar eventos del botón
        button.MouseButton1Down:Connect(function()
            self.unifiedInput.actionStates[buttonInfo.action] = true
            button.BackgroundTransparency = 0.2 -- Efecto visual de presionado
        end)
        
        button.MouseButton1Up:Connect(function()
            self.unifiedInput.actionStates[buttonInfo.action] = false
            button.BackgroundTransparency = 0.5 -- Restaurar transparencia
        end)
        
        -- Manejar caso en que el dedo se desliza fuera del botón
        button.MouseLeave:Connect(function()
            self.unifiedInput.actionStates[buttonInfo.action] = false
            button.BackgroundTransparency = 0.5
        end)
    end
end

-- Crear joystick virtual para movimiento
function TouchControls:createVirtualJoystick()
    -- Obtener tamaño de pantalla
    local screenSize = self.deviceInfo.screenSize
    local joystickSize = math.min(screenSize.X, screenSize.Y) * 0.15 -- 15% del lado más pequeño
    local padding = joystickSize * 0.5
    
    -- Crear base del joystick
    local joystickBase = Instance.new("Frame")
    joystickBase.Name = "JoystickBase"
    joystickBase.Size = UDim2.new(0, joystickSize, 0, joystickSize)
    joystickBase.Position = UDim2.new(0, padding, 1, -joystickSize-padding)
    joystickBase.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
    joystickBase.BackgroundTransparency = 0.7
    joystickBase.Parent = self.ui.controlsFrame
    
    -- Añadir esquinas redondeadas a la base
    local baseCorner = Instance.new("UICorner")
    baseCorner.CornerRadius = UDim.new(0.5, 0) -- Circular
    baseCorner.Parent = joystickBase
    
    -- Crear stick del joystick
    local joystickStick = Instance.new("Frame")
    joystickStick.Name = "JoystickStick"
    joystickStick.Size = UDim2.new(0, joystickSize * 0.5, 0, joystickSize * 0.5)
    joystickStick.Position = UDim2.new(0.5, -joystickSize * 0.25, 0.5, -joystickSize * 0.25)
    joystickStick.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
    joystickStick.BackgroundTransparency = 0.5
    joystickStick.Parent = joystickBase
    
    -- Añadir esquinas redondeadas al stick
    local stickCorner = Instance.new("UICorner")
    stickCorner.CornerRadius = UDim.new(0.5, 0) -- Circular
    stickCorner.Parent = joystickStick
    
    -- Guardar referencias
    self.ui.joystick = {
        base = joystickBase,
        stick = joystickStick,
        defaultPosition = joystickStick.Position,
        radius = joystickSize * 0.25, -- Radio máximo de movimiento del stick
        active = false,
        touchId = nil
    }
    
    -- Conectar eventos del joystick
    joystickBase.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            self.ui.joystick.active = true
            self.ui.joystick.touchId = input.Touch.TouchId
            self:updateJoystickPosition(input.Position)
        end
    end)
    
    -- Manejar movimiento del joystick
    self:connectJoystickMovement()
    
    -- Manejar fin de input del joystick
    joystickBase.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch and 
           input.Touch.TouchId == self.ui.joystick.touchId then
            self:resetJoystick()
        end
    end)
end

-- Conectar eventos para manejar movimiento del joystick
function TouchControls:connectJoystickMovement()
    -- Usar RenderStepped para actualización suave
    table.insert(self.connections, RunService.RenderStepped:Connect(function()
        if not self.ui.joystick.active then return end
        
        -- Buscar el touch actual
        for _, touch in pairs(UserInputService:GetTouches()) do
            if touch.TouchId == self.ui.joystick.touchId then
                self:updateJoystickPosition(touch.Position)
                break
            end
        end
    end))
end

-- Actualizar posición del joystick y mapear a acciones de movimiento
function TouchControls:updateJoystickPosition(touchPosition)
    if not self.ui.joystick.active then return end
    
    local base = self.ui.joystick.base
    local stick = self.ui.joystick.stick
    local radius = self.ui.joystick.radius
    
    -- Convertir posición de toque a posición relativa a la base
    local basePosition = base.AbsolutePosition
    local baseSize = base.AbsoluteSize
    local baseCenter = basePosition + baseSize/2
    
    -- Calcular vector desde el centro de la base hasta la posición del toque
    local touchVector = touchPosition - baseCenter
    
    -- Limitar la distancia al radio máximo
    local distance = touchVector.Magnitude
    local direction = touchVector.Unit
    local clampedDistance = math.min(distance, radius)
    local finalVector = direction * clampedDistance
    
    -- Actualizar posición del stick
    stick.Position = UDim2.new(0.5, finalVector.X, 0.5, finalVector.Y)
    
    -- Mapear posición a acciones de movimiento
    local normalizedX = finalVector.X / radius -- -1 a 1
    local normalizedY = finalVector.Y / radius -- -1 a 1
    
    -- Umbral para activar movimiento
    local threshold = 0.3
    
    -- Actualizar estados de acción
    self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.MOVE_FORWARD] = (normalizedY < -threshold)
    self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.MOVE_BACKWARD] = (normalizedY > threshold)
    self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.MOVE_LEFT] = (normalizedX < -threshold)
    self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.MOVE_RIGHT] = (normalizedX > threshold)
end

-- Resetear joystick a posición por defecto
function TouchControls:resetJoystick()
    if not self.ui.joystick then return end
    
    self.ui.joystick.active = false
    self.ui.joystick.touchId = nil
    self.ui.joystick.stick.Position = self.ui.joystick.defaultPosition
    
    -- Resetear estados de movimiento
    self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.MOVE_FORWARD] = false
    self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.MOVE_BACKWARD] = false
    self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.MOVE_LEFT] = false
    self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.MOVE_RIGHT] = false
end

-- Configurar eventos táctiles para gestos
function TouchControls:setupTouchEvents()
    -- Limpiar conexiones anteriores
    for _, conn in pairs(self.connections) do
        conn:Disconnect()
    end
    self.connections = {}
    
    -- Conectar eventos táctiles básicos
    table.insert(self.connections, UserInputService.TouchStarted:Connect(function(touch, gameProcessedEvent)
        if gameProcessedEvent then return end
        self:handleTouchStarted(touch)
    end))
    
    table.insert(self.connections, UserInputService.TouchMoved:Connect(function(touch, gameProcessedEvent)
        if gameProcessedEvent then return end
        self:handleTouchMoved(touch)
    end))
    
    table.insert(self.connections, UserInputService.TouchEnded:Connect(function(touch, gameProcessedEvent)
        if gameProcessedEvent then return end
        self:handleTouchEnded(touch)
    end))
    
    -- Conectar eventos de gestos específicos
    table.insert(self.connections, UserInputService.TouchTap:Connect(function(touchPositions, gameProcessedEvent)
        if gameProcessedEvent then return end
        self:handleTouchTap(touchPositions)
    end))
    
    table.insert(self.connections, UserInputService.TouchSwipe:Connect(function(swipeDirection, numberOfTouches, gameProcessedEvent)
        if gameProcessedEvent then return end
        self:handleTouchSwipe(swipeDirection, numberOfTouches)
    end))
    
    -- Usar RenderStepped para detectar gestos continuos (pinch, pan, long press)
    table.insert(self.connections, RunService.RenderStepped:Connect(function()
        self:updateContinuousGestures()
    end))
end

-- Manejar inicio de toque
function TouchControls:handleTouchStarted(touch)
    local touchId = touch.TouchId
    local position = touch.Position
    local currentTime = tick()
    
    -- Registrar información del toque
    self.gestureState.touchStartTime = currentTime
    self.gestureState.touchStartPositions[touchId] = position
    self.gestureState.touchCurrentPositions[touchId] = position
    self.gestureState.activeTouches[touchId] = true
    
    -- Verificar si es potencialmente un double tap
    if currentTime - self.gestureState.lastTapTime < self.config.doubleTapMaxDelay then
        local lastPos = self.gestureState.lastTapPosition
        local distance = (position - lastPos).Magnitude
        
        if distance < self.config.tapMaxMovement * 2 then
            -- Es un double tap
            self:triggerGesture(GESTURE_TYPES.DOUBLE_TAP, {
                position = position,
                lastPosition = lastPos,
                timeBetweenTaps = currentTime - self.gestureState.lastTapTime
            })
        end
    end
    
    -- Verificar inicio de long press
    spawn(function()
        local touchStillActive = true
        local startPos = position
        local startTime = currentTime
        
        -- Esperar duración mínima para long press
        wait(self.config.longPressMinDuration)
        
        -- Verificar si el toque sigue activo y no se ha movido mucho
        if self.gestureState.activeTouches[touchId] then
            local currentPos = self.gestureState.touchCurrentPositions[touchId]
            local distance = (currentPos - startPos).Magnitude
            
            if distance < self.config.tapMaxMovement then
                -- Iniciar long press
                self.gestureState.isLongPressing = true
                
                -- Disparar evento de inicio de long press
                self:triggerGesture(GESTURE_TYPES.LONG_PRESS, {
                    position = currentPos,
                    startPosition = startPos,
                    duration = tick() - startTime,
                    ended = false
                })
                
                -- Esperar a que termine el toque
                while self.gestureState.activeTouches[touchId] do
                    wait(0.05)
                end
                
                -- Disparar evento de fin de long press
                self:triggerGesture(GESTURE_TYPES.LONG_PRESS, {
                    position = self.gestureState.touchCurrentPositions[touchId] or currentPos,
                    startPosition = startPos,
                    duration = tick() - startTime,
                    ended = true
                })
                
                self.gestureState.isLongPressing = false
            end
        end
    end)
end

-- Manejar movimiento de toque
function TouchControls:handleTouchMoved(touch)
    local touchId = touch.TouchId
    local position = touch.Position
    local startPosition = self.gestureState.touchStartPositions[touchId]
    
    if not startPosition then return end
    
    -- Actualizar posición actual
    self.gestureState.touchCurrentPositions[touchId] = position
    
    -- Verificar inicio de pinch si hay exactamente 2 toques activos
    if not self.gestureState.isPinching then
        self:checkPinchStart()
    end
end

-- Manejar fin de toque
function TouchControls:handleTouchEnded(touch)
    local touchId = touch.TouchId
    local position = touch.Position
    local startPosition = self.gestureState.touchStartPositions[touchId]
    local startTime = self.gestureState.touchStartTime
    
    if not startPosition then return end
    
    local currentTime = tick()
    local duration = currentTime - startTime
    local distance = (position - startPosition).Magnitude
    
    -- Verificar si es un tap simple (no double tap, no long press)
    if distance < self.config.tapMaxMovement and duration < self.config.longPressMinDuration and not self.gestureState.isLongPressing then
        -- Registrar para potencial double tap
        self.gestureState.lastTapTime = currentTime
        self.gestureState.lastTapPosition = position
        
        -- Disparar evento de tap
        self:triggerGesture(GESTURE_TYPES.TAP, {
            position = position,
            duration = duration
        })
    end
    
    -- Verificar fin de pinch
    self:checkPinchEnd()
    
    -- Limpiar estado del toque
    self.gestureState.activeTouches[touchId] = nil
    self.gestureState.touchStartPositions[touchId] = nil
    self.gestureState.touchCurrentPositions[touchId] = nil
end

-- Manejar evento de tap
function TouchControls:handleTouchTap(touchPositions)
    -- UserInputService ya detectó un tap, podemos usarlo directamente
    self:triggerGesture(GESTURE_TYPES.TAP, {
        position = touchPositions[1],
        timestamp = tick()
    })
end

-- Manejar evento de swipe
function TouchControls:handleTouchSwipe(swipeDirection, numberOfTouches)
    -- Convertir dirección de swipe a formato interno
    local direction
    if swipeDirection == Enum.SwipeDirection.Up then
        direction = SWIPE_DIRECTIONS.UP
    elseif swipeDirection == Enum.SwipeDirection.Down then
        direction = SWIPE_DIRECTIONS.DOWN
    elseif swipeDirection == Enum.SwipeDirection.Left then
        direction = SWIPE_DIRECTIONS.LEFT
    elseif swipeDirection == Enum.SwipeDirection.Right then
        direction = SWIPE_DIRECTIONS.RIGHT
    end
    
    -- UserInputService ya detectó un swipe, podemos usarlo directamente
    self:triggerGesture(GESTURE_TYPES.SWIPE, {
        direction = direction,
        numberOfTouches = numberOfTouches
    })
end

-- Verificar inicio de pinch
function TouchControls:checkPinchStart()
    -- Necesitamos exactamente 2 toques activos para pinch
    local activeTouchCount = 0
    local touchIds = {}
    
    for id, active in pairs(self.gestureState.activeTouches) do
        if active then
            activeTouchCount = activeTouchCount + 1
            table.insert(touchIds, id)
        end
    end
    
    if activeTouchCount == 2 then
        -- Calcular distancia inicial entre los dos toques
        local pos1 = self.gestureState.touchCurrentPositions[touchIds[1]]
        local pos2 = self.gestureState.touchCurrentPositions[touchIds[2]]
        
        if pos1 and pos2 then
            self.gestureState.isPinching = true
            self.gestureState.pinchTouchIds = touchIds
            self.gestureState.initialPinchDistance = (pos2 - pos1).Magnitude
            self.gestureState.lastPinchDistance = self.gestureState.initialPinchDistance
        end
    end
end

-- Actualizar gesto de pinch
function TouchControls:updatePinchGesture()
    if not self.gestureState.isPinching then return end
    
    local touchIds = self.gestureState.pinchTouchIds
    if not touchIds or #touchIds ~= 2 then return end
    
    -- Verificar que ambos toques sigan activos
    if not (self.gestureState.activeTouches[touchIds[1]] and self.gestureState.activeTouches[touchIds[2]]) then
        self.gestureState.isPinching = false
        return
    end
    
    -- Calcular distancia actual entre los dos toques
    local pos1 = self.gestureState.touchCurrentPositions[touchIds[1]]
    local pos2 = self.gestureState.touchCurrentPositions[touchIds[2]]
    
    if pos1 and pos2 then
        local currentDistance = (pos2 - pos1).Magnitude
        local delta = currentDistance - self.gestureState.lastPinchDistance
        
        -- Si el cambio es significativo, trigger el gesto
        if math.abs(delta) > 1 then
            local scale = currentDistance / self.gestureState.initialPinchDistance
            local center = pos1:Lerp(pos2, 0.5) -- Punto medio entre los dos toques
            
            self:triggerGesture(GESTURE_TYPES.PINCH, {
                center = center,
                scale = scale,
                delta = delta,
                distance = currentDistance,
                touchIds = touchIds
            })
            
            self.gestureState.lastPinchDistance = currentDistance
        end
    end
end

-- Verificar fin de pinch
function TouchControls:checkPinchEnd()
    if not self.gestureState.isPinching then return end
    
    local touchIds = self.gestureState.pinchTouchIds
    if not touchIds or #touchIds ~= 2 then return end
    
    -- Verificar si alguno de los toques ha terminado
    if not (self.gestureState.activeTouches[touchIds[1]] and self.gestureState.activeTouches[touchIds[2]]) then
        self.gestureState.isPinching = false
    end
end

-- Actualizar gestos continuos (llamado en RenderStepped)
function TouchControls:updateContinuousGestures()
    -- Actualizar pinch si está activo
    if self.gestureState.isPinching then
        self:updatePinchGesture()
    end
    
    -- Otros gestos continuos se pueden actualizar aquí
end

-- Disparar un gesto y llamar a los callbacks registrados
function TouchControls:triggerGesture(gestureType, gestureData)
    -- Añadir timestamp al gesto
    gestureData.timestamp = tick()
    
    -- Llamar a todos los callbacks registrados para este tipo de gesto
    for _, callback in ipairs(self.callbacks[gestureType] or {}) do
        callback(gestureData)
    end
    
    -- Debug
    -- print("Gesto detectado: " .. gestureType)
    -- for k, v in pairs(gestureData) do
    --     if type(v) ~= "userdata" then -- Evitar imprimir objetos complejos
    --         print("  " .. k .. ": " .. tostring(v))
    --     end
    -- end
end

-- Registrar un callback para un tipo de gesto
function TouchControls:registerGestureCallback(gestureType, callback)
    if not self.callbacks[gestureType] then
        self.callbacks[gestureType] = {}
    end
    
    table.insert(self.callbacks[gestureType], callback)
    return #self.callbacks[gestureType] -- Devolver ID para poder eliminar después
end

-- Eliminar un callback registrado
function TouchControls:unregisterGestureCallback(gestureType, callbackId)
    if self.callbacks[gestureType] and callbackId <= #self.callbacks[gestureType] then
        table.remove(self.callbacks[gestureType], callbackId)
        return true
    end
    return false
end

-- Registrar callbacks predeterminados para mapear gestos a acciones
function TouchControls:registerDefaultCallbacks()
    -- Tap para interactuar con UI
    self:registerGestureCallback(GESTURE_TYPES.TAP, function(data)
        self.unifiedInput:triggerActionBriefly(self.unifiedInput.ACTIONS.UI_INTERACT)
    end)
    
    -- Double tap para toggle autoplayer
    self:registerGestureCallback(GESTURE_TYPES.DOUBLE_TAP, function(data)
        self.unifiedInput:triggerActionBriefly(self.unifiedInput.ACTIONS.TOGGLE_AUTOPLAYER)
    end)
    
    -- Swipe para esquivar
    self:registerGestureCallback(GESTURE_TYPES.SWIPE, function(data)
        if data.direction == SWIPE_DIRECTIONS.LEFT then
            self.unifiedInput:triggerActionBriefly(self.unifiedInput.ACTIONS.DODGE_LEFT)
        elseif data.direction == SWIPE_DIRECTIONS.RIGHT then
            self.unifiedInput:triggerActionBriefly(self.unifiedInput.ACTIONS.DODGE_RIGHT)
        elseif data.direction == SWIPE_DIRECTIONS.DOWN then
            self.unifiedInput:triggerActionBriefly(self.unifiedInput.ACTIONS.DODGE_BACK)
        elseif data.direction == SWIPE_DIRECTIONS.UP then
            self.unifiedInput:triggerActionBriefly(self.unifiedInput.ACTIONS.MOVE_FORWARD)
        end
    end)
    
    -- Pinch para escalar UI
    self:registerGestureCallback(GESTURE_TYPES.PINCH, function(data)
        -- Implementar escalado de UI si es necesario
    end)
    
    -- Long press para bloquear
    self:registerGestureCallback(GESTURE_TYPES.LONG_PRESS, function(data)
        if not data.ended then
            self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.BLOCK] = true
        else
            self.unifiedInput.actionStates[self.unifiedInput.ACTIONS.BLOCK] = false
        end
    end)
end

-- Mostrar/ocultar controles táctiles
function TouchControls:setControlsVisible(visible)
    self.config.controlsVisible = visible
    if self.ui.controlsFrame then
        self.ui.controlsFrame.Visible = visible
    end
end

-- Limpiar conexiones y destruir UI
function TouchControls:destroy()
    -- Limpiar conexiones
    for _, conn in pairs(self.connections) do
        conn:Disconnect()
    end
    self.connections = {}
    
    -- Destruir UI
    if self.ui.screenGui then
        self.ui.screenGui:Destroy()
    end
    
    print("Controles táctiles destruidos.")
end



--==============================================================================
-- MÓDULO: PerformanceManager
--==============================================================================

local PerformanceManager = {}
PerformanceManager.__index = PerformanceManager

function PerformanceManager.new(platformDetection, adaptiveConfig)
    local self = setmetatable({}, PerformanceManager)
    
    -- Referencias a otros sistemas
    self.platformDetection = platformDetection
    self.adaptiveConfig = adaptiveConfig
    
    -- Información del dispositivo
    self.deviceInfo = platformDetection:getDeviceInfo()
    
    -- Configuración de rendimiento
    self.config = {
        enabled = true,
        targetFPS = 60,
        minFPS = 20,
        adjustmentInterval = 5, -- segundos
        adjustmentFactor = 0.1, -- Cuánto ajustar cada vez
        
        -- Parámetros ajustables
        adjustableParams = {
            updateInterval = {min = 0.05, max = 0.2, current = adaptiveConfig:getConfig().updateInterval},
            screenAnalysisInterval = {min = 0.05, max = 0.25, current = adaptiveConfig:getConfig().screenAnalysisInterval},
            maxMemorySize = {min = 500, max = 2000, current = adaptiveConfig:getConfig().maxMemorySize},
            batchSize = {min = 16, max = 64, current = adaptiveConfig:getConfig().batchSize},
            -- Añadir más parámetros si es necesario
        }
    }
    
    -- Estado de rendimiento
    self.state = {
        currentFPS = 60,
        averageFPS = 60,
        lastAdjustmentTime = 0,
        performanceLevel = "Optimal" -- Optimal, Stressed, Critical
    }
    
    -- Conexiones de eventos
    self.connections = {}
    
    -- Inicializar
    self:initialize()
    
    return self
end

-- Inicializar el gestor de rendimiento
function PerformanceManager:initialize()
    if not self.config.enabled then return end
    
    print("Inicializando gestor de rendimiento...")
    
    -- Conectar evento para monitorear FPS
    self:setupFPSTracking()
    
    -- Iniciar ciclo de ajuste
    self:startAdjustmentCycle()
    
    print("Gestor de rendimiento inicializado.")
end

-- Configurar seguimiento de FPS
function PerformanceManager:setupFPSTracking()
    -- Usar Stats service para obtener FPS
    local fpsHistory = {}
    local maxHistory = 30 -- Promediar sobre los últimos 30 frames
    
    table.insert(self.connections, RunService.RenderStepped:Connect(function(deltaTime)
        local currentFPS = 1 / deltaTime
        self.state.currentFPS = currentFPS
        
        -- Actualizar historial de FPS
        table.insert(fpsHistory, currentFPS)
        if #fpsHistory > maxHistory then
            table.remove(fpsHistory, 1)
        end
        
        -- Calcular FPS promedio
        local sum = 0
        for _, fps in ipairs(fpsHistory) do
            sum = sum + fps
        end
        self.state.averageFPS = sum / #fpsHistory
        
        -- Determinar nivel de rendimiento
        if self.state.averageFPS < self.config.minFPS then
            self.state.performanceLevel = "Critical"
        elseif self.state.averageFPS < self.config.targetFPS * 0.8 then
            self.state.performanceLevel = "Stressed"
        else
            self.state.performanceLevel = "Optimal"
        end
    end))
end

-- Iniciar ciclo de ajuste de rendimiento
function PerformanceManager:startAdjustmentCycle()
    spawn(function()
        while self.config.enabled do
            wait(self.config.adjustmentInterval)
            
            local currentTime = tick()
            if currentTime - self.state.lastAdjustmentTime >= self.config.adjustmentInterval then
                self:adjustPerformance()
                self.state.lastAdjustmentTime = currentTime
            end
        end
    end)
end

-- Ajustar parámetros según el rendimiento actual
function PerformanceManager:adjustPerformance()
    local currentLevel = self.state.performanceLevel
    local adjustmentFactor = self.config.adjustmentFactor
    
    print("Ajustando rendimiento... Nivel actual: " .. currentLevel .. ", FPS Promedio: " .. string.format("%.1f", self.state.averageFPS))
    
    -- Ajustar parámetros según el nivel
    for paramName, paramInfo in pairs(self.config.adjustableParams) do
        local currentValue = paramInfo.current
        local newValue = currentValue
        
        if currentLevel == "Critical" then
            -- Reducir calidad/aumentar intervalos agresivamente
            newValue = currentValue * (1 + adjustmentFactor * 2)
        elseif currentLevel == "Stressed" then
            -- Reducir calidad/aumentar intervalos moderadamente
            newValue = currentValue * (1 + adjustmentFactor)
        elseif currentLevel == "Optimal" then
            -- Intentar aumentar calidad/reducir intervalos ligeramente
            newValue = currentValue * (1 - adjustmentFactor * 0.5)
        end
        
        -- Asegurar que el nuevo valor esté dentro de los límites
        newValue = math.clamp(newValue, paramInfo.min, paramInfo.max)
        
        -- Si el valor cambió, actualizarlo
        if math.abs(newValue - currentValue) > 0.001 then
            paramInfo.current = newValue
            
            -- Actualizar la configuración en AdaptiveConfig
            self.adaptiveConfig:updateConfigValue(paramName, newValue)
            
            print(string.format("  Ajustado %s: %.3f -> %.3f", paramName, currentValue, newValue))
        end
    end
end

-- Obtener el nivel de rendimiento actual
function PerformanceManager:getPerformanceLevel()
    return self.state.performanceLevel
end

-- Obtener FPS promedio
function PerformanceManager:getAverageFPS()
    return self.state.averageFPS
end

-- Activar/desactivar el gestor de rendimiento
function PerformanceManager:setEnabled(enabled)
    self.config.enabled = enabled
    if enabled then
        self:initialize()
    else
        self:destroy()
    end
end

-- Limpiar conexiones
function PerformanceManager:destroy()
    for _, conn in pairs(self.connections) do
        conn:Disconnect()
    end
    self.connections = {}
    print("Gestor de rendimiento destruido.")
end

--==============================================================================
-- MÓDULO: VisualConfig
--==============================================================================

local VisualConfig = {}
VisualConfig.__index = VisualConfig

-- Tipos de elementos de configuración visual
local ELEMENT_TYPES = {
    SLIDER = "Slider",
    TOGGLE = "Toggle",
    DROPDOWN = "Dropdown",
    BUTTON = "Button",
    LABEL = "Label",
    INPUT_FIELD = "InputField"
}

function VisualConfig.new(responsiveUI, adaptiveConfig)
    local self = setmetatable({}, VisualConfig)
    
    -- Referencias a otros sistemas
    self.responsiveUI = responsiveUI
    self.adaptiveConfig = adaptiveConfig
    
    -- Elementos de UI creados
    self.uiElements = {}
    
    -- Configuración de la UI
    self.uiConfig = {
        panelName = "Configuración",
        elementHeight = responsiveUI.dimensions.elementHeight,
        elementSpacing = responsiveUI.dimensions.elementSpacing,
        padding = responsiveUI.dimensions.padding,
        style = responsiveUI.style,
        fontSize = responsiveUI.dimensions.fontSize
    }
    
    -- Definición de los elementos de configuración
    self.configDefinition = {
        -- Ejemplo: Tasa de Exploración (Slider)
        {
            id = "explorationRate",
            type = ELEMENT_TYPES.SLIDER,
            label = "Tasa Exploración",
            min = 0.01,
            max = 1.0,
            step = 0.01,
            configKey = "explorationRate",
            tooltip = "Probabilidad de elegir una acción aleatoria."
        },
        -- Ejemplo: Errores Aleatorios (Toggle)
        {
            id = "randomMistakes",
            type = ELEMENT_TYPES.TOGGLE,
            label = "Errores Aleatorios",
            configKey = "randomMistakes",
            tooltip = "Simular errores ocasionales en las acciones."
        },
        -- Ejemplo: Nivel de Log (Dropdown)
        {
            id = "logLevel",
            type = ELEMENT_TYPES.DROPDOWN,
            label = "Nivel de Log",
            options = {"Debug", "Info", "Warning", "Error", "Success"},
            configKey = "logLevel", -- Necesita manejo especial en UI
            tooltip = "Nivel mínimo de mensajes a mostrar en el log."
        },
        -- Ejemplo: Botón para guardar configuración
        {
            id = "saveConfig",
            type = ELEMENT_TYPES.BUTTON,
            label = "Guardar Configuración",
            action = function() self:saveConfiguration() end,
            tooltip = "Guardar la configuración actual."
        },
        -- Ejemplo: Botón para cargar configuración
        {
            id = "loadConfig",
            type = ELEMENT_TYPES.BUTTON,
            label = "Cargar Configuración",
            action = function() self:loadConfiguration() end,
            tooltip = "Cargar la configuración guardada."
        },
        -- Añadir más elementos según sea necesario...
    }
    
    -- Inicializar
    self:initialize()
    
    return self
end

-- Inicializar la configuración visual
function VisualConfig:initialize()
    print("Inicializando configuración visual...")
    
    -- Crear los elementos de UI en el panel de configuración
    self:createConfigElements()
    
    -- Conectar callback para actualizar UI cuando cambie la config
    self.adaptiveConfig:setConfigChangedCallback(function(key, value, oldValue)
        self:updateElementValue(key, value)
    end)
    
    print("Configuración visual inicializada.")
end

-- Crear los elementos de UI en el panel de configuración
function VisualConfig:createConfigElements()
    local panel = self.responsiveUI.ui.panels[self.uiConfig.panelName]
    if not panel then
        warn("Panel de configuración no encontrado!")
        return
    end
    
    -- Limpiar elementos anteriores
    for _, element in pairs(self.uiElements) do
        if element.frame then element.frame:Destroy() end
    end
    self.uiElements = {}
    
    -- Crear layout si no existe
    local layout = panel:FindFirstChildOfClass("UIListLayout")
    if not layout then
        layout = Instance.new("UIListLayout")
        layout.Padding = UDim.new(0, self.uiConfig.elementSpacing)
        layout.SortOrder = Enum.SortOrder.LayoutOrder
        layout.Parent = panel
    end
    
    -- Crear cada elemento definido
    for i, definition in ipairs(self.configDefinition) do
        local elementFrame = self:createElement(definition, i)
        if elementFrame then
            elementFrame.Parent = panel
        end
    end
    
    -- Actualizar tamaño del canvas
    panel.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + self.uiConfig.padding)
end

-- Crear un elemento de configuración individual
function VisualConfig:createElement(definition, layoutOrder)
    local frame = Instance.new("Frame")
    frame.Name = definition.id .. "_Frame"
    frame.Size = UDim2.new(1, 0, 0, self.uiConfig.elementHeight)
    frame.BackgroundTransparency = 1
    frame.LayoutOrder = layoutOrder
    
    local uiElement = { frame = frame, definition = definition }
    
    -- Crear etiqueta
    local label = Instance.new("TextLabel")
    label.Name = "Label"
    label.Size = UDim2.new(0.4, 0, 1, 0) -- 40% del ancho para la etiqueta
    label.Position = UDim2.new(0, 0, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = definition.label .. ":"
    label.TextColor3 = self.uiConfig.style.textColor
    label.TextSize = self.uiConfig.fontSize
    label.Font = self.uiConfig.style.font
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame
    uiElement.label = label
    
    -- Crear control específico según el tipo
    local control = nil
    if definition.type == ELEMENT_TYPES.SLIDER then
        control = self:createSlider(frame, definition, uiElement)
    elseif definition.type == ELEMENT_TYPES.TOGGLE then
        control = self:createToggle(frame, definition, uiElement)
    elseif definition.type == ELEMENT_TYPES.DROPDOWN then
        control = self:createDropdown(frame, definition, uiElement)
    elseif definition.type == ELEMENT_TYPES.BUTTON then
        control = self:createButton(frame, definition, uiElement)
    -- Añadir más tipos si es necesario
    end
    
    if control then
        control.Position = UDim2.new(0.45, 0, 0, 0) -- Posición del control (55% restante)
        control.Size = UDim2.new(0.55, 0, 1, 0)
        control.Parent = frame
        uiElement.control = control
    end
    
    -- Guardar referencia al elemento
    self.uiElements[definition.id] = uiElement
    
    return frame
end

-- Crear control Slider
function VisualConfig:createSlider(parentFrame, definition, uiElement)
    local sliderFrame = Instance.new("Frame")
    sliderFrame.Name = "SliderControl"
    sliderFrame.BackgroundColor3 = self.uiConfig.style.buttonColor
    
    -- Añadir esquinas redondeadas
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 3)
    corner.Parent = sliderFrame
    
    -- Crear barra de progreso del slider
    local progressBar = Instance.new("Frame")
    progressBar.Name = "ProgressBar"
    progressBar.Size = UDim2.new(0, 0, 1, 0) -- Ancho inicial 0
    progressBar.BackgroundColor3 = self.uiConfig.style.buttonActiveColor
    progressBar.Parent = sliderFrame
    uiElement.progressBar = progressBar
    
    -- Crear etiqueta de valor
    local valueLabel = Instance.new("TextLabel")
    valueLabel.Name = "ValueLabel"
    valueLabel.Size = UDim2.new(1, 0, 1, 0)
    valueLabel.BackgroundTransparency = 1
    valueLabel.TextColor3 = self.uiConfig.style.textColor
    valueLabel.TextSize = self.uiConfig.fontSize * 0.9
    valueLabel.Font = self.uiConfig.style.font
    valueLabel.TextXAlignment = Enum.TextXAlignment.Center
    valueLabel.Parent = sliderFrame
    uiElement.valueLabel = valueLabel
    
    -- Obtener valor inicial
    local initialValue = self.adaptiveConfig:getConfig()[definition.configKey]
    self:updateSliderVisuals(uiElement, initialValue)
    
    -- Conectar eventos de input (simplificado, necesita drag)
    sliderFrame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            local relativeX = input.Position.X - sliderFrame.AbsolutePosition.X
            local percentage = math.clamp(relativeX / sliderFrame.AbsoluteSize.X, 0, 1)
            local newValue = definition.min + (definition.max - definition.min) * percentage
            
            -- Ajustar al step
            if definition.step then
                newValue = math.floor(newValue / definition.step + 0.5) * definition.step
            end
            
            newValue = math.clamp(newValue, definition.min, definition.max)
            
            -- Actualizar configuración y UI
            self.adaptiveConfig:updateConfigValue(definition.configKey, newValue)
            self:updateSliderVisuals(uiElement, newValue)
        end
    end)
    
    return sliderFrame
end

-- Actualizar visuales del slider
function VisualConfig:updateSliderVisuals(uiElement, value)
    local definition = uiElement.definition
    local percentage = (value - definition.min) / (definition.max - definition.min)
    
    if uiElement.progressBar then
        uiElement.progressBar.Size = UDim2.new(percentage, 0, 1, 0)
    end
    
    if uiElement.valueLabel then
        uiElement.valueLabel.Text = string.format("%.2f", value)
    end
end

-- Crear control Toggle
function VisualConfig:createToggle(parentFrame, definition, uiElement)
    local toggleButton = Instance.new("TextButton")
    toggleButton.Name = "ToggleControl"
    toggleButton.Text = ""
    
    -- Obtener valor inicial
    local initialValue = self.adaptiveConfig:getConfig()[definition.configKey]
    self:updateToggleVisuals(toggleButton, initialValue)
    
    -- Conectar evento de clic
    toggleButton.MouseButton1Click:Connect(function()
        local currentValue = self.adaptiveConfig:getConfig()[definition.configKey]
        local newValue = not currentValue
        
        -- Actualizar configuración y UI
        self.adaptiveConfig:updateConfigValue(definition.configKey, newValue)
        self:updateToggleVisuals(toggleButton, newValue)
    end)
    
    return toggleButton
end

-- Actualizar visuales del toggle
function VisualConfig:updateToggleVisuals(button, value)
    if value then
        button.Text = "ON"
        button.BackgroundColor3 = self.uiConfig.style.successColor
    else
        button.Text = "OFF"
        button.BackgroundColor3 = self.uiConfig.style.errorColor
    end
    button.TextColor3 = self.uiConfig.style.textColor
    button.TextSize = self.uiConfig.fontSize
    button.Font = self.uiConfig.style.font
end

-- Crear control Dropdown (simplificado)
function VisualConfig:createDropdown(parentFrame, definition, uiElement)
    local dropdownButton = Instance.new("TextButton")
    dropdownButton.Name = "DropdownControl"
    
    -- Obtener valor inicial
    local initialValue = self.adaptiveConfig:getConfig()[definition.configKey] or definition.options[1]
    dropdownButton.Text = tostring(initialValue) .. " ▼"
    dropdownButton.BackgroundColor3 = self.uiConfig.style.buttonColor
    dropdownButton.TextColor3 = self.uiConfig.style.textColor
    dropdownButton.TextSize = self.uiConfig.fontSize
    dropdownButton.Font = self.uiConfig.style.font
    
    -- Crear lista de opciones (inicialmente oculta)
    local optionsFrame = Instance.new("Frame")
    optionsFrame.Name = "OptionsFrame"
    optionsFrame.Size = UDim2.new(1, 0, 0, self.uiConfig.elementHeight * #definition.options)
    optionsFrame.Position = UDim2.new(0, 0, 1, 0) -- Debajo del botón
    optionsFrame.BackgroundColor3 = self.uiConfig.style.tabBarColor
    optionsFrame.BorderSizePixel = 1
    optionsFrame.BorderColor3 = self.uiConfig.style.buttonActiveColor
    optionsFrame.Visible = false
    optionsFrame.Parent = dropdownButton
    uiElement.optionsFrame = optionsFrame
    
    local layout = Instance.new("UIListLayout")
    layout.Parent = optionsFrame
    
    -- Crear botones para cada opción
    for _, optionValue in ipairs(definition.options) do
        local optionButton = Instance.new("TextButton")
        optionButton.Name = tostring(optionValue)
        optionButton.Size = UDim2.new(1, 0, 0, self.uiConfig.elementHeight)
        optionButton.BackgroundColor3 = self.uiConfig.style.buttonColor
        optionButton.Text = tostring(optionValue)
        optionButton.TextColor3 = self.uiConfig.style.textColor
        optionButton.TextSize = self.uiConfig.fontSize
        optionButton.Font = self.uiConfig.style.font
        optionButton.Parent = optionsFrame
        
        optionButton.MouseButton1Click:Connect(function()
            -- Actualizar configuración y UI
            if definition.configKey == "logLevel" then
                self.responsiveUI.logLevel = optionValue
            else
                self.adaptiveConfig:updateConfigValue(definition.configKey, optionValue)
            end
            dropdownButton.Text = tostring(optionValue) .. " ▼"
            optionsFrame.Visible = false -- Ocultar opciones
        end)
    end
    
    -- Conectar evento para mostrar/ocultar opciones
    dropdownButton.MouseButton1Click:Connect(function()
        optionsFrame.Visible = not optionsFrame.Visible
    end)
    
    return dropdownButton
end

-- Crear control Button
function VisualConfig:createButton(parentFrame, definition, uiElement)
    local button = Instance.new("TextButton")
    button.Name = "ButtonControl"
    button.Text = definition.label
    button.BackgroundColor3 = self.uiConfig.style.buttonColor
    button.TextColor3 = self.uiConfig.style.textColor
    button.TextSize = self.uiConfig.fontSize
    button.Font = self.uiConfig.style.font
    
    -- Conectar acción del botón
    if definition.action then
        button.MouseButton1Click:Connect(definition.action)
    end
    
    -- Ajustar tamaño para que ocupe todo el espacio del control
    parentFrame:FindFirstChild("Label"):Destroy() -- Eliminar etiqueta por defecto
    button.Position = UDim2.new(0, 0, 0, 0)
    button.Size = UDim2.new(1, 0, 1, 0)
    
    return button
end

-- Actualizar el valor visual de un elemento cuando cambia la configuración
function VisualConfig:updateElementValue(configKey, newValue)
    for id, uiElement in pairs(self.uiElements) do
        if uiElement.definition.configKey == configKey then
            local definition = uiElement.definition
            
            if definition.type == ELEMENT_TYPES.SLIDER then
                self:updateSliderVisuals(uiElement, newValue)
            elseif definition.type == ELEMENT_TYPES.TOGGLE then
                self:updateToggleVisuals(uiElement.control, newValue)
            elseif definition.type == ELEMENT_TYPES.DROPDOWN then
                if uiElement.control then
                    uiElement.control.Text = tostring(newValue) .. " ▼"
                end
            end
            
            break -- Asumir que solo hay un elemento por configKey
        end
    end
end

-- Funciones para guardar y cargar configuración (necesitan DataPersistence)
function VisualConfig:saveConfiguration()
    if self.dataPersistence then
        local success, message = self.dataPersistence:saveConfig(self.adaptiveConfig:getConfig())
        if success then
            self.responsiveUI:addLog("Configuración guardada con éxito.", "Success")
        else
            self.responsiveUI:addLog("Error al guardar configuración: " .. message, "Error")
        end
    else
        self.responsiveUI:addLog("Sistema de persistencia no disponible.", "Warning")
    end
end

function VisualConfig:loadConfiguration()
    if self.dataPersistence then
        local success, loadedConfig, message = self.dataPersistence:loadConfig()
        if success then
            -- Actualizar configuración en AdaptiveConfig
            for key, value in pairs(loadedConfig) do
                self.adaptiveConfig:updateConfigValue(key, value)
            end
            self.responsiveUI:addLog("Configuración cargada con éxito.", "Success")
        else
            self.responsiveUI:addLog("Error al cargar configuración: " .. message, "Error")
        end
    else
        self.responsiveUI:addLog("Sistema de persistencia no disponible.", "Warning")
    end
end

-- Establecer referencia a DataPersistence (llamado después de inicializar todos los módulos)
function VisualConfig:setDataPersistence(dataPersistence)
    self.dataPersistence = dataPersistence
end



--==============================================================================
-- MÓDULO: DataPersistence
--==============================================================================

local DataPersistence = {}
DataPersistence.__index = DataPersistence

function DataPersistence.new(platformDetection, adaptiveConfig)
    local self = setmetatable({}, DataPersistence)
    
    -- Referencias a otros sistemas
    self.platformDetection = platformDetection
    self.adaptiveConfig = adaptiveConfig
    
    -- Información del dispositivo
    self.deviceInfo = platformDetection:getDeviceInfo()
    
    -- Configuración de persistencia
    self.config = {
        useCompression = adaptiveConfig:getConfig().useCompression,
        useEncryption = adaptiveConfig:getConfig().useEncryption,
        useBackups = adaptiveConfig:getConfig().useBackups,
        maxBackups = adaptiveConfig:getConfig().maxBackups,
        autoSaveInterval = adaptiveConfig:getConfig().autoSaveInterval,
        saveSlots = adaptiveConfig:getConfig().saveSlots
    }
    
    -- Estado de persistencia
    self.state = {
        lastSaveTime = 0,
        saveCount = 0,
        loadCount = 0,
        currentSlot = 1,
        autoSaveEnabled = true
    }
    
    -- Inicializar
    self:initialize()
    
    return self
end

-- Inicializar el sistema de persistencia
function DataPersistence:initialize()
    print("Inicializando sistema de persistencia de datos...")
    
    -- Iniciar ciclo de autoguardado si está habilitado
    if self.state.autoSaveEnabled then
        self:startAutoSaveCycle()
    end
    
    print("Sistema de persistencia inicializado.")
end

-- Iniciar ciclo de autoguardado
function DataPersistence:startAutoSaveCycle()
    spawn(function()
        while self.state.autoSaveEnabled do
            wait(self.config.autoSaveInterval)
            
            -- Guardar configuración actual
            local success, message = self:saveConfig(self.adaptiveConfig:getConfig())
            if success then
                print("Autoguardado completado.")
            else
                warn("Error en autoguardado: " .. message)
            end
        end
    end)
end

-- Guardar configuración
function DataPersistence:saveConfig(configData, slot)
    slot = slot or self.state.currentSlot
    
    if slot < 1 or slot > self.config.saveSlots then
        return false, "Slot de guardado inválido."
    end
    
    -- Añadir metadatos
    local saveData = {
        config = configData,
        metadata = {
            timestamp = os.time(),
            deviceType = self.deviceInfo.deviceType,
            os = self.deviceInfo.os,
            version = "3.0",
            saveCount = self.state.saveCount + 1
        }
    }
    
    -- Convertir a JSON
    local jsonData
    local success, result = pcall(function()
        return HttpService:JSONEncode(saveData)
    end)
    
    if not success then
        return false, "Error al codificar JSON: " .. tostring(result)
    end
    
    jsonData = result
    
    -- Comprimir si está habilitado
    if self.config.useCompression then
        jsonData = self:compressData(jsonData)
    end
    
    -- Encriptar si está habilitado
    if self.config.useEncryption then
        jsonData = self:encryptData(jsonData)
    end
    
    -- Crear copia de seguridad si está habilitado
    if self.config.useBackups then
        self:createBackup(slot)
    end
    
    -- Guardar datos
    local key = "BoxingBetaAutoplayer_Config_" .. slot
    
    success, result = pcall(function()
        -- Usar WriteBinaryString para datos comprimidos/encriptados
        if self.config.useCompression or self.config.useEncryption then
            writefile(key .. ".dat", jsonData)
        else
            -- Usar WriteFile para datos legibles
            writefile(key .. ".json", jsonData)
        end
    end)
    
    if not success then
        return false, "Error al guardar archivo: " .. tostring(result)
    end
    
    -- Actualizar estado
    self.state.lastSaveTime = os.time()
    self.state.saveCount = self.state.saveCount + 1
    
    return true, "Configuración guardada correctamente."
end

-- Cargar configuración
function DataPersistence:loadConfig(slot)
    slot = slot or self.state.currentSlot
    
    if slot < 1 or slot > self.config.saveSlots then
        return false, nil, "Slot de carga inválido."
    end
    
    -- Determinar nombre de archivo
    local key = "BoxingBetaAutoplayer_Config_" .. slot
    local fileName = self.config.useCompression or self.config.useEncryption and key .. ".dat" or key .. ".json"
    
    -- Verificar si el archivo existe
    local success, fileExists = pcall(function()
        return isfile(fileName)
    end)
    
    if not success or not fileExists then
        return false, nil, "Archivo de configuración no encontrado."
    end
    
    -- Leer archivo
    local jsonData
    success, jsonData = pcall(function()
        return readfile(fileName)
    end)
    
    if not success then
        return false, nil, "Error al leer archivo: " .. tostring(jsonData)
    end
    
    -- Desencriptar si está habilitado
    if self.config.useEncryption then
        jsonData = self:decryptData(jsonData)
    end
    
    -- Descomprimir si está habilitado
    if self.config.useCompression then
        jsonData = self:decompressData(jsonData)
    end
    
    -- Decodificar JSON
    local saveData
    success, saveData = pcall(function()
        return HttpService:JSONDecode(jsonData)
    end)
    
    if not success then
        return false, nil, "Error al decodificar JSON: " .. tostring(saveData)
    end
    
    -- Verificar estructura de datos
    if not saveData or not saveData.config or not saveData.metadata then
        return false, nil, "Formato de datos inválido."
    end
    
    -- Actualizar estado
    self.state.loadCount = self.state.loadCount + 1
    
    -- Devolver configuración
    return true, saveData.config, "Configuración cargada correctamente."
end

-- Crear copia de seguridad
function DataPersistence:createBackup(slot)
    local key = "BoxingBetaAutoplayer_Config_" .. slot
    local fileName = self.config.useCompression or self.config.useEncryption and key .. ".dat" or key .. ".json"
    
    -- Verificar si el archivo existe
    local success, fileExists = pcall(function()
        return isfile(fileName)
    end)
    
    if not success or not fileExists then
        return false -- No hay archivo para hacer backup
    end
    
    -- Leer archivo original
    local fileData
    success, fileData = pcall(function()
        return readfile(fileName)
    end)
    
    if not success then
        return false
    end
    
    -- Crear nombre de backup con timestamp
    local backupFileName = key .. "_backup_" .. os.time() .. (self.config.useCompression or self.config.useEncryption and ".dat" or ".json")
    
    -- Guardar backup
    success = pcall(function()
        writefile(backupFileName, fileData)
    end)
    
    if not success then
        return false
    end
    
    -- Eliminar backups antiguos si exceden el máximo
    self:cleanupOldBackups(key)
    
    return true
end

-- Limpiar backups antiguos
function DataPersistence:cleanupOldBackups(baseKey)
    -- Listar archivos
    local success, files = pcall(function()
        return listfiles()
    end)
    
    if not success then
        return false
    end
    
    -- Filtrar backups para este slot
    local backups = {}
    for _, fileName in ipairs(files) do
        if fileName:find(baseKey .. "_backup_") then
            table.insert(backups, {
                name = fileName,
                time = tonumber(fileName:match("_backup_(%d+)")) or 0
            })
        end
    end
    
    -- Ordenar por tiempo (más reciente primero)
    table.sort(backups, function(a, b)
        return a.time > b.time
    end)
    
    -- Eliminar backups antiguos
    for i = self.config.maxBackups + 1, #backups do
        pcall(function()
            delfile(backups[i].name)
        end)
    end
    
    return true
end

-- Comprimir datos (implementación simplificada)
function DataPersistence:compressData(data)
    -- En un entorno real, usaríamos una biblioteca de compresión
    -- Para este ejemplo, simulamos compresión con un marcador
    return "COMPRESSED:" .. data
end

-- Descomprimir datos (implementación simplificada)
function DataPersistence:decompressData(data)
    -- Simulación de descompresión
    if data:sub(1, 11) == "COMPRESSED:" then
        return data:sub(12)
    end
    return data
end

-- Encriptar datos (implementación simplificada)
function DataPersistence:encryptData(data)
    -- En un entorno real, usaríamos una biblioteca de encriptación
    -- Para este ejemplo, simulamos encriptación con un marcador
    return "ENCRYPTED:" .. data
end

-- Desencriptar datos (implementación simplificada)
function DataPersistence:decryptData(data)
    -- Simulación de desencriptación
    if data:sub(1, 10) == "ENCRYPTED:" then
        return data:sub(11)
    end
    return data
end

-- Cambiar slot actual
function DataPersistence:setCurrentSlot(slot)
    if slot >= 1 and slot <= self.config.saveSlots then
        self.state.currentSlot = slot
        return true
    end
    return false
end

-- Activar/desactivar autoguardado
function DataPersistence:setAutoSaveEnabled(enabled)
    self.state.autoSaveEnabled = enabled
    
    if enabled and not self.autoSaveThread then
        self:startAutoSaveCycle()
    else
        -- Detener ciclo de autoguardado si existe
        if self.autoSaveThread then
            self.autoSaveThread:Disconnect()
            self.autoSaveThread = nil
        end
    end
end

-- Obtener información de guardado
function DataPersistence:getSaveInfo(slot)
    slot = slot or self.state.currentSlot
    
    local key = "BoxingBetaAutoplayer_Config_" .. slot
    local fileName = self.config.useCompression or self.config.useEncryption and key .. ".dat" or key .. ".json"
    
    -- Verificar si el archivo existe
    local success, fileExists = pcall(function()
        return isfile(fileName)
    end)
    
    if not success or not fileExists then
        return nil
    end
    
    -- Intentar cargar para obtener metadatos
    local loadSuccess, config, message = self:loadConfig(slot)
    if not loadSuccess then
        return {
            exists = true,
            valid = false,
            error = message
        }
    end
    
    -- Devolver información básica
    return {
        exists = true,
        valid = true,
        timestamp = os.date("%Y-%m-%d %H:%M:%S", config.metadata and config.metadata.timestamp or 0),
        deviceType = config.metadata and config.metadata.deviceType or "Unknown",
        version = config.metadata and config.metadata.version or "Unknown"
    }
end

--==============================================================================
-- MÓDULO: AntiDetection
--==============================================================================

local AntiDetection = {}
AntiDetection.__index = AntiDetection

function AntiDetection.new(platformDetection, adaptiveConfig)
    local self = setmetatable({}, AntiDetection)
    
    -- Referencias a otros sistemas
    self.platformDetection = platformDetection
    self.adaptiveConfig = adaptiveConfig
    
    -- Información del dispositivo
    self.deviceInfo = platformDetection:getDeviceInfo()
    
    -- Configuración anti-detección
    self.config = {
        enabled = true,
        humanPatterns = adaptiveConfig:getConfig().humanPatterns,
        variableTimings = adaptiveConfig:getConfig().variableTimings,
        avoidPerfectTiming = adaptiveConfig:getConfig().avoidPerfectTiming,
        
        -- Configuración específica para PC
        pc = {
            reactionTimeMin = 0.08,
            reactionTimeMax = 0.25,
            mouseJitter = true,
            mouseJitterAmount = 2, -- píxeles
            keyPressVariation = true,
            keyPressMinDuration = 0.05,
            keyPressMaxDuration = 0.15,
            occasionalMisclick = true,
            misclickChance = 0.01, -- 1%
            keyboardRollover = true,
            keyboardRolloverMax = 3,
            occasionalPause = true,
            pauseChance = 0.005, -- 0.5%
            pauseDurationMin = 0.2,
            pauseDurationMax = 1.0,
            comboDelayVariation = 0.02
        },
        
        -- Configuración específica para móvil
        mobile = {
            reactionTimeMin = 0.12,
            reactionTimeMax = 0.35,
            touchInaccuracy = true,
            touchInaccuracyAmount = 5, -- píxeles
            touchDuration = true,
            touchDurationMin = 0.08,
            touchDurationMax = 0.25,
            swipeSpeedMin = 200, -- píxeles por segundo
            swipeSpeedMax = 800,
            touchDelayAfterSwipe = true,
            touchDelayMin = 0.1,
            touchDelayMax = 0.3,
            multiTouchLimitation = true,
            maxSimultaneousTouches = 2,
            occasionalMissTouch = true,
            missTouchChance = 0.02, -- 2%
            touchAreaSize = 30, -- tamaño aproximado de área táctil
            orientationAwareness = true,
            portraitSlowerReaction = 0.05 -- segundos adicionales en modo retrato
        }
    }
    
    -- Estado del sistema anti-detección
    self.state = {
        lastActionTime = 0,
        currentReactionDelay = 0.1,
        actionHistory = {},
        maxHistorySize = 100,
        activeKeys = {},
        activeTouches = {},
        lastMousePosition = Vector2.new(0, 0),
        isPaused = false,
        pauseEndTime = 0,
        isInCombo = false,
        comboStartTime = 0,
        comboActionCount = 0,
        lastSwipeTime = 0,
        patternIndex = 1,
        patternRepeatCount = 0
    }
    
    -- Patrones humanos predefinidos
    self.humanPatterns = {
        pc = {
            mouseMovements = {
                {type = "linear", duration = 0.2},
                {type = "bezier", duration = 0.3},
                {type = "overshoot", duration = 0.25, overshoot = 5},
                {type = "undershoot", duration = 0.22, undershoot = 3},
                {type = "jittery", duration = 0.18, jitterAmount = 2}
            },
            keyPresses = {
                {duration = 0.08, pressure = 0.7},
                {duration = 0.12, pressure = 0.9},
                {duration = 0.06, pressure = 0.5},
                {duration = 0.10, pressure = 0.8},
                {duration = 0.14, pressure = 0.6}
            },
            clickPatterns = {
                {duration = 0.06, pressure = 0.8},
                {duration = 0.09, pressure = 0.7},
                {duration = 0.05, pressure = 0.9},
                {duration = 0.08, pressure = 0.6},
                {duration = 0.07, pressure = 0.8}
            }
        },
        mobile = {
            touches = {
                {duration = 0.15, accuracy = 0.9},
                {duration = 0.22, accuracy = 0.85},
                {duration = 0.18, accuracy = 0.95},
                {duration = 0.25, accuracy = 0.8},
                {duration = 0.12, accuracy = 0.9}
            },
            swipes = {
                {speed = 400, straightness = 0.8},
                {speed = 600, straightness = 0.9},
                {speed = 300, straightness = 0.7},
                {speed = 500, straightness = 0.85},
                {speed = 700, straightness = 0.95}
            },
            multiTouch = {
                {accuracy = 0.7, timingOffset = 0.05},
                {accuracy = 0.8, timingOffset = 0.03},
                {accuracy = 0.75, timingOffset = 0.07},
                {accuracy = 0.85, timingOffset = 0.02},
                {accuracy = 0.65, timingOffset = 0.08}
            }
        }
    }
    
    -- Conexiones de eventos
    self.connections = {}
    
    -- Inicializar
    self:initialize()
    
    return self
end

-- Inicializar el sistema anti-detección
function AntiDetection:initialize()
    if not self.config.enabled then return end
    
    print("Inicializando sistema anti-detección...")
    
    -- Conectar eventos de input para registrar patrones
    self:connectInputEvents()
    
    print("Sistema anti-detección inicializado.")
end

-- Conectar eventos de input
function AntiDetection:connectInputEvents()
    -- Limpiar conexiones anteriores
    for _, conn in pairs(self.connections) do
        conn:Disconnect()
    end
    self.connections = {}
    
    -- Conectar eventos de input
    table.insert(self.connections, UserInputService.InputBegan:Connect(function(input, gameProcessedEvent)
        if not gameProcessedEvent then
            self:recordInputBegan(input)
        end
    end))
    
    table.insert(self.connections, UserInputService.InputEnded:Connect(function(input, gameProcessedEvent)
        self:recordInputEnded(input)
    end))
    
    table.insert(self.connections, UserInputService.InputChanged:Connect(function(input, gameProcessedEvent)
        if not gameProcessedEvent then
            self:recordInputChanged(input)
        end
    end))
end

-- Registrar inicio de input
function AntiDetection:recordInputBegan(input)
    local inputType = input.UserInputType
    local time = tick()
    
    if inputType == Enum.UserInputType.MouseButton1 or inputType == Enum.UserInputType.MouseButton2 then
        -- Clic de mouse
        self:addToActionHistory({
            type = "mouse_click",
            button = inputType,
            position = input.Position,
            time = time
        })
    elseif inputType == Enum.UserInputType.Keyboard then
        -- Tecla
        local keyCode = input.KeyCode
        self.state.activeKeys[keyCode] = time
        
        self:addToActionHistory({
            type = "key_press",
            key = keyCode,
            time = time
        })
    end
end

-- Registrar fin de input
function AntiDetection:recordInputEnded(input)
    local inputType = input.UserInputType
    local time = tick()
    
    if inputType == Enum.UserInputType.MouseButton1 or inputType == Enum.UserInputType.MouseButton2 then
        -- Fin de clic de mouse
        self:addToActionHistory({
            type = "mouse_release",
            button = inputType,
            position = input.Position,
            time = time
        })
    elseif inputType == Enum.UserInputType.Keyboard then
        -- Fin de tecla
        local keyCode = input.KeyCode
        local pressTime = self.state.activeKeys[keyCode]
        
        if pressTime then
            local duration = time - pressTime
            
            self:addToActionHistory({
                type = "key_release",
                key = keyCode,
                time = time,
                duration = duration
            })
            
            self.state.activeKeys[keyCode] = nil
        end
    end
end

-- Registrar cambio de input
function AntiDetection:recordInputChanged(input)
    local inputType = input.UserInputType
    
    if inputType == Enum.UserInputType.MouseMovement then
        -- Movimiento de mouse (registrar solo ocasionalmente)
        if math.random() < 0.05 then
            local position = input.Position
            local lastPosition = self.state.lastMousePosition
            local time = tick()
            
            self:addToActionHistory({
                type = "mouse_move",
                position = position,
                lastPosition = lastPosition,
                time = time,
                delta = position - lastPosition
            })
            
            self.state.lastMousePosition = position
        end
    end
end

-- Añadir acción al historial
function AntiDetection:addToActionHistory(action)
    table.insert(self.state.actionHistory, action)
    
    -- Limitar tamaño del historial
    if #self.state.actionHistory > self.state.maxHistorySize then
        table.remove(self.state.actionHistory, 1)
    end
end

-- Aplicar comportamiento humano a una acción
function AntiDetection:applyHumanBehavior(action)
    if not self.config.enabled then
        return action -- Sin modificaciones si está deshabilitado
    end
    
    -- Verificar si estamos en pausa simulada
    if self.state.isPaused and tick() < self.state.pauseEndTime then
        return nil -- Ignorar acción durante pausa
    end
    
    -- Finalizar pausa si corresponde
    if self.state.isPaused and tick() >= self.state.pauseEndTime then
        self.state.isPaused = false
    end
    
    -- Simular pausa ocasional
    if self:shouldSimulatePause() then
        self:simulatePause()
        return nil -- Ignorar acción durante inicio de pausa
    end
    
    -- Aplicar retraso de reacción
    action = self:applyReactionDelay(action)
    if not action then
        return nil -- Acción retrasada, se procesará después
    end
    
    -- Aplicar comportamiento específico según plataforma
    if self.deviceInfo.isMobile then
        action = self:applyMobileBehavior(action)
    else
        action = self:applyPCBehavior(action)
    end
    
    -- Aplicar patrones humanos si está habilitado
    if self.config.humanPatterns then
        action = self:applyHumanPatterns(action)
    end
    
    return action
end

-- Verificar si se debe simular una pausa
function AntiDetection:shouldSimulatePause()
    if self.deviceInfo.isMobile then
        return self.config.mobile.occasionalMissTouch and math.random() < self.config.mobile.missTouchChance
    else
        return self.config.pc.occasionalPause and math.random() < self.config.pc.pauseChance
    end
end

-- Simular pausa
function AntiDetection:simulatePause()
    self.state.isPaused = true
    
    local pauseDuration
    if self.deviceInfo.isMobile then
        pauseDuration = self:randomRange(self.config.mobile.touchDelayMin, self.config.mobile.touchDelayMax)
    else
        pauseDuration = self:randomRange(self.config.pc.pauseDurationMin, self.config.pc.pauseDurationMax)
    end
    
    self.state.pauseEndTime = tick() + pauseDuration
    
    -- print("Simulando pausa de " .. string.format("%.2f", pauseDuration) .. "s")
end

-- Aplicar retraso de reacción
function AntiDetection:applyReactionDelay(action)
    local currentTime = tick()
    
    -- Si es la primera acción o ha pasado suficiente tiempo, calcular nuevo retraso
    if self.state.lastActionTime == 0 or currentTime - self.state.lastActionTime > 1 then
        if self.deviceInfo.isMobile then
            local baseDelay = self:randomRange(self.config.mobile.reactionTimeMin, self.config.mobile.reactionTimeMax)
            
            -- Añadir retraso adicional en modo retrato si está habilitado
            if self.config.mobile.orientationAwareness and self.deviceInfo.orientation == "Portrait" then
                baseDelay = baseDelay + self.config.mobile.portraitSlowerReaction
            end
            
            self.state.currentReactionDelay = baseDelay
        else
            self.state.currentReactionDelay = self:randomRange(self.config.pc.reactionTimeMin, self.config.pc.reactionTimeMax)
        end
    end
    
    -- Si estamos en un combo, usar retraso diferente
    if self.state.isInCombo then
        local comboProgress = math.min((currentTime - self.state.comboStartTime) / 2, 1) -- Normalizado a 2 segundos máximo
        local comboFactor = 1 - (comboProgress * 0.3) -- Reducir hasta un 30% el tiempo de reacción
        
        self.state.currentReactionDelay = self.state.currentReactionDelay * comboFactor
        
        -- Añadir variación entre acciones de combo
        if self.config.variableTimings then
            if self.deviceInfo.isMobile then
                -- No añadir variación adicional en móvil para combos
            else
                self.state.currentReactionDelay = self.state.currentReactionDelay + 
                    (math.random() * 2 - 1) * self.config.pc.comboDelayVariation
            end
        end
    end
    
    -- Asegurar que el retraso no sea negativo
    self.state.currentReactionDelay = math.max(self.state.currentReactionDelay, 0.01)
    
    -- Programar la acción con retraso
    delay(self.state.currentReactionDelay, function()
        -- Ejecutar acción después del retraso
        self:executeDelayedAction(action)
    end)
    
    -- Actualizar tiempo de última acción
    self.state.lastActionTime = currentTime
    
    -- Devolver nil para indicar que la acción se procesará después
    return nil
end

-- Ejecutar acción retrasada
function AntiDetection:executeDelayedAction(action)
    -- Aquí se implementaría la ejecución real de la acción
    -- Para este ejemplo, simplemente la registramos
    
    -- print("Ejecutando acción retrasada: " .. action.type)
    
    -- Actualizar estado de combo si es necesario
    if action.isComboAction then
        if not self.state.isInCombo then
            self.state.isInCombo = true
            self.state.comboStartTime = tick()
            self.state.comboActionCount = 1
        else
            self.state.comboActionCount = self.state.comboActionCount + 1
        end
    else
        -- Resetear estado de combo
        self.state.isInCombo = false
        self.state.comboActionCount = 0
    end
end

-- Aplicar comportamiento específico de móvil
function AntiDetection:applyMobileBehavior(action)
    local config = self.config.mobile
    
    -- No modificar si no es una acción de toque
    if not action.type:find("touch") then
        return action
    end
    
    -- Simular imprecisión de toque
    if config.touchInaccuracy and action.position then
        local inaccuracy = config.touchInaccuracyAmount
        local offsetX = (math.random() * 2 - 1) * inaccuracy
        local offsetY = (math.random() * 2 - 1) * inaccuracy
        
        action.originalPosition = action.position -- Guardar posición original
        action.position = Vector2.new(
            action.position.X + offsetX,
            action.position.Y + offsetY
        )
    end
    
    -- Simular duración variable de toque
    if config.touchDuration and action.type == "touch_end" then
        local minDuration = config.touchDurationMin
        local maxDuration = config.touchDurationMax
        
        -- Si es un swipe, usar duración diferente
        if action.distance and action.distance > 20 then
            -- Calcular duración basada en distancia y velocidad aleatoria
            local speed = self:randomRange(config.swipeSpeedMin, config.swipeSpeedMax)
            local calculatedDuration = action.distance / speed
            
            -- Limitar a un rango razonable
            action.duration = math.max(minDuration, math.min(calculatedDuration, maxDuration * 1.5))
            
            -- Registrar tiempo de último swipe
            self.state.lastSwipeTime = tick()
        else
            -- Toque normal
            action.duration = self:randomRange(minDuration, maxDuration)
        end
    end
    
    -- Simular retraso después de swipe
    if config.touchDelayAfterSwipe and self.state.lastSwipeTime > 0 then
        local timeSinceSwipe = tick() - self.state.lastSwipeTime
        local minDelay = config.touchDelayMin
        
        if timeSinceSwipe < minDelay then
            -- Ignorar acción si no ha pasado suficiente tiempo
            return nil
        end
    end
    
    -- Simular limitación de multi-touch
    if config.multiTouchLimitation then
        local activeTouchCount = 0
        for _ in pairs(self.state.activeTouches) do
            activeTouchCount = activeTouchCount + 1
        end
        
        if activeTouchCount >= config.maxSimultaneousTouches and action.type == "touch_start" then
            -- Ignorar nuevo toque si ya hay demasiados activos
            return nil
        end
    end
    
    -- Simular toque fallido ocasional
    if config.occasionalMissTouch and action.type == "touch_start" and math.random() < config.missTouchChance then
        -- Modificar posición significativamente para simular toque fallido
        local missOffset = config.touchAreaSize * 2
        action.position = Vector2.new(
            action.position.X + (math.random() * 2 - 1) * missOffset,
            action.position.Y + (math.random() * 2 - 1) * missOffset
        )
    end
    
    return action
end

-- Aplicar comportamiento específico de PC
function AntiDetection:applyPCBehavior(action)
    local config = self.config.pc
    
    -- Simular jitter de mouse
    if config.mouseJitter and (action.type == "mouse_move" or action.type == "mouse_click") and action.position then
        local jitter = config.mouseJitterAmount
        local offsetX = (math.random() * 2 - 1) * jitter
        local offsetY = (math.random() * 2 - 1) * jitter
        
        action.originalPosition = action.position -- Guardar posición original
        action.position = Vector2.new(
            action.position.X + offsetX,
            action.position.Y + offsetY
        )
    end
    
    -- Simular variación en duración de pulsación de teclas
    if config.keyPressVariation and action.type == "key_release" then
        action.duration = self:randomRange(config.keyPressMinDuration, config.keyPressMaxDuration)
    end
    
    -- Simular clic fallido ocasional
    if config.occasionalMisclick and action.type == "mouse_click" and math.random() < config.misclickChance then
        -- Modificar posición significativamente para simular clic fallido
        local missOffset = 20
        action.position = Vector2.new(
            action.position.X + (math.random() * 2 - 1) * missOffset,
            action.position.Y + (math.random() * 2 - 1) * missOffset
        )
    end
    
    -- Simular limitaciones de rollover de teclado
    if config.keyboardRollover and action.type == "key_press" then
        local activeKeyCount = 0
        for _ in pairs(self.state.activeKeys) do
            activeKeyCount = activeKeyCount + 1
        end
        
        if activeKeyCount >= config.keyboardRolloverMax then
            -- Ignorar nueva tecla si ya hay demasiadas activas
            return nil
        end
    end
    
    return action
end

-- Aplicar patrones humanos
function AntiDetection:applyHumanPatterns(action)
    -- Seleccionar conjunto de patrones según plataforma
    local patterns = self.deviceInfo.isMobile and self.humanPatterns.mobile or self.humanPatterns.pc
    
    -- Aplicar patrones según tipo de acción
    if self.deviceInfo.isMobile then
        if action.type:find("touch") then
            -- Aplicar patrones de toque
            local touchPatterns = patterns.touches
            local pattern = touchPatterns[self.state.patternIndex]
            
            -- Modificar acción según patrón
            if pattern and action.type == "touch_end" and action.duration then
                -- Ajustar duración según patrón
                local patternFactor = pattern.duration / 0.2 -- Normalizado a 0.2s
                action.duration = action.duration * patternFactor
                
                -- Ajustar precisión
                if pattern.accuracy and action.position and action.originalPosition then
                    local accuracyFactor = pattern.accuracy
                    action.position = Vector2.new(
                        action.originalPosition.X * accuracyFactor + action.position.X * (1 - accuracyFactor),
                        action.originalPosition.Y * accuracyFactor + action.position.Y * (1 - accuracyFactor)
                    )
                end
                
                -- Avanzar al siguiente patrón
                self:advancePattern(#touchPatterns)
            end
        end
    else
        if action.type:find("mouse") then
            -- Aplicar patrones de mouse
            local mousePatterns = patterns.mouseMovements
            local pattern = mousePatterns[self.state.patternIndex]
            
            -- Modificar acción según patrón
            if pattern and action.type == "mouse_move" then
                -- Implementar curva de movimiento según tipo
                if pattern.type == "bezier" and action.delta then
                    -- Simular curva bezier ajustando delta
                    local curveFactor = 1.2
                    action.delta = Vector2.new(
                        action.delta.X * curveFactor,
                        action.delta.Y * (2 - curveFactor)
                    )
                elseif pattern.type == "overshoot" and action.position and action.originalPosition then
                    -- Simular overshooting
                    local overshootFactor = pattern.overshoot / 100
                    local direction = (action.position - action.originalPosition).Unit
                    action.position = action.position + direction * overshootFactor
                end
                
                -- Avanzar al siguiente patrón
                self:advancePattern(#mousePatterns)
            end
        elseif action.type:find("key") then
            -- Aplicar patrones de teclado
            local keyPatterns = patterns.keyPresses
            local pattern = keyPatterns[self.state.patternIndex]
            
            -- Modificar acción según patrón
            if pattern and action.type == "key_release" and action.duration then
                -- Ajustar duración según patrón
                local patternFactor = pattern.duration / 0.2 -- Normalizado a 0.2s
                action.duration = action.duration * patternFactor
                
                -- Avanzar al siguiente patrón
                self:advancePattern(#keyPatterns)
            end
        end
    end
    
    return action
end

-- Avanzar al siguiente patrón
function AntiDetection:advancePattern(patternCount)
    self.state.patternRepeatCount = self.state.patternRepeatCount + 1
    
    -- Cambiar de patrón cada cierto número de repeticiones
    if self.state.patternRepeatCount >= 3 then
        self.state.patternIndex = self.state.patternIndex + 1
        self.state.patternRepeatCount = 0
        
        -- Volver al primer patrón si llegamos al final
        if self.state.patternIndex > patternCount then
            self.state.patternIndex = 1
        end
    end
end

-- Generar número aleatorio en un rango
function AntiDetection:randomRange(min, max)
    return min + math.random() * (max - min)
end

-- Procesar acción antes de ejecutarla
function AntiDetection:processAction(actionType, actionData)
    -- Crear objeto de acción
    local action = {
        type = actionType,
        time = tick()
    }
    
    -- Añadir datos específicos de la acción
    for key, value in pairs(actionData or {}) do
        action[key] = value
    end
    
    -- Aplicar comportamiento humano
    local modifiedAction = self:applyHumanBehavior(action)
    
    -- Si la acción fue cancelada o retrasada
    if not modifiedAction then
        return false
    end
    
    -- Devolver acción modificada
    return true, modifiedAction
end

-- Verificar si una acción debe ser ejecutada (considerando anti-detección)
function AntiDetection:shouldExecuteAction(actionType, actionData)
    local success, _ = self:processAction(actionType, actionData)
    return success
end

-- Obtener tiempo de retraso para una acción
function AntiDetection:getActionDelay(actionType, actionData)
    -- Calcular retraso base según plataforma
    local baseDelay
    
    if self.deviceInfo.isMobile then
        baseDelay = self:randomRange(self.config.mobile.reactionTimeMin, self.config.mobile.reactionTimeMax)
        
        -- Ajustar según tipo de acción
        if actionType:find("swipe") then
            baseDelay = baseDelay * 1.2 -- Swipes son más lentos
        elseif actionType:find("tap") and actionData and actionData.isComboAction then
            baseDelay = baseDelay * 0.8 -- Taps en combo son más rápidos
        end
    else
        baseDelay = self:randomRange(self.config.pc.reactionTimeMin, self.config.pc.reactionTimeMax)
        
        -- Ajustar según tipo de acción
        if actionType == "mouse_click" and actionData and actionData.isComboAction then
            baseDelay = baseDelay * 0.8 -- Clics en combo son más rápidos
        elseif actionType == "key_press" and actionData and actionData.isComboAction then
            baseDelay = baseDelay * 0.85 -- Teclas en combo son más rápidas
        end
    end
    
    -- Aplicar variación si está habilitada
    if self.config.variableTimings then
        local variationFactor = 0.2 -- 20% de variación
        baseDelay = baseDelay * (1 + (math.random() * 2 - 1) * variationFactor)
    end
    
    -- Evitar timing perfecto si está habilitado
    if self.config.avoidPerfectTiming then
        -- Asegurar que nunca sea exactamente un valor redondo
        if math.abs(baseDelay - math.floor(baseDelay * 10) / 10) < 0.002 then
            baseDelay = baseDelay + 0.003
        end
    end
    
    return baseDelay
end

-- Actualizar configuración
function AntiDetection:updateConfig(newConfig)
    -- Actualizar configuración general
    if newConfig.enabled ~= nil then
        self.config.enabled = newConfig.enabled
    end
    
    if newConfig.humanPatterns ~= nil then
        self.config.humanPatterns = newConfig.humanPatterns
    end
    
    if newConfig.variableTimings ~= nil then
        self.config.variableTimings = newConfig.variableTimings
    end
    
    if newConfig.avoidPerfectTiming ~= nil then
        self.config.avoidPerfectTiming = newConfig.avoidPerfectTiming
    end
    
    -- Actualizar configuración específica de plataforma
    if self.deviceInfo.isMobile then
        -- Actualizar configuración de móvil
        for key, value in pairs(newConfig.mobile or {}) do
            if self.config.mobile[key] ~= nil then
                self.config.mobile[key] = value
            end
        end
    else
        -- Actualizar configuración de PC
        for key, value in pairs(newConfig.pc or {}) do
            if self.config.pc[key] ~= nil then
                self.config.pc[key] = value
            end
        end
    end
end

-- Limpiar conexiones al destruir
function AntiDetection:destroy()
    for _, conn in pairs(self.connections) do
        conn:Disconnect()
    end
    self.connections = {}
    
    print("Sistema Anti-Detección destruido.")
end


--==============================================================================
-- INICIALIZACIÓN Y CÓDIGO PRINCIPAL
--==============================================================================

-- Clase principal del Autoplayer
local BoxingBetaAutoplayer = {}
BoxingBetaAutoplayer.__index = BoxingBetaAutoplayer

function BoxingBetaAutoplayer.new()
    local self = setmetatable({}, BoxingBetaAutoplayer)
    
    -- Estado general
    self.isEnabled = false
    self.isInitialized = false
    self.startTime = os.time()
    
    -- Estadísticas
    self.stats = {
        learning = {
            iterations = 0,
            explorationRate = CONFIG.explorationRate,
            memorySize = 0,
            qTableSize = 0,
            wins = 0,
            losses = 0,
            performanceMetrics = {
                successRate = 0,
                avgRewardPerAction = 0
            }
        },
        execution = {
            totalActions = 0,
            avgTimeBetweenActions = 0,
            lastActionTime = 0
        }
    }
    
    -- Inicializar módulos
    self:initializeModules()
    
    return self
end

-- Inicializar todos los módulos
function BoxingBetaAutoplayer:initializeModules()
    print("Inicializando BoxingBetaAutoplayer v3.0 Universal...")
    
    -- Inicializar módulos en orden de dependencia
    self.platformDetection = PlatformDetection.new()
    self.adaptiveConfig = AdaptiveConfig.new(self.platformDetection)
    self.responsiveUI = ResponsiveUI.new(self.platformDetection, self.adaptiveConfig)
    self.unifiedInput = UnifiedInput.new(self.platformDetection)
    self.touchControls = TouchControls.new(self.platformDetection, self.unifiedInput)
    self.performanceManager = PerformanceManager.new(self.platformDetection, self.adaptiveConfig)
    self.visualConfig = VisualConfig.new(self.responsiveUI, self.adaptiveConfig)
    self.dataPersistence = DataPersistence.new(self.platformDetection, self.adaptiveConfig)
    self.antiDetection = AntiDetection.new(self.platformDetection, self.adaptiveConfig)
    
    -- Inicializar el motor de aprendizaje
    self.learningEngine = LearningEngine.new(self.adaptiveConfig:getConfig())
    -- Inicializar la interfaz de configuración del aprendizaje
    self.learningConfigUI = LearningConfigUI.new(self.learningEngine, self.responsiveUI, self.adaptiveConfig)
    
    -- Establecer referencias cruzadas
    self.responsiveUI.mainModule = self
    self.visualConfig:setDataPersistence(self.dataPersistence)
    
    -- Crear UI
    self.responsiveUI:createUI()
    
    -- Conectar eventos principales
    self:connectEvents()
    
    -- Marcar como inicializado
    self.isInitialized = true
    
    print("BoxingBetaAutoplayer v3.0 Universal inicializado correctamente.")
    
    -- Mostrar mensaje de bienvenida
    self.responsiveUI:addLog("BoxingBetaAutoplayer v3.0 Universal inicializado correctamente.", "Success")
    self.responsiveUI:addLog("Plataforma detectada: " .. self.platformDetection:getDeviceInfo().deviceType, "Info")
    
    -- Actualizar estadísticas iniciales
    self:updateStats()
end

-- Actualizar estadísticas
function BoxingBetaAutoplayer:updateStats()
    -- Actualizar estadísticas de tiempo de ejecución
    self.stats.execution.avgTimeBetweenActions = self.stats.execution.totalActions > 0 
        and (os.time() - self.startTime) / self.stats.execution.totalActions 
        or 0

    -- Actualizar estadísticas de aprendizaje desde el motor
    if self.learningEngine then
        local learningStats = self.learningEngine:getStats()
        self.stats.learning.iterations = learningStats.iterations
        self.stats.learning.explorationRate = learningStats.explorationRate
        self.stats.learning.memorySize = learningStats.memorySize
        self.stats.learning.qTableSize = learningStats.qTableSize
        self.stats.learning.wins = learningStats.wins
        self.stats.learning.losses = learningStats.losses
        self.stats.learning.performanceMetrics = learningStats.performanceMetrics
    end

    -- Actualizar UI con estadísticas
    self.responsiveUI:updateStats(self.stats.learning, self.stats.execution)
end

function BoxingBetaAutoplayer:connectEvents()
    -- Actualizar estadísticas periódicamente
    spawn(function()
        while true do
            wait(1)
            if self.isInitialized then
                self:updateStats()
            end
        end
    end)

    -- Entrenar el modelo periódicamente cuando está activado
    spawn(function()
        while true do
            wait(5) -- Entrenar cada 5 segundos
            if self.isInitialized and self.isEnabled and self.learningEngine then
                self.learningEngine:trainBatch()
                self.responsiveUI:addLog("Entrenamiento de lote completado.", "Debug")
            end
        end
    end)

    -- ...otros eventos según sea necesario...
end

-- Ciclo principal del autoplayer
function BoxingBetaAutoplayer:runCycle()
    if not self.isEnabled then return end

    -- Percibir estado del juego
    local gameState = self:perceiveGameState()

    -- Decidir acción usando el motor de aprendizaje
    local action = self:decideAction(gameState)

    -- Ejecutar acción
    if action then
        local actionResult = self:executeAction(action)

        -- Calcular recompensa y almacenar experiencia
        if self.learningEngine and self.lastGameState then
            local reward = self.learningEngine:computeReward(gameState, self.lastGameState, action, actionResult)
            self.learningEngine:storeExperience(self.lastGameState, action, reward, gameState)

            -- Registrar victoria o derrota si corresponde
            if actionResult and actionResult.gameOver then
                if actionResult.victory then
                    self.learningEngine:recordWin()
                    self.responsiveUI:addLog("¡Victoria registrada!", "Success")
                else
                    self.learningEngine:recordLoss()
                    self.responsiveUI:addLog("Derrota registrada.", "Warning")
                end
            end
        end

        -- Actualizar estadísticas
        self.stats.execution.totalActions = self.stats.execution.totalActions + 1
        self.stats.execution.lastActionTime = os.time()
    end

    -- Guardar estado actual para la próxima iteración
    self.lastGameState = gameState
end

-- Decidir acción usando el motor de aprendizaje
function BoxingBetaAutoplayer:decideAction(gameState)
    -- Usar el motor de aprendizaje para seleccionar la acción
    if self.learningEngine then
        local action = self.learningEngine:selectAction(gameState)
        self.responsiveUI:addLog("Acción seleccionada: " .. action, "Debug")
        return action
    else
        -- Fallback a selección aleatoria si no hay motor de aprendizaje
        local actions = {"Jab", "Uppercut", "Hook", "Block", "Dodge"}
        local randomIndex = math.random(1, #actions)
        return actions[randomIndex]
    end
end

-- Ejecutar acción (implementación simplificada)
function BoxingBetaAutoplayer:executeAction(action)
    -- Aquí iría la lógica para ejecutar la acción en el juego
    -- Para este ejemplo, solo registramos la acción

    -- Aplicar anti-detección
    local shouldExecute = self.antiDetection:shouldExecuteAction(action, {isComboAction = false})

    if shouldExecute then
        self.responsiveUI:addLog("Ejecutando acción: " .. action, "Debug")

        -- Aquí iría el código para ejecutar la acción en el juego
        -- ...

        -- Simular resultado de la acción
        local actionResult = {
            hitLanded = math.random() > 0.5,
            dodgeSuccessful = action:find("Dodge") and math.random() > 0.3,
            blockSuccessful = action == "Block" and math.random() > 0.2,
            damageDealt = math.random(5, 15),
            damageTaken = math.random(0, 10),
            gameOver = math.random() > 0.95,
            victory = math.random() > 0.5,
            perfectTiming = math.random() > 0.8,  -- Nuevo: timing perfecto
            counterAttack = math.random() > 0.9,  -- Nuevo: contraataque
            combo = math.random() > 0.7,          -- Nuevo: combo
            comboLength = math.random(2, 5),      -- Nuevo: longitud del combo
            comboTiming = math.random() > 0.6 and "perfect" or "good"  -- Nuevo: timing del combo
        }

        return actionResult
    end

    return nil
end

function BoxingBetaAutoplayer:connectEvents()
    -- Actualizar estadísticas periódicamente
    spawn(function()
        while true do
            wait(1)
            if self.isInitialized then
                self:updateStats()
            end
        end
    end)

    -- Entrenar el modelo periódicamente cuando está activado
    spawn(function()
        while true do
            wait(5) -- Entrenar cada 5 segundos
            if self.isInitialized and self.isEnabled and self.learningEngine then
                self.learningEngine:trainBatch()
                self.responsiveUI:addLog("Entrenamiento de lote completado.", "Debug")
            end
        end
    end)

    -- ...otros eventos según sea necesario...
end

function BoxingBetaAutoplayer:updateStats()
    -- Actualizar estadísticas de tiempo de ejecución
    self.stats.execution.avgTimeBetweenActions = self.stats.execution.totalActions > 0 
        and (os.time() - self.startTime) / self.stats.execution.totalActions 
        or 0

    -- Actualizar estadísticas de aprendizaje desde el motor
    if self.learningEngine then
        local learningStats = self.learningEngine:getStats()
        self.stats.learning.iterations = learningStats.iterations
        self.stats.learning.explorationRate = learningStats.explorationRate
        self.stats.learning.memorySize = learningStats.memorySize
        self.stats.learning.qTableSize = learningStats.qTableSize
        self.stats.learning.wins = learningStats.wins
        self.stats.learning.losses = learningStats.losses
        self.stats.learning.performanceMetrics = learningStats.performanceMetrics
    end

    -- Actualizar UI con estadísticas
    self.responsiveUI:updateStats(self.stats.learning, self.stats.execution)
end

function BoxingBetaAutoplayer:destroy()
    print("Destruyendo BoxingBetaAutoplayer...")

    -- Detener autoplayer
    self:stop()

    -- Destruir módulos en orden inverso
    if self.learningConfigUI then
        self.learningConfigUI:destroy()
    end

    if self.learningEngine then
        -- Guardar tabla Q antes de destruir
        self.learningEngine:saveQTable("BoxingBetaAutoplayer_QTable.json")
    end

    if self.antiDetection then self.antiDetection:destroy() end
    if self.dataPersistence then end -- No tiene método destroy
    if self.visualConfig then end -- No tiene método destroy
    if self.performanceManager then self.performanceManager:destroy() end
    if self.touchControls then self.touchControls:destroy() end
    if self.unifiedInput then self.unifiedInput:destroy() end
    if self.responsiveUI then end -- No tiene método destroy explícito
    if self.adaptiveConfig then end -- No tiene método destroy
    if self.platformDetection then end -- No tiene método destroy

    print("BoxingBetaAutoplayer destruido.")
end

--==============================================================================

-- Crear instancia global
_G.BoxingBetaAutoplayer = BoxingBetaAutoplayer.new()

-- Mensaje de confirmación
print("BoxingBetaAutoplayer v3.0 Universal cargado correctamente.")
print("Usa la interfaz para activar/desactivar el autoplayer.")

-- Mostrar UI
_G.BoxingBetaAutoplayer.responsiveUI:show()

return _G.BoxingBetaAutoplayer

