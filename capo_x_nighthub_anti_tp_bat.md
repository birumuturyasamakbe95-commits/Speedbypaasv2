print("DEOBF BY NOXA DUELS ON TOP")

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local NetworkClient = game:GetService("NetworkClient")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer
local environment = if getgenv then getgenv() else \_G
local RUNTIME_KEY = "\__HOOK_ANTI_ANTI_RECONSTRUCTION"
local previousRuntime = environment\[RUNTIME_KEY\]
if type(previousRuntime) == "table" and type(previousRuntime.destroy) == "function" then
pcall(previousRuntime.destroy)
end
local runtime = {
alive = true,
enabled = false,
awaitingKey = false,
boundKey = Enum.KeyCode.Q,
character = nil,
rootPart = nil,
fakeRoot = nil,
repRootOwner = nil,
stepConnection = nil,
connections = {},
settingsRestore = {},
captureGeneration = 0,
}
environment\[RUNTIME_KEY\] = runtime
local function connect(signal, callback)
local connection = signal:Connect(callback)
table.insert(runtime.connections, connection)
return connection
end
local function disconnect(connection)
if connection then
pcall(function()
connection:Disconnect()
end)
end
end
local function create(className, properties, parent)
local object = Instance.new(className)
for property, value in pairs(properties or {}) do
object\[property\] = value
end
if parent then
object.Parent = parent
end
return object
end
local function corner(parent, radius)
return create("UICorner", {
CornerRadius = typeof(radius) == "UDim" and radius or UDim.new(0, radius),
}, parent)
end
local function stroke(parent, color, transparency, thickness)
return create("UIStroke", {
ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
Color = color,
Transparency = transparency,
Thickness = thickness,
}, parent)
end
local function tween(object, duration, goals)
local animation = TweenService:Create(
object,
TweenInfo.new(duration, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
goals
)
animation:Play()
return animation
end
local function isBasePart(instance)
if not instance then
return false
end
local ok, result = pcall(function()
return instance:IsA("BasePart")
end)
return ok and result == true
end
local function getCurrentRoot(character)
character = character or LocalPlayer.Character
if not character then
return nil
end
local ok, root = pcall(function()
return character:FindFirstChild("HumanoidRootPart")
end)
if ok and isBasePart(root) then
return root
end
return nil
end
local function findGlobalFunction(...)
for index = 1, select("#", ...) do
local name = select(index, ...)
local value = rawget(environment, name)
if type(value) == "function" then
return value
end
end
return nil
end
local function setHidden(instance, property, value)
if not instance then
return false
end
local setter = findGlobalFunction(
"sethiddenproperty",
"set_hidden_property",
"sethiddenprop",
"set_hidden_prop"
)
if setter then
local ok = pcall(setter, instance, property, value)
if ok then
return true
end
end
return pcall(function()
instance\[property\] = value
end)
end
local function getHidden(instance, property)
if not instance then
return false, nil
end
local getter = findGlobalFunction(
"gethiddenproperty",
"get_hidden_property",
"gethiddenprop",
"get_hidden_prop"
)
if getter then
local ok, value = pcall(getter, instance, property)
if ok then
return true, value
end
end
local ok, value = pcall(function()
return instance\[property\]
end)
return ok, value
end
local function rememberSetting(instance, property)
local ok, value = pcall(function()
return instance\[property\]
end)
if ok then
table.insert(runtime.settingsRestore, {
instance = instance,
property = property,
value = value,
})
end
end
local function applyPublicSetting(instance, property, value)
if not instance then
return false
end
rememberSetting(instance, property)
return pcall(function()
instance\[property\] = value
end)
end
local function configurePhysics()
setHidden(LocalPlayer, "MaximumSimulationRadius", math.huge)
setHidden(LocalPlayer, "SimulationRadius", math.huge)
pcall(function()
local networkSettings = settings().Network
applyPublicSetting(
networkSettings,
"InterpolationThrottling",
Enum.InterpolationThrottlingMode.Disabled
)
end)
pcall(function()
local physicsSettings = settings().Physics
applyPublicSetting(
physicsSettings,
"PhysicsEnvironmentalThrottle",
Enum.EnviromentalPhysicsThrottle.Disabled
)
applyPublicSetting(physicsSettings, "AllowSleep", false)
end)
pcall(function()
NetworkClient:SetOutgoingKBPSLimit(math.huge)
end)
end
configurePhysics()

local FAKE_ROOT_NAME = "DavidDesyncRoot"
local FAKE_ROOT_Y = -2500
local FAKE_ROOT_VELOCITY = Vector3.new(0, -1000, 0)

local function fakeRootIsUsable()
local fake = runtime.fakeRoot
if not isBasePart(fake) then
return false
end
local ok, parent = pcall(function()
return fake.Parent
end)
return ok and parent \~= nil
end

local function destroyFakeRoot()
local fake = runtime.fakeRoot
runtime.fakeRoot = nil
if fake then
pcall(function()
fake:Destroy()
end)
end
end

local function restoreReplicationRoot()
local owner = runtime.repRootOwner or runtime.rootPart
if isBasePart(owner) then
setHidden(owner, "PhysicsRepRootPart", owner)
end
runtime.repRootOwner = nil
end

local function createFakeRoot(rootPart)
destroyFakeRoot()
local fake = create("Part", {
Name = FAKE_ROOT_NAME,
Size = Vector3.new(2, 2, 1),
Anchored = true,
CanCollide = false,
CanTouch = false,
CanQuery = false,
Transparency = 1,
CFrame = CFrame.new(0, FAKE_ROOT_Y, 0),
AssemblyLinearVelocity = FAKE_ROOT_VELOCITY,
}, Workspace)
local ok, position = pcall(function()
return rootPart.Position
end)
if ok then
fake.CFrame = CFrame.new(position.X, FAKE_ROOT_Y, position.Z)
end
runtime.fakeRoot = fake
return fake
end

local function assignFakeReplicationRoot(rootPart, fake)
if not isBasePart(rootPart) or not isBasePart(fake) then
return false
end
setHidden(rootPart, "PhysicsRepRootPart", rootPart)
runtime.repRootOwner = rootPart
return setHidden(rootPart, "PhysicsRepRootPart", fake)
end

local function stepDesync()
if not runtime.alive or not runtime.enabled then
return
end
local root = runtime.rootPart
if not isBasePart(root) then
root = getCurrentRoot(runtime.character)
runtime.rootPart = root
end
if not root then
return
end
if not fakeRootIsUsable() then
local fake = createFakeRoot(root)
assignFakeReplicationRoot(root, fake)
return
end
local fake = runtime.fakeRoot
local ok, rootPosition, fakePosition = pcall(function()
return root.Position, fake.Position
end)
if ok and (
math.abs(rootPosition.X - fakePosition.X) > 0.01
or math.abs(rootPosition.Z - fakePosition.Z) > 0.01
or math.abs(fakePosition.Y - FAKE_ROOT_Y) > 0.01
) then
pcall(function()
fake.CFrame = CFrame.new(rootPosition.X, FAKE_ROOT_Y, rootPosition.Z)
end)
end
pcall(function()
fake.Anchored = true
fake.AssemblyLinearVelocity = FAKE_ROOT_VELOCITY
end)
local gotValue, current = getHidden(root, "PhysicsRepRootPart")
if not gotValue or current \~= fake then
setHidden(root, "PhysicsRepRootPart", fake)
end
end

local function stopStepConnection()
disconnect(runtime.stepConnection)
runtime.stepConnection = nil
end

local function startStepConnection()
stopStepConnection()
runtime.stepConnection = RunService.Stepped:Connect(stepDesync)
end

local function bindCharacter(character)
local oldRoot = runtime.rootPart
runtime.character = character
runtime.rootPart = getCurrentRoot(character)
if runtime.enabled then
if isBasePart(oldRoot) and oldRoot \~= runtime.rootPart then
setHidden(oldRoot, "PhysicsRepRootPart", oldRoot)
end
destroyFakeRoot()
local root = runtime.rootPart
if not root and character then
local ok, waitedRoot = pcall(function()
return character:WaitForChild("HumanoidRootPart", 8)
end)
if ok and isBasePart(waitedRoot) then
root = waitedRoot
runtime.rootPart = root
end
end
if root then
local fake = createFakeRoot(root)
assignFakeReplicationRoot(root, fake)
startStepConnection()
end
end
end

bindCharacter(LocalPlayer.Character)
connect(LocalPlayer.CharacterAdded, function(character)
task.defer(bindCharacter, character)
end)

local COLORS = {
main = Color3.fromRGB(0, 0, 0),
row = Color3.fromRGB(15, 15, 15),
track = Color3.fromRGB(25, 25, 25),
button = Color3.fromRGB(20, 20, 20),
text = Color3.fromRGB(255, 255, 255),
muted = Color3.fromRGB(150, 150, 150),
accent = Color3.fromRGB(255, 215, 0),
rowStroke = Color3.fromRGB(50, 50, 50),
}

local HookAntiAntiUI = Instance.new("ScreenGui")
HookAntiAntiUI.Name = "HookAntiAntiUI"
HookAntiAntiUI.IgnoreGuiInset = true
HookAntiAntiUI.ResetOnSpawn = false
HookAntiAntiUI.DisplayOrder = 999
HookAntiAntiUI.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
HookAntiAntiUI.Parent = game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")

local Main = Instance.new("Frame")
Main.Name = "Main"
Main.Active = true
Main.ClipsDescendants = true
Main.Position = UDim2.new(0.5, -130, 0.5, -70)
Main.Size = UDim2.new(0, 260, 0, 140)
Main.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
Main.BackgroundTransparency = 0.05
Main.BorderSizePixel = 0
Main.Parent = HookAntiAntiUI

local UICorner = Instance.new("UICorner")
UICorner.Name = "UICorner"
UICorner.CornerRadius = UDim.new(0, 12)
UICorner.Parent = Main

local UIStroke = Instance.new("UIStroke")
UIStroke.Name = "UIStroke"
UIStroke.Color = Color3.fromRGB(255, 215, 0)
UIStroke.Thickness = 1.5
UIStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
UIStroke.Parent = Main

local UIScale = Instance.new("UIScale")
UIScale.Name = "UIScale"
UIScale.Parent = Main

local Frame = Instance.new("Frame")
Frame.Name = "HeaderFrame"
Frame.ZIndex = 10
Frame.Position = UDim2.new(0, 10, 0, 4)
Frame.Size = UDim2.new(1, -20, 0, 32)
Frame.BackgroundTransparency = 1
Frame.Parent = Main

local TextLabel = Instance.new("TextLabel")
TextLabel.Name = "TitleLabel"
TextLabel.ZIndex = 11
TextLabel.Position = UDim2.new(0, 0, 0, 0)
TextLabel.Size = UDim2.new(1, 0, 1, 0)
TextLabel.BackgroundTransparency = 1
TextLabel.Text = "Capo X Nighthub Anti Tp bat"
TextLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TextLabel.TextSize = 13
TextLabel.Font = Enum.Font.GothamBlack
TextLabel.TextXAlignment = Enum.TextXAlignment.Left
TextLabel.Parent = Frame

local TitleGradient = Instance.new("UIGradient")
TitleGradient.Name = "TitleGradient"
TitleGradient.Color = ColorSequence.new({
ColorSequenceKeypoint.new(0, Color3.fromRGB(212, 175, 55)),  -- Gold
ColorSequenceKeypoint.new(0.25, Color3.fromRGB(255, 255, 255)), -- Sarı beyaz
ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 255, 0)),   -- Sarı
ColorSequenceKeypoint.new(0.75, Color3.fromRGB(255, 255, 255)), -- Beyaz
ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 215, 0))     -- Gold
})
TitleGradient.Rotation = 0
TitleGradient.Parent = TextLabel

local Frame2 = Instance.new("Frame")
Frame2.Name = "ContentFrame"
Frame2.ZIndex = 5
Frame2.Position = UDim2.new(0, 10, 0, 38)
Frame2.Size = UDim2.new(1, -20, 1, -44)
Frame2.BackgroundTransparency = 1
Frame2.Parent = Main

local UIListLayout = Instance.new("UIListLayout")
UIListLayout.Name = "UIListLayout"
UIListLayout.Padding = UDim.new(0, 6)
UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
UIListLayout.Parent = Frame2

local Frame3 = Instance.new("Frame")
Frame3.Name = "ToggleRow"
Frame3.ZIndex = 5
Frame3.ClipsDescendants = true
Frame3.Size = UDim2.new(1, 0, 0, 40)
Frame3.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
Frame3.BorderSizePixel = 0
Frame3.Parent = Frame2

local UICorner4 = Instance.new("UICorner")
UICorner4.CornerRadius = UDim.new(0, 8)
UICorner4.Parent = Frame3

local UIStroke3 = Instance.new("UIStroke")
UIStroke3.Color = Color3.fromRGB(40, 40, 40)
UIStroke3.Thickness = 1
UIStroke3.Parent = Frame3

local TextLabel2 = Instance.new("TextLabel")
TextLabel2.ZIndex = 6
TextLabel2.Position = UDim2.new(0, 10, 0, 4)
TextLabel2.Size = UDim2.new(1, -60, 0, 16)
TextLabel2.BackgroundTransparency = 1
TextLabel2.Text = "Enable Anti Anti"
TextLabel2.TextColor3 = Color3.fromRGB(255, 255, 255)
TextLabel2.TextSize = 11
TextLabel2.Font = Enum.Font.GothamBold
TextLabel2.TextXAlignment = Enum.TextXAlignment.Left
TextLabel2.Parent = Frame3

local TextLabel3 = Instance.new("TextLabel")
TextLabel3.ZIndex = 6
TextLabel3.Position = UDim2.new(0, 10, 0, 20)
TextLabel3.Size = UDim2.new(1, -60, 0, 14)
TextLabel3.BackgroundTransparency = 1
TextLabel3.Text = "OFF"
TextLabel3.TextColor3 = COLORS.muted
TextLabel3.TextSize = 9
TextLabel3.Font = Enum.Font.GothamBold
TextLabel3.TextXAlignment = Enum.TextXAlignment.Left
TextLabel3.Parent = Frame3

local Frame4 = Instance.new("Frame")
Frame4.ZIndex = 7
Frame4.AnchorPoint = Vector2.new(1, 0.5)
Frame4.Position = UDim2.new(1, -8, 0.5, 0)
Frame4.Size = UDim2.new(0, 36, 0, 18)
Frame4.BackgroundColor3 = COLORS.track
Frame4.BorderSizePixel = 0
Frame4.Parent = Frame3

local UICorner5 = Instance.new("UICorner")
UICorner5.CornerRadius = UDim.new(0, 9)
UICorner5.Parent = Frame4

local Frame5 = Instance.new("Frame")
Frame5.ZIndex = 8
Frame5.Position = UDim2.new(0, 2, 0, 2)
Frame5.Size = UDim2.new(0, 14, 0, 14)
Frame5.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
Frame5.BorderSizePixel = 0
Frame5.Parent = Frame4

local UICorner6 = Instance.new("UICorner")
UICorner6.CornerRadius = UDim.new(1, 0)
UICorner6.Parent = Frame5

local ToggleHit = Instance.new("TextButton")
ToggleHit.ZIndex = 9
ToggleHit.Size = UDim2.new(1, 0, 1, 0)
ToggleHit.BackgroundTransparency = 1
ToggleHit.Text = ""
ToggleHit.Parent = Frame3

local Frame6 = Instance.new("Frame")
Frame6.Name = "KeybindRow"
Frame6.ZIndex = 5
Frame6.ClipsDescendants = true
Frame6.Size = UDim2.new(1, 0, 0, 40)
Frame6.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
Frame6.BorderSizePixel = 0
Frame6.Parent = Frame2

local UICorner7 = Instance.new("UICorner")
UICorner7.CornerRadius = UDim.new(0, 8)
UICorner7.Parent = Frame6

local UIStroke6 = Instance.new("UIStroke")
UIStroke6.Color = Color3.fromRGB(40, 40, 40)
UIStroke6.Thickness = 1
UIStroke6.Parent = Frame6

local TextLabel4 = Instance.new("TextLabel")
TextLabel4.ZIndex = 6
TextLabel4.Position = UDim2.new(0, 10, 0, 0)
TextLabel4.Size = UDim2.new(1, -70, 1, 0)
TextLabel4.BackgroundTransparency = 1
TextLabel4.Text = "Keybind"
TextLabel4.TextColor3 = Color3.fromRGB(255, 255, 255)
TextLabel4.TextSize = 11
TextLabel4.Font = Enum.Font.GothamBold
TextLabel4.TextXAlignment = Enum.TextXAlignment.Left
TextLabel4.Parent = Frame6

local KeybindBtn = Instance.new("TextButton")
KeybindBtn.ZIndex = 7
KeybindBtn.AnchorPoint = Vector2.new(1, 0.5)
KeybindBtn.Position = UDim2.new(1, -8, 0.5, 0)
KeybindBtn.Size = UDim2.new(0, 60, 0, 22)
KeybindBtn.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
KeybindBtn.Text = "Q"
KeybindBtn.TextColor3 = Color3.fromRGB(255, 215, 0)
KeybindBtn.TextSize = 10
KeybindBtn.Font = Enum.Font.GothamBlack
KeybindBtn.AutoButtonColor = false
KeybindBtn.Parent = Frame6

local UICorner8 = Instance.new("UICorner")
UICorner8.CornerRadius = UDim.new(0, 5)
UICorner8.Parent = KeybindBtn

runtime.gui = HookAntiAntiUI
runtime.refs = {
screenGui = HookAntiAntiUI,
main = Main,
mainScale = UIScale,
mainStroke = UIStroke,
header = Frame,
title = TextLabel,
content = Frame2,
toggleRow = Frame3,
toggleLabel = TextLabel2,
statusLabel = TextLabel3,
toggleTrack = Frame4,
toggleKnob = Frame5,
toggleHit = ToggleHit,
keybindRow = Frame6,
keybindLabel = TextLabel4,
keybindButton = KeybindBtn,
}

local function ripple(row)
if not runtime.alive or not row or not row.Parent then
return
end
local mousePosition = UserInputService:GetMouseLocation()
local absolutePosition = row.AbsolutePosition
local absoluteSize = row.AbsoluteSize
local x = mousePosition.X - absolutePosition.X
local y = mousePosition.Y - absolutePosition.Y
local diameter = math.max(absoluteSize.X, absoluteSize.Y) \* 1.35
local image = create("ImageLabel", {
Name = "Ripple",
BackgroundTransparency = 1,
Image = "rbxassetid://266543268",
ImageColor3 = COLORS.accent,
ImageTransparency = 0.40,
AnchorPoint = Vector2.new(0.5, 0.5),
Position = UDim2.new(0, x, 0, y),
Size = UDim2.new(0, 0, 0, 0),
ZIndex = 30,
}, row)
local animation = tween(image, 0.45, {
Size = UDim2.new(0, diameter, 0, diameter),
ImageTransparency = 1,
})
animation.Completed:Connect(function()
if image then
image:Destroy()
end
end)
end

local function applyEnabledVisual(value, instant)
TextLabel3.Text = value and "ACTIVE" or "OFF"
TextLabel3.TextColor3 = value and COLORS.accent or COLORS.muted
local trackColor = value and Color3.fromRGB(180, 140, 0) or COLORS.track
local knobPosition = value and UDim2.new(1, -16, 0, 2) or UDim2.new(0, 2, 0, 2)
if instant then
Frame4.BackgroundColor3 = trackColor
Frame5.Position = knobPosition
else
tween(Frame4, 0.18, { BackgroundColor3 = trackColor })
tween(Frame5, 0.18, { Position = knobPosition })
end
end

local function setEnabled(value)
if not runtime.alive then
return false
end
value = value == true
if runtime.enabled == value then
applyEnabledVisual(value, false)
return value
end
runtime.enabled = value
applyEnabledVisual(value, false)
if value then
local root = getCurrentRoot(runtime.character)
runtime.rootPart = root
if not root then
runtime.enabled = false
applyEnabledVisual(false, false)
return false
end
local fake = createFakeRoot(root)
assignFakeReplicationRoot(root, fake)
startStepConnection()
else
stopStepConnection()
restoreReplicationRoot()
destroyFakeRoot()
end
return runtime.enabled
end

local function toggleEnabled()
return setEnabled(not runtime.enabled)
end

connect(ToggleHit.MouseButton1Click, function()
ripple(Frame3)
toggleEnabled()
end)

connect(KeybindBtn.MouseButton1Click, function()
if not runtime.alive or runtime.awaitingKey then
return
end
ripple(Frame6)
runtime.awaitingKey = true
runtime.captureGeneration += 1
local generation = runtime.captureGeneration
task.spawn(function()
for \_, text in ipairs({".", "..", "..."}) do
if not runtime.alive or not runtime.awaitingKey or generation \~= runtime.captureGeneration then
return
end
KeybindBtn.Text = text
task.wait(0.15)
end
end)
end)

connect(UserInputService.InputBegan, function(input, gameProcessed)
if not runtime.alive then
return
end
if runtime.awaitingKey then
if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode \~= Enum.KeyCode.Unknown then
if input.KeyCode \~= Enum.KeyCode.Escape then
runtime.boundKey = input.KeyCode
end
runtime.awaitingKey = false
runtime.captureGeneration += 1
KeybindBtn.Text = runtime.boundKey.Name
end
return
end
if not gameProcessed and input.KeyCode == runtime.boundKey then
ripple(Frame3)
toggleEnabled()
end
end)

local dragging = false
local dragInput = nil
local dragStart = nil
local startPosition = nil

connect(Main.InputBegan, function(input)
if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
dragging = true
dragInput = input
dragStart = input.Position
startPosition = Main.Position
local changedConnection
changedConnection = input.Changed:Connect(function()
if input.UserInputState == Enum.UserInputState.End then
dragging = false
dragInput = nil
disconnect(changedConnection)
end
end)
end
end)

connect(Main.InputChanged, function(input)
if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
dragInput = input
end
end)

connect(UserInputService.InputChanged, function(input)
if not dragging or input \~= dragInput or not dragStart or not startPosition then
return
end
local delta = input.Position - dragStart
Main.Position = UDim2.new(
startPosition.X.Scale,
startPosition.X.Offset + delta.X,
startPosition.Y.Scale,
startPosition.Y.Offset + delta.Y
)
end)

local function destroy()
if not runtime.alive then
return
end
runtime.alive = false
runtime.enabled = false
runtime.awaitingKey = false
runtime.captureGeneration += 1
stopStepConnection()
restoreReplicationRoot()
destroyFakeRoot()
for \_, connection in ipairs(runtime.connections) do
disconnect(connection)
end
table.clear(runtime.connections)
for index = #runtime.settingsRestore, 1, -1 do
local entry = runtime.settingsRestore\[index\]
pcall(function()
entry.instance\[entry.property\] = entry.value
end)
end
table.clear(runtime.settingsRestore)
if runtime.gui then
pcall(function()
runtime.gui:Destroy()
end)
end
if environment\[RUNTIME_KEY\] == runtime then
environment\[RUNTIME_KEY\] = nil
end
end

runtime.setEnabled = setEnabled
runtime.toggle = toggleEnabled
runtime.step = stepDesync
runtime.bindCharacter = bindCharacter
runtime.setHidden = setHidden
runtime.getHidden = getHidden
runtime.destroy = destroy
runtime.getBoundKey = function()
return runtime.boundKey
end
runtime.setBoundKey = function(keyCode)
if keyCode and keyCode \~= Enum.KeyCode.Unknown then
runtime.boundKey = keyCode
KeybindBtn.Text = keyCode.Name
return true
end
return false
end

applyEnabledVisual(false, true)