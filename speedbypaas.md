local Services = {
    Players = game:GetService("Players"),
    UserInput = game:GetService("UserInputService"),
    RunService = game:GetService("RunService"),
    Tween = game:GetService("TweenService"),
    Http = game:GetService("HttpService"),
}

local LocalPlayer = Services.Players.LocalPlayer or Services.Players.PlayerAdded:Wait()
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

--=====================================================================
-- SECTION 2: GLOBAL CONFIGURATION
--=====================================================================
local CONSTANTS = {
    -- File
    CONFIG_FILE = "MiroSpeedBypass.json",
    GUI_NAME = "MiroSpeedBypass",
    
    -- Bypass Logic
    DEPTH = 296,
    SPAM_DELAY = 0.12,
    MIN_POWER = 1,
    MAX_POWER = 500000,
    
    -- Default Values
    DEFAULT_POWER_PC = 97000,
    DEFAULT_POWER_MOBILE = 65000,
    DEFAULT_WINDOW_INDEX = 6,
    
    -- Animation
    TWEEN_DURATION = 0.15,
}

--=====================================================================
-- SECTION 3: THEME SYSTEM
--=====================================================================
local Theme = {
    Colors = {
        Background = Color3.fromRGB(0, 0, 0),
        BackgroundDark = Color3.fromRGB(0, 0, 0),
        BackgroundMedium = Color3.fromRGB(15, 15, 15),
        BackgroundInput = Color3.fromRGB(10, 10, 10),
        BackgroundButton = Color3.fromRGB(20, 20, 20),
        BackgroundButtonLight = Color3.fromRGB(30, 30, 30),
        
        TextPrimary = Color3.fromRGB(255, 215, 0), -- Gold Color
        TextSecondary = Color3.fromRGB(200, 200, 200),
        TextAccent = Color3.fromRGB(255, 215, 0), -- Gold Color
        TextWarning = Color3.fromRGB(255, 200, 100),
        
        StrokeDefault = Color3.fromRGB(80, 80, 80),
        StrokeDark = Color3.fromRGB(70, 70, 70),
        StrokeInput = Color3.fromRGB(60, 60, 60),
        StrokeLight = Color3.fromRGB(90, 90, 90),
        
        Accent = Color3.fromRGB(218, 165, 32),
        Enabled = Color3.fromRGB(255, 215, 0),
    },
    Transparency = {
        Main = 0,
        TitleBar = 0,
        Panel = 0.78,
        Active = 0.52,
        Button = 0.78,
    },
}

--=====================================================================
-- SECTION 4: WINDOW SIZES REGISTRY
--=====================================================================
local WindowSizes = {
    [1] = UDim2.new(0, 220, 0, 180),
    [2] = UDim2.new(0, 230, 0, 205),
    [3] = UDim2.new(0, 250, 0, 210),
    [4] = UDim2.new(0, 260, 0, 235),
    [5] = UDim2.new(0, 280, 0, 240),
    [6] = UDim2.new(0, 290, 0, 265),
    [7] = UDim2.new(0, 310, 0, 270),
    [8] = UDim2.new(0, 320, 0, 295),
    [9] = UDim2.new(0, 260, 0, 235),
    [10] = UDim2.new(0, 230, 0, 205),
}

--=====================================================================
-- SECTION 5: STATE MANAGER CLASS
--=====================================================================
local StateManager = {}
StateManager.__index = StateManager

function StateManager.new()
    local self = setmetatable({}, StateManager)
    self._state = {
        enabled = false,
        powerValue = CONSTANTS.DEFAULT_POWER_PC,
        mode = "PC",
        toggleKey = Enum.KeyCode.V,
        backgroundVisible = false,
        backgroundStyle = 1,
        windowSizeIndex = CONSTANTS.DEFAULT_WINDOW_INDEX,
        isVisible = true,
    }
    self._listeners = {}
    self._bypass = {
        running = false,
        bomb = nil,
        thread = nil,
        startedAt = 0,
    }
    return self
end

function StateManager:get(key)
    return self._state[key]
end

function StateManager:set(key, value)
    local oldValue = self._state[key]
    self._state[key] = value
    if oldValue ~= value then
        self:_notify(key, value, oldValue)
    end
    return value
end

function StateManager:toggle(key)
    return self:set(key, not self._state[key])
end

function StateManager:subscribe(key, callback)
    if not self._listeners[key] then
        self._listeners[key] = {}
    end
    table.insert(self._listeners[key], callback)
    return function()
        for i, cb in ipairs(self._listeners[key]) do
            if cb == callback then
                table.remove(self._listeners[key], i)
                break
            end
        end
    end
end

function StateManager:_notify(key, newValue, oldValue)
    if self._listeners[key] then
        for _, callback in ipairs(self._listeners[key]) do
            task.spawn(callback, newValue, oldValue)
        end
    end
end

-- Bypass sub-state accessors
function StateManager:isBypassRunning() return self._bypass.running end
function StateManager:getBypassData() return self._bypass end
function StateManager:setBypassField(field, value) self._bypass[field] = value end

--=====================================================================
-- SECTION 6: CONFIG PERSISTENCE
--=====================================================================
local ConfigStore = {}

function ConfigStore.save(state)
    local data = {
        Enabled = state:get("enabled"),
        PowerValue = state:get("powerValue"),
        Mode = state:get("mode"),
        ToggleKey = state:get("toggleKey").Name,
        BackgroundVisible = false,
        BackgroundStyle = state:get("backgroundStyle"),
        WindowSizeIndex = state:get("windowSizeIndex"),
        IsVisible = state:get("isVisible"),
    }
    pcall(function()
        writefile(CONSTANTS.CONFIG_FILE, Services.Http:JSONEncode(data))
    end)
end

function ConfigStore.load(state)
    if not isfile(CONSTANTS.CONFIG_FILE) then return end
    
    local success, data = pcall(function()
        return Services.Http:JSONDecode(readfile(CONSTANTS.CONFIG_FILE))
    end)
    
    if not success or not data then return end
    
    state:set("enabled", data.Enabled or false)
    state:set("powerValue", data.PowerValue or CONSTANTS.DEFAULT_POWER_PC)
    state:set("mode", data.Mode or "PC")
    state:set("backgroundVisible", false)
    state:set("backgroundStyle", data.BackgroundStyle or 1)
    state:set("windowSizeIndex", data.WindowSizeIndex or CONSTANTS.DEFAULT_WINDOW_INDEX)
    state:set("isVisible", data.IsVisible ~= false)
    
    if data.ToggleKey then
        state:set("toggleKey", Enum.KeyCode[data.ToggleKey] or Enum.KeyCode.V)
    end
end

--=====================================================================
-- SECTION 7: UI FACTORY (Element Creation Helpers)
--=====================================================================
local UIFactory = {}

function UIFactory.corner(parent, radius)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius or 4)
    c.Parent = parent
    return c
end

function UIFactory.stroke(parent, color, thickness)
    local s = Instance.new("UIStroke")
    s.Color = color or Theme.Colors.StrokeDefault
    s.Thickness = thickness or 1.5
    s.Parent = parent
    return s
end

function UIFactory.padding(parent, l, t, r, b)
    local p = Instance.new("UIPadding")
    if l then p.PaddingLeft = UDim.new(0, l) end
    if t then p.PaddingTop = UDim.new(0, t) end
    if r then p.PaddingRight = UDim.new(0, r) end
    if b then p.PaddingBottom = UDim.new(0, b) end
    p.Parent = parent
    return p
end

function UIFactory.frame(props)
    local f = Instance.new("Frame")
    for k, v in pairs(props) do f[k] = v end
    return f
end

function UIFactory.label(props)
    local l = Instance.new("TextLabel")
    for k, v in pairs(props) do l[k] = v end
    return l
end

function UIFactory.button(props)
    local b = Instance.new("TextButton")
    for k, v in pairs(props) do b[k] = v end
    b.AutoButtonColor = false
    return b
end

function UIFactory.textbox(props)
    local t = Instance.new("TextBox")
    for k, v in pairs(props) do t[k] = v end
    return t
end

--=====================================================================
-- SECTION 8: DRAG HANDLER
--=====================================================================
local function makeDraggable(frame)
    local dragging, dragStart, startPos = false, nil, nil
    
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
        end
    end)
    
    frame.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    
    frame.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
        end
    end)
end

--=====================================================================
-- SECTION 9: BYPASS ENGINE
--=====================================================================
local BypassEngine = {}
BypassEngine.__index = BypassEngine

function BypassEngine.new(state)
    local self = setmetatable({}, BypassEngine)
    self._state = state
    return self
end

function BypassEngine:_buildBomb(power)
    local spammed = {}
    table.insert(spammed, {})
    local z = spammed[1]
    
    for _ = 1, CONSTANTS.DEPTH do
        local nextTable = {}
        table.insert(z, nextTable)
        z = nextTable
    end
    
    local mainTable = {}
    local reps = math.floor(power / (CONSTANTS.DEPTH + 2))
    for _ = 1, reps do
        table.insert(mainTable, spammed)
    end
    
    return mainTable
end

function BypassEngine:start(power)
    self:stop()
    
    local bypass = self._state:getBypassData()
    bypass.running = true
    bypass.bomb = self:_buildBomb(power)
    
    bypass.thread = task.spawn(function()
        while bypass.running do
            if bypass.bomb then
                pcall(function()
                    game.RobloxReplicatedStorage.SetPlayerBlockList:FireServer(bypass.bomb)
                end)
            end
            task.wait(CONSTANTS.SPAM_DELAY)
        end
    end)
end

function BypassEngine:stop()
    local bypass = self._state:getBypassData()
    bypass.running = false
    if bypass.thread then
        pcall(task.cancel, bypass.thread)
    end
    bypass.bomb = nil
    bypass.thread = nil
end

function BypassEngine:restart(power)
    if self._state:get("enabled") then
        self:start(power)
    end
end

--=====================================================================
-- SECTION 10: UI CONTROLLER
--=====================================================================
local UIController = {}
UIController.__index = UIController

function UIController.new(state)
    local self = setmetatable({}, UIController)
    self._state = state
    self._refs = {}
    self._connections = {}
    self:_build()
    return self
end

function UIController:_build()
    -- Cleanup existing
    local existing = PlayerGui:FindFirstChild(CONSTANTS.GUI_NAME)
    if existing then existing:Destroy() end
    
    -- Root ScreenGui
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = CONSTANTS.GUI_NAME
    screenGui.ResetOnSpawn = false
    screenGui.Parent = PlayerGui
    
    -- Meta value (kept for compatibility)
    local metaValue = Instance.new("BoolValue")
    metaValue.Name = "__g"
    metaValue.Parent = screenGui
    
    -- Main Frame (Siyah düz arka plan)
    local mainFrame = UIFactory.frame({
        Name = "Frame",
        ClipsDescendants = true,
        Position = UDim2.new(0.5, -145, 0.5, -132),
        Size = WindowSizes[self._state:get("windowSizeIndex")],
        BackgroundColor3 = Theme.Colors.Background,
        BackgroundTransparency = Theme.Transparency.Main,
        Parent = screenGui,
    })
    
    local mainStroke = UIFactory.stroke(mainFrame, Theme.Colors.StrokeDefault, 1.5)
    UIFactory.corner(mainFrame, 8)
    
    -- Title Bar
    local titleBar = UIFactory.frame({
        Name = "Frame",
        ZIndex = 2,
        Size = UDim2.new(1, 0, 0, 35),
        BackgroundColor3 = Theme.Colors.BackgroundDark,
        BackgroundTransparency = Theme.Transparency.TitleBar,
        Parent = mainFrame,
    })
    UIFactory.corner(titleBar, 4)
    
    -- Başlık ve Gold renk ayarı
    local titleText = UIFactory.label({
        ZIndex = 3,
        Position = UDim2.new(0, 8, 0, 0),
        Size = UDim2.new(1, -110, 1, 0),
        BackgroundTransparency = 1,
        Text = "Capo X Night Speed Bypaas",
        TextColor3 = Theme.Colors.TextPrimary, -- Gold renk
        TextSize = 12,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = titleBar,
    })
    
    -- Control Buttons Helper
    local function makeTitleButton(text, xOffset, parent)
        local btn = UIFactory.button({
            ZIndex = 5,
            Position = UDim2.new(1, xOffset, 0.5, -10),
            Size = UDim2.new(0, 22, 0, 20),
            BackgroundColor3 = Theme.Colors.BackgroundButtonLight,
            BackgroundTransparency = Theme.Transparency.Button,
            Text = text,
            TextColor3 = Color3.fromRGB(255, 215, 0), -- Gold renk
            TextSize = 12,
            Font = Enum.Font.GothamBold,
            Parent = parent,
        })
        UIFactory.corner(btn, 4)
        UIFactory.stroke(btn, Theme.Colors.StrokeLight, 1)
        return btn
    end
    
    local minimizeBtn = makeTitleButton("_", -28, titleBar)
    local resizeDownBtn = makeTitleButton("-", -54, titleBar)
    local resizeUpBtn = makeTitleButton("+", -80, titleBar)
    
    -- Content Container
    local contentFrame = UIFactory.frame({
        Name = "Frame",
        ZIndex = 2,
        Position = UDim2.new(0, 10, 0, 42),
        Size = UDim2.new(1, -20, 1, -49),
        BackgroundTransparency = 1,
        Parent = mainFrame,
    })
    
    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 6)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = contentFrame
    
    -- Row Builder Helper
    local function buildRow(order, height, labelText, labelScale)
        local row = UIFactory.frame({
            ZIndex = 3,
            LayoutOrder = order,
            Size = UDim2.new(1, 0, 0, height),
            BackgroundColor3 = Theme.Colors.BackgroundMedium,
            BackgroundTransparency = Theme.Transparency.Panel,
            Parent = contentFrame,
        })
        UIFactory.corner(row, 6)
        
        local label = UIFactory.label({
            ZIndex = 4,
            Size = UDim2.new(labelScale or 0.5, 0, 1, 0),
            BackgroundTransparency = 1,
            Text = labelText,
            TextColor3 = Theme.Colors.TextPrimary,
            TextSize = 11,
            Font = Enum.Font.Gotham,
            TextXAlignment = Enum.TextXAlignment.Left,
            Parent = row,
        })
        UIFactory.padding(label, 8)
        
        return row, label
    end
    
    -- STATUS ROW
    local statusRow, _ = buildRow(1, 34, "")
    local statusStroke = UIFactory.stroke(statusRow, Theme.Colors.StrokeDark, 1)
    local statusBtn = UIFactory.button({
        ZIndex = 4,
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text = "STATUS: DISABLED",
        TextColor3 = Theme.Colors.TextSecondary,
        TextSize = 12,
        Font = Enum.Font.GothamBold,
        Parent = statusRow,
    })
    
    -- POWER ROW
    local powerRow = buildRow(2, 32, "Power:")
    local powerInput = UIFactory.textbox({
        ZIndex = 4,
        Position = UDim2.new(0.52, 0, 0.5, -11),
        Size = UDim2.new(0.44, 0, 0, 22),
        BackgroundColor3 = Theme.Colors.BackgroundInput,
        BackgroundTransparency = Theme.Transparency.Button,
        Text = tostring(self._state:get("powerValue")),
        TextColor3 = Theme.Colors.TextAccent,
        TextSize = 12,
        Font = Enum.Font.GothamBold,
        PlaceholderText = "Click to edit",
        Parent = powerRow,
    })
    UIFactory.corner(powerInput, 4)
    UIFactory.stroke(powerInput, Theme.Colors.StrokeInput, 1)
    
    -- MODE ROW
    local modeRow = buildRow(3, 28, "Mode:", 0.28)
    
    local function makeModeButton(text, xPos, parent)
        local btn = UIFactory.button({
            ZIndex = 4,
            Position = UDim2.new(xPos, 0, 0.5, -10),
            Size = UDim2.new(0.3, -6, 0, 20),
            BackgroundColor3 = Theme.Colors.BackgroundButton,
            BackgroundTransparency = Theme.Transparency.Button,
            Text = text,
            TextColor3 = Theme.Colors.TextSecondary,
            TextSize = 10,
            Font = Enum.Font.GothamBold,
            Parent = parent,
        })
        UIFactory.corner(btn, 5)
        local stroke = UIFactory.stroke(btn, Theme.Colors.StrokeDark, 1)
        return btn, stroke
    end
    
    local mobileBtn, mobileStroke = makeModeButton("Mobile", 0.4, modeRow)
    local pcBtn, pcStroke = makeModeButton("PC", 0.7, modeRow)
    
    -- TOGGLE KEY ROW
    local keyRow = buildRow(4, 28, "Toggle Key:", 0.55)
    local keyBtn = UIFactory.button({
        ZIndex = 4,
        Position = UDim2.new(0.56, 0, 0.5, -10),
        Size = UDim2.new(0.44, -6, 0, 20),
        BackgroundColor3 = Theme.Colors.BackgroundButton,
        BackgroundTransparency = Theme.Transparency.Button,
        Text = self._state:get("toggleKey").Name,
        TextColor3 = Theme.Colors.TextPrimary,
        TextSize = 11,
        Font = Enum.Font.GothamBold,
        Parent = keyRow,
    })
    UIFactory.corner(keyBtn, 5)
    UIFactory.stroke(keyBtn, Theme.Colors.StrokeDark, 1)
    
    makeDraggable(mainFrame)
    
    -- Store references
    self._refs = {
        screenGui = screenGui,
        mainFrame = mainFrame,
        mainStroke = mainStroke,
        statusBtn = statusBtn,
        statusStroke = statusStroke,
        powerInput = powerInput,
        mobileBtn = mobileBtn,
        mobileStroke = mobileStroke,
        pcBtn = pcBtn,
        pcStroke = pcStroke,
        keyBtn = keyBtn,
        minimizeBtn = minimizeBtn,
        resizeDownBtn = resizeDownBtn,
        resizeUpBtn = resizeUpBtn,
        titleBar = titleBar,
        contentFrame = contentFrame,
    }
    
    -- Apply initial visibility state
    if not self._state:get("isVisible") then
        contentFrame.Visible = false
        mainFrame.Size = UDim2.new(0, WindowSizes[self._state:get("windowSizeIndex")].X.Offset, 0, 35)
    end
end

function UIController:getRefs()
    return self._refs
end

-- UI Update Methods
function UIController:refreshStatus()
    local refs = self._refs
    local enabled = self._state:get("enabled")
    local color = enabled and Theme.Colors.Enabled or Theme.Colors.StrokeDefault
    
    refs.mainStroke.Color = color
    refs.statusBtn.Text = enabled and "STATUS: ENABLED" or "STATUS: DISABLED"
    refs.statusBtn.TextColor3 = enabled and Theme.Colors.Enabled or Theme.Colors.TextSecondary
    refs.statusStroke.Color = color
end

function UIController:refreshMode()
    local refs = self._refs
    local mode = self._state:get("mode")
    
    local function setActive(btn, stroke)
        btn.BackgroundColor3 = Theme.Colors.Accent
        btn.BackgroundTransparency = Theme.Transparency.Active
        btn.TextColor3 = Theme.Colors.TextPrimary
        stroke.Color = Theme.Colors.TextAccent
    end
    
    local function setInactive(btn, stroke)
        btn.BackgroundColor3 = Theme.Colors.BackgroundButton
        btn.BackgroundTransparency = Theme.Transparency.Button
        btn.TextColor3 = Theme.Colors.TextSecondary
        stroke.Color = Theme.Colors.StrokeDark
    end
    
    if mode == "PC" then
        setActive(refs.pcBtn, refs.pcStroke)
        setInactive(refs.mobileBtn, refs.mobileStroke)
    else
        setActive(refs.mobileBtn, refs.mobileStroke)
        setInactive(refs.pcBtn, refs.pcStroke)
    end
end

function UIController:refreshWindowSize()
    local refs = self._refs
    local idx = self._state:get("windowSizeIndex")
    if self._state:get("isVisible") then
        refs.mainFrame.Size = WindowSizes[idx]
    else
        refs.mainFrame.Size = UDim2.new(0, WindowSizes[idx].X.Offset, 0, 35)
    end
end

function UIController:destroy()
    for _, conn in ipairs(self._connections) do
        pcall(function() conn:Disconnect() end)
    end
    if self._refs.screenGui then
        self._refs.screenGui:Destroy()
    end
end

--=====================================================================
-- SECTION 11: EVENT BINDING (Controller Layer)
--=====================================================================
local function bindEvents(ui, state, bypass)
    local refs = ui:getRefs()
    
    -- STATUS TOGGLE
    refs.statusBtn.MouseButton1Click:Connect(function()
        local enabled = state:toggle("enabled")
        ui:refreshStatus()
        
        if enabled then
            state:setBypassField("startedAt", os.clock())
            bypass:start(state:get("powerValue"))
        else
            bypass:stop()
        end
        ConfigStore.save(state)
    end)
    
    -- POWER INPUT
    refs.powerInput.FocusLost:Connect(function()
        local val = tonumber(refs.powerInput.Text)
        if val then
            val = math.clamp(math.floor(val + 0.5), CONSTANTS.MIN_POWER, CONSTANTS.MAX_POWER)
            state:set("powerValue", val)
            refs.powerInput.Text = tostring(val)
            bypass:restart(val)
        else
            refs.powerInput.Text = tostring(state:get("powerValue"))
        end
        ConfigStore.save(state)
    end)
    
    -- MODE BUTTONS
    refs.mobileBtn.MouseButton1Click:Connect(function()
        state:set("mode", "Mobile")
        state:set("powerValue", CONSTANTS.DEFAULT_POWER_MOBILE)
        refs.powerInput.Text = tostring(CONSTANTS.DEFAULT_POWER_MOBILE)
        ui:refreshMode()
        bypass:restart(state:get("powerValue"))
        ConfigStore.save(state)
    end)
    
    refs.pcBtn.MouseButton1Click:Connect(function()
        state:set("mode", "PC")
        state:set("powerValue", CONSTANTS.DEFAULT_POWER_PC)
        refs.powerInput.Text = tostring(CONSTANTS.DEFAULT_POWER_PC)
        ui:refreshMode()
        bypass:restart(state:get("powerValue"))
        ConfigStore.save(state)
    end)
    
    -- KEYBIND CAPTURE
    refs.keyBtn.MouseButton1Click:Connect(function()
        refs.keyBtn.Text = "..."
        refs.keyBtn.TextColor3 = Theme.Colors.TextWarning
        
        local conn
        conn = Services.UserInput.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Keyboard then
                state:set("toggleKey", input.KeyCode)
                refs.keyBtn.Text = input.KeyCode.Name
                refs.keyBtn.TextColor3 = Theme.Colors.TextPrimary
                conn:Disconnect()
                ConfigStore.save(state)
            end
        end)
    end)
    
    -- MINIMIZE
    refs.minimizeBtn.MouseButton1Click:Connect(function()
        local visible = state:toggle("isVisible")
        refs.contentFrame.Visible = visible
        ui:refreshWindowSize()
        ConfigStore.save(state)
    end)
    
    -- RESIZE DOWN
    refs.resizeDownBtn.MouseButton1Click:Connect(function()
        local idx = state:get("windowSizeIndex")
        idx = (idx > 1) and (idx - 1) or #WindowSizes
        state:set("windowSizeIndex", idx)
        ui:refreshWindowSize()
        ConfigStore.save(state)
    end)
    
    -- RESIZE UP
    refs.resizeUpBtn.MouseButton1Click:Connect(function()
        local idx = state:get("windowSizeIndex")
        idx = (idx < #WindowSizes) and (idx + 1) or 1
        state:set("windowSizeIndex", idx)
        ui:refreshWindowSize()
        ConfigStore.save(state)
    end)
end

--=====================================================================
-- SECTION 12: KEYBIND LISTENER
--=====================================================================
local function setupKeybind(ui, state, bypass)
    Services.UserInput.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Keyboard 
           and input.KeyCode == state:get("toggleKey") then
            local enabled = state:toggle("enabled")
            ui:refreshStatus()
            
            if enabled then
                state:setBypassField("startedAt", os.clock())
                bypass:start(state:get("powerValue"))
            else
                bypass:stop()
            end
            ConfigStore.save(state)
        end
    end)
end

--=====================================================================
-- SECTION 13: APPLICATION BOOTSTRAP
--=====================================================================
local function bootstrap()
    if not Services.Players.LocalPlayer then
        Services.Players.PlayerAdded:Wait()
    end
    
    -- Initialize state
    local state = StateManager.new()
    ConfigStore.load(state)
    
    -- Initialize bypass engine
    local bypass = BypassEngine.new(state)
    
    -- Build UI
    local ui = UIController.new(state)
    
    -- Bind all events
    bindEvents(ui, state, bypass)
    setupKeybind(ui, state, bypass)
    
    -- Apply loaded state to UI
    ui:refreshStatus()
    ui:refreshMode()
    
    -- Start bypass if it was enabled
    if state:get("enabled") then
        state:setBypassField("startedAt", os.clock())
        bypass:start(state:get("powerValue"))
    end
    
    return { state = state, ui = ui, bypass = bypass }
end

-- Launch Application
local App = bootstrap()

-- Expose for external access (optional)
_G.Speedbypaas = App