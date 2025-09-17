-- Painel de Habilidades para Roblox - VERSÃO COMPLETA COM COOLDOWN
-- Coloque este script em StarterPlayer > StarterPlayerScripts
-- Pressione G para abrir/fechar o painel

print("🔄 Iniciando Painel de Habilidades Avançado...")

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local mouse = player:GetMouse()

-- Aguardar o PlayerGui estar disponível
local playerGui
repeat
    wait()
    playerGui = player:FindFirstChild("PlayerGui")
until playerGui

print("✅ PlayerGui encontrado!")

-- Variáveis de controle
local isGuiOpen = false
local teleportEnabled = false
local cooldownModifierEnabled = false
local speedEnabled = false
local noclipEnabled = false
local infiniteJumpEnabled = false
local espEnabled = false
local aimAssistEnabled = false
local flyEnabled = false
local teleportKey = "F"
local currentSpeed = 50
local originalWalkSpeed = 16
local originalJumpPower = 50
local gui = nil
local mainFrame = nil

-- Variáveis para funcionalidades avançadas
local noclipConnection = nil
local flyConnection = nil
local espBoxes = {}
local flySpeed = 50
local cooldownMultiplier = 0.5
local originalWait = wait
local hookActive = false

-- Sistema de hook do wait() melhorado
local function setupCooldownHook()
    if hookActive then return end
    hookActive = true
    
    -- Hook global wait function
    local env = getfenv()
    if env then
        env.wait = function(duration)
            if duration and type(duration) == "number" and duration > 0 then
                local modifiedDuration = duration * cooldownMultiplier
                print("⏱️ Wait interceptado: " .. duration .. "s -> " .. modifiedDuration .. "s")
                return originalWait(modifiedDuration)
            else
                return originalWait(duration)
            end
        end
        print("✅ Hook do wait() ativo!")
    end
    
    -- Hook task.wait também
    local originalTaskWait = task.wait
    task.wait = function(duration)
        if duration and type(duration) == "number" and duration > 0 then
            local modifiedDuration = duration * cooldownMultiplier
            print("⏱️ Task.wait interceptado: " .. duration .. "s -> " .. modifiedDuration .. "s")
            return originalTaskWait(modifiedDuration)
        else
            return originalTaskWait(duration)
        end
    end
    
    print("✅ Sistema de cooldown configurado!")
end

local function removeCooldownHook()
    if not hookActive then return end
    hookActive = false
    
    -- Restaurar wait original
    local env = getfenv()
    if env then
        env.wait = originalWait
    end
    
    -- Restaurar task.wait original
    task.wait = function(duration)
        return originalWait(duration)
    end
    
    print("❌ Hook do cooldown removido!")
end

-- Função para criar efeito visual de teleporte
local function createTeleportEffect(position)
    local character = player.Character
    if not character then return end
    
    local effect = Instance.new("Part")
    effect.Name = "TeleportEffect"
    effect.Shape = Enum.PartType.Ball
    effect.Material = Enum.Material.Neon
    effect.BrickColor = BrickColor.new("Bright blue")
    effect.Size = Vector3.new(0.1, 0.1, 0.1)
    effect.Position = position
    effect.Anchored = true
    effect.CanCollide = false
    effect.Parent = workspace
    
    local tweenInfo = TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    local tween = TweenService:Create(effect, tweenInfo, {
        Size = Vector3.new(10, 10, 10),
        Transparency = 1
    })
    
    tween:Play()
    tween.Completed:Connect(function()
        effect:Destroy()
    end)
end

-- Função de teleporte
local function teleportToMousePosition()
    if not teleportEnabled then return end
    
    local character = player.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then 
        print("❌ Character ou RootPart não encontrado para teleporte")
        return 
    end
    
    local rootPart = character.HumanoidRootPart
    local hit = mouse.Hit
    
    if hit then
        local targetPosition = hit.Position + Vector3.new(0, 5, 0)
        rootPart.CFrame = CFrame.new(targetPosition)
        createTeleportEffect(targetPosition)
        print("✅ Teleportado para: " .. tostring(targetPosition))
    end
end

-- Sistema de velocidade
local function updateSpeed()
    local character = player.Character
    if not character then return end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    
    if speedEnabled then
        humanoid.WalkSpeed = currentSpeed
        print("💨 Velocidade alterada para: " .. currentSpeed)
    else
        humanoid.WalkSpeed = originalWalkSpeed
        print("🚶 Velocidade restaurada para: " .. originalWalkSpeed)
    end
end

-- Sistema de Noclip
local function toggleNoclip()
    if noclipEnabled then
        noclipConnection = RunService.Stepped:Connect(function()
            local character = player.Character
            if character then
                for _, part in pairs(character:GetDescendants()) do
                    if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                        part.CanCollide = false
                    end
                end
            end
        end)
        print("👻 Noclip ATIVADO")
    else
        if noclipConnection then
            noclipConnection:Disconnect()
            noclipConnection = nil
        end
        
        local character = player.Character
        if character then
            for _, part in pairs(character:GetDescendants()) do
                if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                    part.CanCollide = true
                end
            end
        end
        print("🚶 Noclip DESATIVADO")
    end
end

-- Sistema de ESP
local function createESP(targetPlayer)
    if not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
        return
    end
    
    local character = targetPlayer.Character
    local rootPart = character.HumanoidRootPart
    
    local espBox = Instance.new("BoxHandleAdornment")
    espBox.Name = "ESPBox_" .. targetPlayer.Name
    espBox.Adornee = rootPart
    espBox.Size = rootPart.Size + Vector3.new(1, 3, 1)
    espBox.Color3 = targetPlayer.TeamColor and targetPlayer.TeamColor.Color or Color3.new(1, 0, 0)
    espBox.Transparency = 0.7
    espBox.AlwaysOnTop = true
    espBox.ZIndex = 10
    espBox.Parent = rootPart
    
    local espLabel = Instance.new("BillboardGui")
    espLabel.Name = "ESPLabel_" .. targetPlayer.Name
    espLabel.Adornee = rootPart
    espLabel.Size = UDim2.new(0, 100, 0, 50)
    espLabel.StudsOffset = Vector3.new(0, 3, 0)
    espLabel.AlwaysOnTop = true
    espLabel.Parent = rootPart
    
    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = targetPlayer.Name
    textLabel.TextColor3 = Color3.new(1, 1, 1)
    textLabel.TextStrokeTransparency = 0
    textLabel.TextScaled = true
    textLabel.Font = Enum.Font.GothamBold
    textLabel.Parent = espLabel
    
    espBoxes[targetPlayer] = {espBox, espLabel}
end

local function toggleESP()
    if espEnabled then
        for _, otherPlayer in pairs(Players:GetPlayers()) do
            if otherPlayer ~= player then
                createESP(otherPlayer)
            end
        end
        
        Players.PlayerAdded:Connect(function(newPlayer)
            if espEnabled then
                newPlayer.CharacterAdded:Connect(function()
                    wait(1)
                    createESP(newPlayer)
                end)
            end
        end)
        
        print("👁️ ESP ATIVADO")
    else
        for targetPlayer, espElements in pairs(espBoxes) do
            for _, element in pairs(espElements) do
                if element and element.Parent then
                    element:Destroy()
                end
            end
        end
        espBoxes = {}
        print("👁️ ESP DESATIVADO")
    end
end

-- Sistema de Fly
local function toggleFly()
    if flyEnabled then
        local character = player.Character
        if not character or not character:FindFirstChild("HumanoidRootPart") then return end
        
        local rootPart = character.HumanoidRootPart
        local bodyVelocity = Instance.new("BodyVelocity")
        bodyVelocity.MaxForce = Vector3.new(4000, 4000, 4000)
        bodyVelocity.Velocity = Vector3.new(0, 0, 0)
        bodyVelocity.Parent = rootPart
        
        local bodyAngularVelocity = Instance.new("BodyAngularVelocity")
        bodyAngularVelocity.MaxTorque = Vector3.new(4000, 4000, 4000)
        bodyAngularVelocity.AngularVelocity = Vector3.new(0, 0, 0)
        bodyAngularVelocity.Parent = rootPart
        
        flyConnection = RunService.Heartbeat:Connect(function()
            local camera = workspace.CurrentCamera
            local moveVector = Vector3.new(0, 0, 0)
            
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then
                moveVector = moveVector + camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then
                moveVector = moveVector - camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then
                moveVector = moveVector - camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then
                moveVector = moveVector + camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
                moveVector = moveVector + Vector3.new(0, 1, 0)
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
                moveVector = moveVector - Vector3.new(0, 1, 0)
            end
            
            bodyVelocity.Velocity = moveVector * flySpeed
        end)
        
        print("✈️ Fly ATIVADO (WASD + Space/Shift)")
    else
        if flyConnection then
            flyConnection:Disconnect()
            flyConnection = nil
        end
        
        local character = player.Character
        if character and character:FindFirstChild("HumanoidRootPart") then
            local rootPart = character.HumanoidRootPart
            for _, obj in pairs(rootPart:GetChildren()) do
                if obj:IsA("BodyVelocity") or obj:IsA("BodyAngularVelocity") then
                    obj:Destroy()
                end
            end
        end
        print("🚶 Fly DESATIVADO")
    end
end

-- Sistema de Aim Assist
local function getClosestPlayer()
    local closestPlayer = nil
    local shortestDistance = math.huge
    
    for _, otherPlayer in pairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character and otherPlayer.Character:FindFirstChild("Head") then
            local head = otherPlayer.Character.Head
            local screenPos, onScreen = workspace.CurrentCamera:WorldToScreenPoint(head.Position)
            
            if onScreen then
                local mousePos = Vector2.new(mouse.X, mouse.Y)
                local screenPoint = Vector2.new(screenPos.X, screenPos.Y)
                local distance = (mousePos - screenPoint).Magnitude
                
                if distance < shortestDistance then
                    closestPlayer = otherPlayer
                    shortestDistance = distance
                end
            end
        end
    end
    
    return closestPlayer
end

local aimConnection = nil
local function toggleAimAssist()
    if aimAssistEnabled then
        aimConnection = RunService.Heartbeat:Connect(function()
            local closestPlayer = getClosestPlayer()
            if closestPlayer and closestPlayer.Character and closestPlayer.Character:FindFirstChild("Head") then
                local head = closestPlayer.Character.Head
                local camera = workspace.CurrentCamera
                local targetCFrame = CFrame.lookAt(camera.CFrame.Position, head.Position)
                camera.CFrame = camera.CFrame:Lerp(targetCFrame, 0.1)
            end
        end)
        print("🎯 Aim Assist ATIVADO")
    else
        if aimConnection then
            aimConnection:Disconnect()
            aimConnection = nil
        end
        print("🎯 Aim Assist DESATIVADO")
    end
end

-- Funções para atualizar status visual
local function updateTeleportStatus(statusLabel, button)
    if teleportEnabled then
        statusLabel.Text = "STATUS: ATIVO"
        statusLabel.TextColor3 = Color3.new(0, 1, 0)
        button.BackgroundColor3 = Color3.new(0, 0.7, 0)
        button.Text = "DESATIVAR"
    else
        statusLabel.Text = "STATUS: DESATIVADO"
        statusLabel.TextColor3 = Color3.new(1, 0, 0)
        button.BackgroundColor3 = Color3.new(0.7, 0, 0)
        button.Text = "ATIVAR"
    end
end

local function updateCooldownStatus(statusLabel, button)
    if cooldownModifierEnabled then
        local percent = math.floor(cooldownMultiplier * 100)
        statusLabel.Text = "STATUS: ATIVO (" .. percent .. "%)"
        statusLabel.TextColor3 = Color3.new(0, 1, 0)
        button.BackgroundColor3 = Color3.new(0, 0.7, 0)
        button.Text = "DESATIVAR"
    else
        statusLabel.Text = "STATUS: DESATIVADO"
        statusLabel.TextColor3 = Color3.new(1, 0, 0)
        button.BackgroundColor3 = Color3.new(0.7, 0, 0)
        button.Text = "ATIVAR"
    end
end

local function updateSpeedStatus(statusLabel, button)
    if speedEnabled then
        statusLabel.Text = "STATUS: ATIVO (" .. currentSpeed .. ")"
        statusLabel.TextColor3 = Color3.new(0, 1, 0)
        button.BackgroundColor3 = Color3.new(0, 0.7, 0)
        button.Text = "DESATIVAR"
    else
        statusLabel.Text = "STATUS: DESATIVADO"
        statusLabel.TextColor3 = Color3.new(1, 0, 0)
        button.BackgroundColor3 = Color3.new(0.7, 0, 0)
        button.Text = "ATIVAR"
    end
end

local function updateNoclipStatus(statusLabel, button)
    if noclipEnabled then
        statusLabel.Text = "STATUS: ATIVO"
        statusLabel.TextColor3 = Color3.new(0, 1, 0)
        button.BackgroundColor3 = Color3.new(0, 0.7, 0)
        button.Text = "DESATIVAR"
    else
        statusLabel.Text = "STATUS: DESATIVADO"
        statusLabel.TextColor3 = Color3.new(1, 0, 0)
        button.BackgroundColor3 = Color3.new(0.7, 0, 0)
        button.Text = "ATIVAR"
    end
end

local function updateJumpStatus(statusLabel, button)
    if infiniteJumpEnabled then
        statusLabel.Text = "STATUS: ATIVO"
        statusLabel.TextColor3 = Color3.new(0, 1, 0)
        button.BackgroundColor3 = Color3.new(0, 0.7, 0)
        button.Text = "DESATIVAR"
    else
        statusLabel.Text = "STATUS: DESATIVADO"
        statusLabel.TextColor3 = Color3.new(1, 0, 0)
        button.BackgroundColor3 = Color3.new(0.7, 0, 0)
        button.Text = "ATIVAR"
    end
end

local function updateESPStatus(statusLabel, button)
    if espEnabled then
        statusLabel.Text = "STATUS: ATIVO"
        statusLabel.TextColor3 = Color3.new(0, 1, 0)
        button.BackgroundColor3 = Color3.new(0, 0.7, 0)
        button.Text = "DESATIVAR"
    else
        statusLabel.Text = "STATUS: DESATIVADO"
        statusLabel.TextColor3 = Color3.new(1, 0, 0)
        button.BackgroundColor3 = Color3.new(0.7, 0, 0)
        button.Text = "ATIVAR"
    end
end

local function updateAimStatus(statusLabel, button)
    if aimAssistEnabled then
        statusLabel.Text = "STATUS: ATIVO"
        statusLabel.TextColor3 = Color3.new(0, 1, 0)
        button.BackgroundColor3 = Color3.new(0, 0.7, 0)
        button.Text = "DESATIVAR"
    else
        statusLabel.Text = "STATUS: DESATIVADO"
        statusLabel.TextColor3 = Color3.new(1, 0, 0)
        button.BackgroundColor3 = Color3.new(0.7, 0, 0)
        button.Text = "ATIVAR"
    end
end

local function updateFlyStatus(statusLabel, button)
    if flyEnabled then
        statusLabel.Text = "STATUS: ATIVO (" .. flySpeed .. ")"
        statusLabel.TextColor3 = Color3.new(0, 1, 0)
        button.BackgroundColor3 = Color3.new(0, 0.7, 0)
        button.Text = "DESATIVAR"
    else
        statusLabel.Text = "STATUS: DESATIVADO"
        statusLabel.TextColor3 = Color3.new(1, 0, 0)
        button.BackgroundColor3 = Color3.new(0.7, 0, 0)
        button.Text = "ATIVAR"
    end
end

-- Função para detectar tecla personalizada
local function setupCustomKey(keyLabel)
    keyLabel.Text = "Pressione uma tecla..."
    keyLabel.TextColor3 = Color3.new(1, 1, 0)
    
    local connection
    connection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.UserInputType == Enum.UserInputType.Keyboard then
            teleportKey = input.KeyCode.Name
            keyLabel.Text = "TECLA: " .. teleportKey
            keyLabel.TextColor3 = Color3.new(1, 1, 1)
            connection:Disconnect()
        elseif input.UserInputType == Enum.UserInputType.MouseButton1 then
            teleportKey = "MouseButton1"
            keyLabel.Text = "TECLA: BOTÃO ESQUERDO"
            keyLabel.TextColor3 = Color3.new(1, 1, 1)
            connection:Disconnect()
        elseif input.UserInputType == Enum.UserInputType.MouseButton2 then
            teleportKey = "MouseButton2"
            keyLabel.Text = "TECLA: BOTÃO DIREITO"
            keyLabel.TextColor3 = Color3.new(1, 1, 1)
            connection:Disconnect()
        end
    end)
end

-- Função para criar a GUI
local function createGui()
    print("🎨 Criando interface expandida...")
    
    if gui then
        gui:Destroy()
    end
    
    -- ScreenGui principal
    gui = Instance.new("ScreenGui")
    gui.Name = "SkillsPanel"
    gui.Parent = playerGui
    gui.ResetOnSpawn = false
    
    -- Frame principal
    mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 420, 0, 700)
    mainFrame.Position = UDim2.new(0.5, -210, 0.5, -350)
    mainFrame.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
    mainFrame.BorderSizePixel = 0
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.Parent = gui
    mainFrame.Visible = true
    
    -- ScrollingFrame
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1, 0, 1, -50)
    scrollFrame.Position = UDim2.new(0, 0, 0, 45)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.ScrollBarThickness = 8
    scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 1200)
    scrollFrame.Parent = mainFrame
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 15)
    corner.Parent = mainFrame
    
    -- Título
    local title = Instance.new("TextLabel")
    title.Name = "Title"
    title.Size = UDim2.new(1, 0, 0, 45)
    title.Position = UDim2.new(0, 0, 0, 0)
    title.BackgroundColor3 = Color3.new(0.2, 0.2, 0.2)
    title.BorderSizePixel = 0
    title.Text = "PAINEL DE HABILIDADES PRO"
    title.TextColor3 = Color3.new(1, 1, 1)
    title.TextScaled = true
    title.Font = Enum.Font.GothamBold
    title.Parent = mainFrame
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 15)
    titleCorner.Parent = title
    
    -- Função para criar seções
    local function createSection(name, emoji, yPos, height)
        local section = Instance.new("Frame")
        section.Name = name .. "Section"
        section.Size = UDim2.new(1, -20, 0, height)
        section.Position = UDim2.new(0, 10, 0, yPos)
        section.BackgroundColor3 = Color3.new(0.15, 0.15, 0.15)
        section.BorderSizePixel = 0
        section.Parent = scrollFrame
        
        local sectionCorner = Instance.new("UICorner")
        sectionCorner.CornerRadius = UDim.new(0, 10)
        sectionCorner.Parent = section
        
        local sectionTitle = Instance.new("TextLabel")
        sectionTitle.Size = UDim2.new(1, 0, 0, 25)
        sectionTitle.Position = UDim2.new(0, 0, 0, 5)
        sectionTitle.BackgroundTransparency = 1
        sectionTitle.Text = emoji .. " " .. name:upper()
        sectionTitle.TextColor3 = Color3.new(0.8, 0.8, 1)
        sectionTitle.TextScaled = true
        sectionTitle.Font = Enum.Font.Gotham
        sectionTitle.Parent = section
        
        return section
    end
    
    -- Criar seções
    local teleportSection = createSection("Teleporte", "🚀", 5, 160)
    local speedSection = createSection("Velocidade", "💨", 175, 140)
    local cooldownSection = createSection("Modificador de Cooldown", "⏱️", 325, 140)
    local noclipSection = createSection("Noclip", "👻", 475, 100)
    local jumpSection = createSection("Pulo Infinito", "🦘", 585, 100)
    local espSection = createSection("ESP", "👁️", 695, 100)
    local aimSection = createSection("Aim Assist", "🎯", 805, 100)
    local flySection = createSection("Fly", "✈️", 915, 140)
    
    -- SEÇÃO TELEPORTE
    local teleportStatus = Instance.new("TextLabel")
    teleportStatus.Size = UDim2.new(1, -10, 0, 20)
    teleportStatus.Position = UDim2.new(0, 5, 0, 30)
    teleportStatus.BackgroundTransparency = 1
    teleportStatus.Text = "STATUS: DESATIVADO"
    teleportStatus.TextColor3 = Color3.new(1, 0, 0)
    teleportStatus.TextScaled = true
    teleportStatus.Font = Enum.Font.Gotham
    teleportStatus.Parent = teleportSection
    
    local teleportToggle = Instance.new("TextButton")
    teleportToggle.Size = UDim2.new(0.45, 0, 0, 35)
    teleportToggle.Position = UDim2.new(0.05, 0, 0, 55)
    teleportToggle.BackgroundColor3 = Color3.new(0.7, 0, 0)
    teleportToggle.BorderSizePixel = 0
    teleportToggle.Text = "ATIVAR"
    teleportToggle.TextColor3 = Color3.new(1, 1, 1)
    teleportToggle.TextScaled = true
    teleportToggle.Font = Enum.Font.GothamBold
    teleportToggle.Parent = teleportSection
    
    local teleportCorner = Instance.new("UICorner")
    teleportCorner.CornerRadius = UDim.new(0, 8)
    teleportCorner.Parent = teleportToggle
    
    local teleportNow = Instance.new("TextButton")
    teleportNow.Size = UDim2.new(0.45, 0, 0, 35)
    teleportNow.Position = UDim2.new(0.5, 0, 0, 55)
    teleportNow.BackgroundColor3 = Color3.new(0, 0.5, 0.8)
    teleportNow.BorderSizePixel = 0
    teleportNow.Text = "TELEPORTAR AGORA"
    teleportNow.TextColor3 = Color3.new(1, 1, 1)
    teleportNow.TextScaled = true
    teleportNow.Font = Enum.Font.GothamBold
    teleportNow.Parent = teleportSection
    
    local nowCorner = Instance.new("UICorner")
    nowCorner.CornerRadius = UDim.new(0, 8)
    nowCorner.Parent = teleportNow
    
    local keyLabel = Instance.new("TextLabel")
    keyLabel.Size = UDim2.new(1, -10, 0, 20)
    keyLabel.Position = UDim2.new(0, 5, 0, 95)
    keyLabel.BackgroundTransparency = 1
    keyLabel.Text = "TECLA: " .. teleportKey
    keyLabel.TextColor3 = Color3.new(1, 1, 1)
    keyLabel.TextScaled = true
    keyLabel.Font = Enum.Font.Gotham
    keyLabel.Parent = teleportSection
    
    local chooseKey = Instance.new("TextButton")
    chooseKey.Size = UDim2.new(1, -10, 0, 30)
    chooseKey.Position = UDim2.new(0, 5, 0, 120)
    chooseKey.BackgroundColor3 = Color3.new(0.3, 0.3, 0.3)
    chooseKey.BorderSizePixel = 0
    chooseKey.Text = "ESCOLHER NOVA TECLA"
    chooseKey.TextColor3 = Color3.new(1, 1, 1)
    chooseKey.TextScaled = true
    chooseKey.Font = Enum.Font.Gotham
    chooseKey.Parent = teleportSection
    
    local chooseCorner = Instance.new("UICorner")
    chooseCorner.CornerRadius = UDim.new(0, 8)
    chooseCorner.Parent = chooseKey
    
    -- SEÇÃO VELOCIDADE
    local speedStatus = Instance.new("TextLabel")
    speedStatus.Size = UDim2.new(1, -10, 0, 20)
    speedStatus.Position = UDim2.new(0, 5, 0, 30)
    speedStatus.BackgroundTransparency = 1
    speedStatus.Text = "STATUS: DESATIVADO"
    speedStatus.TextColor3 = Color3.new(1, 0, 0)
    speedStatus.TextScaled = true
    speedStatus.Font = Enum.Font.Gotham
    speedStatus.Parent = speedSection
    
    local speedToggle = Instance.new("TextButton")
    speedToggle.Size = UDim2.new(0.45, 0, 0, 35)
    speedToggle.Position = UDim2.new(0.05, 0, 0, 55)
    speedToggle.BackgroundColor3 = Color3.new(0.7, 0, 0)
    speedToggle.BorderSizePixel = 0
    speedToggle.Text = "ATIVAR"
    speedToggle.TextColor3 = Color3.new(1, 1, 1)
    speedToggle.TextScaled = true
    speedToggle.Font = Enum.Font.GothamBold
    speedToggle.Parent = speedSection
    
    local speedCorner = Instance.new("UICorner")
    speedCorner.CornerRadius = UDim.new(0, 8)
    speedCorner.Parent = speedToggle
    
    local speedBox = Instance.new("TextBox")
    speedBox.Size = UDim2.new(0.45, 0, 0, 35)
    speedBox.Position = UDim2.new(0.5, 0, 0, 55)
    speedBox.BackgroundColor3 = Color3.new(0.2, 0.2, 0.2)
    speedBox.BorderSizePixel = 0
    speedBox.Text = "50"
    speedBox.PlaceholderText = "Velocidade"
    speedBox.TextColor3 = Color3.new(1, 1, 1)
    speedBox.TextScaled = true
    speedBox.Font = Enum.Font.Gotham
    speedBox.Parent = speedSection
    
    local speedBoxCorner = Instance.new("UICorner")
    speedBoxCorner.CornerRadius = UDim.new(0, 8)
    speedBoxCorner.Parent = speedBox
    
    local speedInfo = Instance.new("TextLabel")
    speedInfo.Size = UDim2.new(1, -10, 0, 35)
    speedInfo.Position = UDim2.new(0, 5, 0, 95)
    speedInfo.BackgroundTransparency = 1
    speedInfo.Text = "Normal: 16 | Recomendado: 30-80"
    speedInfo.TextColor3 = Color3.new(0.8, 0.8, 0.8)
    speedInfo.TextScaled = true
    speedInfo.Font = Enum.Font.Gotham
    speedInfo.Parent = speedSection
    
    -- SEÇÃO COOLDOWN
    local cooldownStatus = Instance.new("TextLabel")
    cooldownStatus.Size = UDim2.new(1, -10, 0, 20)
    cooldownStatus.Position = UDim2.new(0, 5, 0, 30)
    cooldownStatus.BackgroundTransparency = 1
    cooldownStatus.Text = "STATUS: DESATIVADO"
    cooldownStatus.TextColor3 = Color3.new(1, 0, 0)
    cooldownStatus.TextScaled = true
    cooldownStatus.Font = Enum.Font.Gotham
    cooldownStatus.Parent = cooldownSection
    
    local cooldownToggle = Instance.new("TextButton")
    cooldownToggle.Size = UDim2.new(0.45, 0, 0, 35)
    cooldownToggle.Position = UDim2.new(0.05, 0, 0, 55)
    cooldownToggle.BackgroundColor3 = Color3.new(0.7, 0, 0)
    cooldownToggle.BorderSizePixel = 0
    cooldownToggle.Text = "ATIVAR"
    cooldownToggle.TextColor3 = Color3.new(1, 1, 1)
    cooldownToggle.TextScaled = true
    cooldownToggle.Font = Enum.Font.GothamBold
    cooldownToggle.Parent = cooldownSection
    
    local cooldownCorner = Instance.new("UICorner")
    cooldownCorner.CornerRadius = UDim.new(0, 8)
    cooldownCorner.Parent = cooldownToggle
    
    local cooldownBox = Instance.new("TextBox")
    cooldownBox.Size = UDim2.new(0.45, 0, 0, 35)
    cooldownBox.Position = UDim2.new(0.5, 0, 0, 55)
    cooldownBox.BackgroundColor3 = Color3.new(0.2, 0.2, 0.2)
    cooldownBox.BorderSizePixel = 0
    cooldownBox.Text = "50"
    cooldownBox.PlaceholderText = "Porcentagem"
    cooldownBox.TextColor3 = Color3.new(1, 1, 1)
    cooldownBox.TextScaled = true
    cooldownBox.Font = Enum.Font.Gotham
    cooldownBox.Parent = cooldownSection
    
    local cooldownBoxCorner = Instance.new("UICorner")
    cooldownBoxCorner.CornerRadius = UDim.new(0, 8)
    cooldownBoxCorner.Parent = cooldownBox
    
    local cooldownInfo = Instance.new("TextLabel")
    cooldownInfo.Size = UDim2.new(1, -10, 0, 35)
    cooldownInfo.Position = UDim2.new(0, 5, 0, 95)
    cooldownInfo.BackgroundTransparency = 1
    cooldownInfo.Text = "50% = metade do tempo | 10% = muito rápido"
    cooldownInfo.TextColor3 = Color3.new(0.8, 0.8, 0.8)
    cooldownInfo.TextScaled = true
    cooldownInfo.Font = Enum.Font.Gotham
    cooldownInfo.Parent = cooldownSection
    
    -- SEÇÃO NOCLIP
    local noclipStatus = Instance.new("TextLabel")
    noclipStatus.Size = UDim2.new(1, -10, 0, 20)
    noclipStatus.Position = UDim2.new(0, 5, 0, 30)
    noclipStatus.BackgroundTransparency = 1
    noclipStatus.Text = "STATUS: DESATIVADO"
    noclipStatus.TextColor3 = Color3.new(1, 0, 0)
    noclipStatus.TextScaled = true
    noclipStatus.Font = Enum.Font.Gotham
    noclipStatus.Parent = noclipSection
    
    local noclipToggle = Instance.new("TextButton")
    noclipToggle.Size = UDim2.new(0.8, 0, 0, 40)
    noclipToggle.Position = UDim2.new(0.1, 0, 0, 50)
    noclipToggle.BackgroundColor3 = Color3.new(0.7, 0, 0)
    noclipToggle.BorderSizePixel = 0
    noclipToggle.Text = "ATIVAR"
    noclipToggle.TextColor3 = Color3.new(1, 1, 1)
    noclipToggle.TextScaled = true
    noclipToggle.Font = Enum.Font.GothamBold
    noclipToggle.Parent = noclipSection
    
    local noclipCorner = Instance.new("UICorner")
    noclipCorner.CornerRadius = UDim.new(0, 8)
    noclipCorner.Parent = noclipToggle
    
    -- SEÇÃO PULO INFINITO
    local jumpStatus = Instance.new("TextLabel")
    jumpStatus.Size = UDim2.new(1, -10, 0, 20)
    jumpStatus.Position = UDim2.new(0, 5, 0, 30)
    jumpStatus.BackgroundTransparency = 1
    jumpStatus.Text = "STATUS: DESATIVADO"
    jumpStatus.TextColor3 = Color3.new(1, 0, 0)
    jumpStatus.TextScaled = true
    jumpStatus.Font = Enum.Font.Gotham
    jumpStatus.Parent = jumpSection
    
    local jumpToggle = Instance.new("TextButton")
    jumpToggle.Size = UDim2.new(0.8, 0, 0, 40)
    jumpToggle.Position = UDim2.new(0.1, 0, 0, 50)
    jumpToggle.BackgroundColor3 = Color3.new(0.7, 0, 0)
    jumpToggle.BorderSizePixel = 0
    jumpToggle.Text = "ATIVAR"
    jumpToggle.TextColor3 = Color3.new(1, 1, 1)
    jumpToggle.TextScaled = true
    jumpToggle.Font = Enum.Font.GothamBold
    jumpToggle.Parent = jumpSection
    
    local jumpCorner = Instance.new("UICorner")
    jumpCorner.CornerRadius = UDim.new(0, 8)
    jumpCorner.Parent = jumpToggle
    
    -- SEÇÃO ESP
    local espStatus = Instance.new("TextLabel")
    espStatus.Size = UDim2.new(1, -10, 0, 20)
    espStatus.Position = UDim2.new(0, 5, 0, 30)
    espStatus.BackgroundTransparency = 1
    espStatus.Text = "STATUS: DESATIVADO"
    espStatus.TextColor3 = Color3.new(1, 0, 0)
    espStatus.TextScaled = true
    espStatus.Font = Enum.Font.Gotham
    espStatus.Parent = espSection
    
    local espToggle = Instance.new("TextButton")
    espToggle.Size = UDim2.new(0.8, 0, 0, 40)
    espToggle.Position = UDim2.new(0.1, 0, 0, 50)
    espToggle.BackgroundColor3 = Color3.new(0.7, 0, 0)
    espToggle.BorderSizePixel = 0
    espToggle.Text = "ATIVAR"
    espToggle.TextColor3 = Color3.new(1, 1, 1)
    espToggle.TextScaled = true
    espToggle.Font = Enum.Font.GothamBold
    espToggle.Parent = espSection
    
    local espCorner = Instance.new("UICorner")
    espCorner.CornerRadius = UDim.new(0, 8)
    espCorner.Parent = espToggle
    
    -- SEÇÃO AIM ASSIST
    local aimStatus = Instance.new("TextLabel")
    aimStatus.Size = UDim2.new(1, -10, 0, 20)
    aimStatus.Position = UDim2.new(0, 5, 0, 30)
    aimStatus.BackgroundTransparency = 1
    aimStatus.Text = "STATUS: DESATIVADO"
    aimStatus.TextColor3 = Color3.new(1, 0, 0)
    aimStatus.TextScaled = true
    aimStatus.Font = Enum.Font.Gotham
    aimStatus.Parent = aimSection
    
    local aimToggle = Instance.new("TextButton")
    aimToggle.Size = UDim2.new(0.8, 0, 0, 40)
    aimToggle.Position = UDim2.new(0.1, 0, 0, 50)
    aimToggle.BackgroundColor3 = Color3.new(0.7, 0, 0)
    aimToggle.BorderSizePixel = 0
    aimToggle.Text = "ATIVAR"
    aimToggle.TextColor3 = Color3.new(1, 1, 1)
    aimToggle.TextScaled = true
    aimToggle.Font = Enum.Font.GothamBold
    aimToggle.Parent = aimSection
    
    local aimCorner = Instance.new("UICorner")
    aimCorner.CornerRadius = UDim.new(0, 8)
    aimCorner.Parent = aimToggle
    
    -- SEÇÃO FLY
    local flyStatus = Instance.new("TextLabel")
    flyStatus.Size = UDim2.new(1, -10, 0, 20)
    flyStatus.Position = UDim2.new(0, 5, 0, 30)
    flyStatus.BackgroundTransparency = 1
    flyStatus.Text = "STATUS: DESATIVADO"
    flyStatus.TextColor3 = Color3.new(1, 0, 0)
    flyStatus.TextScaled = true
    flyStatus.Font = Enum.Font.Gotham
    flyStatus.Parent = flySection
    
    local flyToggle = Instance.new("TextButton")
    flyToggle.Size = UDim2.new(0.45, 0, 0, 35)
    flyToggle.Position = UDim2.new(0.05, 0, 0, 55)
    flyToggle.BackgroundColor3 = Color3.new(0.7, 0, 0)
    flyToggle.BorderSizePixel = 0
    flyToggle.Text = "ATIVAR"
    flyToggle.TextColor3 = Color3.new(1, 1, 1)
    flyToggle.TextScaled = true
    flyToggle.Font = Enum.Font.GothamBold
    flyToggle.Parent = flySection
    
    local flyCorner = Instance.new("UICorner")
    flyCorner.CornerRadius = UDim.new(0, 8)
    flyCorner.Parent = flyToggle
    
    local flyBox = Instance.new("TextBox")
    flyBox.Size = UDim2.new(0.45, 0, 0, 35)
    flyBox.Position = UDim2.new(0.5, 0, 0, 55)
    flyBox.BackgroundColor3 = Color3.new(0.2, 0.2, 0.2)
    flyBox.BorderSizePixel = 0
    flyBox.Text = "50"
    flyBox.PlaceholderText = "Velocidade"
    flyBox.TextColor3 = Color3.new(1, 1, 1)
    flyBox.TextScaled = true
    flyBox.Font = Enum.Font.Gotham
    flyBox.Parent = flySection
    
    local flyBoxCorner = Instance.new("UICorner")
    flyBoxCorner.CornerRadius = UDim.new(0, 8)
    flyBoxCorner.Parent = flyBox
    
    local flyInfo = Instance.new("TextLabel")
    flyInfo.Size = UDim2.new(1, -10, 0, 35)
    flyInfo.Position = UDim2.new(0, 5, 0, 95)
    flyInfo.BackgroundTransparency = 1
    flyInfo.Text = "Use WASD + Space/Shift para voar"
    flyInfo.TextColor3 = Color3.new(0.8, 0.8, 0.8)
    flyInfo.TextScaled = true
    flyInfo.Font = Enum.Font.Gotham
    flyInfo.Parent = flySection
    
    -- BOTÃO FECHAR
    local closeButton = Instance.new("TextButton")
    closeButton.Size = UDim2.new(0, 25, 0, 25)
    closeButton.Position = UDim2.new(1, -35, 0, 10)
    closeButton.BackgroundColor3 = Color3.new(0.8, 0, 0)
    closeButton.BorderSizePixel = 0
    closeButton.Text = "X"
    closeButton.TextColor3 = Color3.new(1, 1, 1)
    closeButton.TextScaled = true
    closeButton.Font = Enum.Font.GothamBold
    closeButton.Parent = mainFrame
    
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 12)
    closeCorner.Parent = closeButton
    
    print("Interface completa criada!")
    
    -- EVENTOS DOS BOTÕES
    
    -- Teleporte
    teleportToggle.MouseButton1Click:Connect(function()
        teleportEnabled = not teleportEnabled
        updateTeleportStatus(teleportStatus, teleportToggle)
        print("Teleporte: " .. (teleportEnabled and "ATIVADO" or "DESATIVADO"))
    end)
    
    teleportNow.MouseButton1Click:Connect(function()
        teleportToMousePosition()
    end)
    
    chooseKey.MouseButton1Click:Connect(function()
        setupCustomKey(keyLabel)
    end)
    
    -- Velocidade
    speedToggle.MouseButton1Click:Connect(function()
        speedEnabled = not speedEnabled
        updateSpeedStatus(speedStatus, speedToggle)
        updateSpeed()
        print("Velocidade: " .. (speedEnabled and "ATIVADA" or "DESATIVADA"))
    end)
    
    speedBox.FocusLost:Connect(function()
        local value = tonumber(speedBox.Text)
        if value and value > 0 and value <= 500 then
            currentSpeed = value
            updateSpeedStatus(speedStatus, speedToggle)
            if speedEnabled then
                updateSpeed()
            end
        else
            speedBox.Text = tostring(currentSpeed)
        end
    end)
    
    -- Cooldown
    cooldownToggle.MouseButton1Click:Connect(function()
        cooldownModifierEnabled = not cooldownModifierEnabled
        updateCooldownStatus(cooldownStatus, cooldownToggle)
        
        if cooldownModifierEnabled then
            setupCooldownHook()
        else
            removeCooldownHook()
        end
        print("Cooldown Modifier: " .. (cooldownModifierEnabled and "ATIVADO" or "DESATIVADO"))
    end)
    
    cooldownBox.FocusLost:Connect(function()
        local value = tonumber(cooldownBox.Text)
        if value and value > 0 and value <= 100 then
            cooldownMultiplier = value / 100
            updateCooldownStatus(cooldownStatus, cooldownToggle)
            print("Cooldown definido para: " .. value .. "%")
        else
            cooldownBox.Text = tostring(math.floor(cooldownMultiplier * 100))
        end
    end)
    
    -- Noclip
    noclipToggle.MouseButton1Click:Connect(function()
        noclipEnabled = not noclipEnabled
        updateNoclipStatus(noclipStatus, noclipToggle)
        toggleNoclip()
    end)
    
    -- Pulo Infinito
    jumpToggle.MouseButton1Click:Connect(function()
        infiniteJumpEnabled = not infiniteJumpEnabled
        updateJumpStatus(jumpStatus, jumpToggle)
        print("Pulo Infinito: " .. (infiniteJumpEnabled and "ATIVADO" or "DESATIVADO"))
    end)
    
    -- ESP
    espToggle.MouseButton1Click:Connect(function()
        espEnabled = not espEnabled
        updateESPStatus(espStatus, espToggle)
        toggleESP()
    end)
    
    -- Aim Assist
    aimToggle.MouseButton1Click:Connect(function()
        aimAssistEnabled = not aimAssistEnabled
        updateAimStatus(aimStatus, aimToggle)
        toggleAimAssist()
    end)
    
    -- Fly
    flyToggle.MouseButton1Click:Connect(function()
        flyEnabled = not flyEnabled
        updateFlyStatus(flyStatus, flyToggle)
        toggleFly()
    end)
    
    flyBox.FocusLost:Connect(function()
        local value = tonumber(flyBox.Text)
        if value and value > 0 and value <= 200 then
            flySpeed = value
            updateFlyStatus(flyStatus, flyToggle)
        else
            flyBox.Text = tostring(flySpeed)
        end
    end)
    
    -- Fechar
    closeButton.MouseButton1Click:Connect(function()
        toggleGui()
    end)
    
    -- Inicializar status
    updateTeleportStatus(teleportStatus, teleportToggle)
    updateSpeedStatus(speedStatus, speedToggle)
    updateCooldownStatus(cooldownStatus, cooldownToggle)
    updateNoclipStatus(noclipStatus, noclipToggle)
    updateJumpStatus(jumpStatus, jumpToggle)
    updateESPStatus(espStatus, espToggle)
    updateAimStatus(aimStatus, aimToggle)
    updateFlyStatus(flyStatus, flyToggle)
    
    print("Interface totalmente configurada!")
    return mainFrame
end

-- Função para abrir/fechar GUI
function toggleGui()
    if not gui or not gui.Parent then
        local newMainFrame = createGui()
        isGuiOpen = true
        newMainFrame.Visible = true
        print("GUI criada e exibida!")
    else
        isGuiOpen = not isGuiOpen
        if mainFrame then
            mainFrame.Visible = isGuiOpen
        end
    end
end

-- Sistema de input
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    -- Abrir/fechar painel com G
    if input.KeyCode == Enum.KeyCode.G then
        toggleGui()
    end
    
    -- Teleporte com tecla customizada
    if teleportEnabled then
        if input.KeyCode.Name == teleportKey then
            teleportToMousePosition()
        elseif teleportKey == "MouseButton1" and input.UserInputType == Enum.UserInputType.MouseButton1 then
            teleportToMousePosition()
        elseif teleportKey == "MouseButton2" and input.UserInputType == Enum.UserInputType.MouseButton2 then
            teleportToMousePosition()
        end
    end
    
    -- Atalhos
    if input.KeyCode == Enum.KeyCode.N then
        noclipEnabled = not noclipEnabled
        toggleNoclip()
        print("Noclip: " .. (noclipEnabled and "ATIVADO" or "DESATIVADO"))
    end
    
    if input.KeyCode == Enum.KeyCode.J then
        infiniteJumpEnabled = not infiniteJumpEnabled
        print("Pulo Infinito: " .. (infiniteJumpEnabled and "ATIVADO" or "DESATIVADO"))
    end
    
    if input.KeyCode == Enum.KeyCode.V then
        espEnabled = not espEnabled
        toggleESP()
        print("ESP: " .. (espEnabled and "ATIVADO" or "DESATIVADO"))
    end
    
    if input.KeyCode == Enum.KeyCode.B then
        aimAssistEnabled = not aimAssistEnabled
        toggleAimAssist()
        print("Aim Assist: " .. (aimAssistEnabled and "ATIVADO" or "DESATIVADO"))
    end
    
    if input.KeyCode == Enum.KeyCode.H then
        flyEnabled = not flyEnabled
        toggleFly()
        print("Fly: " .. (flyEnabled and "ATIVADO" or "DESATIVADO"))
    end
    
    if input.KeyCode == Enum.KeyCode.C then
        cooldownModifierEnabled = not cooldownModifierEnabled
        if cooldownModifierEnabled then
            setupCooldownHook()
        else
            removeCooldownHook()
        end
        print("Cooldown Modifier: " .. (cooldownModifierEnabled and "ATIVADO" or "DESATIVADO"))
    end
end)

-- Sistema de respawn
player.CharacterAdded:Connect(function(character)
    print("Character respawned...")
    
    local humanoid = character:WaitForChild("Humanoid")
    originalWalkSpeed = humanoid.WalkSpeed
    originalJumpPower = humanoid.JumpPower
    
    wait(2)
    
    if isGuiOpen then
        createGui()
        if gui and mainFrame then
            mainFrame.Visible = true
        end
    end
    
    -- Reconfigurar sistemas ativos
    if cooldownModifierEnabled then
        setupCooldownHook()
    end
    
    if speedEnabled then
        updateSpeed()
    end
    
    if noclipEnabled then
        toggleNoclip()
    end
    
    if espEnabled then
        toggleESP()
    end
    
    if aimAssistEnabled then
        toggleAimAssist()
    end
    
    if flyEnabled then
        toggleFly()
    end
end)

-- Configuração inicial
if player.Character then
    local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
    if humanoid then
        originalWalkSpeed = humanoid.WalkSpeed
        originalJumpPower = humanoid.JumpPower
    end
end

-- Sistema de pulo infinito global
UserInputService.JumpRequest:Connect(function()
    if infiniteJumpEnabled then
        local character = player.Character
        if character and character:FindFirstChildOfClass("Humanoid") then
            character:FindFirstChildOfClass("Humanoid"):ChangeState(Enum.HumanoidStateType.Jumping)
        end
    end
end)

wait(1)
print("PAINEL DE HABILIDADES PRO CARREGADO!")
print("CONTROLES:")
print("   G - Abrir/Fechar Painel")
print("   N - Toggle Noclip")
print("   J - Toggle Pulo Infinito") 
print("   V - Toggle ESP")
print("   B - Toggle Aim Assist")
print("   H - Toggle Fly")
print("   C - Toggle Cooldown Modifier")
print("   F - Teleporte (padrão)")

-- Criar GUI automaticamente
spawn(function()
    wait(2)
    if not isGuiOpen then
        toggleGui()
    end
end)
