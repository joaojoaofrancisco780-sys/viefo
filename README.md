local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")

math.randomseed(tick())

-- ====== CONFIGURAÇÕES ======
local PROMPT_ACTION_TEXT = "Tentar Lockpick"
local PROMPT_OBJECT_TEXT = "Carro"
local PROMPT_MAX_DISTANCE = 10
local PROMPT_HOLD_DURATION = 0
local PROMPT_NAME = "LockpickPrompt"

local AUTO_FARM_INTERVAL = 2
local AUTO_FARM_AMOUNT = 10
-- ===========================

-- Cria/garante RemoteEvent em ReplicatedStorage
local function ensureRemoteEvent(name)
    local ev = ReplicatedStorage:FindFirstChild(name)
    if ev and ev:IsA("RemoteEvent") then
        return ev
    end
    ev = Instance.new("RemoteEvent")
    ev.Name = name
    ev.Parent = ReplicatedStorage
    return ev
end

local ToggleAutoFarmEvent = ensureRemoteEvent("ToggleAutoFarm")
local RequestOpenLockpickEvent = ensureRemoteEvent("RequestOpenLockpick")
local OpenLockpickGuiEvent = ensureRemoteEvent("OpenLockpickGui")
local AttemptLockpickEvent = ensureRemoteEvent("AttemptLockpick")
local LockpickResultEvent = ensureRemoteEvent("LockpickResult")

-- Leaderstats helpers
local function ensureLeaderstats(player)
    local stats = player:FindFirstChild("leaderstats")
    if not stats then
        stats = Instance.new("Folder")
        stats.Name = "leaderstats"
        stats.Parent = player
    end
    local cash = stats:FindFirstChild("Cash")
    if not cash then
        cash = Instance.new("IntValue")
        cash.Name = "Cash"
        cash.Value = 0
        cash.Parent = stats
    end
    local lockpicks = stats:FindFirstChild("Lockpicks")
    if not lockpicks then
        lockpicks = Instance.new("IntValue")
        lockpicks.Name = "Lockpicks"
        lockpicks.Value = 3
        lockpicks.Parent = stats
    end
    return cash, lockpicks
end

local function playerHasLockpicks(player)
    local stats = player:FindFirstChild("leaderstats")
    if not stats then return 0 end
    local lp = stats:FindFirstChild("Lockpicks")
    if not lp then return 0 end
    return lp.Value
end

local function consumeLockpick(player)
    local stats = player:FindFirstChild("leaderstats")
    if not stats then return false end
    local lp = stats:FindFirstChild("Lockpicks")
    if not lp then return false end
    if lp.Value <= 0 then return false end
    lp.Value = lp.Value - 1
    return true
end

-- AutoFarm (servidor-side)
local autoFarmPlayers = {} -- [player] = true

local function startAutoFarmFor(player)
    if autoFarmPlayers[player] then return end
    autoFarmPlayers[player] = true
    spawn(function()
        while autoFarmPlayers[player] and player.Parent do
            local cash = ensureLeaderstats(player)
            cash.Value = cash.Value + AUTO_FARM_AMOUNT
            -- feedback opcional via evento (cliente recebe por LockpickResultEvent aqui)
            LockpickResultEvent:FireClient(player, true, "AutoFarm: +"..tostring(AUTO_FARM_AMOUNT))
            wait(AUTO_FARM_INTERVAL)
        end
        autoFarmPlayers[player] = nil
    end)
end

local function stopAutoFarmFor(player)
    autoFarmPlayers[player] = nil
end

ToggleAutoFarmEvent.OnServerEvent:Connect(function(player)
    if autoFarmPlayers[player] then
        stopAutoFarmFor(player)
        LockpickResultEvent:FireClient(player, false, "AutoFarm desativado.")
        print("[Server] AutoFarm desligado para", player.Name)
    else
        startAutoFarmFor(player)
        LockpickResultEvent:FireClient(player, true, "AutoFarm ativado.")
        print("[Server] AutoFarm ligado para", player.Name)
    end
end)

Players.PlayerRemoving:Connect(function(player)
    autoFarmPlayers[player] = nil
end)

-- ===== ProximityPrompt / carros =====

local function getCarReferencePart(car)
    if not car or not car:IsA("Model") then return nil end
    if car.PrimaryPart and car.PrimaryPart:IsA("BasePart") then
        return car.PrimaryPart
    end
    for _, v in ipairs(car:GetChildren()) do
        if v:IsA("BasePart") then
            return v
        end
    end
    return nil
end

local function ensurePromptForCar(car)
    if not car or not car:IsA("Model") then return end
    local refPart = getCarReferencePart(car)
    if not refPart then
        warn("[Server] Carro sem BasePart:", car.Name)
        return
    end

    local existing = refPart:FindFirstChild(PROMPT_NAME)
    if existing and existing:IsA("ProximityPrompt") then
        existing.ActionText = PROMPT_ACTION_TEXT
        existing.ObjectText = PROMPT_OBJECT_TEXT
        existing.MaxActivationDistance = PROMPT_MAX_DISTANCE
        existing.HoldDuration = PROMPT_HOLD_DURATION
        return
    end

    local prompt = Instance.new("ProximityPrompt")
    prompt.Name = PROMPT_NAME
    prompt.ActionText = PROMPT_ACTION_TEXT
    prompt.ObjectText = PROMPT_OBJECT_TEXT
    prompt.RequiresLineOfSight = false
    prompt.MaxActivationDistance = PROMPT_MAX_DISTANCE
    prompt.HoldDuration = PROMPT_HOLD_DURATION
    prompt.Parent = refPart

    prompt.Triggered:Connect(function(player)
        local lockedVal = car:FindFirstChild("Locked")
        if not lockedVal or not lockedVal:IsA("BoolValue") then
            OpenLockpickGuiEvent:FireClient(player, false, "Veículo sem BoolValue 'Locked'.")
            return
        end
        if not lockedVal.Value then
            OpenLockpickGuiEvent:FireClient(player, false, "Carro já desbloqueado.")
            return
        end
        -- autoriza abrir GUI no cliente passando o car.Name
        OpenLockpickGuiEvent:FireClient(player, car.Name)
        print("[Server] Prompt acionado por", player.Name, "no carro", car.Name)
    end)
end

local function initPromptsForCars()
    local carsFolder = Workspace:FindFirstChild("Cars")
    if not carsFolder then
        warn("[Server] Crie Workspace.Cars com Models que tenham BoolValue 'Locked' e uma BasePart.")
        return
    end
    for _, car in ipairs(carsFolder:GetChildren()) do
        if car:IsA("Model") then
            ensurePromptForCar(car)
        end
    end
    carsFolder.ChildAdded:Connect(function(child)
        if child:IsA("Model") then
            wait(0.05)
            ensurePromptForCar(child)
        end
    end)
end

initPromptsForCars()

-- Quando cliente pede via botão, procura o carro travado mais próximo
RequestOpenLockpickEvent.OnServerEvent:Connect(function(player)
    local char = player.Character
    if not char then
        OpenLockpickGuiEvent:FireClient(player, false, "Nenhum personagem.")
        return
    end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then
        OpenLockpickGuiEvent:FireClient(player, false, "Sem HumanoidRootPart.")
        return
    end
    local carsFolder = Workspace:FindFirstChild("Cars")
    if not carsFolder then
        OpenLockpickGuiEvent:FireClient(player, false, "Nenhum carro configurado.")
        return
    end

    local nearest = nil
    local nearestDist = math.huge
    for _, car in ipairs(carsFolder:GetChildren()) do
        if car:IsA("Model") then
            local lockedVal = car:FindFirstChild("Locked")
            local ref = getCarReferencePart(car)
            if lockedVal and lockedVal.Value and ref then
                local d = (ref.Position - hrp.Position).Magnitude
                if d < nearestDist then
                    nearestDist = d
                    nearest = car
                end
            end
        end
    end

    if nearest and nearestDist <= PROMPT_MAX_DISTANCE then
        OpenLockpickGuiEvent:FireClient(player, nearest.Name)
    else
        OpenLockpickGuiEvent:FireClient(player, false, "Nenhum carro travado por perto.")
    end
end)

-- Processa tentativa de lockpick
AttemptLockpickEvent.OnServerEvent:Connect(function(player, carName, clientSaidSuccess)
    if typeof(carName) ~= "string" then
        LockpickResultEvent:FireClient(player, false, "Dados inválidos.")
        return
    end
    local carsFolder = Workspace:FindFirstChild("Cars")
    if not carsFolder then
        LockpickResultEvent:FireClient(player, false, "Nenhum carro configurado.")
        return
    end
    local car = carsFolder:FindFirstChild(carName)
    if not car then
        LockpickResultEvent:FireClient(player, false, "Carro não encontrado.")
        return
    end
    local lockedVal = car:FindFirstChild("Locked")
    if not lockedVal or not lockedVal:IsA("BoolValue") then
        LockpickResultEvent:FireClient(player, false, "Carro sem estado de trava.")
        return
    end
    if not lockedVal.Value then
        LockpickResultEvent:FireClient(player, false, "Carro já está desbloqueado.")
        return
    end
    if playerHasLockpicks(player) <= 0 then
        LockpickResultEvent:FireClient(player, false, "Sem lockpicks.")
        return
    end
    if not consumeLockpick(player) then
        LockpickResultEvent:FireClient(player, false, "Falha ao consumir lockpick.")
        return
    end

    local clientFactor = clientSaidSuccess and 1 or 0
    local baseChance = 0.35
    local bonus = clientFactor * 0.45
    local successChance = math.clamp(baseChance + bonus, 0, 1)

    if math.random() < successChance then
        lockedVal.Value = false
        local ownerId = car:FindFirstChild("OwnerUserId")
        if not ownerId then
            ownerId = Instance.new("IntValue")
            ownerId.Name = "OwnerUserId"
            ownerId.Value = player.UserId
            ownerId.Parent = car
        else
            ownerId.Value = player.UserId
        end
        LockpickResultEvent:FireClient(player, true, "Lockpick bem-sucedido! Carro desbloqueado.")
        print("[Server] Carro desbloqueado por", player.Name, "=>", car.Name)
    else
        LockpickResultEvent:FireClient(player, false, "Lockpick falhou.")
    end
end)

-- ===== Injeção do LocalScript no PlayerGui =====
-- Código cliente (UI + input + minigame) em string para ser atribuído ao LocalScript.Source
local clientSource = [[
-- LocalScript gerado automaticamente pelo servidor.
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ContextActionService = game:GetService("ContextActionService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local ToggleAutoFarmEvent = ReplicatedStorage:WaitForChild("ToggleAutoFarm")
local RequestOpenLockpickEvent = ReplicatedStorage:WaitForChild("RequestOpenLockpick")
local OpenLockpickGuiEvent = ReplicatedStorage:WaitForChild("OpenLockpickGui")
local AttemptLockpickEvent = ReplicatedStorage:WaitForChild("AttemptLockpick")
local LockpickResultEvent = ReplicatedStorage:WaitForChild("LockpickResult")

-- UI (simples)
local UI_BG = Color3.fromRGB(20, 22, 25)
local UI_ACCENT = Color3.fromRGB(45, 150, 255)
local UI_TEXT = Color3.fromRGB(230,230,230)
local UI_MUTED = Color3.fromRGB(170,170,170)

-- evita duplicação caso já exista (reconexões)
if playerGui:FindFirstChild("AutoFarmLockpickUI") then
    playerGui.AutoFarmLockpickUI:Destroy()
end

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AutoFarmLockpickUI"
screenGui.Parent = playerGui
screenGui.ResetOnSpawn = false

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 230, 0, 120)
mainFrame.Position = UDim2.new(0, 12, 0, 80)
mainFrame.BackgroundColor3 = UI_BG
mainFrame.BorderSizePixel = 0
mainFrame.Parent = screenGui
local corner = Instance.new("UICorner", mainFrame)
corner.CornerRadius = UDim.new(0, 8)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -12, 0, 28)
title.Position = UDim2.new(0, 6, 0, 6)
title.BackgroundTransparency = 1
title.Text = "Ferramentas"
title.TextColor3 = UI_TEXT
title.Font = Enum.Font.SourceSansSemibold
title.TextSize = 18
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = mainFrame

local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(0.9, 0, 0, 36)
toggleBtn.Position = UDim2.new(0.05, 0, 0, 36)
toggleBtn.BackgroundColor3 = UI_ACCENT
toggleBtn.TextColor3 = Color3.new(1,1,1)
toggleBtn.Font = Enum.Font.SourceSansBold
toggleBtn.TextSize = 16
toggleBtn.Text = "Alternar AutoFarm (G)"
toggleBtn.Parent = mainFrame
local btnCorner = Instance.new("UICorner", toggleBtn)
btnCorner.CornerRadius = UDim.new(0, 6)

local lockpickBtn = Instance.new("TextButton")
lockpickBtn.Size = UDim2.new(0.9, 0, 0, 28)
lockpickBtn.Position = UDim2.new(0.05, 0, 0, 78)
lockpickBtn.BackgroundColor3 = Color3.fromRGB(60,60,60)
lockpickBtn.TextColor3 = UI_TEXT
lockpickBtn.Font = Enum.Font.SourceSans
lockpickBtn.TextSize = 14
lockpickBtn.Text = "Tentar Lockpick (Interagir ou botão)"
lockpickBtn.Parent = mainFrame
local lpCorner = Instance.new("UICorner", lockpickBtn)
lpCorner.CornerRadius = UDim.new(0, 6)

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(1, -12, 0, 18)
statusLabel.Position = UDim2.new(0, 6, 0, 108)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = ""
statusLabel.TextColor3 = UI_MUTED
statusLabel.Font = Enum.Font.SourceSans
statusLabel.TextSize = 14
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.Parent = mainFrame

-- Minigame UI
local lpFrame = Instance.new("Frame")
lpFrame.Size = UDim2.new(0, 420, 0, 140)
lpFrame.Position = UDim2.new(0.5, -210, 0.66, -70)
lpFrame.AnchorPoint = Vector2.new(0.5, 0)
lpFrame.BackgroundColor3 = UI_BG
lpFrame.BorderSizePixel = 0
lpFrame.Parent = screenGui
lpFrame.Visible = false
local lpCorner = Instance.new("UICorner", lpFrame)
lpCorner.CornerRadius = UDim.new(0, 10)

local lpTitle = Instance.new("TextLabel")
lpTitle.Size = UDim2.new(1, -24, 0, 28)
lpTitle.Position = UDim2.new(0, 12, 0, 8)
lpTitle.BackgroundTransparency = 1
lpTitle.Text = "Lockpick"
lpTitle.TextColor3 = UI_TEXT
lpTitle.Font = Enum.Font.SourceSansSemibold
lpTitle.TextSize = 18
lpTitle.Parent = lpFrame

local lpBar = Instance.new("Frame")
lpBar.Size = UDim2.new(0.92, 0, 0, 36)
lpBar.Position = UDim2.new(0.04, 0, 0, 46)
lpBar.BackgroundColor3 = Color3.fromRGB(200,200,200)
lpBar.Parent = lpFrame
local lpBarCorner = Instance.new("UICorner", lpBar)
lpBarCorner.CornerRadius = UDim.new(0, 6)

local lpMover = Instance.new("Frame")
lpMover.Size = UDim2.new(0.06, 0, 1, 0)
lpMover.Position = UDim2.new(0, 0, 0, 0)
lpMover.BackgroundColor3 = Color3.fromRGB(255,80,80)
lpMover.Parent = lpBar
local lpMoverCorner = Instance.new("UICorner", lpMover)
lpMoverCorner.CornerRadius = UDim.new(0, 4)

local lpTarget = Instance.new("Frame")
lpTarget.Size = UDim2.new(0.14, 0, 1, 0)
lpTarget.Position = UDim2.new(0.43, 0, 0, 0)
lpTarget.BackgroundColor3 = Color3.fromRGB(100,220,120)
lpTarget.Transparency = 0.15
lpTarget.Parent = lpBar
local lpTargetCorner = Instance.new("UICorner", lpTarget)
lpTargetCorner.CornerRadius = UDim.new(0, 6)

local lpInfo = Instance.new("TextLabel")
lpInfo.Size = UDim2.new(1, -24, 0, 28)
lpInfo.Position = UDim2.new(0, 12, 0, 88)
lpInfo.BackgroundTransparency = 1
lpInfo.Text = "Clique para tentar"
lpInfo.TextColor3 = UI_TEXT
lpInfo.Font = Enum.Font.SourceSans
lpInfo.TextSize = 16
lpInfo.Parent = lpFrame

-- Minigame state
local lpRunning = false
local lpDirection = 1
local lpSpeed = 0.9
local currentCarName = nil

local function showLockpickGui(carName)
    currentCarName = carName
    lpFrame.Visible = true
    lpRunning = true
    lpMover.Position = UDim2.new(0, 0, 0, 0)
    lpDirection = 1
    lpInfo.Text = "Clique para tentar"
    statusLabel.Text = ""
end

local function hideLockpickGui()
    lpRunning = false
    currentCarName = nil
    lpFrame.Visible = false
end

OpenLockpickGuiEvent.OnClientEvent:Connect(function(carNameOrFalse, maybeMsg)
    if carNameOrFalse == false then
        statusLabel.Text = maybeMsg or "Nenhum carro perto."
        delay(2, function() if statusLabel then statusLabel.Text = "" end end)
        return
    end
    showLockpickGui(carNameOrFalse)
end)

lpBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 and lpRunning and currentCarName then
        lpRunning = false
        local moverCenter = lpMover.AbsolutePosition.X + lpMover.AbsoluteSize.X * 0.5
        local targetStart = lpTarget.AbsolutePosition.X
        local targetEnd = lpTarget.AbsolutePosition.X + lpTarget.AbsoluteSize.X
        local inside = moverCenter >= targetStart and moverCenter <= targetEnd
        AttemptLockpickEvent:FireServer(currentCarName, inside)
        hideLockpickGui()
        statusLabel.Text = "Tentando..."
    end
end)

RunService.Heartbeat:Connect(function(dt)
    if lpRunning and lpMover and lpMover.Parent then
        local pos = lpMover.Position.X.Scale
        pos = pos + lpDirection * lpSpeed * dt
        if pos <= 0 then
            pos = 0
            lpDirection = 1
        elseif pos + lpMover.Size.X.Scale >= 1 then
            pos = 1 - lpMover.Size.X.Scale
            lpDirection = -1
        end
        lpMover.Position = UDim2.new(pos, 0, 0, 0)
    end
end)

LockpickResultEvent.OnClientEvent:Connect(function(success, message)
    if success then
        statusLabel.Text = message or "Desbloqueado!"
    else
        statusLabel.Text = message or "Falhou."
    end
    delay(2, function()
        if statusLabel then statusLabel.Text = "" end
    end)
end)

-- Bind G para ToggleAutoFarm
local function onToggleAction(name, state)
    if state == Enum.UserInputState.Begin then
        ToggleAutoFarmEvent:FireServer()
        statusLabel.Text = "Toggle AutoFarm solicitado..."
        delay(1.5, function() if statusLabel then statusLabel.Text = "" end end)
    end
end
ContextActionService:BindAction("ToggleAutoFarmKey", onToggleAction, false, Enum.KeyCode.G)

toggleBtn.MouseButton1Click:Connect(function()
    ToggleAutoFarmEvent:FireServer()
    statusLabel.Text = "Toggle AutoFarm solicitado..."
    delay(1.5, function() if statusLabel then statusLabel.Text = "" end end)
end)

lockpickBtn.MouseButton1Click:Connect(function()
    RequestOpenLockpickEvent:FireServer()
    statusLabel.Text = "Procurando carro travado..."
end)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.Escape then
        if lpFrame.Visible then
            hideLockpickGui()
        end
    end
end)
]]

-- Injeta LocalScript no PlayerGui com o código acima
local function injectLocalScriptToPlayer(player)
    -- tenta pegar PlayerGui; aguarda um pouco se necessário
    local playerGui = player:FindFirstChild("PlayerGui") or player:WaitForChild("PlayerGui", 10)
    if not playerGui then
        warn("[Server] Não foi possível obter PlayerGui para", player.Name)
        return
    end

    -- evita duplicar
    local existing = playerGui:FindFirstChild("AutoFarmLockpickUI")
    if existing then
        existing:Destroy()
    end
    local existingScript = playerGui:FindFirstChild("AutoFarmLockpickClient")
    if existingScript then
        existingScript:Destroy()
    end

    local localScript = Instance.new("LocalScript")
    localScript.Name = "AutoFarmLockpickClient"
    localScript.Source = clientSource
    localScript.Parent = playerGui

    print("[Server] LocalScript injetado em PlayerGui para", player.Name)
end

-- Injetar para quem já está conectado (útil no Studio se rodar o script em hot-reload)
for _, pl in ipairs(Players:GetPlayers()) do
    ensureLeaderstats(pl)
    injectLocalScriptToPlayer(pl)
end

Players.PlayerAdded:Connect(function(player)
    ensureLeaderstats(player)
    -- injeta quando PlayerGui estiver pronto
    player.CharacterAdded:Connect(function()
        wait(0.1)
        injectLocalScriptToPlayer(player)
    end)
    -- tenta também logo na entrada
    injectLocalScriptToPlayer(player)
end)
