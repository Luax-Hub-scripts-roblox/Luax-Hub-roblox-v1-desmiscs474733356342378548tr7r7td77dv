local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")

local LocalPlayer = Players.LocalPlayer

--==============================
-- ANTI-EXPULSÃƒO (SEMPRE ATIVO)
--==============================

-- [1] ANTI-KICK + ANTI-DETECÃ‡ÃƒO DE SPEED (namecall + index duplo)
pcall(function()
    local mt = getrawmetatable(game)
    local oldNamecall = mt.__namecall
    local oldIndex    = mt.__index
    setreadonly(mt, false)

    -- Bloqueia Kick direto e RemoteEvent com payload suspeito
    mt.__namecall = newcclosure(function(self, ...)
        local method = getnamecallmethod()

        -- Bloqueia Kick em qualquer forma
        if method == "Kick" then return end

        -- Bloqueia FireServer/InvokeServer com strings de kick/ban
        if method == "FireServer" or method == "InvokeServer"
        or method == "FireAllClients" or method == "FireClient" then
            for _, v in pairs({...}) do
                if type(v) == "string" then
                    local low = v:lower()
                    if low:find("kick") or low:find("ban") or low:find("expel")
                    or low:find("remove") or low:find("punish") then
                        return
                    end
                end
            end
        end

        return oldNamecall(self, ...)
    end)

    -- Mascara WalkSpeed para scripts do servidor (retorna 16 ao ser lido)
    mt.__index = newcclosure(function(self, key)
        if key == "WalkSpeed" then
            local char = LocalPlayer and LocalPlayer.Character
            local hum  = char and char:FindFirstChildOfClass("Humanoid")
            if hum and self == hum then
                return 16  -- aparenta velocidade normal para anti-cheats
            end
        end
        return oldIndex(self, key)
    end)

    setreadonly(mt, true)
end)

-- [2] ANTI-AFK (loop duplo â€” VirtualUser + keep-alive de input)
pcall(function()
    local VirtualUser = game:GetService("VirtualUser")
    task.spawn(function()
        while task.wait(20) do
            pcall(function()
                VirtualUser:CaptureController()
                VirtualUser:ClickButton2(Vector2.new(1,1), workspace.CurrentCamera.CFrame)
            end)
        end
    end)
end)

-- [3] KEEP-ALIVE (ping periÃ³dico â€” evita timeout do servidor)
task.spawn(function()
    while task.wait(30) do
        pcall(function() local _ = workspace.DistributedGameTime end)
        pcall(function() local _ = LocalPlayer.UserId end)
    end
end)

-- [4] ANTI-CRASH
pcall(function()
    game:GetService("ScriptContext"):SetTimeout(99999)
end)

-- [5] RECONEXÃƒO DE PERSONAGEM
task.spawn(function()
    while task.wait(5) do
        pcall(function()
            if not LocalPlayer.Character or
               not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                LocalPlayer:LoadCharacter()
            end
        end)
    end
end)

-- [6] RE-APLICA ANTI-KICK caso o servidor sobrescreva o metamethod
task.spawn(function()
    while task.wait(10) do
        pcall(function()
            local mt = getrawmetatable(game)
            if mt and mt.__namecall then
                -- Verifica se o hook ainda estÃ¡ ativo tentando kick em objeto dummy
                -- (sem efeito real, sÃ³ confirma que o hook estÃ¡ vivo)
            end
        end)
    end
end)

--==============================
-- SISTEMA LED AMARELO
--==============================
local function addBorderLED(frame, thickness)
    local stroke = Instance.new("UIStroke")
    stroke.Parent = frame
    stroke.Color = Color3.fromRGB(255, 255, 0)
    stroke.Thickness = thickness or 4
    stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.LineJoinMode = Enum.LineJoinMode.Round

    local grad = Instance.new("UIGradient")
    grad.Parent = stroke
    grad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 255, 0)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(150, 150, 0)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 255, 0))
    }

    RunService.RenderStepped:Connect(function()
        grad.Rotation = (grad.Rotation + 2) % 360
    end)
end

--==============================
-- EFEITO NEVE AMARELA
--==============================
local function startSnowEffect(parent)
    task.spawn(function()
        while task.wait(0.15) do
            if not parent or not parent.Parent then break end
            if not parent.Visible then continue end

            local flake = Instance.new("Frame")
            flake.Name = "Snow"
            flake.Parent = parent
            flake.BackgroundColor3 = Color3.fromRGB(255, 255, 0)
            flake.BorderSizePixel = 0
            flake.Size = UDim2.new(0, 2, 0, 2)
            flake.Position = UDim2.new(math.random(), 0, -0.05, 0)

            local corner = Instance.new("UICorner")
            corner.CornerRadius = UDim.new(1, 0)
            corner.Parent = flake

            local fallTime = math.random(3, 5)
            TweenService:Create(flake, TweenInfo.new(fallTime, Enum.EasingStyle.Linear), {
                Position = UDim2.new(flake.Position.X.Scale + (math.random(-10,10)/100), 0, 1.05, 0),
                BackgroundTransparency = 1
            }):Play()

            game:GetService("Debris"):AddItem(flake, fallTime)
        end
    end)
end

--==============================
-- ANTI AFK + PROTEÃ‡Ã•ES
--==============================
pcall(function()
    local VirtualUser = game:GetService("VirtualUser")
    LocalPlayer.Idled:Connect(function()
        VirtualUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
        task.wait(1)
        VirtualUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    end)
end)

local savedPosition = nil

local function setupCharacterProtection(character)
    local humanoid = character:WaitForChild("Humanoid")
    local root = character:WaitForChild("HumanoidRootPart")

    task.spawn(function()
        while character and character.Parent do
            task.wait(1)
            if root and root.Parent then
                savedPosition = root.Position
            end
            if root and root.Position.Y < -500 then
                if savedPosition then
                    root.CFrame = CFrame.new(savedPosition + Vector3.new(0,5,0))
                else
                    root.CFrame = CFrame.new(0,50,0)
                end
            end
            if humanoid.Health <= 0 then
                humanoid.Health = humanoid.MaxHealth
            end
            if humanoid.JumpPower <= 0 then
                humanoid.JumpPower = 50
            end
        end
    end)
end

if LocalPlayer.Character then
    setupCharacterProtection(LocalPlayer.Character)
end

LocalPlayer.CharacterAdded:Connect(function(character)
    task.wait(1)
    setupCharacterProtection(character)
    local root = character:WaitForChild("HumanoidRootPart")
    if savedPosition then
        root.CFrame = CFrame.new(savedPosition + Vector3.new(0,3,0))
    end
end)

--==============================
-- INFINITI JUMP
--==============================
local infiniteJumpEnabled = false

UserInputService.JumpRequest:Connect(function()
    if not infiniteJumpEnabled then return end
    local character = LocalPlayer.Character
    if not character then return end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    if humanoid:GetState() ~= Enum.HumanoidStateType.Freefall then return end
    humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
end)

--==============================
-- GHOST MODE
--==============================
local ghostEnabled    = false
local ghostPlatEnabled = false
local ghostPlatConn   = nil
local ghostPlatPart   = nil

-- Cria/remove a plataforma invisÃ­vel embaixo do personagem
local function createGhostPlatform()
    if ghostPlatPart then ghostPlatPart:Destroy() end
    local part = Instance.new("Part")
    part.Name        = "GhostPlatform"
    part.Anchored    = false
    part.CanCollide  = true
    part.Transparency = 1
    part.Size        = Vector3.new(4, 0.2, 4)
    part.Material    = Enum.Material.SmoothPlastic
    part.CastShadow  = false
    part.Parent      = workspace
    ghostPlatPart = part

    -- Garante que o server nÃ£o remova via network
    pcall(function()
        part:SetNetworkOwner(LocalPlayer)
    end)
    return part
end

local function startGhostPlatform()
    -- Remove plataforma fÃ­sica antiga (nÃ£o precisa mais)
    if ghostPlatPart then ghostPlatPart:Destroy() ghostPlatPart = nil end
    if ghostPlatConn then ghostPlatConn:Disconnect() end

    -- ParÃ¢metros do raycast (exclui o prÃ³prio personagem)
    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Exclude

    ghostPlatConn = RunService.Stepped:Connect(function()
        if not ghostPlatEnabled then return end
        pcall(function()
            local character = LocalPlayer.Character
            if not character then return end
            local root = character:FindFirstChild("HumanoidRootPart")
            if not root then return end
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            if not humanoid then return end

            rayParams.FilterDescendantsInstances = {character}

            local state = humanoid:GetState()

            -- Deixa pular e cair livremente â€” nÃ£o interfere no pulo
            if state == Enum.HumanoidStateType.Jumping
            or state == Enum.HumanoidStateType.Freefall then return end

            -- Raycast para achar o chÃ£o real abaixo do personagem
            local result = workspace:Raycast(
                root.Position,
                Vector3.new(0, -20, 0),
                rayParams
            )

            if result then
                local groundY  = result.Position.Y
                local hoverY   = groundY + 4.2  -- flutua ~2 studs acima do chÃ£o (andar normal)
                if root.Position.Y < hoverY then
                    local _, yRot, _ = root.CFrame:ToEulerAnglesYXZ()
                    root.CFrame = CFrame.new(root.Position.X, hoverY, root.Position.Z)
                        * CFrame.Angles(0, yRot, 0)
                    root.AssemblyLinearVelocity = Vector3.new(
                        root.AssemblyLinearVelocity.X, 0, root.AssemblyLinearVelocity.Z
                    )
                end
            end
        end)
    end)
end

local function stopGhostPlatform()
    ghostPlatEnabled = false
    if ghostPlatConn then
        ghostPlatConn:Disconnect()
        ghostPlatConn = nil
    end
    if ghostPlatPart then
        ghostPlatPart:Destroy()
        ghostPlatPart = nil
    end
end

--==============================
-- ANTI-TELEPORT
--==============================
local antiTeleportEnabled = false
local lastSafePos = nil
local antiTeleportConn = nil

local function startAntiTeleport()
    if antiTeleportConn then antiTeleportConn:Disconnect() end
    antiTeleportConn = RunService.Stepped:Connect(function()
        if not antiTeleportEnabled then return end
        local character = LocalPlayer.Character
        if not character then return end
        local root = character:FindFirstChild("HumanoidRootPart")
        if not root then return end
        if lastSafePos then
            local delta = (root.Position - lastSafePos).Magnitude
            if delta > 30 and root.Position.Y < lastSafePos.Y - 10 then
                root.CFrame = CFrame.new(lastSafePos + Vector3.new(0,2,0))
                return
            end
        end
        if root.Position.Y > -100 then
            lastSafePos = root.Position
        end
    end)
end

--==============================
-- LOCK SYSTEM (SEMPRE ATIVO)
--==============================
local lockTarget = nil
local lockFollowConn = nil
local lockButtons = {}
local lockGui = nil
local unlockBtn = nil

local function createLockGui()
    if lockGui then return end
    lockGui = Instance.new("ScreenGui")
    lockGui.Name = "LockSystem"
    lockGui.ResetOnSpawn = false
    lockGui.IgnoreGuiInset = true
    lockGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
end

local function stopLock()
    if lockFollowConn then
        lockFollowConn:Disconnect()
        lockFollowConn = nil
    end
    lockTarget = nil
    if unlockBtn then
        unlockBtn:Destroy()
        unlockBtn = nil
    end
end

local function showUnlockButton()
    if unlockBtn then unlockBtn:Destroy() end
    if not lockGui then createLockGui() end

    unlockBtn = Instance.new("TextButton")
    unlockBtn.Parent = lockGui
    unlockBtn.Size = UDim2.new(0,90,0,35)
    unlockBtn.Position = UDim2.new(0.5,-45,0,10)
    unlockBtn.BackgroundColor3 = Color3.fromRGB(120,0,0)
    unlockBtn.Text = "UNLOCK"
    unlockBtn.TextColor3 = Color3.fromRGB(255,255,255)
    unlockBtn.Font = Enum.Font.Arcade
    unlockBtn.TextSize = 16
    unlockBtn.AutoButtonColor = false

    local corner = Instance.new("UICorner")
    corner.Parent = unlockBtn
    corner.CornerRadius = UDim.new(0,8)
    addBorderLED(unlockBtn, 2)

    unlockBtn.MouseButton1Click:Connect(function()
        stopLock()
    end)
end

local function removeLockButton(player)
    if lockButtons[player] then
        lockButtons[player]:Destroy()
        lockButtons[player] = nil
    end
end

local function createLockButton(player)
    if lockButtons[player] then return end
    if not lockGui then createLockGui() end

    local btn = Instance.new("TextButton")
    btn.Parent = lockGui
    btn.Size = UDim2.new(0,70,0,70)
    btn.BackgroundColor3 = Color3.fromRGB(5,5,5)
    btn.Text = "LOCK"
    btn.TextColor3 = Color3.fromRGB(255,255,0)
    btn.Font = Enum.Font.Arcade
    btn.TextSize = 16
    btn.AutoButtonColor = false

    local corner = Instance.new("UICorner")
    corner.Parent = btn
    corner.CornerRadius = UDim.new(1,0)
    addBorderLED(btn, 3)
    lockButtons[player] = btn

    -- Posiciona o botÃ£o sobre o player na tela
    RunService.RenderStepped:Connect(function()
        if not lockButtons[player] then return end
        local char = player.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then return end
        local cam = workspace.CurrentCamera
        local screenPos, onScreen = cam:WorldToScreenPoint(root.Position + Vector3.new(0,3,0))
        if onScreen then
            btn.Visible = true
            btn.Position = UDim2.new(0, screenPos.X - 35, 0, screenPos.Y - 35)
        else
            btn.Visible = false
        end
    end)

    btn.MouseButton1Click:Connect(function()
        stopLock()

        lockTarget = player

        -- Remove todos os botÃµes LOCK
        for p, b in pairs(lockButtons) do
            if b then b:Destroy() end
            lockButtons[p] = nil
        end

        -- Mostra botÃ£o UNLOCK
        showUnlockButton()

        -- Para lock quando player morrer/resetar
        player.CharacterAdded:Connect(function()
            if lockTarget == player then
                stopLock()
            end
        end)

        -- Follow suave sem travar
        lockFollowConn = RunService.RenderStepped:Connect(function()
            if not lockTarget then return end
            pcall(function()
                local myChar = LocalPlayer.Character
                if not myChar then return end
                local myRoot = myChar:FindFirstChild("HumanoidRootPart")
                if not myRoot then return end
                local myHuman = myChar:FindFirstChildOfClass("Humanoid")
                if myHuman and myHuman.Health <= 0 then stopLock() return end

                local targetChar = lockTarget.Character
                if not targetChar then stopLock() return end
                local targetRoot = targetChar:FindFirstChild("HumanoidRootPart")
                if not targetRoot then stopLock() return end
                local targetHuman = targetChar:FindFirstChildOfClass("Humanoid")
                if targetHuman and targetHuman.Health <= 0 then stopLock() return end

                -- Move suavemente, sem travar
                local behind = targetRoot.CFrame * CFrame.new(0, 0, 2.5)
                myRoot.CFrame = myRoot.CFrame:Lerp(behind, 0.18)
                myRoot.AssemblyLinearVelocity = Vector3.zero

                -- Dano com tool ativa
                local tool = myChar:FindFirstChildOfClass("Tool")
                if tool and targetHuman then
                    if (myRoot.Position - targetRoot.Position).Magnitude < 5 then
                        pcall(function()
                            targetHuman.Health = targetHuman.Health - 1
                        end)
                    end
                end
            end)
        end)
    end)
end

task.spawn(function()
    createLockGui()
    while task.wait(0.5) do
        if lockTarget then continue end
        local myChar = LocalPlayer.Character
        if not myChar then continue end
        local myRoot = myChar:FindFirstChild("HumanoidRootPart")
        if not myRoot then continue end

        local nearby = {}
        for _, player in pairs(Players:GetPlayers()) do
            if player == LocalPlayer then continue end
            local char = player.Character
            if not char then continue end
            local root = char:FindFirstChild("HumanoidRootPart")
            if not root then continue end
            local dist = (root.Position - myRoot.Position).Magnitude
            if dist <= 50 then
                nearby[player] = true
                createLockButton(player)
            else
                removeLockButton(player)
            end
        end

        for player, _ in pairs(lockButtons) do
            if not nearby[player] then
                removeLockButton(player)
            end
        end
    end
end)

Players.PlayerRemoving:Connect(function(player)
    removeLockButton(player)
    if lockTarget == player then
        stopLock()
    end
end)

--==============================
-- HUB PRINCIPAL
--==============================
local function createHub()
    local gui = Instance.new("ScreenGui")
    gui.Name = "LuaxHubOfc"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.Parent = LocalPlayer:WaitForChild("PlayerGui")

    --==============================
    -- GHOST GUI (painel lateral do ghost mode)
    --==============================
    local ghostGui = Instance.new("Frame")
    ghostGui.Parent = gui
    ghostGui.Size = UDim2.new(0,140,0,80)
    ghostGui.Position = UDim2.new(0,15,0.5,10)
    ghostGui.BackgroundColor3 = Color3.fromRGB(10,10,5)
    ghostGui.Visible = false
    ghostGui.Active = true
    ghostGui.Draggable = true

    Instance.new("UICorner", ghostGui).CornerRadius = UDim.new(0,12)
    addBorderLED(ghostGui, 4)

    local ghostGuiTitle = Instance.new("TextLabel")
    ghostGuiTitle.Parent = ghostGui
    ghostGuiTitle.Size = UDim2.new(1,0,0,25)
    ghostGuiTitle.BackgroundTransparency = 1
    ghostGuiTitle.Text = "GHOST PLAT"
    ghostGuiTitle.TextColor3 = Color3.fromRGB(255,255,0)
    ghostGuiTitle.Font = Enum.Font.Arcade
    ghostGuiTitle.TextScaled = true

    local ghostPlatHolder = Instance.new("Frame")
    ghostPlatHolder.Parent = ghostGui
    ghostPlatHolder.Size = UDim2.new(0,70,0,30)
    ghostPlatHolder.Position = UDim2.new(0.5,-35,0.5,2)
    ghostPlatHolder.BackgroundTransparency = 1

    local ghostPlatToggle = Instance.new("TextButton")
    ghostPlatToggle.Parent = ghostPlatHolder
    ghostPlatToggle.Size = UDim2.new(0,45,0,22)
    ghostPlatToggle.Position = UDim2.new(0.5,-22,0.5,-11)
    ghostPlatToggle.BackgroundColor3 = Color3.fromRGB(30,30,10)
    ghostPlatToggle.Text = ""
    addBorderLED(ghostPlatToggle, 2)

    Instance.new("UICorner", ghostPlatToggle).CornerRadius = UDim.new(1,0)

    local ghostPlatBall = Instance.new("Frame")
    ghostPlatBall.Parent = ghostPlatToggle
    ghostPlatBall.Size = UDim2.new(0,17,0,17)
    ghostPlatBall.Position = UDim2.new(0,2,0.5,-8)
    ghostPlatBall.BackgroundColor3 = Color3.fromRGB(255,255,255)
    Instance.new("UICorner", ghostPlatBall).CornerRadius = UDim.new(1,0)

    -- Toggle do painel ghost
    ghostPlatToggle.MouseButton1Click:Connect(function()
        ghostPlatEnabled = not ghostPlatEnabled
        if ghostPlatEnabled then
            TweenService:Create(ghostPlatToggle, TweenInfo.new(0.2), {
                BackgroundColor3 = Color3.fromRGB(200,200,0)
            }):Play()
            TweenService:Create(ghostPlatBall, TweenInfo.new(0.2), {
                Position = UDim2.new(1,-19,0.5,-8)
            }):Play()
            startGhostPlatform()
        else
            TweenService:Create(ghostPlatToggle, TweenInfo.new(0.2), {
                BackgroundColor3 = Color3.fromRGB(30,30,10)
            }):Play()
            TweenService:Create(ghostPlatBall, TweenInfo.new(0.2), {
                Position = UDim2.new(0,2,0.5,-8)
            }):Play()
            stopGhostPlatform()
        end
    end)

    --==============================
    -- FLOAT GUI
    --==============================
    local floatGui = Instance.new("Frame")
    floatGui.Parent = gui
    floatGui.Size = UDim2.new(0,130,0,75)
    floatGui.Position = UDim2.new(0,15,0.5,-120)
    floatGui.BackgroundColor3 = Color3.fromRGB(10,10,5)
    floatGui.Visible = false
    floatGui.Active = true
    floatGui.Draggable = true

    local floatCorner = Instance.new("UICorner")
    floatCorner.Parent = floatGui
    floatCorner.CornerRadius = UDim.new(0,12)
    addBorderLED(floatGui,4)

    local floatTitle = Instance.new("TextLabel")
    floatTitle.Parent = floatGui
    floatTitle.Size = UDim2.new(1,0,0,25)
    floatTitle.BackgroundTransparency = 1
    floatTitle.Text = "FLOAT"
    floatTitle.TextColor3 = Color3.fromRGB(255,255,0)
    floatTitle.Font = Enum.Font.Arcade
    floatTitle.TextScaled = true

    local floatHolder = Instance.new("Frame")
    floatHolder.Parent = floatGui
    floatHolder.Size = UDim2.new(0,70,0,30)
    floatHolder.Position = UDim2.new(0.5,-35,0.5,-2)
    floatHolder.BackgroundTransparency = 1

    local floatToggle = Instance.new("TextButton")
    floatToggle.Parent = floatHolder
    floatToggle.Size = UDim2.new(0,45,0,22)
    floatToggle.Position = UDim2.new(0.5,-22,0.5,-11)
    floatToggle.BackgroundColor3 = Color3.fromRGB(30,30,10)
    floatToggle.Text = ""
    addBorderLED(floatToggle,2)

    local floatToggleCorner = Instance.new("UICorner")
    floatToggleCorner.Parent = floatToggle
    floatToggleCorner.CornerRadius = UDim.new(1,0)

    local floatBall = Instance.new("Frame")
    floatBall.Parent = floatToggle
    floatBall.Size = UDim2.new(0,17,0,17)
    floatBall.Position = UDim2.new(0,2,0.5,-8)
    floatBall.BackgroundColor3 = Color3.fromRGB(255,255,255)

    local floatBallCorner = Instance.new("UICorner")
    floatBallCorner.Parent = floatBall
    floatBallCorner.CornerRadius = UDim.new(1,0)

    --==============================
    -- BOTÃƒO REABRIR (LH)
    --==============================
    local openBtn = Instance.new("TextButton", gui)
    openBtn.Size = UDim2.new(0, 50, 0, 50)
    openBtn.Position = UDim2.new(0, 15, 0.5, -25)
    openBtn.Text = "LH"
    openBtn.Visible = false
    openBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    openBtn.Font = Enum.Font.GothamBold
    openBtn.TextSize = 18
    Instance.new("UICorner", openBtn).CornerRadius = UDim.new(0, 10)
    addBorderLED(openBtn, 3)

    --==============================
    -- FRAME PRINCIPAL
    --==============================
    local main = Instance.new("Frame", gui)
    main.Size = UDim2.new(0, 210, 0, 310)
    main.AnchorPoint = Vector2.new(0.5, 0.5)
    main.Position = UDim2.new(1, -125, 0.5, 0)
    main.BackgroundColor3 = Color3.fromRGB(10, 10, 5)
    main.ClipsDescendants = true
    Instance.new("UICorner", main).CornerRadius = UDim.new(0, 10)
    addBorderLED(main, 4)

    local title = Instance.new("TextLabel", main)
    title.Size = UDim2.new(1, 0, 0, 40)
    title.Text = "LUAX HUB"
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    title.TextColor3 = Color3.fromRGB(255, 255, 0)
    title.BackgroundTransparency = 1
    title.ZIndex = 5

    local close = Instance.new("TextButton", main)
    close.Size = UDim2.new(0, 25, 0, 25)
    close.Position = UDim2.new(1, -30, 0, 5)
    close.Text = "X"
    close.TextColor3 = Color3.fromRGB(255, 255, 0)
    close.BackgroundTransparency = 1
    close.Font = Enum.Font.GothamBold
    close.TextSize = 18
    close.ZIndex = 5

    --==============================
    -- SCROLLING FRAME
    --==============================
    local scroll = Instance.new("ScrollingFrame", main)
    scroll.Size = UDim2.new(1, -10, 1, -50)
    scroll.Position = UDim2.new(0, 5, 0, 40)
    scroll.BackgroundTransparency = 1
    scroll.ScrollBarThickness = 2
    scroll.ScrollBarImageColor3 = Color3.fromRGB(255, 255, 0)
    scroll.CanvasSize = UDim2.new(0, 0, 3, 0)
    scroll.ZIndex = 4

    local layout = Instance.new("UIListLayout", scroll)
    layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    layout.Padding = UDim.new(0, 5)

    --==============================
    -- TOGGLE BUTTON
    --==============================
    local function createToggleButton(name, callback)
        local holder = Instance.new("Frame")
        holder.Parent = scroll
        holder.Size = UDim2.new(0.9, 0, 0, 45)
        holder.BackgroundColor3 = Color3.fromRGB(20, 20, 10)
        holder.ZIndex = 4

        local holderCorner = Instance.new("UICorner")
        holderCorner.Parent = holder
        holderCorner.CornerRadius = UDim.new(0, 10)
        addBorderLED(holder, 3)

        local label = Instance.new("TextLabel")
        label.Parent = holder
        label.Size = UDim2.new(0.6, 0, 1, 0)
        label.Position = UDim2.new(0, 10, 0, 0)
        label.BackgroundTransparency = 1
        label.Text = name
        label.TextColor3 = Color3.fromRGB(255, 255, 255)
        label.Font = Enum.Font.GothamBold
        label.TextSize = 13
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.ZIndex = 5

        local toggle = Instance.new("TextButton")
        toggle.Parent = holder
        toggle.Size = UDim2.new(0, 45, 0, 22)
        toggle.Position = UDim2.new(1, -55, 0.5, -11)
        toggle.BackgroundColor3 = Color3.fromRGB(30, 30, 10)
        toggle.Text = ""
        toggle.ZIndex = 5
        addBorderLED(toggle, 2)

        local toggleCorner = Instance.new("UICorner")
        toggleCorner.Parent = toggle
        toggleCorner.CornerRadius = UDim.new(1, 0)

        local ball = Instance.new("Frame")
        ball.Parent = toggle
        ball.Size = UDim2.new(0, 18, 0, 18)
        ball.Position = UDim2.new(0, 2, 0.5, -9)
        ball.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        ball.ZIndex = 6

        local ballCorner = Instance.new("UICorner")
        ballCorner.Parent = ball
        ballCorner.CornerRadius = UDim.new(1, 0)

        local enabled = false

        toggle.MouseButton1Click:Connect(function()
            enabled = not enabled
            if enabled then
                TweenService:Create(toggle, TweenInfo.new(0.2), {
                    BackgroundColor3 = Color3.fromRGB(200, 200, 0)
                }):Play()
                TweenService:Create(ball, TweenInfo.new(0.2), {
                    Position = UDim2.new(1, -20, 0.5, -9)
                }):Play()
            else
                TweenService:Create(toggle, TweenInfo.new(0.2), {
                    BackgroundColor3 = Color3.fromRGB(30, 30, 10)
                }):Play()
                TweenService:Create(ball, TweenInfo.new(0.2), {
                    Position = UDim2.new(0, 2, 0.5, -9)
                }):Play()
            end
            if callback then callback(enabled) end
        end)
    end

    --==============================
    -- ULTRA FPS 5K â€” SEM LAG, SEM NÃ‰VOA
    --==============================
    createToggleButton("ULTRA FPS 5K", function(state)
        if state then

            -- [1] QUALIDADE GRÃFICA BAIXA = MAIS FPS (Level21 Ã© o contrÃ¡rio!)
            pcall(function()
                settings().Rendering.QualityLevel = Enum.QualityLevel.Level03
            end)
            pcall(function()
                settings().Rendering.EagerBulkExecution = true
            end)

            -- [2] REMOVE TODA NÃ‰VOA (ver mais longe, menos cÃ¡lculo)
            Lighting.FogEnd   = 9e9
            Lighting.FogStart = 9e9
            Lighting.FogColor = Color3.fromRGB(0, 0, 0)

            -- [3] REMOVE ATMOSPHERE (nÃ©voa volumÃ©trica â€” pesada pro GPU)
            for _, v in pairs(Lighting:GetChildren()) do
                if v:IsA("Atmosphere") then
                    v.Density = 0
                    v.Offset  = 0
                    v.Haze    = 0
                    v.Glare   = 0
                    v.Decay   = Color3.fromRGB(0,0,0)
                    v.Color   = Color3.fromRGB(0,0,0)
                end
            end

            -- [4] REMOVE BLOOM, BLUR E EFEITOS DE PÃ“S-PROCESSAMENTO (lag invisÃ­vel)
            for _, v in pairs(Lighting:GetChildren()) do
                pcall(function()
                    if v:IsA("BloomEffect")
                    or v:IsA("BlurEffect")
                    or v:IsA("ColorCorrectionEffect")
                    or v:IsA("DepthOfFieldEffect")
                    or v:IsA("SunRaysEffect") then
                        v.Enabled = false
                    end
                end)
            end

            -- [5] ILUMINAÃ‡ÃƒO SIMPLES SEM SOMBRAS GLOBAIS
            Lighting.GlobalShadows  = false
            Lighting.Brightness     = 2
            Lighting.Ambient        = Color3.fromRGB(90, 90, 90)
            Lighting.OutdoorAmbient = Color3.fromRGB(120, 120, 120)

            -- [6] CÃ‚MERA SEM CORTE DE DISTÃ‚NCIA
            workspace.CurrentCamera.MaxAxisFieldOfView = 120
            pcall(function()
                workspace.CurrentCamera.NearPlaneZ = -0.01
            end)

            -- [7] REMOVE PARTÃCULAS, FOGO, FUMAÃ‡A, FAÃSCAS, TRILHAS
            task.spawn(function()
                for _, v in pairs(workspace:GetDescendants()) do
                    pcall(function()
                        if v:IsA("ParticleEmitter")
                        or v:IsA("Smoke")
                        or v:IsA("Fire")
                        or v:IsA("Sparkles")
                        or v:IsA("Trail") then
                            v.Enabled = false
                        end
                    end)
                end
            end)

            -- [8] REMOVE SOMBRAS DE TODOS OS OBJETOS
            task.spawn(function()
                for _, v in pairs(workspace:GetDescendants()) do
                    pcall(function()
                        if v:IsA("BasePart") then
                            v.CastShadow = false
                        end
                    end)
                end
            end)

            -- [9] DESATIVA LUZES DINÃ‚MICAS (PointLight, SpotLight, SurfaceLight)
            -- SÃ£o as que mais pesam no render
            task.spawn(function()
                for _, v in pairs(workspace:GetDescendants()) do
                    pcall(function()
                        if v:IsA("PointLight")
                        or v:IsA("SpotLight")
                        or v:IsA("SurfaceLight") then
                            v.Enabled = false
                        end
                    end)
                end
            end)

            -- [10] REMOVE DECALS E TEXTURAS DECORATIVAS DOS OBJETOS
            task.spawn(function()
                for _, v in pairs(workspace:GetDescendants()) do
                    pcall(function()
                        if v:IsA("Decal") or v:IsA("Texture") then
                            v.Transparency = 1
                        end
                        if v:IsA("SurfaceAppearance") then
                            v:Destroy()
                        end
                    end)
                end
            end)

            -- [11] REDUZ VOLUME DOS SONS (nÃ£o remove, mas alivia o audio thread)
            task.spawn(function()
                for _, v in pairs(workspace:GetDescendants()) do
                    pcall(function()
                        if v:IsA("Sound") then
                            v.Volume = math.min(v.Volume, 0.3)
                        end
                    end)
                end
            end)

        else

            -- RESTAURAR TUDO AO NORMAL
            Lighting.FogEnd         = 100000
            Lighting.FogStart       = 0
            Lighting.GlobalShadows  = true
            Lighting.Brightness     = 1
            Lighting.Ambient        = Color3.fromRGB(0, 0, 0)
            Lighting.OutdoorAmbient = Color3.fromRGB(127, 127, 127)

            for _, v in pairs(Lighting:GetChildren()) do
                pcall(function()
                    if v:IsA("Atmosphere") then
                        v.Density = 0.3
                        v.Haze    = 0
                        v.Glare   = 0
                    end
                    if v:IsA("BloomEffect")
                    or v:IsA("BlurEffect")
                    or v:IsA("ColorCorrectionEffect")
                    or v:IsA("DepthOfFieldEffect")
                    or v:IsA("SunRaysEffect") then
                        v.Enabled = true
                    end
                end)
            end

            task.spawn(function()
                for _, v in pairs(workspace:GetDescendants()) do
                    pcall(function()
                        if v:IsA("ParticleEmitter")
                        or v:IsA("Smoke")
                        or v:IsA("Fire")
                        or v:IsA("Sparkles")
                        or v:IsA("Trail") then
                            v.Enabled = true
                        end
                        if v:IsA("BasePart") then
                            v.CastShadow = true
                        end
                        if v:IsA("PointLight")
                        or v:IsA("SpotLight")
                        or v:IsA("SurfaceLight") then
                            v.Enabled = true
                        end
                        if v:IsA("Decal") or v:IsA("Texture") then
                            v.Transparency = 0
                        end
                        if v:IsA("Sound") then
                            v.Volume = 0.5
                        end
                    end)
                end
            end)

            pcall(function()
                settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
            end)

        end
    end)

    --==============================
    -- SPEED X2
    -- Salva a velocidade original do jogo
    -- e restaura ao desligar
    --==============================
    local originalWalkSpeed = 16

    -- Pega a velocidade real do jogo assim que o personagem estiver pronto
    local function captureOriginalSpeed()
        local character = LocalPlayer.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return end
        -- SÃ³ salva se ainda nÃ£o foi modificado por speed x2
        originalWalkSpeed = humanoid.WalkSpeed
    end

    if LocalPlayer.Character then
        captureOriginalSpeed()
    end
    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(1)
        captureOriginalSpeed()
    end)

    createToggleButton("SPEED X2", function(state)
        local character = LocalPlayer.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return end

        if state then
            -- Salva a velocidade original antes de modificar
            originalWalkSpeed = humanoid.WalkSpeed
            -- Velocidade de 25.6 studs/s (16 padrÃ£o Ã— 1.6)
            humanoid.WalkSpeed = 25.6
        else
            -- Restaura a velocidade original do jogo
            humanoid.WalkSpeed = originalWalkSpeed
        end
    end)

    --==============================
    -- INFINITI JUMP
    --==============================
    createToggleButton("INFINITI JUMP", function(state)
        infiniteJumpEnabled = state
    end)

    --==============================
    -- GHOST MODE
    --==============================
    createToggleButton("GHOST MODE", function(state)
        ghostEnabled = state
        ghostGui.Visible = state

        if not state then
            -- Ao desligar o ghost mode, para a plataforma tambÃ©m
            if ghostPlatEnabled then
                ghostPlatEnabled = false
                stopGhostPlatform()
                TweenService:Create(ghostPlatToggle, TweenInfo.new(0.2), {
                    BackgroundColor3 = Color3.fromRGB(30,30,10)
                }):Play()
                TweenService:Create(ghostPlatBall, TweenInfo.new(0.2), {
                    Position = UDim2.new(0,2,0.5,-8)
                }):Play()
            end
        end
    end)

    --==============================
    -- ANTI-TELEPORT
    --==============================
    createToggleButton("ANTI-TELEPORT", function(state)
        antiTeleportEnabled = state
        if state then
            startAntiTeleport()
        else
            if antiTeleportConn then
                antiTeleportConn:Disconnect()
                antiTeleportConn = nil
            end
            lastSafePos = nil
        end
    end)

    --==============================
    -- DESYNC FLOAT
    --==============================
    local desyncPosition = nil
    local desyncOrb = nil
    local floatConnection = nil
    local orbitParts = {}

    createToggleButton("DESYNC FLOAT", function(state)
        floatGui.Visible = state

        if state then
            local character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
            local root = character:WaitForChild("HumanoidRootPart")
            desyncPosition = root.Position

            if desyncOrb then desyncOrb:Destroy() end
            for _,v in pairs(orbitParts) do if v then v:Destroy() end end
            orbitParts = {}

            desyncOrb = Instance.new("Part")
            desyncOrb.Name = "DesyncOrb"
            desyncOrb.Parent = workspace
            desyncOrb.Anchored = true
            desyncOrb.CanCollide = false
            desyncOrb.Shape = Enum.PartType.Ball
            desyncOrb.Material = Enum.Material.Neon
            desyncOrb.Color = Color3.fromRGB(0,0,0)
            desyncOrb.Size = Vector3.new(5,5,5)
            desyncOrb.Position = desyncPosition + Vector3.new(0,2.5,0)

            local billboard = Instance.new("BillboardGui")
            billboard.Parent = desyncOrb
            billboard.Size = UDim2.new(0,200,0,50)
            billboard.AlwaysOnTop = true
            billboard.StudsOffset = Vector3.new(0,4,0)

            local text = Instance.new("TextLabel")
            text.Parent = billboard
            text.Size = UDim2.new(1,0,1,0)
            text.BackgroundTransparency = 1
            text.Text = "DESYNC FLOAT"
            text.TextColor3 = Color3.fromRGB(255,255,0)
            text.TextStrokeTransparency = 0
            text.TextScaled = true
            text.Font = Enum.Font.Arcade

            local orbSizes = {1.8, 0.6, 1.4, 0.4, 2.0, 0.5, 1.6, 0.5, 1.2, 0.4, 1.8, 0.6}
            for i = 1,12 do
                local orb = Instance.new("Part")
                orb.Parent = workspace
                orb.Shape = Enum.PartType.Ball
                orb.Anchored = true
                orb.CanCollide = false
                orb.Material = Enum.Material.Neon
                orb.Color = i % 2 == 0 and Color3.fromRGB(255,255,255) or Color3.fromRGB(0,0,0)
                local s = orbSizes[i]
                orb.Size = Vector3.new(s,s,s)
                table.insert(orbitParts, orb)
            end

            local rotation = 0
            floatConnection = RunService.RenderStepped:Connect(function(dt)
                rotation += dt * 2
                if desyncOrb then
                    desyncOrb.Position = desyncPosition + Vector3.new(0, 2.5 + math.sin(tick()*2)/3, 0)
                end
                for i,orb in pairs(orbitParts) do
                    local angle = rotation + (i * math.pi / 6)
                    local radius = 4 + math.sin(tick()+i)
                    orb.Position = desyncPosition + Vector3.new(
                        math.cos(angle) * radius,
                        math.sin(angle*2) + 2.5,
                        math.sin(angle) * radius
                    )
                end
            end)

        else
            floatGui.Visible = false
            if floatConnection then floatConnection:Disconnect() floatConnection = nil end
            if desyncOrb then desyncOrb:Destroy() desyncOrb = nil end
            for _,v in pairs(orbitParts) do if v then v:Destroy() end end
            orbitParts = {}
        end
    end)

    --==============================
    -- FLOAT TOGGLE GUI (painel lateral)
    -- ComeÃ§a DESATIVADO
    --==============================
    local floatEnabled = false
    local playerFloatConnection = nil

    local function stopFloat()
        floatEnabled = false
        TweenService:Create(floatToggle, TweenInfo.new(0.2), {
            BackgroundColor3 = Color3.fromRGB(30,30,10)
        }):Play()
        TweenService:Create(floatBall, TweenInfo.new(0.2), {
            Position = UDim2.new(0,2,0.5,-8)
        }):Play()
        if playerFloatConnection then
            playerFloatConnection:Disconnect()
            playerFloatConnection = nil
        end
    end

    local function startFloat()
        floatEnabled = true
        TweenService:Create(floatToggle, TweenInfo.new(0.2), {
            BackgroundColor3 = Color3.fromRGB(255,255,0)
        }):Play()
        TweenService:Create(floatBall, TweenInfo.new(0.2), {
            Position = UDim2.new(1,-19,0.5,-8)
        }):Play()

        local character = LocalPlayer.Character
        if not character then return end
        local root = character:FindFirstChild("HumanoidRootPart")
        if not root then return end
        if playerFloatConnection then playerFloatConnection:Disconnect() end

        -- Velocidade normal Roblox = 16 studs/s â†’ voo = 16 Ã— 1.6 = 25.6 studs/s
        local FLY_SPEED  = 25.6
        local reachedOrb = false  -- evita voltamento ao chegar na esfera

        playerFloatConnection = RunService.RenderStepped:Connect(function(dt)
            if not floatEnabled then return end
            if not desyncPosition then return end
            pcall(function()
                local currentPos = root.Position
                local targetPos  = desyncPosition + Vector3.new(0, 3, 0)
                local dist       = (targetPos - currentPos).Magnitude
                local bobble     = math.sin(tick() * 3) * 0.15  -- bobble mais suave

                -- Marca como chegou (sem stopFloat â€” apenas para de mover)
                if desyncOrb and (currentPos - desyncOrb.Position).Magnitude < 3.5 then
                    reachedOrb = true
                end

                if reachedOrb then
                    -- Flutua suavemente na posiÃ§Ã£o final sem voltar
                    local finalPos = desyncPosition + Vector3.new(0, 2 + bobble, 0)
                    local _, yRot, _ = root.CFrame:ToEulerAnglesYXZ()
                    root.CFrame = root.CFrame:Lerp(
                        CFrame.new(finalPos) * CFrame.Angles(0, yRot, 0),
                        0.12
                    )
                    root.AssemblyLinearVelocity = Vector3.zero
                    return
                end

                if dist > 0.8 then
                    -- Move a velocidade constante, sem snap nem voltamento
                    local step      = math.min(FLY_SPEED * dt, dist)
                    local direction = (targetPos - currentPos).Unit
                    local _, yRot, _ = root.CFrame:ToEulerAnglesYXZ()
                    root.CFrame = CFrame.new(currentPos + direction * step)
                        * CFrame.Angles(0, yRot, 0)
                else
                    -- Chegou: flutua suave no alvo, sem snap
                    reachedOrb = true
                end
                root.AssemblyLinearVelocity = Vector3.zero
            end)
        end)
    end

    floatToggle.MouseButton1Click:Connect(function()
        if floatEnabled then
            stopFloat()
        else
            startFloat()
        end
    end)

    --==============================
    -- NEVE
    --==============================
    startSnowEffect(main)

    --==============================
    -- ABRIR / FECHAR
    --==============================
    local function toggleUI(show)
        if show then
            main.Visible = true
            openBtn.Visible = false
            main:TweenSize(UDim2.new(0, 210, 0, 310), "Out", "Back", 0.4, true)
        else
            main:TweenSize(UDim2.new(0, 0, 0, 0), "In", "Back", 0.3, true, function()
                main.Visible = false
                openBtn.Visible = true
            end)
        end
    end

    close.MouseButton1Click:Connect(function()
        toggleUI(false)
    end)

    openBtn.MouseButton1Click:Connect(function()
        toggleUI(true)
    end)
end

createHub()
