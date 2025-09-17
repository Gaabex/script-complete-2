-- Painel de Habilidades para Roblox - VERSÃO COM COOLDOWN
-- Coloque este script em StarterPlayer > StarterPlayerScripts
-- Pressione G para abrir/fechar o painel

print("🔄 Iniciando Painel de Habilidades Avançado...")

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

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
local hookedRemotes = {}
local cooldownConnections = {}
local flySpeed = 50
local cooldownMultiplier = 0.5 -- 0.5 = metade do cooldown, 0.1 = 10% do cooldown original

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
    
    -- Animação do efeito
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

-- Sistema de Pulo Infinito
local function setupInfiniteJump()
    if infiniteJumpEnabled then
        UserInputService.JumpRequest:Connect(function()
            local character = player.Character
            if character and character:FindFirstChildOfClass("Humanoid") then
                local humanoid = character:FindFirstChildOfClass("Humanoid")
                humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
            end
        end)
        print("🦘 Pulo Infinito ATIVADO")
    else
        print("🚶 Pulo Infinito DESATIVADO")
    end
end

-- Sistema de ESP (visualizar jogadores através de paredes)
local function createESP(targetPlayer)
    if not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
        return
    end
    
    local character = targetPlayer.Character
    local rootPart = character.HumanoidRootPart
    
    -- Criar caixa de ESP
    local espBox = Instance.new("BoxHandleAdornment")
    espBox.Name = "ESPBox_" .. targetPlayer.Name
    espBox.Adornee = rootPart
    espBox.Size = rootPart.Size + Vector3.new(1, 3, 1)
    espBox.Color3 = targetPlayer.TeamColor and targetPlayer.TeamColor.Color or Color3.new(1, 0, 0)
    espBox.Transparency = 0.7
    espBox.AlwaysOnTop = true
    espBox.ZIndex = 10
    espBox.Parent = rootPart
    
    -- Criar label com nome
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
        
        -- Criar ESP para novos jogadores
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
        -- Remover todas as caixas ESP
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

-- Sistema de modificador de cooldown (NOVO)
local originalCooldowns = {}

local function setupCooldownModifier()
    cooldownConnections = {}
    hookedRemotes = {}
    originalCooldowns = {}
    
    local character = player.Character
    if not character then return end
    
    local function hookTool(tool)
        if not tool:IsA("Tool") then return end
        
        -- Hook para LocalScripts que controlam cooldown
        for _, descendant in pairs(tool:GetDescendants()) do
            if descendant:IsA("LocalScript") then
                local source = descendant.Source
                if string.find(string.lower(source), "cooldown") or string.find(string.lower(source), "wait") then
                    print("🕒 Detectado script com cooldown em: " .. tool.Name)
                end
            end
            
            -- Hook para RemoteEvents e RemoteFunctions
            if descendant:IsA("RemoteEvent") then
                if not hookedRemotes[descendant] then
                    hookedRemotes[descendant] = descendant.FireServer
                    
                    descendant.FireServer = function(self, ...)
                        local args = {...}
                        
                        -- Tentar interceptar argumentos de cooldown
                        for i, arg in pairs(args) do
                            if type(arg) == "number" and arg > 0.1 and arg < 60 then
                                -- Provavelmente um tempo de cooldown
                                local newCooldown = arg * cooldownMultiplier
                                args[i] = newCooldown
                                print("⏱️ Cooldown modificado: " .. arg .. "s -> " .. newCooldown .. "s")
                                break
                            end
                        end
                        
                        return hookedRemotes[descendant](self, unpack(args))
                    end
                end
            end
        end
        
        -- Hook para NumberValues que podem representar cooldown
        for _, descendant in pairs(tool:GetDescendants()) do
            if descendant:IsA("NumberValue") and (string.find(string.lower(descendant.Name), "cooldown") or 
                string.find(string.lower(descendant.Name), "delay") or string.find(string.lower(descendant.Name), "wait")) then
                
                if not originalCooldowns[descendant] then
                    originalCooldowns[descendant] = descendant.Value
                end
                
                descendant.Value = originalCooldowns[descendant] * cooldownMultiplier
                print("🔧 Cooldown Value modificado: " .. descendant.Name .. " -> " .. descendant.Value)
                
                -- Monitorar mudanças
                table.insert(cooldownConnections, descendant.Changed:Connect(function(newValue)
                    if newValue ~= originalCooldowns[descendant] * cooldownMultiplier then
                        wait(0.1)
                        descendant.Value = originalCooldowns[descendant] * cooldownMultiplier
                    end
                end))
            end
        end
    end
    
    -- Hook ferramentas existentes
    for _, tool in pairs(character:GetChildren()) do
        hookTool(tool)
    end
    
    local backpack = player:FindFirstChild("Backpack")
    if backpack then
        for _, tool in pairs(backpack:GetChildren()) do
            hookTool(tool)
        end
    end
    
    -- Hook futuras ferramentas
    table.insert(cooldownConnections, character.ChildAdded:Connect(hookTool))
    if backpack then
        table.insert(cooldownConnections, backpack.ChildAdded:Connect(hookTool))
    end
    
    -- Tentar modificar cooldowns globais conhecidos
    local function modifyGlobalCooldowns()
        local commonCooldownNames = {"Cooldown", "AttackCooldown", "SkillCooldown", "AbilityCooldown", "Delay"}
        
        for _, name in pairs(commonCooldownNames) do
            local foundValues = {}
            
            -- Procurar em ReplicatedStorage
            if game:GetService("ReplicatedStorage"):FindFirstChild(name, true) then
                local value = game:GetService("ReplicatedStorage"):FindFirstChild(name, true)
                if value:IsA("NumberValue") then
                    table.insert(foundValues, value)
                end
            end
            
            -- Procurar em StarterGui
            if game:GetService("StarterGui"):FindFirstChild(name, true) then
                local value = game:GetService("StarterGui"):FindFirstChild(name, true)
                if value:IsA("NumberValue") then
                    table.insert(foundValues, value)
                end
            end
            
            for _, value in pairs(foundValues) do
                if not originalCooldowns[value] then
                    originalCooldowns[value] = value.Value
                end
                value.Value = originalCooldowns[value] * cooldownMultiplier
                print("🌐 Cooldown global modificado: " .. value.Name)
            end
        end
    end
    
    modifyGlobalCooldowns()
    print("✅ Sistema de modificador de cooldown configurado!")
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

-- Função para criar a GUI (atualizada)
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
    
    -- ScrollingFrame para comportar todas as seções
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
    
    -- Função helper para criar seções
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
    
    -- Criar todas as seções (substituindo dano por cooldown)
    local teleportSection = createSection("Teleporte", "🚀", 5, 160)
    local speedSection = createSection("Velocidade", "💨", 175, 140)
    local noclipSection = createSection("Noclip", "👻", 325, 100)
    local jumpSection = createSection("Pulo Infinito", "🦘", 435, 100)
    local espSection = createSection("ESP", "👁️", 545, 100)
    local aimSection = createSection("Aim Assist", "🎯", 655, 100)
    local flySection = createSection("Fly", "✈️", 765, 140)
    local cooldownSection = createSection("Modificador de Cooldown", "⏱️", 915, 140)
    
    -- Configurar seção de teleporte
    local teleportStatus = Instance.new("TextLabel")
    teleportStatus.Size = UDim2.new(1, -10, 0, 20)
    teleportStatus.Position = UDim2.new(0, 5, 0, 30)
    teleportStatus.Backgroun
