
-- MINE A MOUNTAIN HUB | NO STOP FLY BUTTON

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local LP = Players.LocalPlayer
local PG = LP:WaitForChild("PlayerGui")

local TARGET_NAME = "Nocturnite"
local FLY_SPEED = 1000
local LAND_HEIGHT = 12
local ARRIVAL_DISTANCE = 3

local ESP_ENABLED = true
local flying = false
local target
local flyConnection
local savedCollisions = {}
local playerESPs = {}
local targetESP
local targetESPConnection
local minimized = false

-- GUI
local oldGui = PG:FindFirstChild("MineMountainHub")
if oldGui then oldGui:Destroy() end

local gui = Instance.new("ScreenGui")
gui.Name = "MineMountainHub"
gui.ResetOnSpawn = false
gui.Parent = PG

local frame = Instance.new("Frame")
frame.Size = UDim2.fromOffset(170, 154)
frame.Position = UDim2.new(1, -180, 0.5, -77)
frame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
frame.BorderSizePixel = 0
frame.Active = true
frame.Parent = gui
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)

local stroke = Instance.new("UIStroke", frame)
stroke.Color = Color3.fromRGB(140, 70, 255)
stroke.Thickness = 1.5

local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 30)
header.BackgroundColor3 = Color3.fromRGB(38, 28, 55)
header.BorderSizePixel = 0
header.Active = true
header.Parent = frame
Instance.new("UICorner", header).CornerRadius = UDim.new(0, 8)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -40, 1, 0)
title.Position = UDim2.fromOffset(8, 0)
title.BackgroundTransparency = 1
title.Text = "MINE A MOUNTAIN"
title.TextColor3 = Color3.fromRGB(220, 190, 255)
title.TextSize = 11
title.Font = Enum.Font.GothamBold
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

local minimize = Instance.new("TextButton")
minimize.Size = UDim2.fromOffset(26, 24)
minimize.Position = UDim2.new(1, -30, 0, 3)
minimize.BackgroundColor3 = Color3.fromRGB(75, 50, 100)
minimize.Text = "—"
minimize.TextColor3 = Color3.new(1, 1, 1)
minimize.TextSize = 15
minimize.Font = Enum.Font.GothamBold
minimize.Parent = header
Instance.new("UICorner", minimize).CornerRadius = UDim.new(0, 5)

local content = Instance.new("Frame")
content.Size = UDim2.new(1, -14, 1, -38)
content.Position = UDim2.fromOffset(7, 35)
content.BackgroundTransparency = 1
content.Parent = frame

local layout = Instance.new("UIListLayout", content)
layout.Padding = UDim.new(0, 5)
layout.SortOrder = Enum.SortOrder.LayoutOrder

local function makeButton(text, order)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, 0, 0, 27)
    b.BackgroundColor3 = Color3.fromRGB(48, 35, 67)
    b.BorderSizePixel = 0
    b.Text = text
    b.TextColor3 = Color3.fromRGB(245, 240, 255)
    b.TextSize = 11
    b.Font = Enum.Font.GothamBold
    b.LayoutOrder = order
    b.Parent = content
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 5)
    return b
end

local flyButton = makeButton("FLY TO NOCTURNITE", 1)
local rescanButton = makeButton("RESCAN TARGET", 2)
local espButton = makeButton("ESP: ON", 3)

-- DRAG GUI
local dragging, dragStart, startPosition, dragInput

header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        dragStart = input.Position
        startPosition = frame.Position

        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

header.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input == dragInput then
        local d = input.Position - dragStart
        frame.Position = UDim2.new(
            startPosition.X.Scale,
            startPosition.X.Offset + d.X,
            startPosition.Y.Scale,
            startPosition.Y.Offset + d.Y
        )
    end
end)

minimize.MouseButton1Click:Connect(function()
    minimized = not minimized
    content.Visible = not minimized
    frame.Size = minimized
        and UDim2.fromOffset(170, 34)
        or UDim2.fromOffset(170, 154)
    minimize.Text = minimized and "+" or "—"
end)

-- FIND TARGET
local function findTarget()
    target = nil

    for _, obj in ipairs(Workspace:GetDescendants()) do
        if string.lower(obj.Name) == string.lower(TARGET_NAME) then
            if obj:IsA("BasePart") then
                target = obj
                break
            elseif obj:IsA("Model") then
                target = obj.PrimaryPart
                    or obj:FindFirstChildWhichIsA("BasePart", true)
                if target then break end
            end
        end
    end

    return target
end

-- RESTORE COLLISIONS
local function restoreCollisions()
    for part, original in pairs(savedCollisions) do
        if part and part.Parent then
            part.CanCollide = original
        end
    end
    table.clear(savedCollisions)
end

-- INTERNAL STOP FLY
-- Tetap diperlukan untuk berhenti otomatis.
local function stopFly()
    flying = false

    if flyConnection then
        flyConnection:Disconnect()
        flyConnection = nil
    end

    restoreCollisions()

    local character = LP.Character
    local humanoid = character
        and character:FindFirstChildOfClass("Humanoid")

    if humanoid then
        humanoid.PlatformStand = false
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end

    flyButton.Text = "FLY TO NOCTURNITE"
end

-- FLY
local function startFly()
    if flying then return end

    if not target or not target.Parent then
        findTarget()
    end

    if not target then
        flyButton.Text = "TARGET NOT FOUND"

        task.delay(1.5, function()
            if flyButton.Parent then
                flyButton.Text = "FLY TO NOCTURNITE"
            end
        end)
        return
    end

    local character = LP.Character
    local root = character
        and character:FindFirstChild("HumanoidRootPart")
    local humanoid = character
        and character:FindFirstChildOfClass("Humanoid")

    if not root or not humanoid then return end

    flying = true
    flyButton.Text = "FLYING..."
    humanoid.PlatformStand = true

    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            savedCollisions[part] = part.CanCollide
            part.CanCollide = false
        end
    end

    flyConnection = RunService.Heartbeat:Connect(function(dt)
        if not flying then return end

        if not root.Parent or not target or not target.Parent then
            stopFly()
            return
        end

        local destination = target.Position
            + Vector3.new(0, LAND_HEIGHT, 0)

        local offset = destination - root.Position
        local distance = offset.Magnitude

        if distance <= ARRIVAL_DISTANCE then
            root.CFrame = CFrame.new(destination)
            stopFly()
            return
        end

        local step = math.min(FLY_SPEED * dt, distance)
        root.CFrame = CFrame.new(root.Position + offset.Unit * step)
    end)
end

flyButton.MouseButton1Click:Connect(startFly)

-- PLAYER ESP CLEANUP
local function removePlayerESP(player)
    local data = playerESPs[player]
    if not data then return end

    if data.connection then
        data.connection:Disconnect()
    end

    if data.gui then
        data.gui:Destroy()
    end

    playerESPs[player] = nil
end

local function clearTargetESP()
    if targetESPConnection then
        targetESPConnection:Disconnect()
        targetESPConnection = nil
    end

    if targetESP then
        targetESP:Destroy()
        targetESP = nil
    end
end

-- TARGET ESP
local function createTargetESP()
    clearTargetESP()

    local part = findTarget()
    if not part then return end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "NocturniteESP"
    billboard.Adornee = part
    billboard.Size = UDim2.fromOffset(170, 35)
    billboard.StudsOffsetWorldSpace = Vector3.new(0, 3, 0)
    billboard.AlwaysOnTop = true
    billboard.MaxDistance = 2000
    billboard.Enabled = ESP_ENABLED
    billboard.Parent = PG

    local name = Instance.new("TextLabel")
    name.Size = UDim2.new(1, 0, 0, 19)
    name.BackgroundTransparency = 1
    name.Text = "NOCTURNITE"
    name.TextColor3 = Color3.fromRGB(255, 80, 200)
    name.TextStrokeTransparency = 0
    name.TextStrokeColor3 = Color3.new(0, 0, 0)
    name.Font = Enum.Font.GothamBold
    name.TextSize = 13
    name.Parent = billboard

    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Position = UDim2.fromOffset(0, 18)
    distanceLabel.Size = UDim2.new(1, 0, 0, 16)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.TextColor3 = Color3.new(1, 1, 1)
    distanceLabel.TextStrokeTransparency = 0
    distanceLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
    distanceLabel.Font = Enum.Font.GothamBold
    distanceLabel.TextSize = 11
    distanceLabel.Parent = billboard

    targetESP = billboard

    targetESPConnection = RunService.RenderStepped:Connect(function()
        if not part.Parent then
            clearTargetESP()
            return
        end

        billboard.Enabled = ESP_ENABLED

        local character = LP.Character
        local root = character
            and character:FindFirstChild("HumanoidRootPart")

        if root then
            local dist = (root.Position - part.Position).Magnitude
            distanceLabel.Text = string.format("[%.0f studs]", dist)
        end
    end)
end

-- PLAYER ESP
local function addPlayerESP(player)
    if player == LP or playerESPs[player] then return end

    local function attach(character)
        removePlayerESP(player)

        if not ESP_ENABLED or not character then return end

        local root = character:FindFirstChild("HumanoidRootPart")
            or character:WaitForChild("HumanoidRootPart", 5)

        if not root then return end

        local billboard = Instance.new("BillboardGui")
        billboard.Name = "PlayerESP"
        billboard.Adornee = root
        billboard.Size = UDim2.fromOffset(150, 32)
        billboard.StudsOffsetWorldSpace = Vector3.new(0, -3, 0)
        billboard.AlwaysOnTop = true
        billboard.MaxDistance = 2000
        billboard.Enabled = ESP_ENABLED
        billboard.Parent = PG

        local name = Instance.new("TextLabel")
        name.Size = UDim2.new(1, 0, 0, 17)
        name.BackgroundTransparency = 1
        name.Text = player.DisplayName
        name.TextColor3 = Color3.fromRGB(255, 45, 45)
        name.TextStrokeTransparency = 0.2
        name.TextStrokeColor3 = Color3.new(0, 0, 0)
        name.TextSize = 12
        name.Font = Enum.Font.GothamBold
        name.Parent = billboard

        local distLabel = Instance.new("TextLabel")
        distLabel.Position = UDim2.fromOffset(0, 16)
        distLabel.Size = UDim2.new(1, 0, 0, 16)
        distLabel.BackgroundTransparency = 1
        distLabel.TextColor3 = Color3.new(1, 1, 1)
        distLabel.TextStrokeTransparency = 0.1
        distLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
        distLabel.TextSize = 10
        distLabel.Font = Enum.Font.GothamBold
        distLabel.Parent = billboard

        local connection = RunService.RenderStepped:Connect(function()
            if not billboard.Parent then return end

            billboard.Enabled = ESP_ENABLED
            if not ESP_ENABLED then return end

            local myChar = LP.Character
            local myRoot = myChar
                and myChar:FindFirstChild("HumanoidRootPart")
            local theirChar = player.Character
            local theirRoot = theirChar
                and theirChar:FindFirstChild("HumanoidRootPart")

            if myRoot and theirRoot then
                local dist = (myRoot.Position - theirRoot.Position).Magnitude
                distLabel.Text = string.format("[%.0f studs]", dist)
            end
        end)

        playerESPs[player] = {
            gui = billboard,
            connection = connection
        }
    end

    if player.Character then
        task.spawn(attach, player.Character)
    end

    player.CharacterAdded:Connect(function(character)
        task.spawn(attach, character)
    end)
end

for _, player in ipairs(Players:GetPlayers()) do
    addPlayerESP(player)
end

Players.PlayerAdded:Connect(addPlayerESP)
Players.PlayerRemoving:Connect(removePlayerESP)

-- ESP TOGGLE
espButton.MouseButton1Click:Connect(function()
    ESP_ENABLED = not ESP_ENABLED
    espButton.Text = ESP_ENABLED and "ESP: ON" or "ESP: OFF"

    for _, data in pairs(playerESPs) do
        if data.gui then
            data.gui.Enabled = ESP_ENABLED
        end
    end

    if targetESP then
        targetESP.Enabled = ESP_ENABLED
    end

    if ESP_ENABLED then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LP and not playerESPs[player] then
                addPlayerESP(player)
            end
        end

        createTargetESP()
    end
end)

-- RESCAN TARGET
rescanButton.MouseButton1Click:Connect(function()
    createTargetESP()

    rescanButton.Text = target
        and "TARGET FOUND"
        or "TARGET NOT FOUND"

    task.delay(1.5, function()
        if rescanButton.Parent then
            rescanButton.Text = "RESCAN TARGET"
        end
    end)
end)

-- INITIAL SCAN
findTarget()
createTargetESP()

LP.CharacterAdded:Connect(function()
    stopFly()
end)
