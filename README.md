-- V/n ULTIMATE SUITE v7.0 - AIMBOT + ESP + POV + RADAR BUTTON
-- ==================== SERVICES ====================
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- Kiểm tra Drawing Library
if not Drawing then
    game:GetService("StarterGui"):SetCore("SendNotification", {
        Title = "V/n ERROR",
        Text = "Executor không hỗ trợ Drawing Library!",
        Duration = 5
    })
    return
end

print("🚀 V/n Ultimate Suite v7.0 đang khởi động...")

-- ==================== CONFIG ====================
local Config = {
    Aimbot = {
        Enabled = true,
        Strength = 0.75,
        Smoothing = 0.2,
        FOV = 150,
        Bone = "Head",
        TeamCheck = true,
        WallCheck = true,
        AutoAim = true,
        Prediction = 0.15,
        FirstPerson = true,
        ThirdPerson = true,
        MaxDistance = 500
    },
    
    POVCircle = {
        Enabled = true,
        OuterRadius = 150,
        InnerRadius = 75,
        ColorNormal = Color3.fromRGB(0, 0, 0),
        ColorTarget = Color3.fromRGB(0, 255, 100),
        Thickness = 2,
        MinRadius = 50,
        MaxRadius = 200
    },
    
    WallCheck = {
        Enabled = true,
        IgnoreTools = true,
        MaxDistance = 1000,
        ToolNames = {"Gun", "Sword", "Tool", "Weapon", "AK", "M4", "Pistol", "Rifle", "Shotgun", "Sniper", "Revolver", "SMG"}
    },
    
    ESP = {
        Enabled = true,
        MaxDistance = 4000,
        ShowName = true,
        ShowHealth = true,
        ShowDistance = true,
        HealthBar = true,
        TeamColor = true,
        EnemyColor = Color3.fromRGB(255, 50, 50),
        FriendlyColor = Color3.fromRGB(50, 200, 50),
        TextSize = 14,
        ShowBox = true,
        ShowHighlight = true,
    },
    
    UI = {
        ToggleKey = Enum.KeyCode.RightShift,
        AccentColor = Color3.fromRGB(0, 170, 255),
        BackgroundColor = Color3.fromRGB(25, 25, 35),
        TextColor = Color3.fromRGB(240, 240, 240)
    }
}

-- ==================== BIẾN TOÀN CỤC ====================
local Drawings = {
    POV = {Outer = nil, Inner = nil},
    ESP = {}
}

local State = {
    Target = nil,
    LastTargetTime = 0,
    TargetLockTime = 1.0,
    UIVisible = true,
    TargetBehindWall = false
}

-- Biến cho Radar
local RadarInstance = nil
local RadarToggleState = false

-- ==================== WALL CHECK ====================
local function IsTargetBehindWall(targetPosition, targetCharacter)
    if not Config.WallCheck.Enabled then return false end
    
    local myChar = LocalPlayer.Character
    if not myChar then return true end
    
    local myHead = myChar:FindFirstChild("Head")
    if not myHead then return true end
    
    local distance = (targetPosition - myHead.Position).Magnitude
    
    if distance < 5 then return false end
    if distance > Config.WallCheck.MaxDistance then return true end
    
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.FilterDescendantsInstances = {myChar}
    params.IgnoreWater = true
    
    if Config.WallCheck.IgnoreTools then
        for _, player in pairs(Players:GetPlayers()) do
            local char = player.Character
            if char then
                for _, item in pairs(char:GetChildren()) do
                    if item:IsA("Tool") then
                        table.insert(params.FilterDescendantsInstances, item)
                    end
                    if item:IsA("BasePart") then
                        for _, toolName in pairs(Config.WallCheck.ToolNames) do
                            if string.find(item.Name:lower(), toolName:lower()) then
                                table.insert(params.FilterDescendantsInstances, item)
                                break
                            end
                        end
                    end
                end
            end
        end
    end
    
    local origin = myHead.Position + Vector3.new(0, 1, 0)
    local direction = (targetPosition - origin).Unit
    local result = Workspace:Raycast(origin, direction * distance, params)
    
    if result then
        local hit = result.Instance
        if targetCharacter and hit:IsDescendantOf(targetCharacter) then
            return false
        end
        if hit.Material == Enum.Material.Glass or 
           hit.Material == Enum.Material.Water or
           hit.Material == Enum.Material.Air or
           hit.Material == Enum.Material.ForceField then
            return false
        end
        return true
    end
    return false
end

-- ==================== POV CIRCLE ====================
local function CreatePOVCircles()
    if Drawings.POV.Outer then
        Drawings.POV.Outer:Remove()
        Drawings.POV.Inner:Remove()
    end
    
    Drawings.POV.Outer = Drawing.new("Circle")
    Drawings.POV.Inner = Drawing.new("Circle")
    
    local outer = Drawings.POV.Outer
    local inner = Drawings.POV.Inner
    
    outer.Visible = Config.POVCircle.Enabled
    outer.Radius = Config.POVCircle.OuterRadius
    outer.Thickness = Config.POVCircle.Thickness
    outer.Color = Config.POVCircle.ColorNormal
    outer.Filled = false
    outer.Transparency = 0.7
    
    inner.Visible = Config.POVCircle.Enabled
    inner.Radius = Config.POVCircle.InnerRadius
    inner.Thickness = Config.POVCircle.Thickness
    inner.Color = Config.POVCircle.ColorNormal
    inner.Filled = false
    inner.Transparency = 0.7
end

local function UpdatePOVPosition()
    if not Drawings.POV.Outer then return end
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    Drawings.POV.Outer.Position = screenCenter
    Drawings.POV.Inner.Position = screenCenter
end

local function UpdatePOVColor(isTarget)
    if not Drawings.POV.Outer then return end
    local color = isTarget and Config.POVCircle.ColorTarget or Config.POVCircle.ColorNormal
    Drawings.POV.Outer.Color = color
    Drawings.POV.Inner.Color = color
end

-- ==================== FIND BEST TARGET ====================
local function FindBestTarget()
    if not Config.Aimbot.Enabled then return nil end
    
    local bestTarget = nil
    local bestDistance = Config.Aimbot.FOV
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    
    for _, player in pairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        if Config.Aimbot.TeamCheck and player.Team and LocalPlayer.Team and player.Team == LocalPlayer.Team then
            continue
        end
        
        local character = player.Character
        if not character then continue end
        
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid or humanoid.Health <= 0 then continue end
        
        local targetPart = character:FindFirstChild(Config.Aimbot.Bone) or
                          character:FindFirstChild("HumanoidRootPart") or
                          character:FindFirstChild("Torso") or
                          character:FindFirstChild("UpperTorso") or
                          character:FindFirstChild("LowerTorso") or
                          character:FindFirstChild("Head")
        if not targetPart then continue end
        
        local distance = (targetPart.Position - Camera.CFrame.Position).Magnitude
        if distance > Config.Aimbot.MaxDistance then continue end
        
        if Config.Aimbot.WallCheck then
            if IsTargetBehindWall(targetPart.Position, character) then
                continue
            end
        end
        
        local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
        if not onScreen then continue end
        
        local pos2D = Vector2.new(screenPos.X, screenPos.Y)
        local screenDistance = (pos2D - screenCenter).Magnitude
        
        if screenDistance <= Config.POVCircle.OuterRadius then
            if screenDistance < bestDistance then
                bestDistance = screenDistance
                bestTarget = {
                    Player = player,
                    Part = targetPart,
                    Position = targetPart.Position,
                    Distance = distance,
                    ScreenDistance = screenDistance
                }
            end
        end
    end
    return bestTarget
end

-- ==================== AUTO AIMBOT ====================
local function PerformAutoAim()
    if not Config.Aimbot.Enabled or not Config.Aimbot.AutoAim then
        UpdatePOVColor(false)
        return false
    end
    
    local currentTime = tick()
    
    if State.Target then
        local target = State.Target
        if not target.Player or not target.Player.Parent then
            State.Target = nil
            State.TargetBehindWall = false
        else
            local character = target.Player.Character
            if not character then
                State.Target = nil
                State.TargetBehindWall = false
            end
        end
    end
    
    if State.Target then
        local target = State.Target
        local character = target.Player.Character
        
        if character then
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            local targetPart = character:FindFirstChild(Config.Aimbot.Bone) or
                              character:FindFirstChild("HumanoidRootPart")
            
            if humanoid and humanoid.Health > 0 and targetPart then
                local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                
                local isBehindWall = false
                if Config.Aimbot.WallCheck then
                    isBehindWall = IsTargetBehindWall(targetPart.Position, character)
                end
                State.TargetBehindWall = isBehindWall
                
                if onScreen and not isBehindWall then
                    State.Target.Part = targetPart
                    State.Target.Position = targetPart.Position
                    State.LastTargetTime = currentTime
                    
                    local currentCF = Camera.CFrame
                    local targetPos = targetPart.Position
                    if targetPart.Velocity then
                        targetPos = targetPos + (targetPart.Velocity * Config.Aimbot.Prediction)
                    end
                    
                    local desiredDirection = (targetPos - currentCF.Position).Unit
                    local currentDirection = currentCF.LookVector
                    local newDirection = currentDirection:Lerp(desiredDirection, Config.Aimbot.Strength * Config.Aimbot.Smoothing)
                    
                    Camera.CFrame = CFrame.new(currentCF.Position, currentCF.Position + newDirection)
                    UpdatePOVColor(true)
                    return true
                else
                    UpdatePOVColor(false)
                    return false
                end
            else
                State.Target = nil
                State.TargetBehindWall = false
            end
        else
            State.Target = nil
            State.TargetBehindWall = false
        end
    end
    
    if not State.Target or (currentTime - State.LastTargetTime) > State.TargetLockTime then
        local newTarget = FindBestTarget()
        if newTarget then
            State.Target = newTarget
            State.LastTargetTime = currentTime
            State.TargetBehindWall = false
            UpdatePOVColor(true)
            return true
        else
            State.Target = nil
            State.TargetBehindWall = false
            UpdatePOVColor(false)
        end
    end
    
    UpdatePOVColor(false)
    return false
end

-- ==================== ESP ====================
local function CreateESP(player)
    if Drawings.ESP[player] then return end
    
    local drawings = {
        Name = Drawing.new("Text"),
        HealthBar = Drawing.new("Square"),
        HealthBarFill = Drawing.new("Square"),
        HealthText = Drawing.new("Text"),
        DistanceText = Drawing.new("Text"),
        Box = Drawing.new("Square"),
        Highlight = Drawing.new("Square")
    }
    
    for _, drawing in pairs(drawings) do
        drawing.Visible = false
    end
    
    drawings.Name.Size = Config.ESP.TextSize
    drawings.Name.Outline = true
    drawings.HealthText.Size = Config.ESP.TextSize - 2
    drawings.HealthText.Outline = true
    drawings.DistanceText.Size = Config.ESP.TextSize - 2
    drawings.DistanceText.Outline = true
    
    drawings.Box.Thickness = 2
    drawings.Box.Filled = false
    drawings.Box.Transparency = 1
    
    drawings.Highlight.Thickness = 0
    drawings.Highlight.Filled = true
    drawings.Highlight.Transparency = 0.7
    drawings.Highlight.Color = Color3.fromRGB(255, 255, 0)
    
    Drawings.ESP[player] = drawings
end

local function UpdateESP()
    if not Config.ESP.Enabled then
        for _, drawings in pairs(Drawings.ESP) do
            for _, drawing in pairs(drawings) do
                drawing.Visible = false
            end
        end
        return
    end
    
    for player, drawings in pairs(Drawings.ESP) do
        if not player or not player.Parent then
            for _, drawing in pairs(drawings) do
                drawing:Remove()
            end
            Drawings.ESP[player] = nil
            continue
        end
        
        local character = player.Character
        if not character then
            for _, drawing in pairs(drawings) do
                drawing.Visible = false
            end
            continue
        end
        
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local head = character:FindFirstChild("Head")
        
        if humanoid and humanoid.Health > 0 and head then
            local distance = (head.Position - Camera.CFrame.Position).Magnitude
            
            if distance <= Config.ESP.MaxDistance then
                local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
                
                if onScreen then
                    local color = Config.ESP.EnemyColor
                    if Config.ESP.TeamColor and player.Team and LocalPlayer.Team and player.Team == LocalPlayer.Team then
                        color = Config.ESP.FriendlyColor
                    end
                    
                    local rootPart = character:FindFirstChild("HumanoidRootPart") or 
                                     character:FindFirstChild("Torso") or 
                                     character:FindFirstChild("UpperTorso") or
                                     character:FindFirstChild("LowerTorso")
                    
                    local boxHeight = 70
                    local boxWidth = 40
                    
                    if rootPart then
                        local rootPos, rootOnScreen = Camera:WorldToViewportPoint(rootPart.Position)
                        if rootOnScreen then
                            boxHeight = math.abs(screenPos.Y - rootPos.Y) * 2
                            boxWidth = boxHeight * 0.5
                        end
                    end
                    
                    if Config.ESP.ShowHighlight and drawings.Highlight then
                        drawings.Highlight.Visible = true
                        drawings.Highlight.Size = Vector2.new(boxWidth + 8, boxHeight + 8)
                        drawings.Highlight.Position = Vector2.new(screenPos.X - (boxWidth + 8)/2, screenPos.Y - boxHeight/2 - 4)
                        drawings.Highlight.Color = color
                    else
                        if drawings.Highlight then drawings.Highlight.Visible = false end
                    end
                    
                    if Config.ESP.ShowBox and drawings.Box then
                        drawings.Box.Visible = true
                        drawings.Box.Size = Vector2.new(boxWidth, boxHeight)
                        drawings.Box.Position = Vector2.new(screenPos.X - boxWidth/2, screenPos.Y - boxHeight/2)
                        drawings.Box.Color = color
                    else
                        if drawings.Box then drawings.Box.Visible = false end
                    end
                    
                    if Config.ESP.ShowName and drawings.Name then
                        drawings.Name.Visible = true
                        drawings.Name.Text = player.Name
                        drawings.Name.Position = Vector2.new(screenPos.X, screenPos.Y - boxHeight/2 - 20)
                        drawings.Name.Color = color
                    else
                        if drawings.Name then drawings.Name.Visible = false end
                    end
                    
                    if Config.ESP.HealthBar and drawings.HealthBar and drawings.HealthBarFill then
                        local healthPercent = humanoid.Health / humanoid.MaxHealth
                        local barWidth = 50
                        local barHeight = 4
                        
                        drawings.HealthBar.Visible = true
                        drawings.HealthBar.Size = Vector2.new(barWidth, barHeight)
                        drawings.HealthBar.Position = Vector2.new(screenPos.X - barWidth/2, screenPos.Y + boxHeight/2 + 5)
                        drawings.HealthBar.Color = Color3.fromRGB(40, 40, 40)
                        drawings.HealthBar.Filled = true
                        
                        drawings.HealthBarFill.Visible = true
                        drawings.HealthBarFill.Size = Vector2.new(barWidth * healthPercent, barHeight)
                        drawings.HealthBarFill.Position = Vector2.new(screenPos.X - barWidth/2, screenPos.Y + boxHeight/2 + 5)
                        drawings.HealthBarFill.Color = Color3.fromRGB(255 * (1 - healthPercent), 255 * healthPercent, 0)
                        drawings.HealthBarFill.Filled = true
                    else
                        if drawings.HealthBar then drawings.HealthBar.Visible = false end
                        if drawings.HealthBarFill then drawings.HealthBarFill.Visible = false end
                    end
                    
                    if Config.ESP.ShowHealth and drawings.HealthText then
                        drawings.HealthText.Visible = true
                        drawings.HealthText.Text = math.floor(humanoid.Health) .. "/" .. math.floor(humanoid.MaxHealth)
                        drawings.HealthText.Position = Vector2.new(screenPos.X, screenPos.Y + boxHeight/2 + 20)
                        drawings.HealthText.Color = Color3.fromRGB(255, 255, 255)
                    else
                        if drawings.HealthText then drawings.HealthText.Visible = false end
                    end
                    
                    if Config.ESP.ShowDistance and drawings.DistanceText then
                        drawings.DistanceText.Visible = true
                        drawings.DistanceText.Text = math.floor(distance) .. "m"
                        drawings.DistanceText.Position = Vector2.new(screenPos.X, screenPos.Y - boxHeight/2 - 35)
                        drawings.DistanceText.Color = Color3.fromRGB(255, 255, 255)
                    else
                        if drawings.DistanceText then drawings.DistanceText.Visible = false end
                    end
                    
                else
                    for _, drawing in pairs(drawings) do
                        drawing.Visible = false
                    end
                end
            else
                for _, drawing in pairs(drawings) do
                    drawing.Visible = false
                end
            end
        else
            for _, drawing in pairs(drawings) do
                drawing.Visible = false
            end
        end
    end
end

-- ==================== HÀM BẬT/TẮT RADAR ====================
local function ToggleRadar(state)
    RadarToggleState = state
    if state then
        -- Bật Radar: Load script từ link
        if not RadarInstance then
            local success, result = pcall(function()
                -- Load và chạy script radar
                RadarInstance = loadstring(game:HttpGet("https://raw.githubusercontent.com/bonhub2013/radar/refs/heads/main/README.md"))()
            end)
            if not success then
                warn("Lỗi khi tải Radar:", result)
                RadarToggleState = false
                return false
            end
        end
    else
        -- Tắt Radar: Dọn dẹp
        if RadarInstance then
            -- Xóa GUI radar nếu nó tự tạo
            local radarGui = game:GetService("CoreGui"):FindFirstChild("RadarUltimateV3")
            if radarGui then
                radarGui:Destroy()
            end
            RadarInstance = nil
        end
    end
    return true
end

-- ==================== MOBILE UI (CÓ NÚT RADAR) ====================
local function CreateMobileUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "VnMobileUI"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    
    if syn and syn.protect_gui then syn.protect_gui(screenGui) end
    screenGui.Parent = game:GetService("CoreGui")
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 320, 0, 700) -- Tăng chiều cao để chứa nút radar
    mainFrame.Position = UDim2.new(0.02, 0, 0.02, 0)
    mainFrame.BackgroundColor3 = Config.UI.BackgroundColor
    mainFrame.BackgroundTransparency = 0.05
    mainFrame.BorderSizePixel = 0
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.ClipsDescendants = true
    mainFrame.Parent = screenGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = mainFrame
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(0, 0, 0)
    stroke.Thickness = 2
    stroke.Transparency = 0.7
    stroke.Parent = mainFrame
    
    local header = Instance.new("Frame")
    header.Size = UDim2.new(1, 0, 0, 50)
    header.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    header.BorderSizePixel = 0
    header.Parent = mainFrame
    
    local headerCorner = Instance.new("UICorner")
    headerCorner.CornerRadius = UDim.new(0, 12)
    headerCorner.Parent = header
    
    local title = Instance.new("TextLabel")
    title.Text = "🎯 V/n CONTROL v7.0"
    title.Size = UDim2.new(1, -60, 1, 0)
    title.Position = UDim2.new(0, 15, 0, 0)
    title.BackgroundTransparency = 1
    title.TextColor3 = Config.UI.AccentColor
    title.Font = Enum.Font.GothamBold
    title.TextSize = 20
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = header
    
    local closeBtn = Instance.new("TextButton")
    closeBtn.Text = "−"
    closeBtn.Size = UDim2.new(0, 40, 0, 40)
    closeBtn.Position = UDim2.new(1, -45, 0, 5)
    closeBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
    closeBtn.TextColor3 = Config.UI.TextColor
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.TextSize = 20
    closeBtn.AutoButtonColor = false
    closeBtn.Parent = header
    
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 8)
    btnCorner.Parent = closeBtn
    
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1, 0, 1, -55)
    scrollFrame.Position = UDim2.new(0, 0, 0, 55)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.ScrollBarThickness = 4
    scrollFrame.ScrollBarImageColor3 = Config.UI.AccentColor
    scrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
    scrollFrame.Parent = mainFrame
    
    local listLayout = Instance.new("UIListLayout")
    listLayout.Padding = UDim.new(0, 12)
    listLayout.Parent = scrollFrame
    
    -- AIMBOT SECTION
    local aimbotSection = Instance.new("Frame")
    aimbotSection.Name = "AimbotSection"
    aimbotSection.Size = UDim2.new(1, -20, 0, 140)
    aimbotSection.Position = UDim2.new(0, 10, 0, 10)
    aimbotSection.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    aimbotSection.BorderSizePixel = 0
    aimbotSection.Parent = scrollFrame
    
    local sectionCorner = Instance.new("UICorner")
    sectionCorner.CornerRadius = UDim.new(0, 8)
    sectionCorner.Parent = aimbotSection
    
    local sectionTitle = Instance.new("TextLabel")
    sectionTitle.Text = "🎯 AUTO AIMBOT"
    sectionTitle.Size = UDim2.new(1, 0, 0, 40)
    sectionTitle.BackgroundTransparency = 1
    sectionTitle.TextColor3 = Config.UI.AccentColor
    sectionTitle.Font = Enum.Font.GothamBold
    sectionTitle.TextSize = 16
    sectionTitle.TextXAlignment = Enum.TextXAlignment.Left
    sectionTitle.Parent = aimbotSection
    
    local titlePadding = Instance.new("UIPadding")
    titlePadding.PaddingLeft = UDim.new(0, 15)
    titlePadding.Parent = sectionTitle
    
    local aimToggleFrame = Instance.new("Frame")
    aimToggleFrame.Size = UDim2.new(1, -30, 0, 40)
    aimToggleFrame.Position = UDim2.new(0, 15, 0, 40)
    aimToggleFrame.BackgroundTransparency = 1
    aimToggleFrame.Parent = aimbotSection
    
    local aimToggleLabel = Instance.new("TextLabel")
    aimToggleLabel.Text = "Tự Động Aimbot"
    aimToggleLabel.Size = UDim2.new(0.7, 0, 1, 0)
    aimToggleLabel.BackgroundTransparency = 1
    aimToggleLabel.TextColor3 = Config.UI.TextColor
    aimToggleLabel.Font = Enum.Font.Gotham
    aimToggleLabel.TextSize = 14
    aimToggleLabel.TextXAlignment = Enum.TextXAlignment.Left
    aimToggleLabel.Parent = aimToggleFrame
    
    local aimToggleBtn = Instance.new("TextButton")
    aimToggleBtn.Text = Config.Aimbot.Enabled and "BẬT" or "TẮT"
    aimToggleBtn.Size = UDim2.new(0, 60, 0, 30)
    aimToggleBtn.Position = UDim2.new(1, -60, 0.5, -15)
    aimToggleBtn.BackgroundColor3 = Config.Aimbot.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    aimToggleBtn.TextColor3 = Config.UI.TextColor
    aimToggleBtn.Font = Enum.Font.Gotham
    aimToggleBtn.TextSize = 14
    aimToggleBtn.AutoButtonColor = false
    aimToggleBtn.Parent = aimToggleFrame
    
    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 6)
    toggleCorner.Parent = aimToggleBtn
    
    aimToggleBtn.MouseButton1Click:Connect(function()
        Config.Aimbot.Enabled = not Config.Aimbot.Enabled
        aimToggleBtn.Text = Config.Aimbot.Enabled and "BẬT" or "TẮT"
        aimToggleBtn.BackgroundColor3 = Config.Aimbot.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    end)
    
    local wallToggleFrame = Instance.new("Frame")
    wallToggleFrame.Size = UDim2.new(1, -30, 0, 40)
    wallToggleFrame.Position = UDim2.new(0, 15, 0, 90)
    wallToggleFrame.BackgroundTransparency = 1
    wallToggleFrame.Parent = aimbotSection
    
    local wallToggleLabel = Instance.new("TextLabel")
    wallToggleLabel.Text = "Wall Check"
    wallToggleLabel.Size = UDim2.new(0.7, 0, 1, 0)
    wallToggleLabel.BackgroundTransparency = 1
    wallToggleLabel.TextColor3 = Config.UI.TextColor
    wallToggleLabel.Font = Enum.Font.Gotham
    wallToggleLabel.TextSize = 14
    wallToggleLabel.TextXAlignment = Enum.TextXAlignment.Left
    wallToggleLabel.Parent = wallToggleFrame
    
    local wallToggleBtn = Instance.new("TextButton")
    wallToggleBtn.Text = Config.WallCheck.Enabled and "BẬT" or "TẮT"
    wallToggleBtn.Size = UDim2.new(0, 60, 0, 30)
    wallToggleBtn.Position = UDim2.new(1, -60, 0.5, -15)
    wallToggleBtn.BackgroundColor3 = Config.WallCheck.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    wallToggleBtn.TextColor3 = Config.UI.TextColor
    wallToggleBtn.Font = Enum.Font.Gotham
    wallToggleBtn.TextSize = 14
    wallToggleBtn.AutoButtonColor = false
    wallToggleBtn.Parent = wallToggleFrame
    
    local wallCorner = Instance.new("UICorner")
    wallCorner.CornerRadius = UDim.new(0, 6)
    wallCorner.Parent = wallToggleBtn
    
    wallToggleBtn.MouseButton1Click:Connect(function()
        Config.WallCheck.Enabled = not Config.WallCheck.Enabled
        wallToggleBtn.Text = Config.WallCheck.Enabled and "BẬT" or "TẮT"
        wallToggleBtn.BackgroundColor3 = Config.WallCheck.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    end)
    
    -- ESP SECTION
    local espSection = Instance.new("Frame")
    espSection.Name = "ESPSection"
    espSection.Size = UDim2.new(1, -20, 0, 250)
    espSection.Position = UDim2.new(0, 10, 0, 162)
    espSection.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    espSection.BorderSizePixel = 0
    espSection.Parent = scrollFrame
    
    local espCorner = Instance.new("UICorner")
    espCorner.CornerRadius = UDim.new(0, 8)
    espCorner.Parent = espSection
    
    local espTitle = Instance.new("TextLabel")
    espTitle.Text = "👁️ ESP"
    espTitle.Size = UDim2.new(1, 0, 0, 40)
    espTitle.BackgroundTransparency = 1
    espTitle.TextColor3 = Config.UI.AccentColor
    espTitle.Font = Enum.Font.GothamBold
    espTitle.TextSize = 16
    espTitle.TextXAlignment = Enum.TextXAlignment.Left
    espTitle.Parent = espSection
    
    local espPadding = Instance.new("UIPadding")
    espPadding.PaddingLeft = UDim.new(0, 15)
    espPadding.Parent = espTitle
    
    local espToggleFrame = Instance.new("Frame")
    espToggleFrame.Size = UDim2.new(1, -30, 0, 40)
    espToggleFrame.Position = UDim2.new(0, 15, 0, 40)
    espToggleFrame.BackgroundTransparency = 1
    espToggleFrame.Parent = espSection
    
    local espToggleLabel = Instance.new("TextLabel")
    espToggleLabel.Text = "Bật ESP"
    espToggleLabel.Size = UDim2.new(0.7, 0, 1, 0)
    espToggleLabel.BackgroundTransparency = 1
    espToggleLabel.TextColor3 = Config.UI.TextColor
    espToggleLabel.Font = Enum.Font.Gotham
    espToggleLabel.TextSize = 14
    espToggleLabel.TextXAlignment = Enum.TextXAlignment.Left
    espToggleLabel.Parent = espToggleFrame
    
    local espToggleBtn = Instance.new("TextButton")
    espToggleBtn.Text = Config.ESP.Enabled and "BẬT" or "TẮT"
    espToggleBtn.Size = UDim2.new(0, 60, 0, 30)
    espToggleBtn.Position = UDim2.new(1, -60, 0.5, -15)
    espToggleBtn.BackgroundColor3 = Config.ESP.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    espToggleBtn.TextColor3 = Config.UI.TextColor
    espToggleBtn.Font = Enum.Font.Gotham
    espToggleBtn.TextSize = 14
    espToggleBtn.AutoButtonColor = false
    espToggleBtn.Parent = espToggleFrame
    
    local espBtnCorner = Instance.new("UICorner")
    espBtnCorner.CornerRadius = UDim.new(0, 6)
    espBtnCorner.Parent = espToggleBtn
    
    espToggleBtn.MouseButton1Click:Connect(function()
        Config.ESP.Enabled = not Config.ESP.Enabled
        espToggleBtn.Text = Config.ESP.Enabled and "BẬT" or "TẮT"
        espToggleBtn.BackgroundColor3 = Config.ESP.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
        UpdateESP()
    end)
    
    local featuresGrid = Instance.new("Frame")
    featuresGrid.Size = UDim2.new(1, -30, 0, 70)
    featuresGrid.Position = UDim2.new(0, 15, 0, 90)
    featuresGrid.BackgroundTransparency = 1
    featuresGrid.Parent = espSection
    
    local features = {
        {name = "Tên", config = "ShowName", row = 0, col = 0},
        {name = "Thanh Máu", config = "HealthBar", row = 0, col = 1},
        {name = "Số Máu", config = "ShowHealth", row = 1, col = 0},
        {name = "Khoảng Cách", config = "ShowDistance", row = 1, col = 1},
    }
    
    for _, feature in pairs(features) do
        local btn = Instance.new("TextButton")
        btn.Text = feature.name
        btn.Size = UDim2.new(0.48, 0, 0, 30)
        btn.Position = UDim2.new(feature.col * 0.5, 0, feature.row * 0.5, 0)
        btn.BackgroundColor3 = Config.ESP[feature.config] and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
        btn.TextColor3 = Config.UI.TextColor
        btn.Font = Enum.Font.Gotham
        btn.TextSize = 12
        btn.AutoButtonColor = false
        btn.Parent = featuresGrid
        
        local featureCorner = Instance.new("UICorner")
        featureCorner.CornerRadius = UDim.new(0, 6)
        featureCorner.Parent = btn
        
        btn.MouseButton1Click:Connect(function()
            Config.ESP[feature.config] = not Config.ESP[feature.config]
            btn.BackgroundColor3 = Config.ESP[feature.config] and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
            UpdateESP()
        end)
    end
    
    local boxHighlightFrame = Instance.new("Frame")
    boxHighlightFrame.Size = UDim2.new(1, -30, 0, 40)
    boxHighlightFrame.Position = UDim2.new(0, 15, 0, 170)
    boxHighlightFrame.BackgroundTransparency = 1
    boxHighlightFrame.Parent = espSection
    
    local boxBtn = Instance.new("TextButton")
    boxBtn.Text = Config.ESP.ShowBox and "📦 BOX: BẬT" or "📦 BOX: TẮT"
    boxBtn.Size = UDim2.new(0.48, 0, 0, 30)
    boxBtn.Position = UDim2.new(0, 0, 0, 0)
    boxBtn.BackgroundColor3 = Config.ESP.ShowBox and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    boxBtn.TextColor3 = Config.UI.TextColor
    boxBtn.Font = Enum.Font.Gotham
    boxBtn.TextSize = 12
    boxBtn.AutoButtonColor = false
    boxBtn.Parent = boxHighlightFrame
    
    local boxCorner = Instance.new("UICorner")
    boxCorner.CornerRadius = UDim.new(0, 6)
    boxCorner.Parent = boxBtn
    
    boxBtn.MouseButton1Click:Connect(function()
        Config.ESP.ShowBox = not Config.ESP.ShowBox
        boxBtn.Text = Config.ESP.ShowBox and "📦 BOX: BẬT" or "📦 BOX: TẮT"
        boxBtn.BackgroundColor3 = Config.ESP.ShowBox and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
        UpdateESP()
    end)
    
    local highlightBtn = Instance.new("TextButton")
    highlightBtn.Text = Config.ESP.ShowHighlight and "✨ HIGHLIGHT: BẬT" or "✨ HIGHLIGHT: TẮT"
    highlightBtn.Size = UDim2.new(0.48, 0, 0, 30)
    highlightBtn.Position = UDim2.new(0.52, 0, 0, 0)
    highlightBtn.BackgroundColor3 = Config.ESP.ShowHighlight and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    highlightBtn.TextColor3 = Config.UI.TextColor
    highlightBtn.Font = Enum.Font.Gotham
    highlightBtn.TextSize = 12
    highlightBtn.AutoButtonColor = false
    highlightBtn.Parent = boxHighlightFrame
    
    local highlightCorner = Instance.new("UICorner")
    highlightCorner.CornerRadius = UDim.new(0, 6)
    highlightCorner.Parent = highlightBtn
    
    highlightBtn.MouseButton1Click:Connect(function()
        Config.ESP.ShowHighlight = not Config.ESP.ShowHighlight
        highlightBtn.Text = Config.ESP.ShowHighlight and "✨ HIGHLIGHT: BẬT" or "✨ HIGHLIGHT: TẮT"
        highlightBtn.BackgroundColor3 = Config.ESP.ShowHighlight and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
        UpdateESP()
    end)
    
    -- POV SECTION
    local povSection = Instance.new("Frame")
    povSection.Name = "POVSection"
    povSection.Size = UDim2.new(1, -20, 0, 130)
    povSection.Position = UDim2.new(0, 10, 0, 422)
    povSection.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    povSection.BorderSizePixel = 0
    povSection.Parent = scrollFrame
    
    local povCorner = Instance.new("UICorner")
    povCorner.CornerRadius = UDim.new(0, 8)
    povCorner.Parent = povSection
    
    local povTitle = Instance.new("TextLabel")
    povTitle.Text = "⭕ POV CIRCLE"
    povTitle.Size = UDim2.new(1, 0, 0, 40)
    povTitle.BackgroundTransparency = 1
    povTitle.TextColor3 = Config.UI.AccentColor
    povTitle.Font = Enum.Font.GothamBold
    povTitle.TextSize = 16
    povTitle.TextXAlignment = Enum.TextXAlignment.Left
    povTitle.Parent = povSection
    
    local povPadding = Instance.new("UIPadding")
    povPadding.PaddingLeft = UDim.new(0, 15)
    povPadding.Parent = povTitle
    
    local povToggleFrame = Instance.new("Frame")
    povToggleFrame.Size = UDim2.new(1, -30, 0, 40)
    povToggleFrame.Position = UDim2.new(0, 15, 0, 40)
    povToggleFrame.BackgroundTransparency = 1
    povToggleFrame.Parent = povSection
    
    local povToggleLabel = Instance.new("TextLabel")
    povToggleLabel.Text = "Vòng POV"
    povToggleLabel.Size = UDim2.new(0.7, 0, 1, 0)
    povToggleLabel.BackgroundTransparency = 1
    povToggleLabel.TextColor3 = Config.UI.TextColor
    povToggleLabel.Font = Enum.Font.Gotham
    povToggleLabel.TextSize = 14
    povToggleLabel.TextXAlignment = Enum.TextXAlignment.Left
    povToggleLabel.Parent = povToggleFrame
    
    local povToggleBtn = Instance.new("TextButton")
    povToggleBtn.Text = Config.POVCircle.Enabled and "BẬT" or "TẮT"
    povToggleBtn.Size = UDim2.new(0, 60, 0, 30)
    povToggleBtn.Position = UDim2.new(1, -60, 0.5, -15)
    povToggleBtn.BackgroundColor3 = Config.POVCircle.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    povToggleBtn.TextColor3 = Config.UI.TextColor
    povToggleBtn.Font = Enum.Font.Gotham
    povToggleBtn.TextSize = 14
    povToggleBtn.AutoButtonColor = false
    povToggleBtn.Parent = povToggleFrame
    
    local povBtnCorner = Instance.new("UICorner")
    povBtnCorner.CornerRadius = UDim.new(0, 6)
    povBtnCorner.Parent = povToggleBtn
    
    povToggleBtn.MouseButton1Click:Connect(function()
        Config.POVCircle.Enabled = not Config.POVCircle.Enabled
        povToggleBtn.Text = Config.POVCircle.Enabled and "BẬT" or "TẮT"
        povToggleBtn.BackgroundColor3 = Config.POVCircle.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
        if Drawings.POV.Outer then
            Drawings.POV.Outer.Visible = Config.POVCircle.Enabled
            Drawings.POV.Inner.Visible = Config.POVCircle.Enabled
        end
    end)
    
    local radiusFrame = Instance.new("Frame")
    radiusFrame.Size = UDim2.new(1, -30, 0, 40)
    radiusFrame.Position = UDim2.new(0, 15, 0, 90)
    radiusFrame.BackgroundTransparency = 1
    radiusFrame.Parent = povSection
    
    local radiusLabel = Instance.new("TextLabel")
    radiusLabel.Text = "Bán kính: " .. Config.POVCircle.OuterRadius
    radiusLabel.Size = UDim2.new(0.6, 0, 1, 0)
    radiusLabel.BackgroundTransparency = 1
    radiusLabel.TextColor3 = Config.UI.TextColor
    radiusLabel.Font = Enum.Font.Gotham
    radiusLabel.TextSize = 14
    radiusLabel.TextXAlignment = Enum.TextXAlignment.Left
    radiusLabel.Name = "RadiusLabel"
    radiusLabel.Parent = radiusFrame
    
    local radiusInput = Instance.new("TextBox")
    radiusInput.Text = tostring(Config.POVCircle.OuterRadius)
    radiusInput.Size = UDim2.new(0.35, 0, 0.7, 0)
    radiusInput.Position = UDim2.new(0.65, 0, 0.15, 0)
    radiusInput.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
    radiusInput.TextColor3 = Config.UI.TextColor
    radiusInput.Font = Enum.Font.Gotham
    radiusInput.TextSize = 14
    radiusInput.PlaceholderText = "50-200"
    radiusInput.Parent = radiusFrame
    
    radiusInput.FocusLost:Connect(function()
        local num = tonumber(radiusInput.Text)
        if num then
            num = math.clamp(num, Config.POVCircle.MinRadius, Config.POVCircle.MaxRadius)
            Config.POVCircle.OuterRadius = num
            Config.POVCircle.InnerRadius = math.floor(num / 2)
            Config.Aimbot.FOV = num
            radiusLabel.Text = "Bán kính: " .. num
            radiusInput.Text = tostring(num)
            if Drawings.POV.Outer then
                Drawings.POV.Outer.Radius = num
                Drawings.POV.Inner.Radius = math.floor(num / 2)
            end
        end
    end)
    
    -- ==================== NÚT RADAR BUTTON (ĐÃ THÊM) ====================
    local radarSection = Instance.new("Frame")
    radarSection.Name = "RadarToggleSection"
    radarSection.Size = UDim2.new(1, -20, 0, 60)
    radarSection.Position = UDim2.new(0, 10, 0, 562)
    radarSection.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    radarSection.BorderSizePixel = 0
    radarSection.Parent = scrollFrame

    local radarCorner = Instance.new("UICorner")
    radarCorner.CornerRadius = UDim.new(0, 8)
    radarCorner.Parent = radarSection

    local radarTitle = Instance.new("TextLabel")
    radarTitle.Text = "📡 RADAR ULTIMATE V3"
    radarTitle.Size = UDim2.new(1, -30, 0, 30)
    radarTitle.Position = UDim2.new(0, 15, 0, 5)
    radarTitle.BackgroundTransparency = 1
    radarTitle.TextColor3 = Config.UI.AccentColor
    radarTitle.Font = Enum.Font.GothamBold
    radarTitle.TextSize = 16
    radarTitle.TextXAlignment = Enum.TextXAlignment.Left
    radarTitle.Parent = radarSection

    local radarToggleFrame = Instance.new("Frame")
    radarToggleFrame.Size = UDim2.new(1, -30, 0, 30)
    radarToggleFrame.Position = UDim2.new(0, 15, 0, 35)
    radarToggleFrame.BackgroundTransparency = 1
    radarToggleFrame.Parent = radarSection

    local radarToggleLabel = Instance.new("TextLabel")
    radarToggleLabel.Text = "Bật Radar"
    radarToggleLabel.Size = UDim2.new(0.7, 0, 1, 0)
    radarToggleLabel.BackgroundTransparency = 1
    radarToggleLabel.TextColor3 = Config.UI.TextColor
    radarToggleLabel.Font = Enum.Font.Gotham
    radarToggleLabel.TextSize = 14
    radarToggleLabel.TextXAlignment = Enum.TextXAlignment.Left
    radarToggleLabel.Parent = radarToggleFrame

    local radarToggleBtn = Instance.new("TextButton")
    radarToggleBtn.Name = "RadarToggleButton"
    radarToggleBtn.Text = "TẮT"
    radarToggleBtn.Size = UDim2.new(0, 60, 0, 25)
    radarToggleBtn.Position = UDim2.new(1, -60, 0.5, -12.5)
    radarToggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
    radarToggleBtn.TextColor3 = Config.UI.TextColor
    radarToggleBtn.Font = Enum.Font.Gotham
    radarToggleBtn.TextSize = 14
    radarToggleBtn.AutoButtonColor = false
    radarToggleBtn.Parent = radarToggleFrame

    local radarBtnCorner = Instance.new("UICorner")
    radarBtnCorner.CornerRadius = UDim.new(0, 6)
    radarBtnCorner.Parent = radarToggleBtn

    -- Xử lý sự kiện click cho nút Radar
    radarToggleBtn.MouseButton1Click:Connect(function()
        RadarToggleState = not RadarToggleState
        ToggleRadar(RadarToggleState)
        radarToggleBtn.Text = RadarToggleState and "BẬT" or "TẮT"
        radarToggleBtn.BackgroundColor3 = RadarToggleState and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
    end)
    -- ==================== KẾT THÚC NÚT RADAR ====================
    
    closeBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = not mainFrame.Visible
        closeBtn.Text = mainFrame.Visible and "−" or "+"
        State.UIVisible = mainFrame.Visible
    end)
    
    UserInputService.InputBegan:Connect(function(input)
        if input.KeyCode == Config.UI.ToggleKey then
            mainFrame.Visible = not mainFrame.Visible
            closeBtn.Text = mainFrame.Visible and "−" or "+"
            State.UIVisible = mainFrame.Visible
        end
    end)
    
    return screenGui
end

-- ==================== MAIN LOOP ====================
local function MainLoop()
    local lastPrint = 0
    
    while task.wait(0.016) do
        UpdatePOVPosition()
        local isAiming = PerformAutoAim()
        UpdateESP()
        
        local currentTime = tick()
        if currentTime - lastPrint > 3 then
            if isAiming and State.Target then
                print("🎯 Đang aim: " .. State.Target.Player.Name .. 
                      " | Khoảng cách: " .. math.floor(State.Target.Distance) .. "m" ..
                      (State.TargetBehindWall and " [KHUẤT TƯỜNG]" or ""))
            end
            lastPrint = currentTime
        end
    end
end

-- ==================== KHỞI TẠO ====================
print([[
╔══════════════════════════════════════════╗
║    V/n ULTIMATE SUITE v7.0               ║
║    AIMBOT + ESP + POV + RADAR BUTTON     ║
╚══════════════════════════════════════════╝
]])

-- Khởi tạo POV
CreatePOVCircles()

-- Khởi tạo Mobile UI (có nút radar)
CreateMobileUI()

-- Tạo ESP cho player hiện tại
for _, player in pairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        CreateESP(player)
    end
end

-- PlayerAdded handler
Players.PlayerAdded:Connect(function(player)
    if player == LocalPlayer then return end
    CreateESP(player)
    
    player.CharacterAdded:Connect(function()
        task.wait(0.5)
        if Drawings.ESP[player] then
            for _, drawing in pairs(Drawings.ESP[player]) do
                pcall(function() drawing:Remove() end)
            end
            Drawings.ESP[player] = nil
        end
        CreateESP(player)
    end)
    
    if player.Character then
        task.wait(0.5)
        CreateESP(player)
    end
end)

-- PlayerRemoving handler
Players.PlayerRemoving:Connect(function(player)
    if Drawings.ESP[player] then
        for _, drawing in pairs(Drawings.ESP[player]) do
            drawing:Remove()
        end
        Drawings.ESP[player] = nil
    end
end)

-- Anti-Afk
LocalPlayer.Idled:Connect(function()
    game:GetService("VirtualUser"):CaptureController()
    game:GetService("VirtualUser"):ClickButton2(Vector2.new())
end)

-- Chạy main loop
task.spawn(MainLoop)

-- ==================== CLEANUP ====================
local function Cleanup()
    print("[V/n] Đang dọn dẹp...")
    
    -- Cleanup POV
    if Drawings.POV.Outer then
        Drawings.POV.Outer:Remove()
        Drawings.POV.Inner:Remove()
    end
    
    -- Cleanup ESP
    for _, drawings in pairs(Drawings.ESP) do
        for _, drawing in pairs(drawings) do
            pcall(function() drawing:Remove() end)
        end
    end
    
    -- Tắt radar nếu đang chạy
    if RadarToggleState then
        ToggleRadar(false)
    end
    
    print("[V/n] Dọn dẹp hoàn tất!")
end

game:BindToClose(Cleanup)

print("✅ V/n v7.0 đã sẵn sàng!")
print("👉 Nhấn Right Shift để mở UI")
print("🎯 Aimbot chỉ aim trong POV, có Box ESP + Highlight")
print("📡 Nút Radar nằm ở cuối UI - Bấm để bật/tắt")
