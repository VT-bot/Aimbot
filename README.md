-- Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local Player = Players.LocalPlayer

-- Configurações iniciais
local aimbotEnabled = false
local aimbotFOV = 100
local targetTeamCheck = true
local aimMode = "Todos" -- "Jogadores", "Mobs", "Todos"
local fovCircleColor = Color3.fromRGB(255, 0, 0)
local aimingSpeed = 0.2
local hitboxEnabled = false
local hitboxSize = 2

-- GUI
local gui = Instance.new("ScreenGui", Player:WaitForChild("PlayerGui"))
gui.Name = "AimbotMenu"
gui.ResetOnSpawn = false

local frame = Instance.new("Frame", gui)
frame.Size = UDim2.new(0, 280, 0, 370)
frame.Position = UDim2.new(0.03, 0, 0.12, 0)
frame.BackgroundColor3 = Color3.fromRGB(28, 28, 28)
frame.BorderSizePixel = 0
frame.Active = true
frame.Draggable = true
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)

local function createButton(text, y)
    local btn = Instance.new("TextButton", frame)
    btn.Size = UDim2.new(1, -20, 0, 32)
    btn.Position = UDim2.new(0, 10, 0, y)
    btn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Font = Enum.Font.Gotham
    btn.TextSize = 15
    btn.Text = text
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    return btn
end

local function createTextbox(y, placeholder, default)
    local box = Instance.new("TextBox", frame)
    box.Size = UDim2.new(1, -20, 0, 32)
    box.Position = UDim2.new(0, 10, 0, y)
    box.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    box.TextColor3 = Color3.new(1, 1, 1)
    box.Font = Enum.Font.Gotham
    box.PlaceholderText = placeholder
    box.Text = tostring(default)
    box.TextSize = 14
    Instance.new("UICorner", box).CornerRadius = UDim.new(0, 6)
    return box
end

local title = Instance.new("TextLabel", frame)
title.Text = "🎯 Aimbot Menu"
title.Size = UDim2.new(1, 0, 0, 30)
title.Position = UDim2.new(0, 0, 0, 5)
title.BackgroundTransparency = 1
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.Font = Enum.Font.GothamBold
title.TextSize = 20

local toggleBtn = createButton("Ativar Aimbot", 40)
local fovBox = createTextbox(80, "FOV (ex: 100)", aimbotFOV)
local teamBtn = createButton("Time Check: ON", 120)
local colorBtn = createButton("Cor: Vermelho", 160)
local modeBtn = createButton("Modo Mira: Todos", 200)
local hitboxBtn = createButton("Hitbox: OFF", 240)
local hitboxBox = createTextbox(280, "Tamanho Hitbox (Ex: 2.5)", hitboxSize)

-- Desenhos
local fovCircle = Drawing.new("Circle")
fovCircle.Visible = false
fovCircle.Thickness = 1.5
fovCircle.NumSides = 64
fovCircle.Filled = false
fovCircle.Color = fovCircleColor

local aimLine = Drawing.new("Line")
aimLine.Visible = false

-- Função para pegar os alvos
local function getTargets()
    local targets = {}

    if aimMode == "Jogadores" or aimMode == "Todos" then
        for _, plr in pairs(Players:GetPlayers()) do
            if plr ~= Player and plr.Character and plr.Character:FindFirstChild("Head") then
                local humanoid = plr.Character:FindFirstChild("Humanoid")
                if humanoid and humanoid.Health > 0 then
                    if not targetTeamCheck or plr.Team ~= Player.Team then
                        table.insert(targets, plr.Character)
                    end
                end
            end
        end
    end

    if aimMode == "Mobs" or aimMode == "Todos" then
        if workspace:FindFirstChild("Mobs") then
            for _, mob in pairs(workspace.Mobs:GetChildren()) do
                if mob:IsA("Model") and mob:FindFirstChild("Head") and mob:FindFirstChild("Humanoid") then
                    if mob.Humanoid.Health > 0 then
                        table.insert(targets, mob)
                    end
                end
            end
        end
    end

    return targets
end

-- Função para achar o alvo mais próximo no centro da tela e dentro do FOV
local function getClosestTarget()
    local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    local closest = nil
    local shortestDist = math.huge

    for _, char in pairs(getTargets()) do
        local head = char:FindFirstChild("Head")
        if head then
            local pos, onScreen = Camera:WorldToViewportPoint(head.Position)
            if onScreen then
                local dist = (center - Vector2.new(pos.X, pos.Y)).Magnitude
                if dist < aimbotFOV and dist < shortestDist then
                    shortestDist = dist
                    closest = head
                end
            end
        end
    end

    return closest
end

-- Atualiza a mira e os desenhos
RunService.RenderStepped:Connect(function()
    local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    fovCircle.Position = center
    fovCircle.Radius = aimbotFOV
    fovCircle.Color = fovCircleColor
    fovCircle.Visible = aimbotEnabled

    local target = getClosestTarget()

    if aimbotEnabled and target then
        local pos, onScreen = Camera:WorldToViewportPoint(target.Position)
        if onScreen then
            local camPos = Camera.CFrame.Position
            local desiredDir = (target.Position - camPos).Unit
            local newDir = Camera.CFrame.LookVector:Lerp(desiredDir, aimingSpeed)
            Camera.CFrame = CFrame.new(camPos, camPos + newDir)

            aimLine.From = center
            aimLine.To = Vector2.new(pos.X, pos.Y)
            aimLine.Color = fovCircleColor
            aimLine.Visible = true
        else
            aimLine.Visible = false
        end
    else
        aimLine.Visible = false
    end

    -- Ajuste de hitbox
    for _, char in pairs(getTargets()) do
        local head = char:FindFirstChild("Head")
        if head then
            if hitboxEnabled then
                if not head:FindFirstChild("OriginalSize") then
                    local orig = Instance.new("Vector3Value")
                    orig.Name = "OriginalSize"
                    orig.Value = head.Size
                    orig.Parent = head
                end
                head.Size = Vector3.new(hitboxSize, hitboxSize, hitboxSize)
                head.CanCollide = false
            else
                if head:FindFirstChild("OriginalSize") then
                    head.Size = head.OriginalSize.Value
                    head.OriginalSize:Destroy()
                end
            end
        end
    end
end)

-- Botões do menu

toggleBtn.MouseButton1Click:Connect(function()
    aimbotEnabled = not aimbotEnabled
    toggleBtn.Text = aimbotEnabled and "Desativar Aimbot" or "Ativar Aimbot"
end)

fovBox.FocusLost:Connect(function(enter)
    if enter then
        local val = tonumber(fovBox.Text)
        if val and val >= 10 and val <= 500 then
            aimbotFOV = val
        else
            fovBox.Text = tostring(aimbotFOV)
        end
    end
end)

teamBtn.MouseButton1Click:Connect(function()
    targetTeamCheck = not targetTeamCheck
    teamBtn.Text = "Time Check: " .. (targetTeamCheck and "ON" or "OFF")
end)

local colors = {
    {Color3.fromRGB(255, 0, 0), "Vermelho"},
    {Color3.fromRGB(0, 255, 0), "Verde"},
    {Color3.fromRGB(0, 0, 255), "Azul"},
    {Color3.fromRGB(255, 255, 0), "Amarelo"},
}

local colorIndex = 1
colorBtn.MouseButton1Click:Connect(function()
    colorIndex = (colorIndex % #colors) + 1
    fovCircleColor = colors[colorIndex][1]
    colorBtn.Text = "Cor: " .. colors[colorIndex][2]
end)

modeBtn.MouseButton1Click:Connect(function()
    if aimMode == "Jogadores" then
        aimMode = "Mobs"
    elseif aimMode == "Mobs" then
        aimMode = "Todos"
    else
        aimMode = "Jogadores"
    end
    modeBtn.Text = "Modo Mira: " .. aimMode
end)

hitboxBtn.MouseButton1Click:Connect(function()
    hitboxEnabled = not hitboxEnabled
    hitboxBtn.Text = hitboxEnabled and "Hitbox: ON" or "Hitbox: OFF"
end)

hitboxBox.FocusLost:Connect(function(enter)
    if enter then
        local val = tonumber(hitboxBox.Text)
        if val and val >= 1 and val <= 10 then
            hitboxSize = val
        else
            hitboxBox.Text = tostring(hitboxSize)
        end
    end
end)

print("✅ Aimbot script carregado com sucesso!")
