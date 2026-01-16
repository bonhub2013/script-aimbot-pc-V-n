-- V/n wartyoon - Combat System v2.0
-- QUÂN ĐỘI HACK V/n - Direct Activation Edition

--[[
    ⚠️ WARTYOON SYSTEMS:
    Tactical combat interface for private environments only.
]]

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")

-- Core variables
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- Configuration System
local Config = {
    Aimbot = {
        Enabled = false,
        TeamCheck = true,
        VisibleCheck = true,
        MaxDistance = 1800,
        Smoothness = 0.25, -- 75% lock strength
        AimPart = "HumanoidRootPart",
        FOV = 85,
        TriggerKey = Enum.KeyCode.F1
    },
    WallCheck = {
        Enabled = true,
        CheckPoints = {"Head", "HumanoidRootPart", "UpperTorso"},
        TriggerKey = Enum.KeyCode.F3
    },
    ESP = {
        Enabled = false,
        Color = Color3.fromRGB(255, 35, 35),
        TeamColor = Color3.fromRGB(55, 155, 255),
        Thickness = 1.8,
        UseTeamColor = true,
        TriggerKey = Enum.KeyCode.F2
    },
    POVCircle = {
        Enabled = true,
        Radius = 70,
        ColorNoTarget = Color3.fromRGB(20, 20, 20),
        ColorTarget = Color3.fromRGB(45, 255, 45),
        Thickness = 2,
        Filled = false,
        Transparency = 0.65
    },
    Menu = {
        ToggleKey = Enum.KeyCode.RightShift,
        Visible = true
    }
}

-- State Management
local Target = nil
local Circle = nil
local ESPDrawings = {}
local Connections = {}
local MenuUI = nil
local KeyStates = {
    F1 = false,
    F2 = false,
    F3 = false
}

-- Utility: Valid player check
function IsValidPlayer(player)
    if not player then return false end
    if player == LocalPlayer then return false end
    if not player.Character then return false end
    
    local humanoid = player.Character:FindFirstChildWhichIsA("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return false end
    
    return true
end

-- Utility: World to screen conversion
function WorldToScreen(position)
    local screenPos, onScreen = Camera:WorldToViewportPoint(position)
    return Vector2.new(screenPos.X, screenPos.Y), onScreen, screenPos.Z
end

-- Advanced Wall Check with ray optimization
function IsTargetVisible(targetChar)
    if not Config.WallCheck.Enabled then return true end
    
    local origin = Camera.CFrame.Position
    local checkPoints = {}
    
    -- Primary check points
    for _, pointName in ipairs(Config.WallCheck.CheckPoints) do
        local part = targetChar:FindFirstChild(pointName)
        if part then
            table.insert(checkPoints, part.Position)
        end
    end
    
    -- Quick visibility test
    for _, targetPoint in ipairs(checkPoints) do
        local direction = (targetPoint - origin)
        local rayParams = RaycastParams.new()
        rayParams.FilterType = Enum.RaycastFilterType.Blacklist
        rayParams.FilterDescendantsInstances = {LocalPlayer.Character, targetChar}
        rayParams.IgnoreWater = true
        
        local result = Workspace:Raycast(origin, direction, rayParams)
        
        -- No hit or hit target = visible
        if not result or result.Instance:IsDescendantOf(targetChar) then
            return true
        end
    end
    
    return false
end

-- Target acquisition algorithm
function AcquireTarget()
    local bestTarget = nil
    local bestScore = -math.huge
    local mousePos = UserInputService:GetMouseLocation()
    
    for _, player in ipairs(Players:GetPlayers()) do
        if not IsValidPlayer(player) then continue end
        if Config.Aimbot.TeamCheck and player.Team == LocalPlayer.Team then continue end
        
        local character = player.Character
        local aimPart = character:FindFirstChild(Config.Aimbot.AimPart)
        if not aimPart then continue end
        
        local screenPos, onScreen, depth = WorldToScreen(aimPart.Position)
        if not onScreen then continue end
        if depth < 0 then continue end
        
        -- Distance to mouse
        local mouseDistance = (mousePos - screenPos).Magnitude
        if mouseDistance > Config.Aimbot.FOV then continue end
        
        -- Physical distance
        local physicalDistance = (aimPart.Position - Camera.CFrame.Position).Magnitude
        if physicalDistance > Config.Aimbot.MaxDistance then continue end
        
        -- Visibility check
        local isVisible = true
        if Config.Aimbot.VisibleCheck then
            isVisible = IsTargetVisible(character)
        end
        
        if not isVisible then continue end
        
        -- Scoring system (prioritize close to cursor and close distance)
        local score = (Config.Aimbot.FOV - mouseDistance) * 2 + (Config.Aimbot.MaxDistance - physicalDistance) * 0.1
        
        if score > bestScore then
            bestScore = score
            bestTarget = player
        end
    end
    
    return bestTarget
end

-- Aimbot execution (runs every frame when enabled)
function ExecuteAimbot()
    if not Config.Aimbot.Enabled then
        Target = nil
        return
    end
    
    Target = AcquireTarget()
    
    if Target and Target.Character then
        local character = Target.Character
        local aimPart = character:FindFirstChild(Config.Aimbot.AimPart)
        
        if aimPart then
            local currentCFrame = Camera.CFrame
            local targetPosition = aimPart.Position
            local targetDirection = (targetPosition - currentCFrame.Position).Unit
            
            -- Smooth aiming with 75% lock
            local newLookVector = currentCFrame.LookVector:Lerp(targetDirection, Config.Aimbot.Smoothness)
            Camera.CFrame = CFrame.new(currentCFrame.Position, currentCFrame.Position + newLookVector)
        end
    end
end

-- ESP Drawing System
function UpdateESP()
    -- Cleanup disconnected players
    for player, drawings in pairs(ESPDrawings) do
        if not Players:FindFirstChild(player.Name) then
            for _, drawing in ipairs(drawings) do
                drawing:Remove()
            end
            ESPDrawings[player] = nil
        end
    end
    
    if not Config.ESP.Enabled then
        -- Hide all drawings when ESP is off
        for player, drawings in pairs(ESPDrawings) do
            for _, drawing in ipairs(drawings) do
                drawing.Visible = false
            end
        end
        return
    end
    
    -- Update ESP for each player
    for _, player in ipairs(Players:GetPlayers()) do
        if not IsValidPlayer(player) then continue end
        
        local character = player.Character
        if not character then continue end
        
        -- Create drawing objects if needed
        if not ESPDrawings[player] then
            ESPDrawings[player] = {}
            for i = 1, 6 do -- Optimized number of lines
                local line = Drawing.new("Line")
                line.Visible = false
                line.Thickness = Config.ESP.Thickness
                table.insert(ESPDrawings[player], line)
            end
        end
        
        local drawings = ESPDrawings[player]
        
        -- Determine color
        local color = Config.ESP.Color
        if Config.ESP.UseTeamColor and player.Team == LocalPlayer.Team then
            color = Config.ESP.TeamColor
        end
        
        -- Get key body positions
        local bodyPoints = {}
        local keyParts = {
            "Head",
            "HumanoidRootPart",
            "LeftHand",
            "RightHand",
            "LeftFoot", 
            "RightFoot"
        }
        
        for _, partName in ipairs(keyParts) do
            local part = character:FindFirstChild(partName)
            if part then
                local screenPos, onScreen = WorldToScreen(part.Position)
                table.insert(bodyPoints, onScreen and screenPos or nil)
            else
                table.insert(bodyPoints, nil)
            end
        end
        
        -- Draw connections (simple skeleton)
        local connections = {
            {1, 2}, -- Head to body
            {2, 3}, {2, 4}, -- Body to hands
            {2, 5}, {2, 6}  -- Body to feet
        }
        
        for idx, conn in ipairs(connections) do
            local line = drawings[idx]
            local pointA = bodyPoints[conn[1]]
            local pointB = bodyPoints[conn[2]]
            
            if line and pointA and pointB then
                line.From = pointA
                line.To = pointB
                line.Color = color
                line.Visible = true
            elseif line then
                line.Visible = false
            end
        end
    end
end

-- POV Circle System
function UpdatePOVCircle()
    if not Circle then
        Circle = Drawing.new("Circle")
        Circle.Visible = Config.POVCircle.Enabled
        Circle.Radius = Config.POVCircle.Radius
        Circle.Thickness = Config.POVCircle.Thickness
        Circle.Filled = Config.POVCircle.Filled
        Circle.Transparency = Config.POVCircle.Transparency
    end
    
    Circle.Visible = Config.POVCircle.Enabled
    if not Config.POVCircle.Enabled then return end
    
    -- Update position to mouse
    local mousePos = UserInputService:GetMouseLocation()
    Circle.Position = mousePos + Vector2.new(0, 36)
    
    -- Update color based on target status
    if Target then
        Circle.Color = Config.POVCircle.ColorTarget
    else
        Circle.Color = Config.POVCircle.ColorNoTarget
    end
end

-- Input Handler (FIXED F1-F3 toggle system)
function SetupInputHandler()
    -- F1: Aimbot Toggle
    local f1Connection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if input.KeyCode == Config.Aimbot.TriggerKey and not KeyStates.F1 then
            KeyStates.F1 = true
            Config.Aimbot.Enabled = not Config.Aimbot.Enabled
            
            -- Update menu if exists
            if MenuUI then
                updateMenuToggle("AimbotToggle", Config.Aimbot.Enabled)
            end
            
            -- Visual feedback
            warn("[V/n] Aimbot: " .. (Config.Aimbot.Enabled and "ACTIVE" or "INACTIVE"))
        end
    end)
    
    local f1Release = UserInputService.InputEnded:Connect(function(input)
        if input.KeyCode == Config.Aimbot.TriggerKey then
            KeyStates.F1 = false
        end
    end)
    
    -- F2: ESP Toggle
    local f2Connection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if input.KeyCode == Config.ESP.TriggerKey and not KeyStates.F2 then
            KeyStates.F2 = true
            Config.ESP.Enabled = not Config.ESP.Enabled
            
            if MenuUI then
                updateMenuToggle("ESPToggle", Config.ESP.Enabled)
            end
            
            warn("[V/n] ESP: " .. (Config.ESP.Enabled and "ACTIVE" or "INACTIVE"))
        end
    end)
    
    local f2Release = UserInputService.InputEnded:Connect(function(input)
        if input.KeyCode == Config.ESP.TriggerKey then
            KeyStates.F2 = false
        end
    end)
    
    -- F3: Wall Check Toggle
    local f3Connection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if input.KeyCode == Config.WallCheck.TriggerKey and not KeyStates.F3 then
            KeyStates.F3 = true
            Config.WallCheck.Enabled = not Config.WallCheck.Enabled
            
            if MenuUI then
                updateMenuToggle("WallCheckToggle", Config.WallCheck.Enabled)
            end
            
            warn("[V/n] Wall Check: " .. (Config.WallCheck.Enabled and "ACTIVE" or "INACTIVE"))
        end
    end)
    
    local f3Release = UserInputService.InputEnded:Connect(function(input)
        if input.KeyCode == Config.WallCheck.TriggerKey then
            KeyStates.F3 = false
        end
    end)
    
    -- Right Shift: Menu Toggle
    local menuConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if input.KeyCode == Config.Menu.ToggleKey then
            Config.Menu.Visible = not Config.Menu.Visible
            if MenuUI then
                MenuUI.Enabled = Config.Menu.Visible
            end
        end
    end)
    
    -- Store connections
    table.insert(Connections, f1Connection)
    table.insert(Connections, f1Release)
    table.insert(Connections, f2Connection)
    table.insert(Connections, f2Release)
    table.insert(Connections, f3Connection)
    table.insert(Connections, f3Release)
    table.insert(Connections, menuConnection)
end

-- Menu Creation
function CreateMenu()
    if MenuUI then MenuUI:Destroy() end
    
    MenuUI = Instance.new("ScreenGui")
    MenuUI.Name = "VnWartyoonControl"
    MenuUI.ResetOnSpawn = false
    MenuUI.Enabled = Config.Menu.Visible
    MenuUI.Parent = CoreGui
    
    local MainFrame = Instance.new("Frame")
    MainFrame.Size = UDim2.new(0, 280, 0, 300)
    MainFrame.Position = UDim2.new(0.02, 0, 0.02, 0)
    MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
    MainFrame.BorderColor3 = Color3.fromRGB(0, 180, 0)
    MainFrame.BorderSizePixel = 2
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.Parent = MenuUI
    
    -- Title Bar
    local TitleBar = Instance.new("Frame")
    TitleBar.Size = UDim2.new(1, 0, 0, 35)
    TitleBar.BackgroundColor3 = Color3.fromRGB(15, 15, 25)
    TitleBar.BorderSizePixel = 0
    TitleBar.Parent = MainFrame
    
    local Title = Instance.new("TextLabel")
    Title.Text = "V/n WARTYOON"
    Title.Size = UDim2.new(1, -40, 1, 0)
    Title.Position = UDim2.new(0, 10, 0, 0)
    Title.BackgroundTransparency = 1
    Title.TextColor3 = Color3.fromRGB(0, 255, 100)
    Title.Font = Enum.Font.GothamBold
    Title.TextSize = 16
    Title.TextXAlignment = Enum.TextXAlignment.Left
    Title.Parent = TitleBar
    
    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Text = "×"
    CloseBtn.Size = UDim2.new(0, 30, 0, 30)
    CloseBtn.Position = UDim2.new(1, -35, 0, 3)
    CloseBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
    CloseBtn.TextColor3 = Color3.fromRGB(220, 220, 220)
    CloseBtn.Font = Enum.Font.GothamBold
    CloseBtn.TextSize = 20
    CloseBtn.MouseButton1Click:Connect(function()
        Config.Menu.Visible = false
        MenuUI.Enabled = false
    end)
    CloseBtn.Parent = TitleBar
    
    -- Content Area
    local ContentFrame = Instance.new("ScrollingFrame")
    ContentFrame.Size = UDim2.new(1, -10, 1, -45)
    ContentFrame.Position = UDim2.new(0, 5, 0, 40)
    ContentFrame.BackgroundTransparency = 1
    ContentFrame.CanvasSize = UDim2.new(0, 0, 0, 320)
    ContentFrame.ScrollBarThickness = 3
    ContentFrame.ScrollBarImageColor3 = Color3.fromRGB(0, 180, 0)
    ContentFrame.Parent = MainFrame
    
    -- Toggle storage for updates
    local ToggleButtons = {}
    
    local function CreateControl(text, configTable, key, yPos, color)
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(1, 0, 0, 30)
        frame.Position = UDim2.new(0, 0, 0, yPos)
        frame.BackgroundTransparency = 1
        frame.Parent = ContentFrame
        
        local toggle = Instance.new("TextButton")
        toggle.Name = text .. "Toggle"
        toggle.Size = UDim2.new(0, 24, 0, 24)
        toggle.Position = UDim2.new(0, 5, 0, 3)
        toggle.BackgroundColor3 = configTable[key] and Color3.fromRGB(0, 200, 0) or Color3.fromRGB(70, 70, 70)
        toggle.Text = ""
        toggle.MouseButton1Click:Connect(function()
            configTable[key] = not configTable[key]
            toggle.BackgroundColor3 = configTable[key] and Color3.fromRGB(0, 200, 0) or Color3.fromRGB(70, 70, 70)
        end)
        toggle.Parent = frame
        
        ToggleButtons[toggle.Name] = toggle
        
        local label = Instance.new("TextLabel")
        label.Text = text
        label.Size = UDim2.new(1, -35, 1, 0)
        label.Position = UDim2.new(0, 35, 0, 0)
        label.BackgroundTransparency = 1
        label.TextColor3 = color or Color3.fromRGB(220, 220, 220)
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Font = Enum.Font.Gotham
        label.TextSize = 14
        label.Parent = frame
        
        return yPos + 35
    end
    
    local yPosition = 0
    
    -- Section: Core Features
    yPosition = CreateControl("AIMBOT", Config.Aimbot, "Enabled", yPosition, Color3.fromRGB(255, 100, 100))
    yPosition = CreateControl("ESP HIGHLIGHT", Config.ESP, "Enabled", yPosition, Color3.fromRGB(100, 200, 255))
    yPosition = CreateControl("WALL CHECK", Config.WallCheck, "Enabled", yPosition, Color3.fromRGB(100, 255, 100))
    yPosition = CreateControl("TEAM CHECK", Config.Aimbot, "TeamCheck", yPosition, Color3.fromRGB(180, 180, 180))
    yPosition = CreateControl("VISIBILITY CHECK", Config.Aimbot, "VisibleCheck", yPosition, Color3.fromRGB(180, 180, 180))
    yPosition = CreateControl("POV CIRCLE", Config.POVCircle, "Enabled", yPosition, Color3.fromRGB(255, 200, 100))
    
    -- Spacer
    yPosition = yPosition + 10
    
    -- Keybinds Display
    local keybinds = Instance.new("TextLabel")
    keybinds.Text = "HOTKEYS:\nRIGHT SHIFT - Menu\nF1 - Aimbot Toggle\nF2 - ESP Toggle\nF3 - Wall Check Toggle"
    keybinds.Size = UDim2.new(1, 0, 0, 90)
    keybinds.Position = UDim2.new(0, 5, 0, yPosition)
    keybinds.BackgroundTransparency = 1
    keybinds.TextColor3 = Color3.fromRGB(150, 150, 150)
    keybinds.TextXAlignment = Enum.TextXAlignment.Left
    keybinds.TextYAlignment = Enum.TextYAlignment.Top
    keybinds.Font = Enum.Font.Gotham
    keybinds.TextSize = 12
    keybinds.Parent = ContentFrame
    
    ContentFrame.CanvasSize = UDim2.new(0, 0, 0, yPosition + 100)
    
    -- Function to update toggle states
    function updateMenuToggle(toggleName, state)
        local toggleBtn = ToggleButtons[toggleName]
        if toggleBtn then
            toggleBtn.BackgroundColor3 = state and Color3.fromRGB(0, 200, 0) or Color3.fromRGB(70, 70, 70)
        end
    end
end

-- Main System Initialization
function InitializeSystem()
    -- Setup input handler
    SetupInputHandler()
    
    -- Create menu
    CreateMenu()
    
    -- Main render loop
    local mainLoop = RunService.RenderStepped:Connect(function()
        ExecuteAimbot()
        UpdatePOVCircle()
        UpdateESP()
    end)
    
    table.insert(Connections, mainLoop)
    
    -- Initial status report
    print("========================================")
    print("V/n WARTYOON - COMBAT SYSTEM v2.0")
    print("========================================")
    print("Status: OPERATIONAL")
    print("Aimbot Smoothness: 75%")
    print("ESP: Red Highlight Active")
    print("Controls:")
    print("  RIGHT SHIFT - Menu")
    print("  F1 - Aimbot Toggle")
    print("  F2 - ESP Toggle") 
    print("  F3 - Wall Check Toggle")
    print("========================================")
end

-- Cleanup Function
function CleanupSystem()
    -- Disconnect all events
    for _, connection in ipairs(Connections) do
        connection:Disconnect()
    end
    Connections = {}
    
    -- Remove drawings
    if Circle then
        Circle:Remove()
        Circle = nil
    end
    
    for player, drawings in pairs(ESPDrawings) do
        for _, drawing in ipairs(drawings) do
            drawing:Remove()
        end
    end
    ESPDrawings = {}
    
    -- Remove menu
    if MenuUI then
        MenuUI:Destroy()
        MenuUI = nil
    end
    
    warn("[V/n] System shutdown complete.")
end

-- Auto cleanup on teleport
if LocalPlayer then
    LocalPlayer.OnTeleport:Connect(function(state)
        if state == Enum.TeleportState.Started then
            CleanupSystem()
        end
    end)
end

-- Error handling and initialization
local success, errorMsg = pcall(function()
    InitializeSystem()
end)

if not success then
    warn("[V/n INIT ERROR]: " .. tostring(errorMsg))
    CleanupSystem()
end

-- Return control interface
return {
    ToggleAimbot = function()
        Config.Aimbot.Enabled = not Config.Aimbot.Enabled
        warn("Aimbot: " .. (Config.Aimbot.Enabled and "ON" or "OFF"))
    end,
    
    ToggleESP = function()
        Config.ESP.Enabled = not Config.ESP.Enabled
        warn("ESP: " .. (Config.ESP.Enabled and "ON" or "OFF"))
    end,
    
    ToggleWallCheck = function()
        Config.WallCheck.Enabled = not Config.WallCheck.Enabled
        warn("Wall Check: " .. (Config.WallCheck.Enabled and "ON" or "OFF"))
    end,
    
    ShowMenu = function()
        Config.Menu.Visible = true
        if MenuUI then
            MenuUI.Enabled = true
        end
    end,
    
    HideMenu = function()
        Config.Menu.Visible = false
        if MenuUI then
            MenuUI.Enabled = false
        end
    end,
    
    Shutdown = CleanupSystem
}
