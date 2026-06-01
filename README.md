--[[
    Nameless Admin - Painel de Controle Mobile
    Usando Rayfield UI
    Funcionalidades: WalkSpeed, JumpPower, Fly, Highlight, Noclip, Air Walk, Air Swim, Fling, Morph
    Criado por: CriadorYan
]]

-- Carregar Rayfield UI
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- Criar a janela principal
local Window = Rayfield:CreateWindow({
    Name = "🔥 Nameless Admin",
    LoadingTitle = "Nameless Admin",
    LoadingSubtitle = "por CriadorYan",
    ConfigurationSaving = {
        Enabled = false,
    },
    Discord = {
        Enabled = false,
    },
    KeySystem = false,
})

-- Criar abas
local MainTab = Window:CreateTab("🏠 Principal", 4483362458)
local FlyTab = Window:CreateTab("✈️ Voo", 4483362458)
local MovementTab = Window:CreateTab("🏃 Movimento", 4483362458)
local TrollTab = Window:CreateTab("👻 Troll", 4483362458)
local VisualTab = Window:CreateTab("👁️ Visual", 4483362458)

-- Variáveis de estado
local player = game.Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local flying = false
local flightConnection
local flySpeed = 50
local currentNotification = nil
local noclipEnabled = false
local noclipConnection
local airWalkEnabled = false
local airWalkConnection
local airSwimEnabled = false
local airSwimConnection
local flingEnabled = false
local flingConnection

-- Valores padrão
local defaultWalkSpeed = 16
local defaultJumpPower = 50

-- Variáveis para controle de notificações
local lastWalkSpeedNotify = 0
local lastJumpPowerNotify = 0
local notifyCooldown = 1.5

-- Função de notificação otimizada
local function notify(title, content, duration)
    if currentNotification then
        currentNotification = nil
    end
    
    currentNotification = Rayfield:Notify({
        Title = title,
        Content = content,
        Duration = duration or 2,
        Image = 4483362458,
    })
end

-- Função para obter o Humanoid
local function getHumanoid()
    if player.Character and player.Character:FindFirstChild("Humanoid") then
        return player.Character.Humanoid
    end
    return nil
end

-- Função de voo com rotação do personagem junto com a câmera
local function startFly()
    local hum = getHumanoid()
    if not hum then
        notify("Erro", "Humanoid não encontrado!", 2)
        return false
    end
    
    flying = true
    
    -- Configurar controles do personagem
    hum.PlatformStand = false
    hum.AutoRotate = false
    
    -- Forçar ShiftLock automaticamente
    if player.DevCameraOcclusionMode ~= Enum.DevCameraOcclusionMode.Invisicam then
        player.DevCameraOcclusionMode = Enum.DevCameraOcclusionMode.Invisicam
    end
    
    local camera = workspace.CurrentCamera
    local rootPart = hum.Parent:WaitForChild("HumanoidRootPart")
    
    -- Corpo do voo
    local bodyGyro = Instance.new("BodyGyro")
    local bodyVelocity = Instance.new("BodyVelocity")
    
    bodyGyro.P = 9e4
    bodyGyro.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    bodyGyro.CFrame = camera.CFrame
    bodyGyro.Parent = rootPart
    
    bodyVelocity.Parent = rootPart
    bodyVelocity.MaxForce = Vector3.new(9e9, 9e9, 9e9)
    bodyVelocity.Velocity = Vector3.new(0, 0, 0)
    
    -- Sistema de voo com rotação do personagem seguindo a câmera
    flightConnection = game:GetService("RunService").RenderStepped:Connect(function(deltaTime)
        if not flying or not hum.Parent or not rootPart then
            return
        end
        
        -- Atualizar rotação do personagem para seguir a câmera (virada junto)
        local cameraCFrame = camera.CFrame
        
        -- Criar rotação apenas no eixo Y (horizontal) para o personagem
        local cameraDirection = cameraCFrame.LookVector
        local horizontalDirection = Vector3.new(cameraDirection.X, 0, cameraDirection.Z).Unit
        
        if horizontalDirection.Magnitude > 0 then
            -- O personagem vai rotacionar para seguir a direção horizontal da câmera
            local newCFrame = CFrame.new(rootPart.Position, rootPart.Position + horizontalDirection)
            bodyGyro.CFrame = newCFrame
        else
            -- Se estiver olhando diretamente para cima/baixo, manter última rotação horizontal
            bodyGyro.CFrame = CFrame.new(rootPart.Position, rootPart.Position + Vector3.new(cameraCFrame.LookVector.X, 0, cameraCFrame.LookVector.Z).Unit)
        end
        
        -- Obter input do movimento do personagem
        local moveVector = hum.MoveDirection
        
        -- Se não houver movimento, pairar no lugar
        if moveVector.Magnitude == 0 then
            bodyVelocity.Velocity = Vector3.new(0, 0, 0)
            return
        end
        
        -- Calcular direções baseadas na câmera
        local cameraForward = cameraCFrame.LookVector
        local cameraRight = cameraCFrame.RightVector
        
        local velocity = Vector3.new()
        
        -- Movimento para frente/trás (não invertido)
        if moveVector.Z > 0 then
            velocity = velocity + cameraForward * flySpeed
        elseif moveVector.Z < 0 then
            velocity = velocity - cameraForward * flySpeed
        end
        
        -- Movimento lateral (não invertido)
        if moveVector.X > 0 then
            velocity = velocity + cameraRight * flySpeed
        elseif moveVector.X < 0 then
            velocity = velocity - cameraRight * flySpeed
        end
        
        bodyVelocity.Velocity = velocity
    end)
    
    notify("✈️ Voo Ativado", "ShiftLock automático! Personagem vira junto com a câmera", 4)
    return true
end

local function stopFly()
    flying = false
    
    if flightConnection then
        flightConnection:Disconnect()
        flightConnection = nil
    end
    
    -- Restaurar configurações do personagem
    local hum = getHumanoid()
    if hum then
        hum.AutoRotate = true
        hum.PlatformStand = false
        
        if hum.Parent and hum.Parent:FindFirstChild("HumanoidRootPart") then
            local root = hum.Parent.HumanoidRootPart
            if root:FindFirstChild("BodyGyro") then
                root.BodyGyro:Destroy()
            end
            if root:FindFirstChild("BodyVelocity") then
                root.BodyVelocity:Destroy()
            end
        end
    end
    
    -- Restaurar modo de câmera
    if player.DevCameraOcclusionMode == Enum.DevCameraOcclusionMode.Invisicam then
        player.DevCameraOcclusionMode = Enum.DevCameraOcclusionMode.Zoom
    end
    
    notify("Voo Desativado", "Sistema de voo e ShiftLock desligados!", 2)
end

-- Função Noclip
local function enableNoclip()
    noclipEnabled = true
    
    noclipConnection = game:GetService("RunService").Stepped:Connect(function()
        if noclipEnabled and player.Character then
            for _, part in pairs(player.Character:GetDescendants()) do
                if part:IsA("BasePart") and part.CanCollide == true then
                    part.CanCollide = false
                end
            end
        end
    end)
    
    notify("🚫 Noclip", "Atravessar paredes ativado!", 2)
end

local function disableNoclip()
    noclipEnabled = false
    
    if noclipConnection then
        noclipConnection:Disconnect()
        noclipConnection = nil
    end
    
    -- Restaurar colisões
    if player.Character then
        for _, part in pairs(player.Character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = true
            end
        end
    end
    
    notify("🚫 Noclip", "Noclip desativado!", 2)
end

-- Função Air Walk (Andar no Ar)
local function enableAirWalk()
    airWalkEnabled = true
    
    airWalkConnection = game:GetService("RunService").RenderStepped:Connect(function()
        if airWalkEnabled and player.Character and player.Character:FindFirstChild("Humanoid") then
            local hum = player.Character.Humanoid
            local root = player.Character:FindFirstChild("HumanoidRootPart")
            
            if root and hum:GetState() == Enum.HumanoidStateType.Freefall then
                -- Criar plataforma invisível no ar
                local bodyVelocity = Instance.new("BodyVelocity")
                bodyVelocity.Velocity = Vector3.new(0, 0, 0)
                bodyVelocity.MaxForce = Vector3.new(0, math.huge, 0)
                bodyVelocity.Parent = root
                
                game:GetService("Debris"):AddItem(bodyVelocity, 0.1)
            end
        end
    end)
    
    notify("🚶 Air Walk", "Andar no ar ativado! Pule e fique flutuando", 2)
end

local function disableAirWalk()
    airWalkEnabled = false
    
    if airWalkConnection then
        airWalkConnection:Disconnect()
        airWalkConnection = nil
    end
    
    notify("🚶 Air Walk", "Andar no ar desativado!", 2)
end

-- Função Air Swim (Nadar no Ar)
local function enableAirSwim()
    airSwimEnabled = true
    
    airSwimConnection = game:GetService("RunService").RenderStepped:Connect(function()
        if airSwimEnabled and player.Character then
            local hum = player.Character:FindFirstChild("Humanoid")
            if hum then
                -- Forçar estado de natação
                hum:SetStateEnabled(Enum.HumanoidStateType.Swimming, true)
                
                if hum:GetState() ~= Enum.HumanoidStateType.Swimming then
                    -- Criar efeito de natação no ar
                    local root = player.Character:FindFirstChild("HumanoidRootPart")
                    if root then
                        local bodyGyro = Instance.new("BodyGyro")
                        bodyGyro.CFrame = workspace.CurrentCamera.CFrame
                        bodyGyro.P = 3000
                        bodyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
                        bodyGyro.Parent = root
                        
                        game:GetService("Debris"):AddItem(bodyGyro, 0.1)
                    end
                end
            end
        end
    end)
    
    notify("🏊 Air Swim", "Nadar no ar ativado! Movimento de natação no ar", 2)
end

local function disableAirSwim()
    airSwimEnabled = false
    
    if airSwimConnection then
        airSwimConnection:Disconnect()
        airSwimConnection = nil
    end
    
    notify("🏊 Air Swim", "Nadar no ar desativado!", 2)
end

-- Função Fling (Jogar jogadores)
local function enableFling()
    flingEnabled = true
    
    local function flingNearbyPlayers()
        if not flingEnabled then return end
        
        local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
        if not root then return end
        
        for _, otherPlayer in pairs(game.Players:GetPlayers()) do
            if otherPlayer ~= player and otherPlayer.Character then
                local otherRoot = otherPlayer.Character:FindFirstChild("HumanoidRootPart")
                if otherRoot then
                    local distance = (root.Position - otherRoot.Position).Magnitude
                    
                    if distance < 20 then -- Alcance de 20 studs
                        -- Aplicar força de fling
                        local flingForce = Instance.new("BodyVelocity")
                        flingForce.Velocity = Vector3.new(
                            math.random(-5000, 5000),
                            math.random(2000, 5000),
                            math.random(-5000, 5000)
                        )
                        flingForce.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                        flingForce.Parent = otherRoot
                        
                        game:GetService("Debris"):AddItem(flingForce, 0.5)
                    end
                end
            end
        end
    end
    
    flingConnection = game:GetService("RunService").Heartbeat:Connect(flingNearbyPlayers)
    
    notify("💨 Fling", "Fling ativado! Jogadores próximos serão arremessados", 2)
end

local function disableFling()
    flingEnabled = false
    
    if flingConnection then
        flingConnection:Disconnect()
        flingConnection = nil
    end
    
    notify("💨 Fling", "Fling desativado!", 2)
end

-- Função Morph (Transformar em outros jogadores)
local function morphPlayer(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then
        notify("Erro", "Jogador não encontrado!", 2)
        return
    end
    
    -- Clonar aparência do jogador alvo
    local function cloneCharacter()
        local myChar = player.Character
        local targetChar = targetPlayer.Character
        
        if not myChar or not targetChar then return end
        
        -- Remover roupas atuais
        for _, item in pairs(myChar:GetChildren()) do
            if item:IsA("Accessory") or item:IsA("Shirt") or item:IsA("Pants") or item:IsA("ShirtGraphic") then
                item:Destroy()
            end
        end
        
        -- Copiar roupas do alvo
        for _, item in pairs(targetChar:GetChildren()) do
            if item:IsA("Shirt") then
                local newShirt = item:Clone()
                newShirt.Parent = myChar
            elseif item:IsA("Pants") then
                local newPants = item:Clone()
                newPants.Parent = myChar
            elseif item:IsA("Accessory") then
                local newAccessory = item:Clone()
                newAccessory.Parent = myChar
            end
        end
        
        -- Copiar escala do corpo
        if myChar:FindFirstChild("Head") and targetChar:FindFirstChild("Head") then
            local myHead = myChar.Head
            local targetHead = targetChar.Head
            
            -- Copiar cores do corpo
            for _, partName in pairs({"Head", "Torso", "LeftArm", "RightArm", "LeftLeg", "RightLeg"}) do
                local myPart = myChar:FindFirstChild(partName)
                local targetPart = targetChar:FindFirstChild(partName)
                
                if myPart and targetPart and myPart:IsA("BasePart") and targetPart:IsA("BasePart") then
                    myPart.BrickColor = targetPart.BrickColor
                    myPart.Color = targetPart.Color
                end
            end
        end
    end
    
    pcall(cloneCharacter)
    notify("🎭 Morph", "Transformado em: " .. targetPlayer.Name, 3)
end

-- Conectar eventos do personagem
player.CharacterAdded:Connect(function(char)
    character = char
    humanoid = char:WaitForChild("Humanoid")
    
    if flying then
        stopFly()
    end
    
    -- Restaurar Noclip se estava ativo
    if noclipEnabled then
        task.wait(0.1)
        enableNoclip()
    end
end)

-- ===== ABA PRINCIPAL =====

-- Slider de WalkSpeed
MainTab:CreateSlider({
    Name = "Velocidade de Andar",
    Range = {16, 200},
    Increment = 1,
    Suffix = "studs/s",
    CurrentValue = defaultWalkSpeed,
    Flag = "WalkSpeed",
    Callback = function(Value)
        local hum = getHumanoid()
        if hum then
            hum.WalkSpeed = Value
            
            local currentTime = tick()
            if currentTime - lastWalkSpeedNotify > notifyCooldown then
                notify("🏃 Velocidade", Value .. " studs/s", 1.5)
                lastWalkSpeedNotify = currentTime
            end
        end
    end,
})

-- Slider de JumpPower
MainTab:CreateSlider({
    Name = "Poder de Pulo",
    Range = {50, 300},
    Increment = 1,
    Suffix = "power",
    CurrentValue = defaultJumpPower,
    Flag = "JumpPower",
    Callback = function(Value)
        local hum = getHumanoid()
        if hum then
            hum.JumpPower = Value
            hum.UseJumpPower = true
            
            local currentTime = tick()
            if currentTime - lastJumpPowerNotify > notifyCooldown then
                notify("🦘 Pulo", "Power: " .. Value, 1.5)
                lastJumpPowerNotify = currentTime
            end
        end
    end,
})

MainTab:CreateSection("📋 Informações")

MainTab:CreateParagraph({
    Title = "Nameless Admin",
    Content = "Painel de controle criado por CriadorYan\nUse os sliders para ajustar velocidade e pulo\nSistema de voo na aba Voo\nOpções de movimento na aba Movimento"
})

-- ===== ABA DE VOO =====

FlyTab:CreateToggle({
    Name = "Ativar Sistema de Voo",
    CurrentValue = false,
    Flag = "Fly",
    Callback = function(Value)
        if Value then
            startFly()
        else
            if flying then
                stopFly()
            end
        end
    end,
})

FlyTab:CreateSlider({
    Name = "Velocidade do Voo",
    Range = {20, 200},
    Increment = 5,
    Suffix = "studs/s",
    CurrentValue = 50,
    Flag = "FlySpeed",
    Callback = function(Value)
        flySpeed = Value
        notify("✈️ Velocidade", Value .. " studs/s", 1.5)
    end,
})

FlyTab:CreateSection("📱 Controles de Voo")

FlyTab:CreateParagraph({
    Title = "Como Voar:",
    Content = "🎮 Joystick para frente = Voar para frente\n⬆️ Olhar para cima + frente = Subir\n⬇️ Olhar para baixo + frente = Descer\n⬅️➡️ Joystick lados = Voar lateralmente\n🔙 Joystick para trás = Voar para trás\n🔄 Virar câmera = Personagem vira junto"
})

-- ===== ABA DE MOVIMENTO =====

-- Noclip Toggle
MovementTab:CreateToggle({
    Name = "Noclip (Atravessar Paredes)",
    CurrentValue = false,
    Flag = "Noclip",
    Callback = function(Value)
        if Value then
            enableNoclip()
        else
            disableNoclip()
        end
    end,
})

-- Air Walk Toggle
MovementTab:CreateToggle({
    Name = "Andar no Ar (Air Walk)",
    CurrentValue = false,
    Flag = "AirWalk",
    Callback = function(Value)
        if Value then
            enableAirWalk()
        else
            disableAirWalk()
        end
    end,
})

-- Air Swim Toggle
MovementTab:CreateToggle({
    Name = "Nadar no Ar (Air Swim)",
    CurrentValue = false,
    Flag = "AirSwim",
    Callback = function(Value)
        if Value then
            enableAirSwim()
        else
            disableAirSwim()
        end
    end,
})

MovementTab:CreateSection("📋 Descrição")

MovementTab:CreateParagraph({
    Title = "Noclip:",
    Content = "Atravessa paredes e objetos sólidos"
})

MovementTab:CreateParagraph({
    Title = "Air Walk:",
    Content = "Pule e fique flutuando no ar como se estivesse em plataformas invisíveis"
})

MovementTab:CreateParagraph({
    Title = "Air Swim:",
    Content = "Nade no ar como se estivesse na água, com movimentos de natação"
})

-- ===== ABA DE TROLL =====

-- Fling Toggle
TrollTab:CreateToggle({
    Name = "Fling (Arremessar Jogadores)",
    CurrentValue = false,
    Flag = "Fling",
    Callback = function(Value)
        if Value then
            enableFling()
        else
            disableFling()
        end
    end,
})

TrollTab:CreateSection("🎭 Morph")

TrollTab:CreateParagraph({
    Title = "Morph (Transformar-se)",
    Content = "Selecione um jogador abaixo para se transformar nele"
})

-- Função para criar botões de morph para cada jogador
local function updateMorphButtons()
    -- Criar botões para cada jogador
    for _, targetPlayer in pairs(game.Players:GetPlayers()) do
        if targetPlayer ~= player then
            TrollTab:CreateButton({
                Name = "Morph: " .. targetPlayer.Name,
                Callback = function()
                    morphPlayer(targetPlayer)
                end,
            })
        end
    end
end

-- Atualizar botões de morph
updateMorphButtons()

-- Atualizar quando novos jogadores entrarem
game.Players.PlayerAdded:Connect(function(newPlayer)
    if newPlayer ~= player then
        TrollTab:CreateButton({
            Name = "Morph: " .. newPlayer.Name,
            Callback = function()
                morphPlayer(newPlayer)
            end,
        })
    end
end)

TrollTab:CreateSection("📋 Descrição")

TrollTab:CreateParagraph({
    Title = "Fling:",
    Content = "Jogadores que se aproximarem serão arremessados para longe"
})

TrollTab:CreateParagraph({
    Title = "Morph:",
    Content = "Clique no nome de um jogador para se transformar nele"
})

-- ===== ABA VISUAL =====

local highlightEnabled = false

VisualTab:CreateToggle({
    Name = "Highlight de Jogadores",
    CurrentValue = false,
    Flag = "Highlight",
    Callback = function(Value)
        highlightEnabled = Value
        
        if Value then
            notify("👁️ Highlight", "Jogadores destacados!", 2)
            
            local function addHighlight(character)
                if character and not character:FindFirstChild("PlayerHighlight") then
                    local highlight = Instance.new("Highlight")
                    highlight.Parent = character
                    highlight.FillColor = Color3.fromRGB(0, 255, 255)
                    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
                    highlight.FillTransparency = 0.5
                    highlight.OutlineTransparency = 0
                    highlight.Name = "PlayerHighlight"
                end
            end
            
            for _, plr in pairs(game.Players:GetPlayers()) do
                if plr ~= player and plr.Character then
                    addHighlight(plr.Character)
                end
            end
            
            for _, plr in pairs(game.Players:GetPlayers()) do
                if plr ~= player then
                    plr.CharacterAdded:Connect(function(char)
                        if highlightEnabled then
                            addHighlight(char)
                        end
                    end)
                end
            end
            
            game.Players.PlayerAdded:Connect(function(plr)
                plr.CharacterAdded:Connect(function(char)
                    if highlightEnabled then
                        addHighlight(char)
                    end
                end)
            end)
        else
            notify("👁️ Highlight", "Destaques removidos!", 2)
            
            for _, plr in pairs(game.Players:GetPlayers()) do
                if plr.Character then
                    local highlight = plr.Character:FindFirstChild("PlayerHighlight")
                    if highlight then
                        highlight:Destroy()
                    end
                end
            end
        end
    end,
})

VisualTab:CreateSection("ℹ️ Nameless Admin")

VisualTab:CreateParagraph({
    Title = "Criado por CriadorYan",
    Content = "Hub otimizado para mobile\nMúltiplas funcionalidades\nControles intuitivos e responsivos"
})

-- Notificação inicial
notify("🔥 Nameless Admin", "Carregado com sucesso! Criado por CriadorYan", 3)

-- Segurança
game:GetService("RunService").Heartbeat:Connect(function()
    if flying then
        local hum = getHumanoid()
        if not hum or not hum.Parent or not hum.Parent:FindFirstChild("HumanoidRootPart") then
            stopFly()
        end
    end
end)

game:GetService("Players").LocalPlayer.OnTeleport:Connect(function()
    if flying then
        stopFly()
    end
    if flingEnabled then
        disableFling()
    end
end)
