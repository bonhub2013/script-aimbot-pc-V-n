-- V/n ULTIMATE COMBAT SUITE v6.0
-- Hoàn chỉnh: Auto Aimbot + POV + Wall Check + ESP + Mobile UI

-- ==================== KHAI BÁO DỊCH VỤ ====================
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

print("🚀 V/n Ultimate Suite v6.0 đang khởi động...")

-- ==================== CẤU HÌNH HOÀN CHỈNH ====================
local Config = {
    Aimbot = {
        Enabled = true, -- Auto aim không cần phím
        Strength = 0.75, -- 75% lock strength
        Smoothing = 0.2,
        FOV = 200,
        Bone = "Head",
        TeamCheck = true,
        WallCheck = true,
        AutoAim = true, -- Tự động aim
        Prediction = 0.15,
        FirstPerson = true,
        ThirdPerson = true,
        MaxDistance = 500
    },
    
    POVCircle = {
        Enabled = true,
        OuterRadius = 150, -- 50-200
        InnerRadius = 75,
        ColorNormal = Color3.fromRGB(0, 0, 0), -- Đen
        ColorTarget = Color3.fromRGB(0, 255, 100), -- Xanh lá
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
        TextSize = 14
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
    UIVisible = true
}

-- ==================== WALL CHECK THÔNG MINH ====================
local function IsValidTarget(targetPosition, targetCharacter)
    if not Config.WallCheck.Enabled then return true end
    
    local myChar = LocalPlayer.Character
    if not myChar then return false end
    
    local myHead = myChar:FindFirstChild("Head")
    if not myHead then return false end
    
    local distance = (targetPosition - myHead.Position).Magnitude
    if distance > Config.WallCheck.MaxDistance then return false end
    
    -- Tạo raycast parameters
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.FilterDescendantsInstances = {myChar}
    params.IgnoreWater = true
    
    -- Thêm tool/weapon vào blacklist
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
    
    -- Thực hiện raycast
    local origin = myHead.Position + Vector3.new(0, 1, 0)
    local direction = (targetPosition - origin).Unit
    local result = Workspace:Raycast(origin, direction * distance, params)
    
    if result then
        local hit = result.Instance
        
        -- Bỏ qua nếu là character của target
        if targetCharacter and hit:IsDescendantOf(targetCharacter) then
            return true
        end
        
        -- Bỏ qua vật liệu trong suốt
        if hit.Material == Enum.Material.Glass or 
           hit.Material == Enum.Material.Water or
           hit.Material == Enum.Material.Air or
           hit.Material == Enum.Material.ForceField then
            return true
        end
        
        -- Có tường chắn
        return false
    end
    
    return true
end

-- ==================== VÒNG POV HÌNH TRÒN ====================
local function CreatePOVCircles()
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

-- ==================== AUTO AIMBOT 75% ====================
local function FindBestTarget()
    if not Config.Aimbot.Enabled then return nil end
    
    local bestTarget = nil
    local bestDistance = Config.Aimbot.FOV
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    
    for _, player in pairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        
        if Config.Aimbot.TeamCheck then
            if player.Team and LocalPlayer.Team and player.Team == LocalPlayer.Team then
                continue
            end
        end
        
        local character = player.Character
        if not character then continue end
        
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid or humanoid.Health <= 0 then continue end
        
        local targetPart = character:FindFirstChild(Config.Aimbot.Bone) or
                          character:FindFirstChild("HumanoidRootPart") or
                          character:FindFirstChild("Torso")
        if not targetPart then continue end
        
        -- Kiểm tra khoảng cách
        local distance = (targetPart.Position - Camera.CFrame.Position).Magnitude
        if distance > Config.Aimbot.MaxDistance then continue end
        
        -- Wall check
        if Config.Aimbot.WallCheck then
            if not IsValidTarget(targetPart.Position, character) then
                continue
            end
        end
        
        -- Kiểm tra trên màn hình
        local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
        if not onScreen then continue end
        
        local pos2D = Vector2.new(screenPos.X, screenPos.Y)
        local screenDistance = (pos2D - screenCenter).Magnitude
        
        -- Kiểm tra trong FOV và POV circle
        if screenDistance < bestDistance and screenDistance <= Config.POVCircle.OuterRadius then
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
    
    return bestTarget
end

local function PerformAutoAim()
    if not Config.Aimbot.Enabled or not Config.Aimbot.AutoAim then
        UpdatePOVColor(false)
        return false
    end
    
    local currentTime = tick()
    
    -- Kiểm tra target cũ còn hợp lệ không
    if State.Target then
        local target = State.Target
        local character = target.Player.Character
        
        if character then
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            local targetPart = character:FindFirstChild(Config.Aimbot.Bone) or
                              character:FindFirstChild("HumanoidRootPart")
            
            if humanoid and humanoid.Health > 0 and targetPart then
                -- Kiểm tra wall check
                if Config.Aimbot.WallCheck then
                    if not IsValidTarget(targetPart.Position, character) then
                        State.Target = nil
                    end
                end
                
                -- Kiểm tra trên màn hình
                local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                if onScreen then
                    -- Cập nhật target
                    State.Target.Part = targetPart
                    State.Target.Position = targetPart.Position
                    State.LastTargetTime = currentTime
                    
                    -- Thực hiện aim
                    local currentCF = Camera.CFrame
                    local targetPos = targetPart.Position
                    
                    -- Dự đoán chuyển động
                    if targetPart.Velocity then
                        targetPos = targetPos + (targetPart.Velocity * Config.Aimbot.Prediction)
                    end
                    
                    -- 75% lock strength
                    local desiredDirection = (targetPos - currentCF.Position).Unit
                    local currentDirection = currentCF.LookVector
                    local newDirection = currentDirection:Lerp(desiredDirection, Config.Aimbot.Strength * Config.Aimbot.Smoothing)
                    
                    -- Cập nhật camera
                    Camera.CFrame = CFrame.new(currentCF.Position, currentCF.Position + newDirection)
                    
                    UpdatePOVColor(true)
                    return true
                end
            end
        end
        
        State.Target = nil
    end
    
    -- Tìm target mới
    if not State.Target or (currentTime - State.LastTargetTime) > State.TargetLockTime then
        local newTarget = FindBestTarget()
        
        if newTarget then
            State.Target = newTarget
            State.LastTargetTime = currentTime
            UpdatePOVColor(true)
            return true
        else
            State.Target = nil
            UpdatePOVColor(false)
        end
    end
    
    UpdatePOVColor(false)
    return false
end

-- ==================== ESP 4000M ====================
local function CreateESP(player)
    if Drawings.ESP[player] then return end
    
    local drawings = {
        Name = Drawing.new("Text"),
        HealthBar = Drawing.new("Square"),
        HealthBarFill = Drawing.new("Square"),
        HealthText = Drawing.new("Text"),
        DistanceText = Drawing.new("Text")
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
                    -- Xác định màu
                    local color = Config.ESP.EnemyColor
                    if Config.ESP.TeamColor and player.Team and LocalPlayer.Team and player.Team == LocalPlayer.Team then
                        color = Config.ESP.FriendlyColor
                    end
                    
                    -- TÊN
                    if Config.ESP.ShowName then
                        drawings.Name.Visible = true
                        drawings.Name.Text = player.Name
                        drawings.Name.Position = Vector2.new(screenPos.X, screenPos.Y - 35)
                        drawings.Name.Color = color
                    else
                        drawings.Name.Visible = false
                    end
                    
                    -- THANH MÁU
                    if Config.ESP.HealthBar then
                        local healthPercent = humanoid.Health / humanoid.MaxHealth
                        local barWidth = 70
                        local barHeight = 6
                        
                        -- Nền thanh máu
                        drawings.HealthBar.Visible = true
                        drawings.HealthBar.Size = Vector2.new(barWidth, barHeight)
                        drawings.HealthBar.Position = Vector2.new(screenPos.X - barWidth/2, screenPos.Y - 25)
                        drawings.HealthBar.Color = Color3.fromRGB(40, 40, 40)
                        drawings.HealthBar.Filled = true
                        
                        -- Thanh máu thực
                        drawings.HealthBarFill.Visible = true
                        drawings.HealthBarFill.Size = Vector2.new(barWidth * healthPercent, barHeight)
                        drawings.HealthBarFill.Position = Vector2.new(screenPos.X - barWidth/2, screenPos.Y - 25)
                        drawings.HealthBarFill.Color = Color3.fromRGB(
                            255 * (1 - healthPercent),
                            255 * healthPercent,
                            0
                        )
                        drawings.HealthBarFill.Filled = true
                        
                        -- SỐ MÁU
                        if Config.ESP.ShowHealth then
                            drawings.HealthText.Visible = true
                            drawings.HealthText.Text = math.floor(humanoid.Health) .. "/" .. math.floor(humanoid.MaxHealth)
                            drawings.HealthText.Position = Vector2.new(screenPos.X, screenPos.Y - 45)
                            drawings.HealthText.Color = Color3.fromRGB(255, 255, 255)
                        else
                            drawings.HealthText.Visible = false
                        end
                    else
                        drawings.HealthBar.Visible = false
                        drawings.HealthBarFill.Visible = false
                        drawings.HealthText.Visible = false
                    end
                    
                    -- KHOẢNG CÁCH
                    if Config.ESP.ShowDistance then
                        drawings.DistanceText.Visible = true
                        drawings.DistanceText.Text = math.floor(distance) .. "m"
                        drawings.DistanceText.Position = Vector2.new(screenPos.X, screenPos.Y - 55)
                        drawings.DistanceText.Color = Color3.fromRGB(255, 255, 255)
                    else
                        drawings.DistanceText.Visible = false
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

-- ==================== MOBILE-STYLE UI ====================
local function CreateMobileUI()
    -- Tạo ScreenGui
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "VnMobileUI"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    
    if syn and syn.protect_gui then
        syn.protect_gui(screenGui)
    end
    
    screenGui.Parent = game:GetService("CoreGui")
    
    -- Main Container
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 320, 0, 480)
    mainFrame.Position = UDim2.new(0.02, 0, 0.02, 0)
    mainFrame.BackgroundColor3 = Config.UI.BackgroundColor
    mainFrame.BackgroundTransparency = 0.05
    mainFrame.BorderSizePixel = 0
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.ClipsDescendants = true
    mainFrame.Parent = screenGui
    
    -- Rounded corners
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = mainFrame
    
    -- Shadow
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(0, 0, 0)
    stroke.Thickness = 2
    stroke.Transparency = 0.7
    stroke.Parent = mainFrame
    
    -- Header
    local header = Instance.new("Frame")
    header.Size = UDim2.new(1, 0, 0, 50)
    header.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    header.BorderSizePixel = 0
    header.Parent = mainFrame
    
    local headerCorner = Instance.new("UICorner")
    headerCorner.CornerRadius = UDim.new(0, 12)
    headerCorner.Parent = header
    
    -- Title
    local title = Instance.new("TextLabel")
    title.Text = "🎯 V/n CONTROL"
    title.Size = UDim2.new(1, -60, 1, 0)
    title.Position = UDim2.new(0, 15, 0, 0)
    title.BackgroundTransparency = 1
    title.TextColor3 = Config.UI.AccentColor
    title.Font = Enum.Font.GothamBold
    title.TextSize = 22
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = header
    
    -- Close button
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
    
    -- Content area
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
    
    -- ==================== AIMBOT SECTION ====================
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
    sectionTitle.Text = "🎯 AUTO AIMBOT (75%)"
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
    
    -- Aimbot toggle
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
    
    -- Wall check toggle
    local wallToggleFrame = Instance.new("Frame")
    wallToggleFrame.Size = UDim2.new(1, -30, 0, 40)
    wallToggleFrame.Position = UDim2.new(0, 15, 0, 90)
    wallToggleFrame.BackgroundTransparency = 1
    wallToggleFrame.Parent = aimbotSection
    
    local wallToggleLabel = Instance.new("TextLabel")
    wallToggleLabel.Text = "Wall Check (Bỏ qua súng)"
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
    
    -- ==================== ESP SECTION ====================
    local espSection = Instance.new("Frame")
    espSection.Name = "ESPSection"
    espSection.Size = UDim2.new(1, -20, 0, 160)
    espSection.Position = UDim2.new(0, 10, 0, 162)
    espSection.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    espSection.BorderSizePixel = 0
    espSection.Parent = scrollFrame
    
    local espCorner = Instance.new("UICorner")
    espCorner.CornerRadius = UDim.new(0, 8)
    espCorner.Parent = espSection
    
    local espTitle = Instance.new("TextLabel")
    espTitle.Text = "👁️ ESP (4000m)"
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
    
    -- ESP toggle
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
    
    -- ESP features grid
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
    
    -- ==================== POV CIRCLE SECTION ====================
    local povSection = Instance.new("Frame")
    povSection.Name = "POVSection"
    povSection.Size = UDim2.new(1, -20, 0, 130)
    povSection.Position = UDim2.new(0, 10, 0, 332)
    povSection.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    povSection.BorderSizePixel = 0
    povSection.Parent = scrollFrame
    
    local povCorner = Instance.new("UICorner")
    povCorner.CornerRadius = UDim.new(0, 8)
    povCorner.Parent = povSection
    
    local povTitle = Instance.new("TextLabel")
    povTitle.Text = "⭕ POV CIRCLE (50-200)"
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
    
    -- POV toggle
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
    
    -- Radius slider
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
            radiusLabel.Text = "Bán kính: " .. num
            radiusInput.Text = tostring(num)
            
            if Drawings.POV.Outer then
                Drawings.POV.Outer.Radius = num
                Drawings.POV.Inner.Radius = math.floor(num / 2)
            end
        end
    end)
    
    -- ==================== QUICK ACTIONS BAR ====================
    local quickActions = Instance.new("Frame")
    quickActions.Size = UDim2.new(1, -20, 0, 60)
    quickActions.Position = UDim2.new(0, 10, 0, 472)
    quickActions.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
    quickActions.BorderSizePixel = 0
    quickActions.Parent = scrollFrame
    
    local actionsCorner = Instance.new("UICorner")
    actionsCorner.CornerRadius = UDim.new(0, 8)
    actionsCorner.Parent = quickActions
    
    local actions = {
        {text = "🎯 AIM", func = function()
            Config.Aimbot.Enabled = not Config.Aimbot.Enabled
            aimToggleBtn.Text = Config.Aimbot.Enabled and "BẬT" or "TẮT"
            aimToggleBtn.BackgroundColor3 = Config.Aimbot.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
        end},
        {text = "👁️ ESP", func = function()
            Config.ESP.Enabled = not Config.ESP.Enabled
            espToggleBtn.Text = Config.ESP.Enabled and "BẬT" or "TẮT"
            espToggleBtn.BackgroundColor3 = Config.ESP.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
            UpdateESP()
        end},
        {text = "🧱 WALL", func = function()
            Config.WallCheck.Enabled = not Config.WallCheck.Enabled
            wallToggleBtn.Text = Config.WallCheck.Enabled and "BẬT" or "TẮT"
            wallToggleBtn.BackgroundColor3 = Config.WallCheck.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
        end},
        {text = "⭕ POV", func = function()
            Config.POVCircle.Enabled = not Config.POVCircle.Enabled
            if Drawings.POV.Outer then
                Drawings.POV.Outer.Visible = Config.POVCircle.Enabled
                Drawings.POV.Inner.Visible = Config.POVCircle.Enabled
            end
            povToggleBtn.Text = Config.POVCircle.Enabled and "BẬT" or "TẮT"
            povToggleBtn.BackgroundColor3 = Config.POVCircle.Enabled and Config.UI.AccentColor or Color3.fromRGB(60, 60, 80)
        end}
    }
    
    for i, action in pairs(actions) do
        local actionBtn = Instance.new("TextButton")
        actionBtn.Text = action.text
        actionBtn.Size = UDim2.new(0.22, 0, 0.7, 0)
        actionBtn.Position = UDim2.new(0.02 + (i-1)*0.245, 0, 0.15, 0)
        actionBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
        actionBtn.TextColor3 = Config.UI.TextColor
        actionBtn.Font = Enum.Font.Gotham
        actionBtn.TextSize = 12
        actionBtn.AutoButtonColor = false
        actionBtn.Parent = quickActions
        
        local actionCorner = Instance.new("UICorner")
        actionCorner.CornerRadius = UDim.new(0, 6)
        actionCorner.Parent = actionBtn
        
        actionBtn.MouseButton1Click:Connect(action.func)
    end
    
    -- ==================== UI CONTROLS ====================
    -- Close button functionality
    closeBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = not mainFrame.Visible
        closeBtn.Text = mainFrame.Visible and "−" or "+"
        State.UIVisible = mainFrame.Visible
    end)
    
    -- Right Shift toggle
    UserInputService.InputBegan:Connect(function(input)
        if input.KeyCode == Config.UI.ToggleKey then
            mainFrame.Visible = not mainFrame.Visible
            closeBtn.Text = mainFrame.Visible and "−" or "+"
            State.UIVisible = mainFrame.Visible
        end
    end)
    
    return screenGui
end

-- ==================== VÒNG LẶP CHÍNH ====================
local function MainLoop()
    local lastPrint = 0
    
    while task.wait(0.016) do -- 60 FPS
        -- Cập nhật vị trí POV circles
        UpdatePOVPosition()
        
        -- Thực hiện auto aim
        local isAiming = PerformAutoAim()
        
        -- Debug info mỗi 3 giây
        local currentTime = tick()
        if currentTime - lastPrint > 3 then
            if isAiming and State.Target then
                print("🎯 Đang aim: " .. State.Target.Player.Name .. 
                      " | Khoảng cách: " .. math.floor(State.Target.Distance) .. "m")
            end
            lastPrint = currentTime
        end
        
        -- Cập nhật ESP
        UpdateESP()
    end
end

-- ==================== KHỞI ĐỘNG HỆ THỐNG ====================
print([[
╔══════════════════════════════════════════╗
║       V/n ULTIMATE SUITE v6.0            ║
║       HOÀN CHỈNH TÍCH HỢP                ║
║                                          ║
║  ✅ Auto Aimbot 75% (Không cần phím)     ║
║  ✅ 2 Vòng POV (Đen → Xanh) 50-200       ║
║  ✅ Wall Check Thông Minh (Bỏ qua súng)  ║
║  ✅ ESP 4000m với Tên + Thanh Máu       ║
║  ✅ UI Mobile-Style Đẹp Mắt              ║
║                                          ║
║  Điều Khiển:                            ║
║  • Right Shift: Ẩn/Hiện UI              ║
║  • Aimbot TỰ ĐỘNG hoàn toàn             ║
║  • UI có nút bấm to như điện thoại      ║
╚══════════════════════════════════════════╝
]])

-- Khởi tạo
CreatePOVCircles()
CreateMobileUI()

-- Tạo ESP cho players hiện có
for _, player in pairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        CreateESP(player)
    end
end

-- Xử lý player mới
Players.PlayerAdded:Connect(function(player)
    task.wait(1)
    CreateESP(player)
end)

-- Xử lý player rời
Players.PlayerRemoving:Connect(function(player)
    if Drawings.ESP[player] then
        for _, drawing in pairs(Drawings.ESP[player]) do
            drawing:Remove()
        end
        Drawings.ESP[player] = nil
    end
end)

-- Bắt đầu vòng lặp
task.spawn(MainLoop)

-- Dọn dẹp
local function Cleanup()
    print("[V/n] Đang dọn dẹp...")
    
    if Drawings.POV.Outer then
        Drawings.POV.Outer:Remove()
        Drawings.POV.Inner:Remove()
    end
    
    for _, drawings in pairs(Drawings.ESP) do
        for _, drawing in pairs(drawings) do
            pcall(function() drawing:Remove() end)
        end
    end
    
    print("[V/n] Dọn dẹp hoàn tất!")
end

game:BindToClose(Cleanup)

print("✅ V/n đã sẵn sàng hoạt động!")
print("👉 Nhấn Right Shift để mở UI điều khiển")
print("🎯 Aimbot sẽ tự động hoạt động khi có mục tiêu trong tầm nhìn!")
