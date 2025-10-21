-- Aimbot Script para Roblox
-- Use apenas em servidores privados com autorização

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- Configurações do Aimbot
local AimbotSettings = {
    Enabled = false,
    Smoothing = 0.3, -- Suavização do movimento (0.1 = muito suave, 1.0 = instantâneo)
    FOV = 200, -- Campo de visão em pixels
    MaxDistance = 1000, -- Distância máxima para detectar alvos
    TeamCheck = true, -- Verificar se é da mesma equipe
    VisibleCheck = true, -- Verificar se o alvo está visível
    TargetPart = "Head", -- Parte do corpo para mirar
    Keybind = Enum.KeyCode.E -- Tecla para ativar/desativar
}

-- Variáveis
local CurrentTarget = nil
local AimbotConnection = nil

-- Função para verificar se o jogador é válido
local function IsValidTarget(player)
    if not player or not player.Character or not player.Character:FindFirstChild("Humanoid") then
        return false
    end
    
    if player == LocalPlayer then
        return false
    end
    
    if AimbotSettings.TeamCheck and player.Team == LocalPlayer.Team then
        return false
    end
    
    if player.Character.Humanoid.Health <= 0 then
        return false
    end
    
    return true
end

-- Função para verificar se o alvo está visível
local function IsTargetVisible(target)
    if not AimbotSettings.VisibleCheck then
        return true
    end
    
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("Head") then
        return false
    end
    
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    raycastParams.FilterDescendantsInstances = {character}
    
    local origin = character.Head.Position
    local direction = (target.Position - origin).Unit * AimbotSettings.MaxDistance
    
    local raycastResult = workspace:Raycast(origin, direction, raycastParams)
    
    if raycastResult then
        local hitPart = raycastResult.Instance
        return hitPart.Parent == target.Parent
    end
    
    return true
end

-- Função para calcular distância
local function GetDistance(position1, position2)
    return (position1 - position2).Magnitude
end

-- Função para encontrar o melhor alvo
local function FindBestTarget()
    local bestTarget = nil
    local bestDistance = math.huge
    local cameraPosition = Camera.CFrame.Position
    
    for _, player in pairs(Players:GetPlayers()) do
        if IsValidTarget(player) and player.Character then
            local character = player.Character
            local targetPart = character:FindFirstChild(AimbotSettings.TargetPart)
            
            if targetPart then
                local distance = GetDistance(cameraPosition, targetPart.Position)
                
                if distance <= AimbotSettings.MaxDistance then
                    -- Verificar se está dentro do FOV
                    local screenPosition, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                    if onScreen then
                        local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
                        local distanceFromCenter = (Vector2.new(screenPosition.X, screenPosition.Y) - screenCenter).Magnitude
                        
                        if distanceFromCenter <= AimbotSettings.FOV then
                            if IsTargetVisible(targetPart) then
                                if distance < bestDistance then
                                    bestTarget = targetPart
                                    bestDistance = distance
                                end
                            end
                        end
                    end
                end
            end
        end
    end
    
    return bestTarget
end

-- Função principal do aimbot
local function AimbotLoop()
    if not AimbotSettings.Enabled then
        return
    end
    
    local target = FindBestTarget()
    if target then
        CurrentTarget = target
        
        -- Calcular direção para o alvo
        local cameraPosition = Camera.CFrame.Position
        local targetPosition = target.Position
        local direction = (targetPosition - cameraPosition).Unit
        
        -- Aplicar suavização
        local currentLookDirection = Camera.CFrame.LookVector
        local smoothedDirection = currentLookDirection:Lerp(direction, AimbotSettings.Smoothing)
        
        -- Aplicar a nova direção da câmera
        local newCFrame = CFrame.lookAt(cameraPosition, cameraPosition + smoothedDirection)
        Camera.CFrame = newCFrame
    else
        CurrentTarget = nil
    end
end

-- Função para alternar o aimbot
local function ToggleAimbot()
    AimbotSettings.Enabled = not AimbotSettings.Enabled
    
    if AimbotSettings.Enabled then
        print("Aimbot ATIVADO")
        if not AimbotConnection then
            AimbotConnection = RunService.Heartbeat:Connect(AimbotLoop)
        end
    else
        print("Aimbot DESATIVADO")
        if AimbotConnection then
            AimbotConnection:Disconnect()
            AimbotConnection = nil
        end
        CurrentTarget = nil
    end
end

-- Configurar atalho de teclado
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.KeyCode == AimbotSettings.Keybind then
        ToggleAimbot()
    end
end)

-- Interface de usuário simples
local function CreateUI()
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "AimbotUI"
    ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
    
    local Frame = Instance.new("Frame")
    Frame.Size = UDim2.new(0, 200, 0, 100)
    Frame.Position = UDim2.new(0, 10, 0, 10)
    Frame.BackgroundColor3 = Color3.new(0, 0, 0)
    Frame.BackgroundTransparency = 0.3
    Frame.BorderSizePixel = 0
    Frame.Parent = ScreenGui
    
    local StatusLabel = Instance.new("TextLabel")
    StatusLabel.Size = UDim2.new(1, 0, 0.5, 0)
    StatusLabel.Position = UDim2.new(0, 0, 0, 0)
    StatusLabel.BackgroundTransparency = 1
    StatusLabel.Text = "Aimbot: DESATIVADO"
    StatusLabel.TextColor3 = Color3.new(1, 1, 1)
    StatusLabel.TextScaled = true
    StatusLabel.Font = Enum.Font.SourceSansBold
    StatusLabel.Parent = Frame
    
    local KeybindLabel = Instance.new("TextLabel")
    KeybindLabel.Size = UDim2.new(1, 0, 0.5, 0)
    KeybindLabel.Position = UDim2.new(0, 0, 0.5, 0)
    KeybindLabel.BackgroundTransparency = 1
    KeybindLabel.Text = "Pressione E para ativar/desativar"
    KeybindLabel.TextColor3 = Color3.new(0.8, 0.8, 0.8)
    KeybindLabel.TextScaled = true
    KeybindLabel.Font = Enum.Font.SourceSans
    KeybindLabel.Parent = Frame
    
    -- Atualizar status
    spawn(function()
        while ScreenGui.Parent do
            if AimbotSettings.Enabled then
                StatusLabel.Text = "Aimbot: ATIVADO"
                StatusLabel.TextColor3 = Color3.new(0, 1, 0)
            else
                StatusLabel.Text = "Aimbot: DESATIVADO"
                StatusLabel.TextColor3 = Color3.new(1, 0, 0)
            end
            wait(0.1)
        end
    end)
end

-- Inicializar
CreateUI()

print("Aimbot Script carregado!")
print("Pressione E para ativar/desativar")
print("Configurações:")
print("- Suavização:", AimbotSettings.Smoothing)
print("- FOV:", AimbotSettings.FOV)
print("- Distância máxima:", AimbotSettings.MaxDistance)
print("- Verificação de equipe:", AimbotSettings.TeamCheck)
print("- Verificação de visibilidade:", AimbotSettings.VisibleCheck)
