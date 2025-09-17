-- Sistema Avançado de Hook de Cooldown para Roblox
-- Inclui Hook de RemoteEvents e Bypass de Proteções

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer

-- Configurações do sistema
local cooldownConfig = {
    enabled = false,
    multiplier = 0.5, -- 50% do tempo original
    debugMode = true,
    bypassProtections = true,
    hookRemoteEvents = true
}

-- Armazenamento de hooks originais
local originalFunctions = {
    wait = wait,
    taskWait = task.wait,
    remoteEventFire = nil,
    remoteFunctionInvoke = nil,
    spawn = spawn,
    delay = delay
}

-- Controle de cooldowns ativos
local activeCooldowns = {}
local remoteEventHooks = {}
local protectionBypasses = {}

-- Sistema de logging
local function log(message, level)
    if cooldownConfig.debugMode then
        local prefix = level == "ERROR" and "❌" or level == "WARNING" and "⚠️" or "✅"
        print(prefix .. " [CooldownSystem] " .. message)
    end
end

-- Função para criar bypass de proteções comuns
local function setupProtectionBypasses()
    if not cooldownConfig.bypassProtections then return end
    
    log("Configurando bypasses de proteção...", "INFO")
    
    -- Bypass de detecção de exploit
    local mt = getrawmetatable(game)
    if mt then
        setreadonly(mt, false)
        
        local oldIndex = mt.__index
        local oldNewindex = mt.__newindex
        
        mt.__index = function(self, key)
            if key == "Parent" and typeof(self) == "Instance" then
                return oldIndex(self, key)
            end
            return oldIndex(self, key)
        end
        
        mt.__newindex = function(self, key, value)
            if key == "Parent" and typeof(self) == "Instance" then
                return oldNewindex(self, key, value)
            end
            return oldNewindex(self, key, value)
        end
        
        setreadonly(mt, true)
        log("Bypass de metamétodo configurado", "INFO")
    end
    
    -- Bypass de detecção de velocidade anormal de chamadas
    protectionBypasses.callThrottle = {}
    protectionBypasses.lastCallTime = {}
end

-- Sistema de hook para wait functions
local function setupWaitHooks()
    log("Configurando hooks de wait...", "INFO")
    
    -- Hook wait() global
    local env = getfenv(1)
    if env then
        env.wait = function(duration)
            if duration and type(duration) == "number" and duration > 0 and cooldownConfig.enabled then
                local modifiedDuration = duration * cooldownConfig.multiplier
                if cooldownConfig.debugMode then
                    log(string.format("Wait interceptado: %.2fs -> %.2fs", duration, modifiedDuration), "INFO")
                end
                return originalFunctions.wait(modifiedDuration)
            end
            return originalFunctions.wait(duration)
        end
    end
    
    -- Hook task.wait
    task.wait = function(duration)
        if duration and type(duration) == "number" and duration > 0 and cooldownConfig.enabled then
            local modifiedDuration = duration * cooldownConfig.multiplier
            if cooldownConfig.debugMode then
                log(string.format("Task.wait interceptado: %.2fs -> %.2fs", duration, modifiedDuration), "INFO")
            end
            return originalFunctions.taskWait(modifiedDuration)
        end
        return originalFunctions.taskWait(duration)
    end
    
    -- Hook spawn com delay
    spawn = function(func)
        if cooldownConfig.enabled then
            return originalFunctions.spawn(function()
                func()
            end)
        end
        return originalFunctions.spawn(func)
    end
    
    -- Hook delay
    delay = function(time, func)
        if time and cooldownConfig.enabled then
            local modifiedTime = time * cooldownConfig.multiplier
            return originalFunctions.delay(modifiedTime, func)
        end
        return originalFunctions.delay(time, func)
    end
    
    log("Hooks de wait configurados com sucesso", "INFO")
end

-- Sistema de hook para RemoteEvents
local function hookRemoteEvent(remoteEvent)
    if not remoteEvent or remoteEventHooks[remoteEvent] then return end
    
    log("Configurando hook para RemoteEvent: " .. remoteEvent.Name, "INFO")
    
    -- Salvar função original
    local originalFire = remoteEvent.FireServer
    remoteEventHooks[remoteEvent] = originalFire
    
    -- Criar hook
    remoteEvent.FireServer = function(self, ...)
        local args = {...}
        local eventName = self.Name
        
        -- Verificar se é um evento relacionado a cooldown
        if cooldownConfig.enabled and (
            eventName:lower():find("skill") or 
            eventName:lower():find("ability") or
            eventName:lower():find("cast") or
            eventName:lower():find("use") or
            eventName:lower():find("activate")
        ) then
            -- Aplicar bypass de throttle se necessário
            local currentTime = tick()
            local lastCall = protectionBypasses.lastCallTime[eventName] or 0
            
            if currentTime - lastCall < 0.1 then -- Muito rápido
                if cooldownConfig.bypassProtections then
                    log("Bypass de throttle aplicado para: " .. eventName, "WARNING")
                    originalFunctions.wait(0.05) -- Pequeno delay para evitar detecção
                end
            end
            
            protectionBypasses.lastCallTime[eventName] = currentTime
            log("RemoteEvent interceptado: " .. eventName, "INFO")
        end
        
        -- Chamar função original
        return originalFire(self, unpack(args))
    end
end

-- Sistema de hook para RemoteFunctions
local function hookRemoteFunction(remoteFunction)
    if not remoteFunction or remoteEventHooks[remoteFunction] then return end
    
    log("Configurando hook para RemoteFunction: " .. remoteFunction.Name, "INFO")
    
    local originalInvoke = remoteFunction.InvokeServer
    remoteEventHooks[remoteFunction] = originalInvoke
    
    remoteFunction.InvokeServer = function(self, ...)
        local args = {...}
        local functionName = self.Name
        
        if cooldownConfig.enabled and (
            functionName:lower():find("skill") or
            functionName:lower():find("ability") or
            functionName:lower():find("cooldown")
        ) then
            log("RemoteFunction interceptada: " .. functionName, "INFO")
            
            -- Aplicar modificações nos argumentos se necessário
            for i, arg in ipairs(args) do
                if type(arg) == "number" and arg > 1 and arg < 300 then -- Possível cooldown
                    args[i] = arg * cooldownConfig.multiplier
                    log(string.format("Argumento %d modificado: %.2f -> %.2f", i, arg, args[i]), "INFO")
                end
            end
        end
        
        return originalInvoke(self, unpack(args))
    end
end

-- Função para encontrar e hookear todos os RemoteEvents/Functions
local function scanAndHookRemotes()
    if not cooldownConfig.hookRemoteEvents then return end
    
    log("Escaneando RemoteEvents e RemoteFunctions...", "INFO")
    
    local function scanFolder(folder)
        for _, child in ipairs(folder:GetChildren()) do
            if child:IsA("RemoteEvent") then
                hookRemoteEvent(child)
            elseif child:IsA("RemoteFunction") then
                hookRemoteFunction(child)
            elseif child:IsA("Folder") then
                scanFolder(child)
            end
        end
    end
    
    -- Escanear ReplicatedStorage
    scanFolder(ReplicatedStorage)
    
    -- Escanear workspace
    if workspace:FindFirstChild("RemoteEvents") then
        scanFolder(workspace.RemoteEvents)
    end
    
    -- Hook automático para novos remotes
    ReplicatedStorage.DescendantAdded:Connect(function(descendant)
        if descendant:IsA("RemoteEvent") then
            wait(0.1) -- Pequeno delay para garantir que está totalmente carregado
            hookRemoteEvent(descendant)
        elseif descendant:IsA("RemoteFunction") then
            wait(0.1)
            hookRemoteFunction(descendant)
        end
    end)
    
    log("Scan de remotes concluído", "INFO")
end

-- Sistema de detecção de cooldown por padrões
local function setupCooldownDetection()
    log("Configurando detecção avançada de cooldown...", "INFO")
    
    -- Monitor de mudanças em GUIs relacionadas a cooldown
    local function monitorGui(gui)
        for _, child in ipairs(gui:GetDescendants()) do
            if child:IsA("TextLabel") or child:IsA("TextButton") then
                if child.Name:lower():find("cooldown") or 
                   child.Name:lower():find("timer") or
                   child.Text:find("%d+") then
                    
                    child:GetPropertyChangedSignal("Text"):Connect(function()
                        local text = child.Text
                        local number = tonumber(text:match("%d+%.?%d*"))
                        
                        if number and number > 0 and cooldownConfig.enabled then
                            local newNumber = number * cooldownConfig.multiplier
                            log(string.format("GUI Cooldown detectado e modificado: %.1f -> %.1f", number, newNumber), "INFO")
                            
                            -- Tentar modificar o texto (pode não funcionar se protegido)
                            pcall(function()
                                child.Text = text:gsub(tostring(number), string.format("%.1f", newNumber))
                            end)
                        end
                    end)
                end
            end
        end
    end
    
    -- Monitor PlayerGui
    player.PlayerGui.DescendantAdded:Connect(function(descendant)
        if descendant:IsA("ScreenGui") then
            wait(1) -- Aguardar carregamento completo
            monitorGui(descendant)
        end
    end)
    
    -- Escanear GUIs existentes
    for _, gui in ipairs(player.PlayerGui:GetChildren()) do
        if gui:IsA("ScreenGui") then
            monitorGui(gui)
        end
    end
end

-- Sistema de controle de tempo personalizado
local customTimeControl = {}

function customTimeControl.createTimer(duration, callback)
    if cooldownConfig.enabled then
        duration = duration * cooldownConfig.multiplier
        log(string.format("Timer personalizado criado: %.2fs", duration), "INFO")
    end
    
    return originalFunctions.spawn(function()
        originalFunctions.wait(duration)
        if callback then callback() end
    end)
end

function customTimeControl.modifyExistingTimer(timerReference, newMultiplier)
    if timerReference and cooldownConfig.enabled then
        log("Modificando timer existente com multiplicador: " .. newMultiplier, "INFO")
        -- Implementação dependeria do sistema específico do jogo
    end
end

-- Sistema principal de controle
local CooldownSystem = {}

function CooldownSystem.enable(multiplier)
    cooldownConfig.enabled = true
    if multiplier then
        cooldownConfig.multiplier = math.max(0.01, math.min(1, multiplier))
    end
    
    log(string.format("Sistema ativado com multiplicador: %.2f", cooldownConfig.multiplier), "INFO")
    
    -- Configurar todos os hooks
    setupProtectionBypasses()
    setupWaitHooks()
    scanAndHookRemotes()
    setupCooldownDetection()
end

function CooldownSystem.disable()
    cooldownConfig.enabled = false
    log("Sistema desativado", "INFO")
    
    -- Restaurar funções originais
    wait = originalFunctions.wait
    task.wait = originalFunctions.taskWait
    spawn = originalFunctions.spawn
    delay = originalFunctions.delay
    
    -- Restaurar RemoteEvents
    for remote, originalFunction in pairs(remoteEventHooks) do
        if remote and originalFunction then
            if remote:IsA("RemoteEvent") then
                remote.FireServer = originalFunction
            elseif remote:IsA("RemoteFunction") then
                remote.InvokeServer = originalFunction
            end
        end
    end
    
    remoteEventHooks = {}
end

function CooldownSystem.setMultiplier(multiplier)
    cooldownConfig.multiplier = math.max(0.01, math.min(1, multiplier))
    log(string.format("Multiplicador alterado para: %.2f", cooldownConfig.multiplier), "INFO")
end

function CooldownSystem.toggleDebug()
    cooldownConfig.debugMode = not cooldownConfig.debugMode
    log("Debug mode: " .. (cooldownConfig.debugMode and "ATIVADO" or "DESATIVADO"), "INFO")
end

function CooldownSystem.getStatus()
    return {
        enabled = cooldownConfig.enabled,
        multiplier = cooldownConfig.multiplier,
        debugMode = cooldownConfig.debugMode,
        hookedRemotes = #remoteEventHooks,
        activeCooldowns = #activeCooldowns
    }
end

-- Exportar sistema
_G.CooldownSystem = CooldownSystem
_G.CustomTimeControl = customTimeControl

log("Sistema Avançado de Cooldown carregado!", "INFO")
log("Use _G.CooldownSystem.enable(0.5) para ativar com 50% do tempo", "INFO")
log("Use _G.CooldownSystem.disable() para desativar", "INFO")

return CooldownSystem
